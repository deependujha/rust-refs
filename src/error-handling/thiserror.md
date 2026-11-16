# thiserror

Derive macro that writes `Display`, `Error::source` and `From` for your own
error types. For libraries and any code where callers match on the failure.

```toml
[dependencies]
thiserror = "2"
```

## Typical enum

```rust
use std::io;
use std::num::ParseIntError;
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    // #[from]: generates From<io::Error> and returns it from source()
    #[error("io error")]
    Io(#[from] io::Error),

    // a field named `source` is the cause automatically
    #[error("failed to read {}", path.display())]
    Read { path: PathBuf, source: io::Error },

    // #[source]: mark a cause that has a different field name
    #[error("invalid port {raw:?}")]
    BadPort {
        raw: String,
        #[source]
        cause: ParseIntError,
    },

    // tuple fields by position
    #[error("missing key `{0}`")]
    Missing(&'static str),

    // extra format arguments
    #[error("port {value} out of range (max {max})", max = u16::MAX)]
    OutOfRange { value: u64 },
}
```

## Using it

```rust
use std::path::Path;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum LoadError {
    #[error("failed to read {path}")]
    Read { path: String, source: std::io::Error },
    #[error("invalid number")]
    Parse(#[from] std::num::ParseIntError),
    #[error("value must be non-zero")]
    Zero,
}

pub fn load(path: &Path) -> Result<u64, LoadError> {
    // extra context needed: build the variant with map_err
    let text = std::fs::read_to_string(path).map_err(|source| LoadError::Read {
        path: path.display().to_string(),
        source,
    })?;

    // #[from] variant: plain `?`
    let n: u64 = text.trim().parse()?;

    if n == 0 {
        return Err(LoadError::Zero);
    }
    Ok(n)
}

pub fn caller(path: &Path) {
    match load(path) {
        Ok(n) => println!("{n}"),
        Err(LoadError::Zero) => println!("zero, using default"),
        Err(LoadError::Read { path, .. }) => eprintln!("cannot read {path}"),
        Err(e) => eprintln!("{e}"),
    }
}
```

## Attribute reference

| Attribute | Where | Effect |
| --- | --- | --- |
| `#[error("...")]` | variant or struct | `Display` message |
| `#[error(transparent)]` | variant or struct | forward `Display` and `source` to the single inner field |
| `#[from]` | field | `impl From<FieldType>`, and the field becomes the source |
| `#[source]` | field | field is returned from `source()`; no `From` impl |
| field named `source` | field | same as `#[source]` |

## Message format

| Syntax | Meaning |
| --- | --- |
| `{0}`, `{1}` | tuple field, `Display` |
| `{name}` | named field, `Display` |
| `{name:?}` / `{0:?}` | `Debug` |
| `{}` with `, path.display()` | expression on a named field |
| `{}` with `, .0.len()` | expression on a tuple field |
| `{max}` with `, max = u16::MAX` | named extra argument |

## Struct errors

```rust
use thiserror::Error;

#[derive(Debug, Error)]
#[error("line {line}, column {column}: {message}")]
pub struct ParseError {
    pub line: usize,
    pub column: usize,
    pub message: String,
}

#[derive(Debug, Error)]
#[error("request timed out")]
pub struct Timeout;
```

## `transparent`: wrap without adding a message

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("not found: {0}")]
    NotFound(String),

    // catch-all: Display and source come straight from the inner error
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}
```

## Opaque public error, private details

Keeps the variants out of the public API so they can change freely.

```rust
use thiserror::Error;

#[derive(Debug, Error)]
#[error(transparent)]
pub struct StoreError(#[from] ErrorKind);

#[derive(Debug, Error)]
enum ErrorKind {
    #[error("connection lost")]
    Connection(#[from] std::io::Error),
    #[error("key not found: {0}")]
    NotFound(String),
}

impl StoreError {
    pub fn is_not_found(&self) -> bool {
        matches!(self.0, ErrorKind::NotFound(_))
    }
}

pub fn get(key: &str) -> Result<String, StoreError> {
    Err(ErrorKind::NotFound(key.to_string()).into())
}
```

## Generic helper bound

```rust
use std::error::Error;

fn log_error<E: Error + Send + Sync + 'static>(err: E) {
    eprintln!("{err}");
}
```

Any `thiserror` type satisfies this as long as its fields are `Send + Sync`.

## Gotchas

- One `#[from]` per source type per enum. Two variants wrapping `io::Error`
  cannot both be `#[from]`; use `map_err` for one of them.
- A `#[from]` variant can hold only that one field (plus an optional
  backtrace). Need a path or other context alongside it? Use a `source` field
  and `map_err`.
- `?` applies a single `From` conversion. `io::Error -> ErrorKind -> StoreError`
  needs an explicit step: `.map_err(ErrorKind::from)?`.
- Do not write `#[error("io error: {0}")]` on a `#[from]` field. The cause is
  already in `source()`, and chain-printing reporters such as anyhow's `{:#}`
  will print it twice.
- `#[derive(Debug)]` is still required.
- thiserror does not appear in your public API. Replacing the derive with
  hand-written impls later is not a breaking change.
- Adding a variant to a public enum is a breaking change unless the enum is
  marked `#[non_exhaustive]`.
