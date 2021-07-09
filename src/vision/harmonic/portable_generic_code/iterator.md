# Iterator

The async iterator trait can leverage [inline async functions](../async_fn_everywhere/inline_async_fn.md):

```rust
#[repr(inline_async)]
trait AsyncIterator {
    type Item;

    async fn next(&mut self) -> Self::Item;
}
```

Note the name change from `Stream` to `AsyncIterator`.

One implication of this change is that pinning is no longer necessary when driving an async iterator. For example, one could now write an async iterator that recursively walks through a set of URLs like so (presuming `std::async_iter::from_fn` and [async closures](../async_fn_everywhere/async_closures)):

```rust
fn explore(start_url: Url) -> impl AsyncIterator {
    let mut urls = vec![start_url];
    std::async_iter::from_fn(async move || {
        if let Some(url) = urls.pop() {
            let mut successor_urls = fetch_successor_urls(url).await;
            urls.extend(successor_urls);
            Some(url)
        } else {
            None
        }
    })
}
```
