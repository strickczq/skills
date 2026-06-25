# Tooling and Conventions

These are the hard constraints CI will enforce anyway, plus the naming and layout conventions that mark code as community-standard. Following them up front means clippy and rustfmt have nothing to complain about.

## rustfmt — formatting is not a matter of opinion

The community does not bikeshed formatting; `rustfmt` decides, everyone runs it. Write code as if `cargo fmt` has already run: 4-space indent, no manual column alignment (rustfmt will undo it), trailing commas in multiline literals, braces on the same line. Don't invent a personal style — there's exactly one, and it's whatever rustfmt produces.

## clippy — treat its lints as idiom lessons

`cargo clippy` is the closest thing to an automated idiom checker. Write to pass `clippy` clean. Common lints that catch accents directly:

- `needless_range_loop` — you wrote `for i in 0..n`; use an iterator.
- `redundant_clone` — that `.clone()` was unnecessary.
- `manual_map` / `option_map_unit_fn` — use the `Option` combinator.
- `single_match` — use `if let` instead of a one-arm `match`.
- `needless_return` — drop the trailing `return`; use the tail expression.
- `ptr_arg` — you took `&String`/`&Vec<T>`; take `&str`/`&[T]`.
- `len_zero` — use `.is_empty()` not `.len() == 0`.
- `clone_on_copy` — cloning a `Copy` type.
- `or_fun_call` — use `unwrap_or_else` for the lazy default.

When clippy fires, the fix is almost always *the idiomatic form*, not an `#[allow]`. Reach for `#[allow(...)]` only with a justification comment, never to silence a lint you don't feel like fixing.

## Naming — RFC 430 (the std API guidelines)

This is what makes names "look like Rust." Non-negotiable in published code:

| Item | Convention | Example |
|---|---|---|
| types, traits, enum variants | `UpperCamelCase` | `HashMap`, `IoError`, `ParseError` |
| functions, methods, variables, modules | `snake_case` | `read_to_string`, `byte_count` |
| constants, statics | `SCREAMING_SNAKE_CASE` | `MAX_LEN`, `DEFAULT_PORT` |
| lifetimes | short lowercase | `'a`, `'src`, `'de` |
| generic type params | short `UpperCamelCase` | `T`, `K`, `V`, `E` |

Method naming conventions that matter:

- **No `get_` prefix** on simple getters. The getter for `name` is `fn name(&self)`, not `get_name`. (Exception: `HashMap::get`-style lookups returning `Option`.)
- **Conversions follow cost conventions:** `as_x` (cheap borrow, `&self → &X`), `to_x` (expensive, `&self → X`, often clones/allocates), `into_x` (consuming, `self → X`). Naming a cheap borrow `to_` or an allocating call `as_` misleads readers.
- **`is_`/`has_`** prefixes for predicates returning `bool`: `is_empty`, `is_active`.
- **`iter`/`iter_mut`/`into_iter`** for the three iteration flavors.

## Module layout — prefer `foo.rs` over `foo/mod.rs`

Modern Rust (2018 edition onward) prefers `src/foo.rs` for a module and `src/foo/` for its submodules, over the old `src/foo/mod.rs`. A directory full of `mod.rs` files is harder to navigate (every tab says `mod.rs`). Keep modules focused; re-export the public surface from the crate root or a `prelude` so users have a clean import path.

## Doc comments — `///` and document the *why* and the contract

- `///` for public items; the first line is a one-sentence summary (renders in the API list).
- Document panics (`# Panics`), errors (`# Errors`), and safety (`# Safety` for `unsafe fn`) in the conventional sections.
- Include runnable examples in ` ```rust ` blocks — they're tested by `cargo test` as doctests, so they can't rot.
- Comment **why**, not **what**. `// increment i` is noise; `// skip the BOM if present` earns its place. Code that needs a what-comment usually needs a better name instead.

## Edition and `#![warn(...)]`

Use a recent edition in `Cargo.toml`. For libraries, consider crate-level `#![warn(missing_docs)]` and denying common footguns. Many crates run `#![deny(clippy::all)]` in CI. Write as though those are on.

## The one-line summary

Run `cargo fmt && cargo clippy` mentally as you write. If imagined-clippy would fire, you've probably written an accent — fix the code, don't suppress the lint.
