# 🌈 Harmonic synthesis

This page is a **work in progress**. It represents a complete vision for where we want async to go. This vision starts with the [experience] that we want async to provide and then dives into details of the technical pieces that need to get done to create that vision. Note that while a lot of the steps needed are fairly clear, several of them also have [significant unknowns or points of controversy](./unresolved_questions.md). We have attempted to highlight those and expect to be working through those points as we go.

## Certainty levels

- 🌈 -- Implemented and stable
- 🌞 -- Everything is looking good
- 🌤️ -- Still some stuff to figure out, but unlikely to see major changes in the design
- 🌥️ -- Got one or two solid leads, but still have to figure out if it will work
- 🌧️ -- No clear path yet

## Groups

| Group                                           | Certainty |
| ----------------------------------------------- | --------- |
| [Use async fn everywhere]                       | 🌤️        |
| [Easily compose, control scheduling]            | 🌥️        |
| [Generic code that is portable across runtimes] | 🌥️        |
| [Use current runtime without generics]          | 🌧️        |
| [Easy to understand what is happening]          | 🌤️        |
| [Easily produce iterators]                      | 🌤️        |
| [Getting started in async Rust is easy]         | 🌤️        |

[use async fn everywhere]: ./async_fn_everywhere.md
[easily compose, control scheduling]: ./compose_control_scheduling.md
[generic code that is portable across runtimes]: ./portable_generic_code.md
[use current runtime without generics]: ./use_current_runtime.md
[easy to understand what is happening]: ./easy_to_understand.md
[easily produce iterators]: ./easily_produce_iterators.md
[getting started in async rust is easy]: ./getting_started.md
