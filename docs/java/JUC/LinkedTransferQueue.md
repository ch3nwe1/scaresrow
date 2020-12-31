# LinkedTransferQueue

## overview

```java
/*
* We use a threshold-based approach to updates, with a slack
* threshold of two -- that is, we update head/tail when the
* current pointer appears to be two or more steps away from the
* first/last node. The slack value is hard-wired: a path greater
* than one is naturally implemented by checking equality of
* traversal pointers except when the list has only one element,
* in which case we keep slack threshold at one. Avoiding tracking
* explicit counts across method calls slightly simplifies an
* already-messy implementation. Using randomization would
* probably work better if there were a low-quality dirt-cheap
* per-thread one available, but even ThreadLocalRandom is too
* heavy for these purposes.
* 
* 
* With such a small slack threshold value, it is not worthwhile
* to augment this with path short-circuiting (i.e., unsplicing
* interior nodes) except in the case of cancellation/removal (see
* below).
*
* We allow both the head and tail fields to be null before any
* nodes are enqueued; initializing upon first append.  This
* simplifies some other logic, as well as providing more
* efficient explicit control paths instead of letting JVMs insert
* implicit NullPointerExceptions when they are null.  While not
* currently fully implemented, we also leave open the possibility
* of re-nulling these fields when empty (which is complicated to
* arrange, for little benefit.)
*
* All enqueue/dequeue operations are handled by the single method
* "xfer" with parameters indicating whether to act as some form
* of offer, put, poll, take, or transfer (each possibly with
* timeout). The relative complexity of using one monolithic
* method outweighs the code bulk and maintenance problems of
* using separate methods for each case.
*
* Operation consists of up to three phases. The first is
* implemented within method xfer, the second in tryAppend, and
* the third in method awaitMatch.
*
* 1. Try to match an existing node
*
*    Starting at head, skip already-matched nodes until finding
*    an unmatched node of opposite mode, if one exists, in which
*    case matching it and returning, also if necessary updating
*    head to one past the matched node (or the node itself if the
*    list has no other unmatched nodes). If the CAS misses, then
*    a loop retries advancing head by two steps until either
*    success or the slack is at most two. By requiring that each
*    attempt advances head by two (if applicable), we ensure that
*    the slack does not grow without bound. Traversals also check
*    if the initial head is now off-list, in which case they
*    start at the new head.
*
*    If no candidates are found and the call was untimed
*    poll/offer, (argument "how" is NOW) return.
*
* 2. Try to append a new node (method tryAppend)
*
*    Starting at current tail pointer, find the actual last node
*    and try to append a new node (or if head was null, establish
*    the first node). Nodes can be appended only if their
*    predecessors are either already matched or are of the same
*    mode. If we detect otherwise, then a new node with opposite
*    mode must have been appended during traversal, so we must
*    restart at phase 1. The traversal and update steps are
*    otherwise similar to phase 1: Retrying upon CAS misses and
*    checking for staleness.  In particular, if a self-link is
*    encountered, then we can safely jump to a node on the list
*    by continuing the traversal at current head.
*
*    On successful append, if the call was ASYNC, return.
*
* 3. Await match or cancellation (method awaitMatch)
*
*    Wait for another thread to match node; instead cancelling if
*    the current thread was interrupted or the wait timed out. On
*    multiprocessors, we use front-of-queue spinning: If a node
*    appears to be the first unmatched node in the queue, it
*    spins a bit before blocking. In either case, before blocking
*    it tries to unsplice any nodes between the current "head"
*    and the first unmatched node.
*
*    Front-of-queue spinning vastly improves performance of
*    heavily contended queues. And so long as it is relatively
*    brief and "quiet", spinning does not much impact performance
*    of less-contended queues.  During spins threads check their
*    interrupt status and generate a thread-local random number
*    to decide to occasionally perform a Thread.yield. While
*    yield has underdefined specs, we assume that it might help,
*    and will not hurt, in limiting impact of spinning on busy
*    systems.  We also use smaller (1/2) spins for nodes that are
*    not known to be front but whose predecessors have not
*    blocked -- these "chained" spins avoid artifacts of
*    front-of-queue rules which otherwise lead to alternating
*    nodes spinning vs blocking. Further, front threads that
*    represent phase changes (from data to request node or vice
*    versa) compared to their predecessors receive additional
*    chained spins, reflecting longer paths typically required to
*    unblock threads during phase changes.
*
*
* ** Unlinking removed interior nodes **
*
* In addition to minimizing garbage retention via self-linking
* described above, we also unlink removed interior nodes. These
* may arise due to timed out or interrupted waits, or calls to
* remove(x) or Iterator.remove.  Normally, given a node that was
* at one time known to be the predecessor of some node s that is
* to be removed, we can unsplice s by CASing the next field of
* its predecessor if it still points to s (otherwise s must
* already have been removed or is now offlist). But there are two
* situations in which we cannot guarantee to make node s
* unreachable in this way: (1) If s is the trailing node of list
* (i.e., with null next), then it is pinned as the target node
* for appends, so can only be removed later after other nodes are
* appended. (2) We cannot necessarily unlink s given a
* predecessor node that is matched (including the case of being
* cancelled): the predecessor may already be unspliced, in which
* case some previous reachable node may still point to s.
* (For further explanation see Herlihy & Shavit "The Art of
* Multiprocessor Programming" chapter 9).  Although, in both
* cases, we can rule out the need for further action if either s
* or its predecessor are (or can be made to be) at, or fall off
* from, the head of list.
*
* Without taking these into account, it would be possible for an
* unbounded number of supposedly removed nodes to remain
* reachable.  Situations leading to such buildup are uncommon but
* can occur in practice; for example when a series of short timed
* calls to poll repeatedly time out but never otherwise fall off
* the list because of an untimed call to take at the front of the
* queue.
*
* When these cases arise, rather than always retraversing the
* entire list to find an actual predecessor to unlink (which
* won't help for case (1) anyway), we record a conservative
* estimate of possible unsplice failures (in "sweepVotes").
* We trigger a full sweep when the estimate exceeds a threshold
* ("SWEEP_THRESHOLD") indicating the maximum number of estimated
* removal failures to tolerate before sweeping through, unlinking
* cancelled nodes that were not unlinked upon initial removal.
* We perform sweeps by the thread hitting threshold (rather than
* background threads or by spreading work to other threads)
* because in the main contexts in which removal occurs, the
* caller is already timed-out, cancelled, or performing a
* potentially O(n) operation (e.g. remove(x)), none of which are
* time-critical enough to warrant the overhead that alternatives
* would impose on other threads.
*
* Because the sweepVotes estimate is conservative, and because
* nodes become unlinked "naturally" as they fall off the head of
* the queue, and because we allow votes to accumulate even while
* sweeps are in progress, there are typically significantly fewer
* such nodes than estimated.  Choice of a threshold value
* balances the likelihood of wasted effort and contention, versus
* providing a worst-case bound on retention of interior nodes in
* quiescent queues. The value defined below was chosen
* empirically to balance these under various timeout scenarios.
*
* Note that we cannot self-link unlinked interior nodes during
* sweeps. However, the associated garbage chains terminate when
* some successor ultimately falls off the head of the list and is
* self-linked.
*/
```

```java
public void put(E e) {
    xfer(e, true, ASYNC, 0);
}
```

```java
public boolean offer(E e, long timeout, TimeUnit unit) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```

```java
public boolean offer(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```

```java
public boolean add(E e) {
    xfer(e, true, ASYNC, 0);
    return true;
}
```

```java
public boolean tryTransfer(E e) {
    return xfer(e, true, NOW, 0) == null;
}
```

```java
public void transfer(E e) throws InterruptedException {
    if (xfer(e, true, SYNC, 0) != null) {
        Thread.interrupted(); // failure possible only due to interrupt
        throw new InterruptedException();
    }
}
```

```java
public boolean tryTransfer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (xfer(e, true, TIMED, unit.toNanos(timeout)) == null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}
```

```java
public E take() throws InterruptedException {
    E e = xfer(null, false, SYNC, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = xfer(null, false, TIMED, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
```

```java
public E poll() {
    return xfer(null, false, NOW, 0);
}
```

```java
//
private E xfer(E e, boolean haveData, int how, long nanos) {
    //如果是存放数据, 元素不能为空
    if (haveData && (e == null))
        throw new NullPointerException();
    Node s = null;                        // the node to append, if needed

    retry:
    for (;;) {                            // restart on append race

        for (Node h = head, p = h; p != null;) { // find & match first node
            boolean isData = p.isData;
            Object item = p.item;
            if (item != p && (item != null) == isData) { // unmatched
                if (isData == haveData)   // can't match
                    break;
                if (p.casItem(item, e)) { // match
                    for (Node q = p; q != h;) {
                        Node n = q.next;  // update by 2 unless singleton
                        if (head == h && casHead(h, n == null ? q : n)) {
                            h.forgetNext();
                            break;
                        }                 // advance and retry
                        if ((h = head)   == null ||
                            (q = h.next) == null || !q.isMatched())
                            break;        // unless slack < 2
                    }
                    LockSupport.unpark(p.waiter);
                    return LinkedTransferQueue.<E>cast(item);
                }
            }
            Node n = p.next;
            p = (p != n) ? n : (h = head); // Use head if p offlist
        }

        if (how != NOW) {                 // No matches available
            if (s == null)
                s = new Node(e, haveData);
            Node pred = tryAppend(s, haveData);
            if (pred == null)
                continue retry;           // lost race vs opposite mode
            if (how != ASYNC)
                return awaitMatch(s, pred, e, (how == TIMED), nanos);
        }
        return e; // not waiting
    }
}
```