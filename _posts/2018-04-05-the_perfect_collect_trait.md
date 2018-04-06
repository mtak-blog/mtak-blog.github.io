---
title: "Writing the Perfect `Collect` Trait"
---
I've been spending some time thinking about garbage collection in rust. I know, shame on me, it's a systems language, we hate garbage collection, but...  even in a systems programming language, garbage collection is still pretty damn useful. Hell, even the linux kernel has [`call_rcu`](http://lse.sourceforge.net/locking/rcu/HOWTO/descrip.html).

### Garbage Collection

Let's start with a definition.

> *the automatic process of making space in a computer's memory by removing data that is no longer required or in use.* - [oed](https://en.oxforddictionaries.com/definition/garbage_collection)

This definition could easily encompass any memory living on the stack (see C's use of [`auto`](http://en.cppreference.com/w/c/language/storage_duration)); however, that's too broad an interpretation to be useful, nowadays garbage collection is usually restricted to memory on the heap.

What about reference counting à la [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html)? This type often lives on the stack even though it's responsible for managing the memory of data living on the heap. I think it'd be easy to get a lot of people to agree that this counts as garbage collection, but they'd probably feel the need to explain how it's different from a typical garbage collection scheme. It's deterministic, there's no runtime involved, etc. It's just not what comes to mind when you hear "garbage collection".

### Lifetimes

The memory managed by `Rc` is tied to a specific (runtime) lifetime. This lifetime ends when the last instance of `Rc` referencing the memory gets `drop`'ed. That's actually pretty specific in the grand scheme of things.

```rust
fn consume_len_x10<T>(x: Rc<Vec<T>>) -> usize {
    let z;
    {
        let y = x;
        z = y.len();
    } // the Vec pointed to by x might be free'd here

    // the Vec can't be free'd for the rest of the function
    let result = z * 5;
    result * 2
}
```

This is really, really cool. All the code after that first closing brace cannot panic (barring overflow), _and_ doesn't call `free`. In fact, the memory once owned by `x` won't be touched at all. Garbage collection, as folks tend to think of it, doesn't provide this strong of a guarantee.

Let's review that example with an imagined garbage collected pointer, `Gc`.

```rust
fn consume_len_x10<T>(x: Gc<Vec<T>>) -> usize {
    let z;
    {
        let y = x;
        z = y.len();
    } // the Vec might be moved, freed, or reused here
    
    let result = z * 5;
    // the Vec might be moved, freed, or reused here
    result * 2
} // the Vec might be moved, freed, or reused here
```

This is also really neat, for different reasons. Garbage collection allows you to batch up deallocation of objects, move their memory around, and avoid incrementing/decrementing reference counts. All of those hidden actions (usually thought of as maybe happening on every line of source) is typically what comes to mind when people think of garbage collection.

There's a clear sacrifice though. The useful concept of a lifetime is lost in this sort of scheme. All we have is a lower bound on the lifetime -- `Vec` won't be `drop`'ed before the first `}`. If that's all we have, then `T` isn't allowed to be just any type as in the `Rc` example. `T` must have some restrictions.

### The Collect Trait

```rust
fn consume_len_x10<T: Collect>(x: Gc<Vec<T>>) -> usize;
```

In the above snippet I've bounded `T` by the trait `Collect` which represents the requirements for a type to be garbage collectible. Let's examine some types and figure out which ones should implement `Collect`.

 - `i32`: Clearly this should implement `Collect`. If we `drop` and `free` any integer _later_ than we expect to `drop` it, that cannot possibly lead to unsafety.
 
 - `String`: This should also implement `Collect`. `String`s own all of their data, so they can never have dangling references. `String`s also don't do any funny self-referential tricks, so they can be freely moved around in memory without invalidating any pointers in the `String`.

 - `&'a i32`: References can also implement `Collect`. The reference may wind up outliving the data it refers to, but that's "ok". `Gc` will only ever move, or `drop` it, never dereference it.

 - `struct C<'a>(&'a i32)`: `C` is another happy `Collect` implementor with one caveat. `C`, might outlive `'a`. If `C` reads from the contained reference in `drop`, then that's a use-after-free -- unless `'a` is `'static`. So the caveat is, "either `C` cannot implement `Drop`, or `'a` is `'static`".

Two things jump out at me:
1. `T: 'static `implies` T: Collect`.

2. `needs_drop::<T>() == false `implies` T: Collect`.

This leaves out types like:
```rust
// where 'a != 'static
type D<'a> = (String, &'a i32); // a tuple
```

Even though, it's pretty clear that if `&'a i32` implements `Collect`, and `String` implements `Collect`, then a tuple of the two should implement `Collect`. If we widened case "2." from `needs_drop` to `nontrivial_drop` for some definition of non-trivial, the tuple case could be handled.

Unfortunately, writing a broad `Collect` trait is impossible in rust (if you know some secret sauce, please share it). Rust gives you two type-level tools for peeking at drop behavior, `Copy` and `'static`. Each implies `Collect`. Or'ing those bounds together is impossible even in nightly rust (compiles with overlapping marker traits, but doesn't do what you'd expect). Even if we could 'or' them together, I don't see a way to generically support the tuple example, without making the `Collect` trait _too_ broad.

### Conclusion

Rust as a language tends to force you to be conservative. Maybe one day we'll have a trait version of `nontrivial_drop::<T>()`, but for now, I'm gonna settle on a definition of `Collect` that's "good enough".

Looking around at various `Gc` like things (e.g. [`crossbeam-epoch`](https://github.com/crossbeam-rs/crossbeam-epoch/), and [`rust-gc`](https://github.com/Manishearth/rust-gc)), `'static` is preferred over `Copy`. I'm happy with that. I can write a garbage collector in rust, and have it work with non-trivial `drop`'s, without any unsafety. I only have to sacrifice support for types that have both trivial `drop`s and non-`'static` lifetimes.