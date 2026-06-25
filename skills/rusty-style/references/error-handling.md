# Error Handling

The accent to kill here is exceptions-and-sentinels thinking. In Rust, errors are ordinary values of type `Result<T, E>` that flow up the call stack. There is no `try`/`catch`, no checked-exception ceremony, and no convention of returning `-1` or `null` on failure. Get comfortable with `Result` and `?` and most of the awkwardness disappears.

## The `?` operator is the spine of error handling

`?` unwraps a `Result`/`Option` on the happy path and early-returns the error on the sad path, after running `From::from` to convert the error type. This is what replaces both exception propagation and manual `if err != nil { return err }` chains.

```rust
// Accent (Go-flavored manual propagation):
fn read_config(path: &Path) -> Result<Config, ConfigError> {
    let contents = match fs::read_to_string(path) {
        Ok(c) => c,
        Err(e) => return Err(ConfigError::Io(e)),
    };
    let config = match toml::from_str(&contents) {
        Ok(c) => c,
        Err(e) => return Err(ConfigError::Parse(e)),
    };
    Ok(config)
}

// Idiomatic:
fn read_config(path: &Path) -> Result<Config, ConfigError> {
    let contents = fs::read_to_string(path)?;
    let config = toml::from_str(&contents)?;
    Ok(config)
}
```

The conversions via `?` work because `ConfigError` implements `From<io::Error>` and `From<toml::Error>` — see the custom error section below.

## `unwrap()` / `expect()` are a code smell in libraries

A library that panics is a library that takes the decision to crash away from its caller. Return `Result` and let the caller decide. `unwrap`/`expect` are acceptable in:

- `main()` and small binaries / scripts / examples
- tests (panicking *is* the failure signal)
- cases where the invariant is genuinely guaranteed by construction, and then prefer `expect("reason this can't fail")` with a message explaining *why* it's safe — this documents the invariant.

```rust
// Accent (panics on any bad input — unacceptable in a lib):
pub fn parse_port(s: &str) -> u16 {
    s.parse().unwrap()
}

// Idiomatic:
pub fn parse_port(s: &str) -> Result<u16, ParseIntError> {
    s.parse()
}
```

## `anyhow` for applications, `thiserror` for libraries

This split is community consensus:

- **Libraries** define their own error types so callers can match on failure modes programmatically. `thiserror` removes the boilerplate of implementing `Error`/`Display`/`From`.
- **Applications / binaries** usually don't need callers to distinguish error kinds — they need a good message and a backtrace. `anyhow::Result<T>` with `.context("...")` is the ergonomic choice.

```rust
// Library error with thiserror:
#[derive(Debug, thiserror::Error)]
pub enum ConfigError {
    #[error("failed to read config file")]
    Io(#[from] std::io::Error),
    #[error("invalid config syntax")]
    Parse(#[from] toml::de::Error),
    #[error("unknown profile: {0}")]
    UnknownProfile(String),
}
```

```rust
// Application code with anyhow + context:
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let config = read_config(&path)
        .with_context(|| format!("loading config from {}", path.display()))?;
    apply(config).context("applying configuration")?;
    Ok(())
}
```

`with_context` (a closure) defers formatting to the error path so the happy path pays nothing; use it when building the message allocates. Use `.context("static str")` for cheap messages.

## `Box<dyn Error>` is a middle ground, not a default

For quick programs you can return `Result<T, Box<dyn std::error::Error>>`. It's better than `unwrap` everywhere but loses the ergonomics of `anyhow` (no easy context, no backtrace). Reach for `anyhow` in real applications.

## Don't reach for panics as control flow

`panic!`, `unwrap`, and friends are for *bugs* — states that should be impossible. They are not for *expected* failures like missing files or bad user input. A useful test: if a fuzzer feeding garbage to your function could trigger the panic, it should have been a `Result`. Reserve panics for "the programmer screwed up" invariant violations.

## `Option` is not an error type, but it composes like one

For "absence" rather than "failure," return `Option<T>`. Convert between the two deliberately: `option.ok_or(MyError::Missing)?` turns absence into an error; `result.ok()` discards the error to get an `Option`. Don't model expected-absence as an error enum variant when `Option` says it more honestly.

## Avoid the stringly-typed error accent

Returning `Result<T, String>` throws away structure — callers can't match, can't chain `From`, can't get a backtrace. It's the dynamic-language habit of "an error is just a message." Use a real error type (even a one-variant `thiserror` enum) or `anyhow::Error`, never `String`, except in throwaway code.
