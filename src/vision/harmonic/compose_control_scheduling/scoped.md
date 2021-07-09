# Scope-based API

Async functions are commonly written with borrowed references as arguments:

```rust
async fn do_something(db: &Db) { ... }
```

but important utilities like `spawn` and `spawn_blocking` require `'static` tasks. Building on non-cancelable traits, we can implement a "scope" API that allows one to introduce an async scope. This scope API should permit one to spawn tasks into a scope, but have various kinds of scopes (e.g., synchronous execution, parallel execution, and so forth). It should ultimately reside in the standard library and hook into different runtimes for scheduling. This will take some experimentation!

```rust
let result = std::async_thread::scope(|s| {
    let job1 = s.spawn(async || {
        11
    });
    let job2 = s.spawn_blocking(|| {
        11
    });

    job1.await + job2.await
}).await;
assert_eq!(result, 22);
```

## Side-stepping the nested await problem

One goal of scopes is to avoid the "nested await" problem, as described in [Barbara battles buffered streams (BBBS)][bbbs]. The idea is like this: the standard combinators which run work "in the background" and which give access to intermediate results should take a scope parameter and spawn that work using scopes[^hard]. Some examples from today's APIs are `FuturesUnordered` and `Stream::buffered`.

[^hard]: This is not a hard rule. But invoking poll manually is best regarded as a risky thing to be managed with care -- not only because of the formal safety guarantees, but because of the possibility for "nested await"-style failures.

[bbbs]: https://rust-lang.github.io/wg-async-foundations/vision/status_quo/barbara_battles_buffered_streams.html
[`buffered`]: https://docs.rs/futures/0.3.15/futures/prelude/stream/trait.StreamExt.html#method.buffered

In the case of [BBBS], the problem arises because of `buffered`, which spawns off concurrent work to process multiple connections. This means that the [`buffered`] combinator would take a `scope` parameter to use for spawning:

```rust
async fn do_work(database: &Database) {
    std::async_thread::scope(|s| {
        let work = do_select(database, FIND_WORK_QUERY)?;
        std::async_iter::from_iter(work)
            .map(|item| do_select(database, work_from_item(item)))
            .buffered(5, scope)
            .for_each(|work_item| process_work_item(database, work_item))
            .await;
    }).await;
}
```

### Concurrency without scopes: Join, select, race, and friends

It is possible to introduce concurrency in ways that both (a) do not require scopes and (b) avoid the "nested await" problem. Any combinator which takes multiple `Async` instances and polls them to completion (or cancels them) before it itself returns is ok. This includes:

- `join`, because the `join(a, b)` doesn't complete until both `a` and `b` have completed;
- `select`, because selecting will cancel the alternatives that are not chosen;
- `race`, which is a variant of select.

This is important because embedded systems often avoid allocators, and the scope API implicitly requires allocation (one can spawn an unbounded number of tasks).

### Could there be a convenient way to access the current scope?

If we wanted to integrate the idea of scopes more deeply, we could have some way to get access to the current scope and reference its lifetime. Let's imagine we added a keyword `scope`, and we said that its lifetime is `'scope`. One would then be able to do something like the following:

Lots of unknowns to work out here, though. For example, suppose you have a function that creates a scope and invokes a closure within. Do we have a way to indicate to the closure that `'scope` in that closure may be different?

It starts to feel like simply passing "scope" values may be simpler, and perhaps we need a way to automate the threading of state instead. Another advantage of passing a `scope` explicitly is that it is clear when parallel tasks may be launched.
