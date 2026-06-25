---
name: rusty-style
description: Write Rust that reads like it was written by an experienced Rust programmer, matching community consensus and idiom — not Rust-flavored Java, C, Python, or Go. Use this skill WHENEVER writing, reviewing, refactoring, or critiquing Rust code, even when the user only asks to "make it work" or "fix the bug." The default for any non-trivial Rust output should be idiomatic Rust, so trigger this skill for new Rust files, Rust functions, type/API design in Rust, error-handling choices, and any time code carries the "accent" of another language (manual indexing, getter/setter walls, stringly-typed data, defensive cloning, unwrap-everywhere). If you catch yourself about to write a `for i in 0..n` loop, a `clone()` to dodge the borrow checker, or an `unwrap()` in library code, that's the signal to consult this skill.
---

# Writing Rusty Rust

The goal of this skill is taste, not correctness. The compiler already enforces correctness. This skill is about writing Rust that an experienced Rustacean would nod at in code review — code that uses the language's grain instead of fighting it, and that doesn't carry the accent of whatever language the author came from.

Most non-idiomatic Rust is correct and compiles fine. It just *reads wrong*: a `for` loop indexing into a `Vec` where an iterator would do, a hierarchy of traits aping Java inheritance, `String` fields where an enum belongs, a wall of `.clone()` calls papering over a borrow the author didn't want to think about. The reader feels the friction even when the compiler doesn't.

## The core instinct

Before writing Rust, internalize the few instincts that separate idiomatic code from translated code:

**Make illegal states unrepresentable.** This is the single most important idea. Rust's type system is a design tool, not just a safety net. If two fields should never both be `Some`, that's an enum, not two `Option`s and a runtime assert. If a string can only be one of four values, that's an enum, not a `String` you match on. Reach for the type system *first*, before writing validation logic — a great deal of defensive code simply disappears when the types make the bad case impossible.

**Move data, don't manage it.** Ownership means values have one clear owner and flow through the program by being moved and borrowed, not copied defensively. When you reach for `.clone()`, pause: is it because the data genuinely needs to exist in two places, or because you're dodging the borrow checker? The latter is an accent. The fix is usually to borrow (`&T`), to restructure who owns what, or to take `&mut`/return ownership rather than clone-and-discard.

**Iterate, don't index.** `for x in &items` over `for i in 0..items.len()`. Adapter chains (`.iter().filter().map().collect()`) over manual accumulation loops with a `mut` accumulator. This isn't golf — iterators express *intent* (filter, then transform, then gather) where index loops express *mechanism*. Idiomatic Rust leans hard on the iterator vocabulary.

**Let errors propagate with `?`.** Errors are values that travel up the stack via `Result` and `?`, not exceptions to catch and not sentinel values to check by hand. Library code returns `Result`; it does not `unwrap`, `panic`, or print-and-exit. `unwrap()` in a library is a code smell; in application `main` or tests it's fine.

**Borrow in signatures, own in storage.** Take `&str` not `&String`, `&[T]` not `&Vec<T>`, `impl AsRef<Path>` not `&PathBuf` — accept the most general borrowed form so callers don't allocate to call you. Store the owned form. This single habit marks experienced Rust immediately.

**Implement traits instead of inventing methods.** Want conversion? `From`/`TryFrom`, not `to_xyz` constructors. Want a default? `Default`. Want display? `Display`. Want to be used in a `for` loop? `IntoIterator`. The standard traits are a shared vocabulary; reimplementing their concepts under custom names is an accent from languages that lack them.

## How to use this skill

For quick work, the instincts above plus the anti-pattern table below are enough. For anything substantial — designing a public API, structuring error handling across a crate, modeling a domain with types — read the relevant reference file. Each is a focused deep-dive:

- `references/error-handling.md` — `Result`, `?`, `thiserror` vs `anyhow`, when panicking is right, custom error types, avoiding `unwrap` accents.
- `references/ownership-and-borrowing.md` — clones vs borrows, lifetimes without fear, `Cow`, `Rc`/`Arc`/`RefCell` and when they signal a design problem, slices over owned collections.
- `references/types-and-api-design.md` — making illegal states unrepresentable, newtypes, builder pattern, accepting `impl Trait` args, the `From`/`Into`/`Default`/`Display` trait vocabulary, avoiding OOP-isms.
- `references/iterators-and-control-flow.md` — iterator adapters, `if let`/`let else`/`match`, combinators on `Option`/`Result`, avoiding index loops and `mut` accumulators.
- `references/tooling-and-conventions.md` — rustfmt, clippy, RFC 430 naming, module layout, doc comments, the conventions CI will enforce anyway.

When reviewing or refactoring existing code, scan for the anti-patterns below, name what's "off" and why, then show the idiomatic version. Don't just say "this isn't idiomatic" — explain which language's accent it carries and what instinct it violates, because that's what actually teaches.

## Anti-pattern quick reference

Each row is a tell. The left column is what translated code does; the right is the rusty instinct. When you see the left, reach for the right.

| Accent (avoid) | Idiomatic (prefer) |
|---|---|
| `for i in 0..v.len() { v[i] }` | `for x in &v` / iterator adapters |
| `.clone()` to satisfy the borrow checker | borrow `&T`, or restructure ownership |
| `unwrap()`/`expect()` in library code | return `Result`, propagate with `?` |
| `String` field with a fixed value set | `enum` |
| two `Option` fields, only one ever set | one `enum` with two variants |
| `if x.is_some() { x.unwrap() }` | `if let Some(x) = x` |
| `match opt { Some(x) => f(x), None => default }` | `opt.map_or(default, f)` |
| getter/setter pair for every field | public fields, or methods only when invariants exist |
| trait hierarchy mimicking class inheritance | composition, generics, enums, small focused traits |
| `&String`, `&Vec<T>`, `&PathBuf` in args | `&str`, `&[T]`, `impl AsRef<Path>` |
| `to_foo()` conversion constructors | `impl From`/`TryFrom` |
| `new()` returning `Self` with 8 args | builder pattern, or `Default` + struct update |
| manual `loop`/`break` accumulating a `Vec` | `.filter().map().collect()` |
| `Rc<RefCell<T>>` reached for reflexively | rethink ownership; it's rarely needed |
| `mod.rs` everywhere, `get_`/`set_` prefixes | RFC 430 naming, `foo.rs` module files |
| index-based access guarded by length checks | `.get(i)` returning `Option`, or iterators |
| comments explaining *what* obvious code does | let names and types speak; comment *why* |

## A note on the goal

Idiomatic Rust is not about cleverness or terseness. It's about expressing intent through the type system and the standard vocabulary so the next reader — and the compiler — can see what you meant. When in doubt, ask: "would this make an experienced Rust programmer reach for the type system, or reach for the keyboard to rewrite it?" Aim for the former.
