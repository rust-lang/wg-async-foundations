# Bridge old, new traits

Introduce "bridge impls" like the following:

```rust
impl<F> Async for F where F: Future {

}
```

Newer runtimes will be built atop the `Async` trait, but older code will still work with them, since everything that implements `Future` implements `Async`.

## Combinators

One tricky case has to do with bridging combinators. If you have a combinator like `Join`:

```rust
struct Join<A, B> { ... }

impl<A, B> Future for Join<A, B>
where
    A: Future,
    B: Future,
{ }
```

This combinator cannot then be used with `Async` values. You cannot (today) add a second impl like the following for coherence reasons:

```rust
impl<A, B> Async for Join<A, B>
where
    A: Async,
    B: Async,
{ }
```

The problem is that this second impl creates multiple routes to implement `Async` for `Join<A, B>` where `A` and `B` are futures. These routes are of course equivalent, but the compiler doesn't know that.

### Solution A: Don't solve it

We might simply introduce new combinators for the `Async` trait. Particularly given the move to [scoped threads](./scoped.md) it is likely that the set of combinators would want to change anyhow.

### Solution B: Specialization

Specialization can be used to resolve this, and it would be a great feature for Rust overall. However, specialization has a number of challenges to overcome. Some related articles:

- [Maximally minimal specialization](https://smallcultfollowing.com/babysteps/blog/2018/02/09/maximally-minimal-specialization-always-applicable-impls/)
- [Supporting blanket impls in specialization](https://smallcultfollowing.com/babysteps/blog/2016/10/24/supporting-blanket-impls-in-specialization/)
