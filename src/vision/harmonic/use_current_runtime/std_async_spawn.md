# std::async_thread::spawn

Ability to spawn parallel, concurrent, and blocking tasks:

- Parallel (e.g., `std::async_thread::spawn`)
- Blocking -- code that may execute
  - Need the ability for blocking tasks, in turn, to enter into async sections and still be able to access borrowed data

Also need the ability to use [scoped](../compose_control_scheduling/scoped.md) tasks to bound and access borrowed references.
