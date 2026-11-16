# Error handling

## Which one

| Situation | Use | Function signature |
| --- | --- | --- |
| Library; callers match on failures | [thiserror](thiserror.md) | `Result<T, MyError>` |
| Application / CLI; errors are reported, not matched | [anyhow](anyhow.md) | `anyhow::Result<T>` |
| No dependencies, or a tiny script | [std](std.md) | `Result<T, Box<dyn Error>>` |
| Library internals plus a binary on top | thiserror in the lib, anyhow in `main` | both |

## The same function three ways

Standard library only:

```rust
use std::error::Error;

fn read_port(path: &str) -> Result<u16, Box<dyn Error + Send + Sync>> {
    let text = std::fs::read_to_string(path)?;
    let port = text.trim().parse::<u16>()?;
    Ok(port)
}
```

thiserror, a typed error the caller can match on:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum PortError {
    #[error("failed to read port file")]
    Io(#[from] std::io::Error),
    #[error("port is not a valid u16")]
    Parse(#[from] std::num::ParseIntError),
}

fn read_port(path: &str) -> Result<u16, PortError> {
    let text = std::fs::read_to_string(path)?;
    let port = text.trim().parse::<u16>()?;
    Ok(port)
}
```

anyhow, an opaque error with context for the person reading the log:

```rust
use anyhow::{Context, Result};

fn read_port(path: &str) -> Result<u16> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read {path}"))?;
    let port = text
        .trim()
        .parse::<u16>()
        .with_context(|| format!("invalid port in {path}"))?;
    Ok(port)
}
```

## Cargo.toml

```toml
[dependencies]
thiserror = "2"
anyhow = "1"
```

## Rules of thumb

- `Result` for failures the caller can reasonably handle; `panic!` for bugs.
- `unwrap()` in tests and examples; `expect("why this cannot fail")` when an
  invariant guarantees success; `?` everywhere else.
- Error messages: lowercase, no trailing period, no "error:" prefix. The
  reporter adds those.
- Either put the cause in the message or return it from `source()`, not both.
  Otherwise chain-printing reporters show it twice.
- Do not expose `anyhow::Error` from a library's public API.
