---
title: "Generalizing Seqlocks"
---
# The swym Algorithm

[swym](https://github.com/mtak-/swym) is a very performant Software Transactional Memory (STM) library. It uses a variation on the per-object Transactional Locking II algorithm. The [paper](https://people.csail.mit.edu/shanir/publications/Transactional_Locking.pdf) does an excellent job explaining the algorithm, but it is not required reading for this article. `swym` is a generalization of seqlocks - one the TL2 paper *almost* achieves, but does not for whatever reason.

## seqlocks

Seqlocks are a form of reader-writer lock with one very important property for read mostly data. **Readers are invisible - they don't mutate any shared state.** This means an arbitrary number of readers can coexist without any cache line invalidation. Seqlocks accomplish this with something normal locks are built to avoid, **they race**. Readers "race" with writers, but have a mechanism for detecting when races occur, and rerunning the read if there is a race. To put this in transactional memory terms, reads are speculative, conflicts are detected, aborted and retried.

Here's the definition of a seqlock:

```rust
pub struct Seqlock<T: Copy> {
    value:      UnsafeCell<T>,
    seq_number: AtomicUsize,
}
```

In addition to the `value` the lock protects, it also contains a sequence number, `seq_number`. This number starts at 0 and increases by two every time the value is changed. Why two? Well, if the value is odd, that means there's a writer currently modifying the value.

#### Seqlock writers 

```rust
pub fn write(&self, new_value: T) {
    // try to make seq_number odd (acquiring the lock)
    // retry if it was already odd
    while self.seq_number.fetch_or(1) & 1 != 0 {
        // std::thread::yield()
    }

    unsafe {
        // not just `*self.value.get() = new_value`
        // due to memory model
        assign(self.value.get(), new_value);
    } 
    
    // adding 1 makes seq_number even releasing the lock
    self.seq_number.fetch_add(1);
    
    // `seq_number` has been effectively incremented by 2
}
```

The writer algorithm (excluding details of memory ordering/fences/overflow):

1. Try to make `seq_number` odd, retry if it already was odd.
2. At this point the writer has exclusive write access, so it can assign `new_value` to `value`.
3. Release the write lock by adding 1 - making it even again.

Crucially, when the writer finishes, there is a *new* sequence number. To see why that's important let's look at the reader side of things.

#### Seqlock readers

```rust
pub fn read(&self) -> T {
    loop {
        // read seq_number before the value
        let seq0 = self.seq_number.load();

        let value = unsafe {
            // not just `*self.value.get()`
            // due to the memory model
            read(self.value.get())
        }; 

        // read seq_number after the value
        let seq1 = self.seq_number.load();

        // if they are equal, and not odd,
        // then the read did not race with a writer
        if seq0 == seq1 && seq0 & 1 == 0 {
            break value
        }
    }
}
```

The reader algorithm (excluding ordering/fence/overflow details):

1. Read `seq_number`.
2. Speculatively read `value`.
3. Read `seq_number` again.
4. If *1* and *3* are the same, and even, then the speculative read did not race with any writer. Otherwise, start back at *1*.

If `write` had not unlocked `seq_number` with a fresh never before existing even number, then a full write could interleave with a `read`.

With the seqlock algorithm sketched out, it's now time to start generalizing it.

## Single-Sequence Multiple Data

Seqlocks are great for providing atomic access to a single value, but they don't compose. If there are two values `t` and `u` each with their own seqlock, and a program wants to "atomically" read both of them, it's out of luck.

To fix that, without losing the ability to perform independent writes to `t` and `u` in parallel, allow me to introduce the Single Sequence Multiple Data (SSMD) structure.

```rust
// always even
static SEQ_NUMBER: AtomicUsize = AtomicUsize::new(0);

pub struct SSMD<T: Copy> {
    value:        UnsafeCell<T>,
    version_lock: AtomicUsize,
}
```

The sequence number is now a private global, and taking its place in the structure is a combined version number and lock.

Take some time to study what's happened here. Every `SSMD` will be sequenced with the *same* sequence number. That sequence number is no longer polluted with this notion of write-locking, yet every object can still be locked independently.

*Sidenote: If you're interested in STMs, the [NOrec](https://anon.cs.rochester.edu/u/scott/papers/2010_ppopp_NOrec.pdf) STM is a very simple STM that uses a single global seqlock for all transactional memory locations. This has obvious downsides for write heavy workloads. NOrec makes up for this with a mechanism to reacquire a sequence number in the middle of a transaction.*

#### SSMD writers

```rust
pub fn write(&self, new_value: T) {
    // try to make seq_number odd (acquiring the lock)
    // retry if it was already odd
    while self.version_lock.fetch_or(1) & 1 != 0 {
        // std::thread::yield()
    }

    unsafe {
        assign(self.value.get(), new_value);
    } 
    
    // increment SEQ_NUMBER by 2 (always even)
    let new_version = SEQ_NUMBER.fetch_add(2) + 2;
    // that new_version is stored into the version lock
    // releasing it.
    self.version_lock.store(new_version);
}
```

Only the last line has changed from `self.seq_number.fetch_add(1)` to instead bump the global sequence number and store the result into `self.version_lock`. In fact, so little has changed that the old reader code (changing `seq_number` to `version_lock`) would still work just fine! The careful observer will also notice that the per-object `version_lock` is able to lag arbitrarily far behind the global `SEQ_NUMBER`. Readers will take advantage of that property.

#### SSMD readers (part 1)

For now, let's forget about composability, and just solve the problem of reading from two `SSMD<T>`s atomically. One possible solution looks like this:

```rust
pub fn read_pair<T, U>(t: &SSMD<T>, u: &SSMD<U>) -> (T, U)
where
    T: Copy,
    U: Copy,
{
    loop {
        // read seq_number before the values
        let seq = SEQ_NUMBER.load();

        // speculatively read from t first
        let t_value = unsafe { read(t.value.get()) };

        // check for races on t
        let t_version = t.version_lock.load();
        if t_version > seq || t_version & 1 != 0 {
            // if there is a race, start over
            continue;
        }

        // repeat the above for u
        let u_value = unsafe { read(u.value.get()) };
        let u_version = u.version_lock.load();
        if u_version > seq || u_version & 1 != 0 {
            continue;
        }
        
        break (t_value, u_value)
    }
}
```

The read algorithm is as follows (ignoring overflows/fences/ordering):

1. Read from the sequence number.
2. Read the first value.
3. Verify the version of that value was set before our read from the sequence number and the write lock is not held. On fail, retry from **1**
4. Read the second value.
5. Verify the version of that value was set before our read from the sequence number and the write lock is not held. On fail, retry from **1**
6. Return both values.

Even if you're experienced at [juggling razor blades](https://www.youtube.com/watch?v=c1gO9aB9nbs), it's significantly less obvious that this algorithm is safe when compared with Seqlocks, and harder still to see that it is an atomic read of both values. Perhaps this is why the original TL2 algorithm had reads from each objects version number *before* and after loads from the value (though it does make TL2's commit algorithm a little simpler).

First, I'll informally prove that no torn reads are ever returned (again ignoring overflow).

**Case 1:** A write lock is held before the reader starts and the sequence number has not yet been bumped.
1. The reader reads the soon to be out of date sequence number.
2. The reader reads the value, possibly torn or more recent than the sequence number.
3. Then while verifying the version number, it either sees:
    - The write lock is still held, in which case it aborts and retries.
    - The sequence number it read is older than the version number (write has finished), in which case it aborts and retries.

**Case 2:** A write lock is held before the reader starts *but* the sequence number has already been bumped.
1. The reader reads the new sequence number.
2. The reader reads the value - this time it's not torn because the sequence number has already been bumped.
3. Then while verifying the version number, it either sees:
    - The write lock is still held, in which case it aborts and retries (a conservative failure).
    - The write lock is no longer held and the version number matches the sequence number read at 1. The read is successful.

Other cases where a write lock is aquired after the read has acquired a sequence number are no different than a regular seqlock.

Now for a handwaving explanation of why the read of the pair is atomic. What matters for atomicity is that both values returned were stored in memory at the exact moment `SEQ_NUMBER` was incremented to the value `fn read` sees. The commit algorithm later in the article will ensure that writes obey this rule.

#### SSMD readers (part 2 - enter transactions)

The `read_pair` algorithm above is screaming for an abstraction. Why stop at reading from just two values atomically? Why not handle an arbitrary number of values?

To do this, a way to encapsulate the read of the sequence number into some newtype wrapper is necessary: `Transaction`.

```rust
pub struct Transaction(usize);
pub struct Error { .. }

impl<T: Copy> SSMD<T> {
    pub fn get(&self, tx: &Transaction)
        -> Result<T, Error>
    {
        let value = unsafe { read(self.value.get()) };
        let version = self.version_lock.load();
        if version > tx.0 || version & 1 != 0 {
            // return an error instead of `continue`.
            Err(Error { .. })
        } else {
            Ok(value)
        }
    }
}
```

`get` is the repeated steps (2 and 3) of the `read_pair` algorithm, except instead of `continue` it returns an error.

Still missing, is the `loop` - a way to run transactions:

```rust
pub fn read_only<F, O>(mut f: F) -> O
where
    F: FnMut(&Transaction) -> Result<O, Error>,
{
    // don't support nested transactions
    assert!(!thread::in_transaction());
    thread::enter_transaction();

    // the outer loop from `read_pair`
    loop {
        let tx = Transaction(SEQ_NUMBER.load());
        let r = f(&tx);
        match r {
            Ok(o) => {
                thread::leave_transaction();
                return o;
            },
            Err(_) => {},
        }
    }
}
```

Unbounded read only transactions are supported! Hurrah! There are a few caveats. Nested transactions are not handled, and reads and writes cannot be mixed.

In `swym`, when [ThreadKey::read](https://docs.rs/swym/0.1.0-preview/swym/thread_key/struct.ThreadKey.html#method.read) is called, the above code is more or less run. No logging, no garbage collection, no commit algorithm, etc. No shared state is modified, **making read only transactions invisible** just like seqlock readers.

## Mixing reads and writes (TCell)

It's time to upgrade `SSMD` into a full fledged [TCell](https://docs.rs/swym/0.1.0-preview/swym/tcell/struct.TCell.html) by adding support for mixing reads and writes in the same transaction. This mixing of reads and writes will eventually require a logging mechanism.

To keep things simple, let's examine a case where all the memory that will be read and written is known ahead of time. In the following example, values are copied from "reads" to "writes" atomically. *Note: It's impossible to write a transactional linked list or most other data structures under such a system.*

```rust
pub fn copy_to(
    reads: &Vec<TCell<usize>>,
    writes: &Vec<TCell<usize>>,
) {
    assert!(!thread::in_transaction());
    thread::enter_transaction();

    let mut values;
    'start: loop {
        // create a transaction
        let tx = Transaction(SEQ_NUMBER.load());

        // grab all of the write locks
        for idx in 0..writes.len() {
            if writes[idx]
                    .version_lock
                    .fetch_or(1) & 1 != 0
            {
                // failed to acquire a write lock
                // so unlock all the locks acquired
                unlock_writes_before(writes, idx);
                continue 'start;
            }
        }

        values = Vec::new();
        // speculatively perform all of the reads
        for src in reads.iter() {
            match src.get(&tx) {
                Ok(o) => values.push(o),
                Err(_) => {
                    // failed to read, so unlock and start
                    // over
                    unlock_writes(writes);
                    continue 'start
                },
            };
        }
        break;
    }

    // perform all of the writes
    for (value, dst) in values.into_iter().zip(writes) {
        unsafe {
            assign(dst.value.get(), value);
        }
    }

    // update the sequence number, and set the version
    // numbers to that value
    let new_version = SEQ_NUMBER.fetch_add(2) + 2;
    for dst in dest.iter() {
        dst.version_lock.store(new_version)
    }
    thread::leave_transaction();
}
```

That's a big soggy ball of code up there, but it's essentially `swym`'s commit algorithm.

The algorithm in words:
1. Grab all of the locks to `TCell`s that will be written - retry if acquiring a lock fails.
2. Perform all of the reads, unlocking/retrying if one fails.
3. Perform all of the writes.
4. Update the sequence number.
5. Unlock every `TCell` in the write list with the new sequence number.

Hopefully the shape of that algorithm is pretty clearly a mashup of `fn write` and `fn read_only`. It looks kinda like the reads could be performed before acquiring the write locks, but that would be subtly broken. Not in any way that's `unsafe`, but it would create an observable smearing of reads across time. After reading, an interleaving transaction could successfully update values that appear in the read set, and by the time this transaction commits, the values it writes would be from out of date reads! That breaks the Single Lock Atomicity (SLA) guarantee that STM's are built around. But... it's not `unsafe`, and it's definitely faster, so perhaps there are uses for it.

When all values in the write set don't depend on a specific read, that read can be performed using [Ordering::Read](https://docs.rs/swym/0.1.0-preview/swym/tx/enum.Ordering.html#variant.Read) without breaking SLA. In at least one [paper](https://www.ssrg.ece.vt.edu/papers/disc16-TR.pdf) this has been referred to as a *reverse-commit anti-dependency* (RCAD). [swym-rbtree](https://github.com/mtak-/swym/tree/master/swym-rbtree) makes heavy use of `Ordering::Read`. Finding the location to insert a new element into the rbtree can use `Ordering::Read` to traverse the tree, as long as the stronger `Ordering::ReadWrite` (SLA) is used for balancing the tree afterwards.

Let's jump ahead to the final commit algorithm.

#### swym's commit algorithm

Transactions in `swym` accumulate a log of all the read `TCell`s and a *redo* log of all the pending writes to be performed along with their destination `TCell`s. Writes are queued up and performed at commit time in an atomic fashion similar to the above function.

```rust
pub struct Transaction<'a> {
    // read from SEQ_NUMBER
    seq: usize,

    // all TCells that have been read from
    read_log: Vec<&'a TCell<usize>>,
    
    // queued up list of writes and destinations
    redo_log: Vec<(usize, &'a TCell<usize>)>,
}
```

Of course `swym` supports more than just `usize`, and it's not limited to `Copy` types, but those details aren't particularly relevant here.

Modifying `copy_to` to use the log information in the transaction is straightforward:

```rust
fn commit(tx: Transaction<'_>) -> Result<(), Error> {
    let Transaction {
        seq,
        read_log,
        redo_log,
    } = tx;

    // grab all of the write locks
    for idx in 0..redo_log.len() {
        if redo_log[idx]
            .1
            .version_lock
            .fetch_or(1) & 1 != 0
        {
            // failed to acquire a write lock
            // so unlock all the locks acquired
            unlock_writes_before(redo_log, idx);
            return Err(Error { .. });
        }
    }

    // validate all of the reads
    for src in read_log {
        let version = src.version_lock.load();
        if (version & 1 != 0 && !redo_log.contains(src))
            || version > seq
        {
            unlock_writes(redo_log);
            return Err(Error { .. });
        }
    }

    // perform all of the writes
    for (value, dst) in redo_log.iter() {
        unsafe {
            assign(dst.value.get(), value);
        }
    }

    // update the sequence number, and set the version
    // numbers to that value
    let new_version = SEQ_NUMBER.fetch_add(2) + 2;
    for (_, dst) in redo_log {
        dst.version_lock.store(new_version)
    }

    thread::leave_transaction();

    Ok(())
}
```

Algorithm:
1. Grab all of the locks to `TCell`s in the redo_log - returning an error on failure.
2. Check that none of the reads have been modified since transaction start, unlocking/retrying if one fails.
3. Perform all of the writes.
4. Update the sequence number.
5. Unlock every `TCell` in the redo log with the new sequence number.

Pretty similar to `copy_to`, eh? All that's left is to add elements to the read and redo logs.

#### TCell::set()

```rust
fn set<'a>(&'a self, tx: &mut Transaction<'a>, value: T) {
    match tx
        .redo_log
        .iter_mut()
        .find(|(_, dst)| std::ptr::eq(dst, self))
    {
        Some((ref mut pending, _)) => *pending = value,
        None => tx.redo_log.push((value, self)),
    }
}
```

Nothing fancy. If the `TCell` is in the redo log already, modify the element to "redo". If not, add the `TCell` and the value to the redo log.

#### TCell::get()

```rust
fn get<'a>(
    &'a self,
    tx: &mut Transaction<'a>,
) -> Result<T, Error> {
    match tx
        .redo_log
        .iter_mut()
        .find(|(_, dst)| std::ptr::eq(dst, self))
    {
        Some((ref mut pending, _)) => {
            tx.read_log.push(self);
            Ok(*pending)
        }
        None => {
            let value = unsafe { read(self.value.get()) };
            let version = self.version_lock.load();
            if version > tx.seq || version & 1 != 0 {
                Err(Error { .. })
            } else {
                tx.read_log.push(self);
                Ok(value)
            }
        }
    }
}
```

`get` is very similar to `SSMD::get` except it adds self to the `read_log` and checks the `redo_log` for any writes made during the transaction. Without that check, Single Lock Atomicity would be violated.


#### ThreadKey::rw

Time for the outer [retry loop](https://docs.rs/swym/0.1.0-preview/swym/thread_key/struct.ThreadKey.html#method.rw) for read write transactions:

```rust
pub fn rw<F, O>(&'a self, mut f: F) -> O
where
    F: FnMut(&mut Transaction<'a>) -> Result<O, Error>,
{
    // don't support nested transactions
    assert!(!thread::in_transaction());
    thread::enter_transaction();

    loop {
        // acquire a new sequence number and
        // initialize the logs
        let mut tx = Transaction {
            seq: SEQ_NUMBER.load(),
            read_log: Vec::new(),
            redo_log: Vec::new(),
        };

        // user code which calls `get`/`set` an arbitrary
        // number of times.
        let r = f(&mut tx);
        
        match r {
            // user code wants to commit
            Ok(r) => {
                // try to commit
                if let Ok(()) = commit(tx) {
                    // success!
                    return r;
                }
            }
            Err(Error { .. }) => {}
        }
        // something failed, retry
    }
}
```

By this point, that code should look pretty reasonable.

## Conclusion

Much of the non-design work that has gone into `swym` was in optimizing the read and redo logs for lookups, and appending. Hardware transactional memory will soon find a place in the commit algorithm. HTM can cut back the number of heavyweight atomic operations from `redo_log.len()` to `1` - albeit an expensive `1` :)

Some topics that may be covered in future articles:
- support for non-`Copy` types
- publication/privatization
- garbage collection

Thanks for sticking with this article to the end! I hope it explained the `swym` algorithm well, even if the details of the implementation are a little different from what was presented.