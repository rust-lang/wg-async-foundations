# Async trait

Today's `Future` trait lacks one fundamental capability compared to synchronous code: there is no way to "block" your caller and be sure that the caller will not continue executing until you agree. In synchronous code, you can use a closure and a destructor to achieve this, which is the technique used for things like `rayon::scope` and crossbeam's scoped threads. In async code, because the `Future` trait has a safe poll function, it is always possible to poll it part way and then `mem::forget` (or otherwise leak) the value; this means that if one cannot have parallel threads executing and using those references.

Async functions are commonly written with borrowed references as arguments:

```rust
async fn do_something(db: &Db) { ... }
```

but important utilities like `spawn` and `spawn_blocking` require `'static` tasks. Without "unfogettable" traits, the only way to circumvent this is with mechanisms like `FuturesUnordered`, which is then subject to footguns as described in [Barbara battles buffered streams](https://rust-lang.github.io/wg-async-foundations/vision/status_quo/barbara_battles_buffered_streams.html).

Introduce a trait like

```rust
trait Async {
    type Output;

    /// # Unsafe conditions
    ///
    /// * Once polled, cannot be moved
    /// * Once polled, destructor must execute before memory is deallocated
    /// * Once polled, must be polled to completion
    unsafe fn poll(
        &mut self,
        context: &mut core::task::Context<'_>,
    ) -> core::task::Ready<Self::Output>;
}
```

## Related work

See also https://github.com/Matthias247/rfcs/pull/1
