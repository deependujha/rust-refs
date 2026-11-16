# anyhow

One error type, `anyhow::Error`, that holds any error plus a stack of context
messages. For applications, CLIs, tests, and scripts, where errors get reported
rather than matched on.

```toml
[dependencies]
anyhow = "1"
```

## Typical use

```rust
use anyhow::{Context, Result};
use std::path::Path;

fn load_port(path: &Path) -> Result<u16> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("failed to load config from {}", path.display()))?;
    let port = text.trim().parse::<u16>().context("invalid port")?;
    Ok(port)
}

fn main() -> Result<()> {
    let port = load_port(Path::new("app.toml"))?;
    println!("{port}");
    Ok(())
}
```

Output when the file is missing (exit code 1):

```text
Error: failed to load config from app.toml

Caused by:
    No such file or directory (os error 2)
```

`anyhow::Result<T>` is `Result<T, anyhow::Error>`. Any
`std::error::Error + Send + Sync + 'static` converts into it with `?`.

## Adding context

```rust
use anyhow::{Context, Result};
use std::collections::HashMap;

fn get_port(cfg: &HashMap<String, String>) -> Result<u16> {
    // on Option: None becomes an error with this message
    let raw = cfg.get("port").context("missing key: port")?;

    // static message: context
    // formatted message: with_context, so format! only runs on failure
    raw.parse().with_context(|| format!("invalid port {raw:?}"))
}
```

## Creating errors: `bail!`, `ensure!`, `anyhow!`

```rust
use anyhow::{anyhow, bail, ensure, Result};

fn parse_mode(raw: &str) -> Result<u8> {
    ensure!(!raw.is_empty(), "mode is empty"); // if !cond { return Err(..) }

    if raw.len() > 1 {
        bail!("mode too long: {raw:?}"); // return Err(anyhow!(..))
    }

    raw.chars()
        .next()
        .and_then(|c| c.to_digit(8))
        .map(|d| d as u8)
        .ok_or_else(|| anyhow!("mode is not an octal digit: {raw:?}")) // an error value
}
```

## Printing

```rust
use anyhow::{Context, Result};

fn load() -> Result<String> {
    std::fs::read_to_string("app.toml").context("failed to load config from app.toml")
}

fn main() {
    if let Err(e) = load() {
        println!("{e}");   // top message only
        println!("{e:#}"); // whole chain on one line
        println!("{e:?}"); // multi-line report, what `main` prints
    }
}
```

| Format | Output |
| --- | --- |
| `{}` | `failed to load config from app.toml` |
| `{:#}` | `failed to load config from app.toml: No such file or directory (os error 2)` |
| `{:?}` | message, then a `Caused by:` list, then a backtrace if enabled |
| `{:#?}` | struct-style `Debug` dump |

`{:#}` is the one for single-line logs.

## Custom report and exit code

```rust
use anyhow::{Context, Result};
use std::process::ExitCode;

fn run() -> Result<()> {
    let path = std::env::args().nth(1).context("usage: app <config>")?;
    let text = std::fs::read_to_string(&path)
        .with_context(|| format!("failed to read {path}"))?;
    println!("{} bytes", text.len());
    Ok(())
}

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("error: {err}");
            for cause in err.chain().skip(1) {
                eprintln!("  caused by: {cause}");
            }
            ExitCode::from(2)
        }
    }
}
```

## Getting the original error back

```rust
use anyhow::{Context, Result};
use std::io;

fn read(path: &str) -> Result<String> {
    std::fs::read_to_string(path).with_context(|| format!("failed to read {path}"))
}

fn read_or_empty(path: &str) -> Result<String> {
    match read(path) {
        Ok(text) => Ok(text),
        Err(err) => match err.downcast_ref::<io::Error>() {
            Some(e) if e.kind() == io::ErrorKind::NotFound => Ok(String::new()),
            _ => Err(err),
        },
    }
}

// search the whole chain, including sources nested inside typed errors
fn is_not_found(err: &anyhow::Error) -> bool {
    err.chain()
        .filter_map(|cause| cause.downcast_ref::<io::Error>())
        .any(|e| e.kind() == io::ErrorKind::NotFound)
}
```

| Method | Returns |
| --- | --- |
| `err.downcast_ref::<E>()` | `Option<&E>` |
| `err.downcast::<E>()` | `Result<E, anyhow::Error>`, takes ownership |
| `err.is::<E>()` | `bool` |
| `err.chain()` | iterator over the error and each cause |
| `err.root_cause()` | the last error in the chain |

## With thiserror

Typed errors in the library, anyhow at the top.

```rust
use anyhow::{Context, Result};
use thiserror::Error;

// library side
#[derive(Debug, Error)]
pub enum StoreError {
    #[error("key not found: {0}")]
    NotFound(String),
    #[error("storage unavailable")]
    Unavailable(#[from] std::io::Error),
}

pub fn get(key: &str) -> Result<String, StoreError> {
    Err(StoreError::NotFound(key.to_string()))
}

// application side
fn greeting() -> Result<String> {
    let name = match get("user.name") {
        Ok(name) => name,
        Err(StoreError::NotFound(_)) => "stranger".to_string(), // handle what you can
        Err(e) => return Err(e).context("failed to look up user name"), // report the rest
    };
    Ok(format!("hello, {name}"))
}
```

## Backtraces

```sh
RUST_BACKTRACE=1 cargo run        # captured where the error was created
RUST_LIB_BACKTRACE=0 cargo run    # panics only, not errors
```

Shown at the end of the `{:?}` output.

## Gotchas

- `anyhow::Error` does not implement `std::error::Error`. It cannot be passed
  where `E: Error` is required. Convert with
  `Box::<dyn Error + Send + Sync>::from(err)` or `err.into_boxed_dyn_error()`.
- Errors must be `Send + Sync + 'static`. An error holding an `Rc`, a
  `MutexGuard`, or a borrowed `&str` will not convert with `?`. Turn it into a
  message first: `.map_err(|e| anyhow!("{e}"))`.
- `{}` prints only the outermost context. Logging with `{}` hides the actual
  cause; use `{:#}` or `{:?}`.
- `context("...")` builds its argument eagerly. Use `with_context(|| format!(...))`
  for anything that allocates.
- Keep it out of library public APIs: callers cannot match on it, and it ties
  them to anyhow.
- `Option::context` needs `use anyhow::Context` in scope, same as for `Result`.
