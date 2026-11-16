# Rust Reference Notes

Short, code-first notes for things I keep looking up.

Each page is laid out the same way:

1. The snippet you most likely came for, at the top.
2. Variations, one heading each.
3. Gotchas at the bottom.

Every Rust block carries its own `use` lines, so it can be pasted as is.

## Topics

- [Error handling](error-handling/README.md)
  - [Standard library](error-handling/std.md): `Result`, `?`, hand-written error types
  - [thiserror](error-handling/thiserror.md): derive error types for libraries
  - [anyhow](error-handling/anyhow.md): one error type with context for applications
