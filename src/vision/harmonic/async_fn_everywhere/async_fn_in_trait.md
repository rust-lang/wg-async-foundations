# Async fn in traits

You should be able to write `async fn` in traits and impls:

```rust
trait Foo {
    async fn bar(&self);
}

impl Foo for MyType {
    async fn bar(&self) {
        ...
    }
}
```

This desugars into a GAT + impl Trait:

```rust
trait Foo {
    type Bar<'me>: Future + 'me;
    fn bar(&self) -> Self::Bar<'_>;
}

impl Foo for MyType {
    type Bar<'me> = impl Future + 'me;
    fn bar(&self) -> Self::Bar<'_> {
        async move {
            ...
        }
    }
}
```

## Unresolved design questions

### What is the name of that GAT we introduce?

- I called it `Bar` here, but that's somewhat arbitrary, perhaps we want to have some generic syntax for naming the method?
- Or for getting the type of the method.
- This problem applies equally to other "`-> impl Trait` in trait" scenarios.
- [Exploration doc](https://hackmd.io/IISsYc0fTGSSm2MiMqby4A)

### Can users easily bound those GATs with `Send`, maybe even in the trait definition?

- People are likely to want to say "I want every future produced by this trait to be Send", and right now that is quite tedious.
- This applies equally to other "`-> impl Trait` in trait" scenarios.

### What about "dyn" traits?

- See the sections on "inline" and dyn" async fn in traits below!
