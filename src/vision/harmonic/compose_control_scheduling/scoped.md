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

One goal of scopes is to avoid the "nested await" problem, as described in [Barbara battles buffered streams (BBBS)][bbbs]. The idea is like this: the standard combinators which run work "in the background" and which give access to intermediate results should take a scope parameter (see the note below about no-std or embedded environments). Some examples from today's APIs are `FuturesUnordered` and `Stream::buffered`. (Combinators like `join` are distinct, because they don't give access to intermediate results; similarly `race` is distinct because of [cancellation](./cancellation.md)).

-- i.e., initiates concurrency -- needs to take a scope parameter. The way to get concurrency, in other words, is to spawn into a scope, and not to construct small "subschedulers". This includes `FuturesUnordered` and `Stream::buffered`, but also more familiar APIs like `r`. By doing this, we ensure that the scheduler is able to poll those concurrent tasks even while the main task is busy doing something else. A good rule of thumb here is that _only the scheduler ever invokes poll_. **All other code just "awaits" things.**[^hard]

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

The `join` combinator would likely be replaced with a method on `scope`:[^variadic]

[^variadic]: Naturally we would want a variadic variation, or perhaps a method macro.

```rust
let (a_result, b_result) = scope.join(a, b).await;
```

This join method would be defined like so:

```rust
impl Scope<'s> {
    async fn join<A, B>(&mut self, a: A, b: B) -> (A::Output, B::Output)
    where
        A: Async + 's,
        B: Async + 's,
    {
        (self.spawn(a).await, b.await)
    }
}
```

## Could there be a convenient way to access the current scope?

If we wanted to integrate the idea of scopes more deeply, we could have some way to get access to the current scope and reference its lifetime. Let's imagine we added a keyword `scope`, and we said that its lifetime is `'scope`. One would then be able to do something like the following:

Lots of unknowns to work out here, though. For example, suppose you have a function that creates a scope and invokes a closure within. Do we have a way to indicate to the closure that `'scope` in that closure may be different?

It starts to feel like simply passing "scope" values may be simpler, and perhaps we need a way to automate the threading of state instead. Another advantage of passing a `scope` explicitly is that it is clear when parallel tasks may be launched.

## What about embedded?

The scope APIs are not especially suitable for embedded environments. Their structure assumes
