---
title: "swym: Are we lock-free yet?"
---

# swym is not lock-free

[swym](https://github.com/mtak-/swym) is a transactional memory library that prioritizes performance. It's not lock-free, but it does have progress guarantees.

This post will explore some of the recent work on swym's progress promises, as well as some comparisons with non-blocking algorithms. I'm not an expert on schedulers or OS's, so please correct me if anything is wrong. It might benefit swym!

## What exactly is progress?

Progress is about a thread or a process gaining access to a resource to perform _some_ work. For concurrent containers that resource is shared memory, and the work is usually wrapped up in a method call (e.g. `conc_vec.push(5)`). swym considers a thread to have made progress if a transaction has successfully committed, encountered an uncaught panic, or returned `AWAIT_RETRY`. Progress contrasts with starvation where access to the resource is continually denied and one or more threads are unable to make any progress.

## Non-blocking guarantees

swym is attempting to be a building block for data structures that compete with [non-blocking](https://en.wikipedia.org/wiki/Non-blocking_algorithm) implementations. The three different flavors of non-blocking algorithms can be roughly summed up like so:

- **Wait-freedom**: All threads will make progress in a finite amount of steps.
- **Lock-freedom**: The process will make progress in a finite amount of steps.
- **Obstruction-freedom**: A thread will make progress in a finite amount of steps if all other threads are preempted at any point in their execution.

Locks, unsurprisingly, are blocking and offer none of the above progress guarantees. But loads of programs use locks somewhere - calls to malloc, println, acquiring file handles, etc. _might_ acquire locks. Even programs with semi-realtime constraints like video games use locks. Barring bugs, these programs still make progress. What's going on?

## The scheduler

With the exception of obstruction-free algorithms, which permit livelock, non-blocking algorithms provide progress guarantees regardless of how the operating system chooses to schedule threads. This is a fantastic property, but it's often not required. Setting aside deadlock, and other programmer errors, locks generally guarantee progress if the scheduler is reasonably *fair*.

Imagine this scenario involving a simple spinlock:
1. thread A acquires a spinlock
2. thread A is preempted by B
3. thread B attempts to acquire the spinlock
4. thread B spins furiously making no progress
5. thread A preempts B and releases the spinlock
6. thread B gets rescheduled and can now acquire the spinlock and make progress

Consider whether or not the scheduler guarantees that thread A will wakeup and release the spinlock. What if thread B has a higher priority than thread A? When a thread is waiting on a resource owned by a thread with a lower priority it's called a [priority inversion](https://en.wikipedia.org/wiki/Priority_inversion). On most OS's the scheduler will still eventually wakeup A and give it a chance to release the spinlock unblocking B. These schedulers are fair. The spinlock is not. iOS has an unfair scheduler which might continually schedule B preventing A from waking up and either thread from making progress - effectively a deadlock (assuming a single core, or pinning etc).

To [fix](https://developer.apple.com/documentation/os/1646466-os_unfair_lock_lock) the [problem on iOS](https://www.mikeash.com/pyblog/friday-qa-2017-10-27-locks-thread-safety-and-swift-2017-edition.html) a lock that is capable of communicating the priority inversion to the scheduler should be used. One way to communicate the inversion might be by parking B. Parking B tells the scheduler to stop giving B any CPU time. Thread A is then allowed to run until it releases the lock at which point A tells the scheduler to unpark B.

If the lock wanted to be fair, it should unpark threads based on when they requested the lock and give ownership of the lock to the next unparked thread - a handoff.

## Blocking guarantees

I'd guess there are also three main types of blocking algorithms, though I couldn't find a list. These properties generally either require a means of communicating priority inversion to the scheduler or the scheduler to be reasonably fair.

- **Fair**: Every thread makes progress if critical sections are finite (`parking_lot::Mutex`).
- **Unfair**: The program makes progress if critical sections are finite (spinlocks).
- **Livelock**: If only one thread is attempting to make progress it will. Under contention, livelock is possible with no threads making progress (C++'s [`std::lock`](https://en.cppreference.com/w/cpp/thread/lock)).

These are very similar to the non-blocking guarantees except that they often require expensive coordination with the scheduler and, under contention, the amount of time spent waiting for access to a resource is substantially greater and more unpredictable. Locks make it easy to have large critical sections whereas lock-free code typically has many single instruction critical sections. Additionally these blocking guarantees don't compose naturally. Care must be taken when combining multiple blocking algorithms in the same application.

## swym until recently

Until recently livelock in swym was possible, but very unlikely to last.

swym runs user code entirely speculatively. Only when the transaction is ready to be committed does it acquire any sort of lock.

```rust
thread_key::get().rw(|tx| {
    // Speculative. No locks are held.
    // ...
    Ok(()) // Just after this, swym
           // acquires a lock for each
           // entry in the write set.
});
```

During commit swym attempts to grab a lock for each entry in the write set. On a modern Intel CPU, swym grabs all of these locks, performs all of the writes, and validates the read set atomically using **hardware transactional memory**. If that hardware transaction is successful, it leaves the write locks in the locked state. After that, the global epoch clock must be bumped, and the locks released with the new epoch. This leaves a small window where scheduler is able to switch out the thread while the locks are _visibly_ held. Therefore, **swym's algorithm is blocking**.

This HTM fast path does not permit livelock; however, the software fallback which is run when there are too many hardware failures _does_ permit livelock. That could be prevented by acquiring the write log locks in a predefined order, but livelock is so unlikely to be persistent that the cost of sorting is not worth it - even for older CPUs.

The biggest pain point with swym's algorithm was that starvation was very likely in certain scenarios. A long running transaction which continually contends with short transactions was unlikely to ever succeed. By the time the long running transaction enters the commit phase, validating the read and write sets would fail. **Constructing a failing test for this was easy.**

## swym is now eventually fair

Taking a queue from [`parking_lot_core`](https://docs.rs/parking_lot_core/0.5.0/parking_lot_core/), swym now has an eventually fair algorithm. If a transaction fails, it is retried after performing some backoff. When a certain threshold is hit, the thread is considered to be "starving". A starving thread broadcasts its presence by grabbing the `starve_lock`. Other threads simply check if the starve lock is held _just_ before committing. If the lock is held, then the non-starving threads will be parked. When the starving thread succeeds, it will either unpark all the threads allowing them to commit, or it will handoff control of the lock to the oldest starving thread.

After grabbing the `starve_lock` a transaction still might fail, but will eventually succeed provided the scheduler itself is fair.

This scheme is a better choice than something like a `RwLock` around every transaction for several reasons:
- Transactions with an empty write set are never parked.
- Threads only "starve" if the backoff algorithm has been unsuccessful.
- Non-starving threads are invisible unless there is a concurrent starving thread.
- Non-starving threads are permitted to run until commit - allowing some overlap with the starving thread.
- It only requires the `Relaxed` memory ordering because this is strictly a backoff algorithm and provides only _eventual_ mutual exclusion.

From a swym user perspective, **transactions must be treated as though they are running with a lock held**. If a mutex is grabbed before starting a transaction, and a second starving thread grabs the same mutex inside of a transaction, deadlock can occur. Furthermore, swym presently requires **both** a way to communicate priority inversion _and_ a fair scheduler. In the future, swym might only require a way to communicate priority inversion.

## Note on performance

Testing the new fairness scheme out on the demo transactional red-black tree data structure revealed that threads rarely starve. It's likely starvation happens when the red-black tree requires a lot of rebalancing. These benchmarks are intended to match the way lock-free data structures might be used. Tests run on a 4 core macbook pro.

Inserting 100,000 elements, using 4 threads, 300 times (a total of 30 million successful transactions) showed about 220 total starvation events which caused 374 transactions to wait for a starving thread to commit. 0.0007% of transactions starved.

Performing the same test using 8 threads (on a 4 core machine - still 30 million transactions) showed about 570 starvation events which caused a ~2200 transactions to wait for a starving thread to commit. 0.0019% of transactions starved.

Overall, there may have been (?) a very very slight performance gain from starvation freedom (<0.5%). Probly because of how the specific benchmark is structured - a fixed amount of work for each thread. The wall clock time of the benchmarks may have decreased, but only because of increased CPU utilization at the very end of the benchmark.

## Conclusion

The new fairness guarantee means there's less gotchas when using swym. It's safe to assume your transactions will eventually commit, and in a reasonable amount of time. I'm _nearly_ satisfied with swyms handling of corner cases, and worst case runtimes. Thanks for reading!
