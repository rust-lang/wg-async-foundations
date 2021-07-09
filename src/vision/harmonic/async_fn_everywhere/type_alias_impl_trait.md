# Type Alias Impl Trait (TAIT)

- **Status:** Development, close to stabilization
- Permits `type Foo = impl Trait` at module and impl level
- Within the defining scope of `Foo` (module or impl, respectively), the TAIT `Foo` must be used only in certain positions:
  - fn return types
  - value of an associated type
- [Stabilization report](https://hackmd.io/o9KO-D8aSb6bJgQ1froVTA)
- [Project board](https://github.com/rust-lang/wg-traits/projects/4)
