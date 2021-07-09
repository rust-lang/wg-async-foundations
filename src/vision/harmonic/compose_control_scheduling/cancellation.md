# Voluntary cancelation

In today's Rust, any async function can be synchronously cancelled at any await point: the code simply stops executing, and destructors are run for any extant variables. This leads to a lot of bugs. (TODO: link to stories)

Under systems like [Swift's proposed structured concurrency model](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md), or with APIs like [.NET's CancellationToken](https://docs.microsoft.com/en-us/dotnet/api/system.threading.cancellationtoken?view=net-5.0), cancellation is "voluntary". What this means is that when a task is cancelled, a flag is set; the task can query this flag but is not otherwise affected. Under structured concurrency systems, this flag is propagated to all chidren (and transitively to their children).

[preemption]: https://tokio.rs/blog/2020-04-preemption

Voluntary cancellation is a requirement for [scoped access](./scoped.md). If there are parallel tasks executing within a scope, and the scope itself is canceled, those parallel tasks must be joined and halted before the memory for the scope can be freed.

One downside of such a system is that cancellation _may not_ take effect. We can make it more likely to work by integrating the cancellation flag into the standard library methods, similar to how tokio encourages ["voluntary preemption"][preemption]. This means that file reads and things will start to report errors (`Err(TaskCanceled)`) once the task has been canceled. This has the advantage that it exercises existing error paths and permits recovery.

## Cancellation and `select`

The `select` macro chooses from N futures and returns the first one that matches. Today, the others are immediately canceled. This behavior doesn't play especially well with voluntary cancellation. There are a few options here:

- We could make `select` signal cancellation for each of the things it is selecting over and then wait for them to finish.
- We could also make `select` continue to take `Future` (not `Async`) values, which effectively makes `Future` a "cancel-safe" trait (or perhaps we introduce a `CancelSafe` marker trait that extends `Async`).
  - This would mean that typical `async fn` could not be given to select, though we might allow people to mark `async fn` as "cancel-safe", in which case they would implement `Future`. They would also not have access to ordinary async fn, though.
    - Effectively, the current `Future` trait becomes the "cancel-safe" form of `Async`. This is a bit odd, since it has other distinctions, like using `Pin`, so it might be preferable to use a 'marker trait'.
  - Of course, users could spawn a task that calls the function and give the handle to select.

## Frequently asked questions

### How does cancellation work in other settings?

Other languages have a combination of a flag to observe when cancellation has been requested with an immediate callback that executes.

### What is the relationship between AsyncDrop and cancellation?

In async Rust today, one signals cancellation of a future by (synchronously) dropping it. This _forces_ the future to stop executing, and drops the values that are on the stack. Experience has shown that this is someting users have a lot of trouble doing correctly, particularly at fine granularities (see e.g. [Alan builds a cache](/vision/status_quo/alan_builds_a_cache.md) or [Barbara gets burned by select](/vision/status_quo/barbara_gets_burned_by_select.md)).

Given `AsyncDrop`, we could adopt a similar convention, where canceling an `Async` is done by (asynchronously) dropping it. This would presumably amend the unsafe contract of the `Async` trait so that the value must be polled to completion _or_ async-dropped. To avoid the footguns we see today, a typical future could simply continue execution from its `AsyncDrop` method (but disregard the result). It might however set an internal flag to true or otherwise allow the user to find out that it has been canceled. It's not clear, though, precisely what value is being added by `AsyncDrop` in this scenario versus the `Async` simply not implementing `AsyncDrop` -- perhaps though it serves as an elegant way to give both an immediate "cancellation" callback and an opportunity to continue.

An alternative is to use a cancellation token of some kind, so that _scopes_ can be canceled and that cancelation can be observed. The main reason to have that token or observation mechanism be "built-in" to some degree is so that it can be observed and used to drive "voluntary cancellation" from I/O routines and the like. Under that model, `AsyncDrop` would be intended more for values (like database handles) that have cleanup to be done, much like `Drop` today, and less as a way to signal cancellation.
