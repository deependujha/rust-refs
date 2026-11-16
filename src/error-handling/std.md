# Standard library

No dependencies: `Result`, `?`, `Box<dyn Error>`, and hand-written error types.

## `?` operator

```rust
use std::fs;
use std::io;

fn read_trimmed(path: &str) -> Result<String, io::Error> {
    let text = fs::read_to_string(path)?; // on Err: return early
    Ok(text.trim().to_string())
}
```

`expr?` expands to roughly:

```rust,ignore
match expr {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)), // note the From conversion
}
```

The `From` call is why `?` can turn an `io::Error` into your own error type.

`?` also works on `Option` inside a function returning `Option`:

```rust
fn second_char_upper(s: &str) -> Option<char> {
    let c = s.chars().nth(1)?;
    Some(c.to_ascii_uppercase())
}
```

## Quick and loose: `Box<dyn Error>`

```rust
use std::error::Error;

type BoxError = Box<dyn Error + Send + Sync + 'static>;

fn run(path: &str) -> Result<u64, BoxError> {
    let text = std::fs::read_to_string(path)?; // any error type converts
    let n: u64 = text.trim().parse()?;
    if n == 0 {
        return Err("value must be non-zero".into()); // &str and String convert too
    }
    Ok(n)
}
```

Add `Send + Sync` so the error can cross threads and `.await` points.

## Custom error type by hand

```rust
use std::fmt;
use std::io;
use std::num::ParseIntError;

#[derive(Debug)]
pub enum ConfigError {
    Io(io::Error),
    BadPort { raw: String, source: ParseIntError },
    Missing(&'static str),
}

// 1. Display: the human-readable message
impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Io(_) => write!(f, "failed to read config"),
            ConfigError::BadPort { raw, .. } => write!(f, "invalid port {raw:?}"),
            ConfigError::Missing(key) => write!(f, "missing key `{key}`"),
        }
    }
}

// 2. Error: expose the underlying cause
impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::Io(e) => Some(e),
            ConfigError::BadPort { source, .. } => Some(source),
            ConfigError::Missing(_) => None,
        }
    }
}

// 3. From: lets `?` convert automatically
impl From<io::Error> for ConfigError {
    fn from(e: io::Error) -> Self {
        ConfigError::Io(e)
    }
}

pub fn load_port(path: &str) -> Result<u16, ConfigError> {
    let text = std::fs::read_to_string(path)?; // io::Error -> ConfigError
    let raw = text
        .lines()
        .find_map(|l| l.strip_prefix("port="))
        .ok_or(ConfigError::Missing("port"))?;
    raw.trim().parse().map_err(|source| ConfigError::BadPort {
        raw: raw.to_string(),
        source,
    })
}
```

[thiserror](thiserror.md) generates steps 1 to 3 from attributes.

## Matching on errors

```rust
use std::fs::File;
use std::io::ErrorKind;

fn open_or_create(path: &str) -> std::io::Result<File> {
    match File::open(path) {
        Ok(f) => Ok(f),
        Err(e) if e.kind() == ErrorKind::NotFound => File::create(path),
        Err(e) => Err(e),
    }
}
```

## `Option` to `Result` and back

```rust
use std::collections::HashMap;
use std::num::ParseIntError;

fn port(cfg: &HashMap<String, String>) -> Result<u16, String> {
    let raw = cfg.get("port").ok_or("missing key: port")?; // Option -> Result
    raw.parse::<u16>()
        .map_err(|e| format!("bad port {raw:?}: {e}"))
}

fn port_lenient(raw: &str) -> Option<u16> {
    raw.parse().ok() // Result -> Option, error discarded
}

// Option<Result<T, E>> -> Result<Option<T>, E>
fn maybe_parse(s: Option<&str>) -> Result<Option<i32>, ParseIntError> {
    s.map(str::parse::<i32>).transpose()
}
```

Use `ok_or_else(|| ...)` when building the error allocates or is expensive.

## Combinators

```rust
fn demo(input: &str) {
    let r = input.parse::<i32>(); // Result<i32, ParseIntError>

    let _ = r.clone().map(|n| n * 2);                    // change the Ok value
    let _ = r.clone().map_err(|e| e.to_string());        // change the Err value
    let _ = r.clone().and_then(|n| "2".parse::<i32>().map(|m| n * m)); // chain fallible step
    let _ = r.clone().or_else(|_| "0".parse::<i32>());   // fallback that may fail

    let _ = r.clone().unwrap_or(0);                      // default value
    let _ = r.clone().unwrap_or_default();               // Default::default()
    let _ = r.clone().unwrap_or_else(|e| {               // compute default from the error
        eprintln!("using 0: {e}");
        0
    });

    let _ = r.clone().ok();                              // Option<i32>
    let _ = r.clone().is_ok_and(|n| n > 0);              // bool
    let _ = r.clone().inspect_err(|e| eprintln!("parse failed: {e}")); // log, pass through
}
```

## `let ... else`

```rust
fn first_word(line: &str) -> Result<&str, String> {
    let Some(word) = line.split_whitespace().next() else {
        return Err("empty line".to_string());
    };
    Ok(word)
}
```

## Iterators of `Result`

```rust
use std::num::ParseIntError;

// stop at the first error
fn parse_all(items: &[&str]) -> Result<Vec<i32>, ParseIntError> {
    items.iter().map(|s| s.parse::<i32>()).collect()
}

// keep the successes, drop the failures
fn parse_valid(items: &[&str]) -> Vec<i32> {
    items.iter().filter_map(|s| s.parse().ok()).collect()
}

// keep both
fn parse_split(items: &[&str]) -> (Vec<i32>, Vec<ParseIntError>) {
    let (ok, err): (Vec<_>, Vec<_>) = items
        .iter()
        .map(|s| s.parse::<i32>())
        .partition(|r| r.is_ok());
    (
        ok.into_iter().map(Result::unwrap).collect(),
        err.into_iter().map(Result::unwrap_err).collect(),
    )
}
```

## Printing the cause chain

`Display` shows only the top-level message. Walk `source()` for the rest.

```rust
use std::error::Error;

fn report(err: &dyn Error) {
    eprintln!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        eprintln!("  caused by: {cause}");
        source = cause.source();
    }
}
```

## Downcasting a `dyn Error`

```rust
use std::error::Error;
use std::io;

fn is_not_found(err: &(dyn Error + 'static)) -> bool {
    err.downcast_ref::<io::Error>()
        .is_some_and(|e| e.kind() == io::ErrorKind::NotFound)
}

fn check(err: Box<dyn Error + Send + Sync>) -> bool {
    is_not_found(err.as_ref())
}
```

## `main` and exit codes

```rust
use std::error::Error;
use std::process::ExitCode;

fn run() -> Result<(), Box<dyn Error>> {
    let n: i32 = "abc".parse()?;
    println!("{n}");
    Ok(())
}

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("error: {e}");
            ExitCode::from(2)
        }
    }
}
```

The shorter form, `fn main() -> Result<(), Box<dyn Error>>`, prints the error
with `Debug` (not `Display`) and exits with code 1.

## `unwrap`, `expect`, `panic!`

| Call | Use when |
| --- | --- |
| `?` | default; let the caller decide |
| `expect("reason")` | failure means a bug; the message states the invariant |
| `unwrap()` | tests, examples, prototypes |
| `unreachable!()` | a branch the type system cannot rule out but logic does |
| `panic!()` | unrecoverable state; continuing would corrupt data |

```rust
use std::net::IpAddr;

fn localhost() -> IpAddr {
    "127.0.0.1".parse().expect("hard-coded address is valid")
}
```

## Gotchas

- `?` only converts through `From`. For anything else use `map_err`.
- `Box<dyn Error>` does not itself implement `Error`, so it cannot be passed
  where a generic `E: Error` is required.
- `impl From<A> for MyError` allows only one variant per source type. Two
  variants that both wrap `io::Error` need `map_err` at the call site.
- `Result<T, String>` works but loses the cause chain and cannot be matched on.
- `#[derive(Debug)]` is required: `Error` has `Debug + Display` as supertraits.
