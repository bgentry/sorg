---
title: How Postgres Makes Transactions Atomic
published_at: 2017-08-06T17:10:18Z
location: San Francisco
hook: TODO
---

Postgres provides strong ACID guarantees with aim of making
data modification safe. More specifically, the "A" in
"ACID" is for _atomicity_, and the property ensures that
when performing a series of operations against the
database, either all the changes commit, or they're all
rolled back.

Since joining a company that uses MongoDB as its primary
data store, and witnessing first-hand the operational
catastrophe that's the inherent result of not having an
atomicity guarantee, I've taken a keen interest in
understanding how it works in databases that have one. This
article will examine how exactly Postgres' atomicity works,
and showcase some of the code that powers it.

A few words of warning: Postgres is under active
development, and these code snippets will become less
accurate as time marches on. It's also worth noting that
Postgres is a tremendously complex beast and I'm glossing
over quite a few details for the purposes of digestibility.
If I didn't, this article would be about a thousand pages
long.

## Managing concurrent access (#mvcc)

In a very naive database, multiple clients trying to access
data at the same time would run into contention as one
client modifies data while another is trying to access it.
A naive solution to this problem might involve each client
locking data it needs and having other clients wait on
reads and writes until they're able to take out locks of
their own. Most modern databases have a better way: MVCC.

Under MVCC (multi-version concurrency control), statements
execute inside of a ***transaction***, and instead of
overwriting data directly, they create new versions of it.
The original data is still available to other clients that
might need it, and any new data stays hidden until the
transaction ***commits***.

When a transaction starts it takes a ***snapshot*** that
represents the state of a database at that moment in time.
Databases eventually remove obsolete data by way of a
background "vacuum" process, but they'll only do so for
information that's no longer needed by any open snapshots.

Postgres manages concurrent access with MVCC. Lets take a
look at how it works.

## Transactions, tuples, and snapshots (#snapshots-transactions)

Here's the data structure that Postgres uses to represent a
transaction (from [proc.c][pgxact]):

``` c
typedef struct PGXACT
{
    TransactionId xid;          /* id of top-level transaction currently being
                                 * executed by this proc, if running and XID
                                 * is assigned; else InvalidTransactionId */

    TransactionId xmin;         /* minimal running XID as it was when we were
                                 * starting our xact, excluding LAZY VACUUM:
                                 * vacuum must not remove tuples deleted by
                                 * xid >= xmin ! */

    ...
} PGXACT;
```

Transactions are identified with a `xid` (transaction, or
"xact" ID), but as an optimization, Postgres will only
assign a transaction a `xid` if other processes will need
to care about it because it's started to modify data.
Readonly transactions can fully execute without ever being
assigned their own `xid`.

`xmin` is always assigned immediately. It's set to the
smallest `xid` of any transactions that are still in
flight, and it's tracked so that a vacuum process doesn't
remove data that the transaction still needs for its
snapshot.

### Lifetime-aware tuples (#tuples)

Rows of data in Postgres are often referred to as
***tuples***. While Postgres uses common lookup structures
like B-trees to make retrievals fast, indexes don't store a
tuple's full set of data or any of its visibility
information. Instead, they store a `tid` (tuple ID) that
can be used to retrieve a row from physical storage,
otherwise known as "the heap". The `tid` gives Postgres a
starting point where it can start scanning the heap until
it finds a valid tuple that satisfies the current
snapshot's visibility.

Here's the Postgres implementation for a "heap tuple" (as
opposed to an "index tuple" which is the structure found in
an index), along with a few other structs that represent
its header information ([from `htup.h`][tuple] [and
`htup_details.h`][tupleheaders]):

``` c
typedef struct HeapTupleData
{
    uint32          t_len;         /* length of *t_data */
    ItemPointerData t_self;        /* SelfItemPointer */
    Oid             t_tableOid;    /* table the tuple came from */
    HeapTupleHeader t_data;        /* -> tuple header and data */
} HeapTupleData;

/* referenced by HeapTupleData */
struct HeapTupleHeaderData
{
    HeapTupleFields t_heap;

    ...
}

/* referenced by HeapTupleHeaderData */
typedef struct HeapTupleFields
{
    TransactionId t_xmin;        /* inserting xact ID */
    TransactionId t_xmax;        /* deleting or locking xact ID */

    ...
} HeapTupleFields;
```

Like a transaction, a tuple tracks its own `xmin`, except
in the tuple's case it's recorded to represent the first
transaction where the tuple becomes visible (i.e. the one
that created it). It may also track `xmax` to be the _last_
transaction where the tuple is visible (i.e. the one that
deleted it).

### Snapshots: xmin, xmax, and xip (#snapshots)

Here's a snapshot ([from snapshot.h][snapshot]):

``` c
typedef struct SnapshotData
{
    /*
     * The remaining fields are used only for MVCC snapshots, and are normally
     * just zeroes in special snapshots.  (But xmin and xmax are used
     * specially by HeapTupleSatisfiesDirty.)
     *
     * An MVCC snapshot can never see the effects of XIDs >= xmax. It can see
     * the effects of all older XIDs except those listed in the snapshot. xmin
     * is stored as an optimization to avoid needing to search the XID arrays
     * for most tuples.
     */
    TransactionId xmin;            /* all XID < xmin are visible to me */
    TransactionId xmax;            /* all XID >= xmax are invisible to me */

    /*
     * For normal MVCC snapshot this contains the all xact IDs that are in
     * progress, unless the snapshot was taken during recovery in which case
     * it's empty. For historic MVCC snapshots, the meaning is inverted, i.e.
     * it contains *committed* transactions between xmin and xmax.
     *
     * note: all ids in xip[] satisfy xmin <= xip[i] < xmax
     */
    TransactionId *xip;
    uint32        xcnt; /* # of xact ids in xip[] */

    ...
}
```

A snapshot's `xmin` is calculated the same way as a
transaction's (i.e. the lowest `xid` amongst running
transactions when the snapshot is created), but for a
different prupose.

A snapshot uses its `xmin` as a lower boundary for data
visibility. Tuples data created or modified by a
transaction with a `xid` _smaller_ than a snapshot's `xmin`
are visible to it.

It also defines an `xmax`, which is set to the last
commited `xid` plus one. `xmax` tracks the upper bound of
visibility; transactions with a `xid` _greater to or equal_
to `xmax` are invisible to the snapshot.

Lastly, a snapshot defines `*xip`, an array of all of the
`xid`s of transactions that were in progress when the
snapshot was created. It's created because even though
`xid`s in `*xip` will be smaller than `xmax` (and might
therefore be considered visible), the snapshot knows to
keep whatever changes they make hidden. Recall that a
snapshot represents the state of a database at a moment in
time, and because these transactions were not committed at
the moment the snapshot was created, their results will
never be considered visible even if they commit
successfully.

## Beginning a transaction (#begin)

TODO: Where the hell does a transaction start?! And how
does that call into GetSnapshotData?!

The meat of beginning a transaction is creating its
snapshot, which is performed by [`GetSnapshotData` in
`procarray.c`][getsnapshotdata]:

``` c
Snapshot
GetSnapshotData(Snapshot snapshot)
{
    /* xmax is always latestCompletedXid + 1 */
    xmax = ShmemVariableCache->latestCompletedXid;
    Assert(TransactionIdIsNormal(xmax));
    TransactionIdAdvance(xmax);

    ...

    snapshot->xmax = xmax;
}
```

This function does a lot of initialization work, but like
we talked about, some of its most important work is set to
the snapshot's `xmin`, `xmax`, and `*xip`. The easiest of
these is `xmax`, which is retrieved from shared memory
managed by the postmaster, which is tracking the `xid`s of
any transactions that complete (more on this later):

Notice that it's the function's responsibility to add one
to the last `xid`. This isn't quite as trivial as
incrementing it because transaction IDs in Postgres are
allowed to wrap. A transaction ID is defined as a simple
unsigned 32-bit integer (from [c.h][xid]):

``` c
typedef uint32 TransactionId;
```

Even though `xid`s are assigned only opportunistically (as
mentioned above, reads don't need one), a system doing a
lot of transaction throughput can easily hit the bounds of
32 bits, so the system needs to be able to wrap to "reset"
the `xid` sequence as necessary. This is handled by some
preprocessor magic (in [transam.h][xidadvance]):

``` c
#define InvalidTransactionId        ((TransactionId) 0)
#define BootstrapTransactionId      ((TransactionId) 1)
#define FrozenTransactionId         ((TransactionId) 2)
#define FirstNormalTransactionId    ((TransactionId) 3)

...

/* advance a transaction ID variable, handling wraparound correctly */
#define TransactionIdAdvance(dest)    \
    do { \
        (dest)++; \
        if ((dest) < FirstNormalTransactionId) \
            (dest) = FirstNormalTransactionId; \
    } while(0)
```

Note that the first few IDs are reserved as special
identifiers, so we always skip those and start at `3`.

Back in `GetSnapshotData`, we get `xmin` and `xip` by
iterating over all running transactions:

``` c
/*
 * Spin over procArray checking xid, xmin, and subxids.  The goal is
 * to gather all active xids, find the lowest xmin, and try to record
 * subxids.
 */
for (index = 0; index < numProcs; index++)
{
    volatile PGXACT *pgxact = &allPgXact[pgprocno];
    TransactionId xid;
    xid = pgxact->xmin; /* fetch just once */

    /*
     * If the transaction has no XID assigned, we can skip it; it
     * won't have sub-XIDs either.  If the XID is >= xmax, we can also
     * skip it; such transactions will be treated as running anyway
     * (and any sub-XIDs will also be >= xmax).
     */
    if (!TransactionIdIsNormal(xid)
        || !NormalTransactionIdPrecedes(xid, xmax))
        continue;

    if (NormalTransactionIdPrecedes(xid, xmin))
        xmin = xid;

    /* Add XID to snapshot. */
    snapshot->xip[count++] = xid;

    ...
}

...

snapshot->xmin = xmin;
```

`xmin`'s purpose isn't to tell us _everything_ we need to
know about what's visible to a snapshot, but it acts as a
useful horizon beyond which we know nothing is visible.
Therefore it's calculated as the minimum `xid` of all
running transactions when a snapshot is created.

## Committing a transaction (#commit)

Transactions committed through [`CommitTransaction` (in
`xact.c`)][commit]. This function is monstrously complex,
but again, I'm going to simplify it and call out a couple
of the most important parts:

``` c
static void
CommitTransaction(void)
{
    ...

    /*
     * We need to mark our XIDs as committed in pg_xact.  This is where we
     * durably commit.
     */
    latestXid = RecordTransactionCommit();

    /*
     * Let others know about no transaction in progress by me. Note that this
     * must be done _before_ releasing locks we hold and _after_
     * RecordTransactionCommit.
     */
    ProcArrayEndTransaction(MyProc, latestXid);

    ...
}
```

Postgres is entirely designed around the idea of
durability, which dictates that even in extreme events like
a crash or power loss, committed transactions should stay
committed. Like many good systems, it uses a write-ahead
log (WAL, or often called a "clog", "xlog", or "transaction
log"in Postgres lingo) to achieve this durability. Every
committed change is written and flushed to disk, and even
in the event of sudden termination, Postgres can replay
what it finds in the WAL to recover any changes didn't make
it into its data files.

`RecordTransactionCommit` from the snippet above handles
getting a change to the WAL:

``` c
static TransactionId
RecordTransactionCommit(void)
{
    bool markXidCommitted = TransactionIdIsValid(xid);

    /*
     * If we haven't been assigned an XID yet, we neither can, nor do we want
     * to write a COMMIT record.
     */
    if (!markXidCommitted)
    {
        ...
    } else {
        XactLogCommitRecord(xactStopTimestamp,
                            nchildren, children, nrels, rels,
                            nmsgs, invalMessages,
                            RelcacheInitFileInval, forceSyncCommit,
                            MyXactFlags,
                            InvalidTransactionId /* plain commit */ );

        ....
    }

    if ((wrote_xlog && markXidCommitted &&
         synchronous_commit > SYNCHRONOUS_COMMIT_OFF) ||
        forceSyncCommit || nrels > 0)
    {
        XLogFlush(XactLastRecEnd);

        /*
         * Now we may update the CLOG, if we wrote a COMMIT record above
         */
        if (markXidCommitted)
            TransactionIdCommitTree(xid, nchildren, children);
    }

    ...
}
```

Another core Postgres philosophy is performance. If a
transaction was never assigned a `xid` because it didn't
affect the state of the database, Postgres skips writing it
to the WAL. If a transaction was aborted, we write it to
the WAL, but don't bother to send a flush because even
though it's completed, it doesn't affect data so it's not
catastrophic if we lose it.

TODO: Confirm that the abort note above is true.

Note also that the WAL is written in two parts. We write
the bulk of the information out in `XactLogCommitRecord`,
and if the transaction committed, we go through in a second
pass with `TransactionIdCommitTree` and set the status of
each record to "committed". It's only after this operation
completes that we can formally say that the transaction was
durably committed.

### Defensive programming (#defensive-programming)

`TransactionIdCommitTree` (in [transam.c][committree], and
its implementation `TransactionIdSetTreeStatus` in
[clog.c][settreestatus]) commits a "tree" because a commit
may have subcommits. I won't get into subcommits at all
here, but it's worth nothing that because
`TransactionIdCommitTree` cannot be guaranteed to be
atomic, each subcommit is recorded as committed separately,
and the parent is recorded as a final step. When Postgres
is reading the WAL on recovery, subcommit records aren't
considered to be committed (even if they're marked as such)
until the parent record is read and confirmed committed.

Once again this is in the name of atomicity; the system
could have successfully recorded every subcommit, but then
crashed before it could write the parent.

### Signaling completion through shared memory (#shared-memory)

With the transaction recorded to WAL, it's safe to signal
its completion to the rest of the system, which is the
second call in `CommitTransaction` above (calls into
[procarray.c][endtransaction]):

``` c
void
ProcArrayEndTransaction(PGPROC *proc, TransactionId latestXid)
{
    /*
     * We must lock ProcArrayLock while clearing our advertised XID, so
     * that we do not exit the set of "running" transactions while someone
     * else is taking a snapshot.  See discussion in
     * src/backend/access/transam/README.
     */
    if (LWLockConditionalAcquire(ProcArrayLock, LW_EXCLUSIVE))
    {
        ProcArrayEndTransactionInternal(proc, pgxact, latestXid);
        LWLockRelease(ProcArrayLock);
    }

    ...
}

static inline void
ProcArrayEndTransactionInternal(PGPROC *proc, PGXACT *pgxact,
                                TransactionId latestXid)
{
    ...

    /* Also advance global latestCompletedXid while holding the lock */
    if (TransactionIdPrecedes(ShmemVariableCache->latestCompletedXid,
                              latestXid))
        ShmemVariableCache->latestCompletedXid = latestXid;
}
```

Remember how when we started a transaction we created a
snapshot and set its `xmax` to `latestCompletedXid + 1`? By
setting `latestCompletedXid` to the `xid` of the
transaction that just committed, we've just made its
results visible to every new transaction that starts from
this point forward.

## Checking visibility (#visibility)

We covered earlier how visibility information is stored on
heap tuples. `heapgettup` (in [heapam.c][gettup]) is the
method responsible for scanning the heap for tuples that
meet a snapshot's visibility criteria:

``` c
static void
heapgettup(HeapScanDesc scan,
           ScanDirection dir,
           int nkeys,
           ScanKey key)
{
    ...

    /*
     * advance the scan until we find a qualifying tuple or run out of stuff
     * to scan
     */
    lpp = PageGetItemId(dp, lineoff);
    for (;;)
    {
        /*
         * if current tuple qualifies, return it.
         */
        valid = HeapTupleSatisfiesVisibility(tuple,
                                             snapshot,
                                             scan->rs_cbuf);

        if (valid)
        {
            return;
        }

        ++lpp;            /* move forward in this page's ItemId array */
        ++lineoff;
    }

    ...
}
```

`HeapTupleSatisfiesVisibility` is a preprocessor macro that
will call into a "satisfies" function in `tqual.c`.
`HeapTupleSatisfiesMVCC` ([`tqual.c`][satisfies]) is one
such function:

``` c
bool
HeapTupleSatisfiesMVCC(HeapTuple htup, Snapshot snapshot,
                       Buffer buffer)
{
    ...

    else if (TransactionIdDidCommit(HeapTupleHeaderGetRawXmin(tuple)))
        SetHintBits(tuple, buffer, HEAP_XMIN_COMMITTED,
                    HeapTupleHeaderGetRawXmin(tuple));

    ...

    /* xmax transaction committed */

    return false;
}
```

And `TransactionIdDidCommit` ([from
`transam.c`][didcommit]):

``` c
bool /* true if given transaction committed */
TransactionIdDidCommit(TransactionId transactionId)
{
    XidStatus xidstatus;

    xidstatus = TransactionLogFetch(transactionId);

    /*
     * If it's marked committed, it's committed.
     */
    if (xidstatus == TRANSACTION_STATUS_COMMITTED)
        return true;

    ...
}
```

Further exploring the implementation of
`TransactionLogFetch` will reveal that it works as
advertised. It calculates a location in the WAL from the
given transaction ID and reaches into the WAL to get that
transaction's commit status. Whether or not the transaction
committed is used to help determine the tuple's visibility.

The key here is that for purposes of consistency, the WAL
is considered the canonical source for commit status [1]
(and by extension, visibility). The same information will
be returned regardless of whether Postgres successfully
committed a transaction hours ago, or seconds before a
crash that the server is just now recovering from.

### Hint bits (#hint-bits)

You may have noticed above that `HeapTupleSatisfiesMVCC`
does one more thing before returning from a visibility
check:

``` c
SetHintBits(tuple, buffer, HEAP_XMIN_COMMITTED,
            HeapTupleHeaderGetRawXmin(tuple));
```

Checking the WAL to see whether a tuple's `xmin` or `xmax`
transactions are committed is an expensive operation. To
avoid having to go to the WAL every time, Postgres will set
a special commit status flags called "hint bits" for a heap
tuple that its scanned. Subsequent operations can check the
tuple's hint bits and are saved a trip to the WAL
themselves.

[commit]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/access/transam/xact.c#L1939
[committree]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/access/transam/transam.c#L259
[didcommit]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/access/transam/transam.c#L124
[endtransaction]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/storage/ipc/procarray.c#L394
[gettup]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/access/heap/heapam.c#L478
[getsnapshotdata]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/storage/ipc/procarray.c#L1507
[pgxact]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/storage/proc.h#L207
[satisfies]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/utils/time/tqual.c#L962
[settreestatus]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/backend/access/transam/clog.c#L148
[snapshot]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/utils/snapshot.h#L52
[tuple]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/access/htup_details.h#L116
[tupleheaders]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/access/htup.h#L62
[xid]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/c.h#L397
[xidadvance]: https://github.com/postgres/postgres/blob/b35006ecccf505d05fd77ce0c820943996ad7ee9/src/include/access/transam.h#L31

[1] Note that changes will eventually be no longer
    available in the WAL, but those will always be beyond a
    snapshot's `xmin` horizon, and therefore the visibility
    check short circuits before having to make a check in
    WAL.