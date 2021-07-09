# Use current runtime without generics

**Certainty level:** 🌧️

When writing sync code, it is possible to simply _access_ I/O and other facilities without needing to thread generics around:

```rust
fn load_socket_addr() -> Result<SocketAddr, Box<dyn Error>> {
    Ok(std::fs::read_to_string("address.txt")?.parse()?)
}
```

This code will work no matter what operating system you run it on.

Similarly, if you don't mind hard-coding your runtime, one can use `tokio` or `async_std` in a similar fashion

```rust
// Pick one:
//
// use tokio as my_runtime;
// use async_std as my_runtime;

async fn load_socket_addr() -> Result<SocketAddr, Box<dyn Error>> {
    Ok(my_runtime::fs::read_to_string("address.txt").await?.parse()?)
}
```

Given [portable generic code], it would be possible to write generic code that feels similar:

```rust
async fn load_socket_addr<F: AsyncFs>() -> Result<SocketAddr, Box<dyn Error>> {
    Ok(F::read_to_string("address.txt").await?.parse()?)
}
```

Alternatively, that might be done with `dyn` trait:

```rust
async fn load_socket_addr(fs: &dyn AsyncFs)) -> Result<SocketAddr, Box<dyn Error>> {
    Ok(F::read_to_string("address.txt").await?.parse()?)
}
```

Either approach is significantly more annoying, both as the author of the library and for folks who invoke your library.

## Preferred experience

The ideal would be that you can write an async function that is "as easy" to use as a non-async one, and have it be portable across runtimes:

```rust
async fn load_socket_addr() -> Result<SocketAddr, Box<dyn Error>> {
    Ok(std::async_fs::read_to_string("address.txt").await?.parse()?)
}
```

### But how to achieve it?

The basic idea is to extract out a "core API" of things that a runtime must provide and to make those functions available as part of the `Context` that `Async` values are invoked with. To avoid the need for generics and monomorphization, this would have to be based purely on `dyn` values. This interface ought to be compatible with no-std runtimes as well, which imposes some challenges. Ideally, we would have traits as described in [portable generic code]

## Frequently asked questions

### What about cap-std?

It's interesting to observe that the `dyn` approach is feeling very close to [cap-std](https://blog.sunfishcode.online/introducing-cap-std/). That might be worth taking into consideration. Some targets, like wasm, may well prefer if we took a more "capability oriented" approach.

[portable generic code]: ../portable_generic_code.md

### What about spawning and scopes?

Given that spawning should occur through scopes, it may be that we don't need a `std::async_thread::spawn` API so much as standards for scopes.
