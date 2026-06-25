# Ownership and Borrowing

This is where the heaviest accents live, because ownership has no analog in GC'd or manually-managed languages. The two failure modes are opposite: people from C/C++ over-reach for raw pointers and `unsafe`; people from Java/Python/JS clone everything to avoid thinking about the borrow checker. Both are fighting the language.

## The clone-to-appease tell

The single most common accent: a `.clone()` whose only purpose is to make the borrow checker stop complaining. This is correct and slow and signals the author didn't internalize ownership.

```rust
// Accent: clone because "I still need `name` later"
fn greet(name: String) -> String {
    let g = format!("Hello, {}", name.clone());
    log(name);  // the clone existed only to avoid moving here
    g
}

// Idiomatic: borrow instead of move
fn greet(name: &str) -> String {
    let g = format!("Hello, {}", name);
    log(name);
    g
}
```

When you write `.clone()`, ask: *does this data genuinely need to exist in two places at once?* If yes, clone is correct and honest. If it's just to avoid a move/borrow conflict, the fix is one of: borrow (`&T`), restructure who owns the value, split the function so borrows don't overlap, or take `&mut self` and mutate in place.

Cheap clones (`Copy` types, `Rc`/`Arc` bumping a refcount) are fine and idiomatic. The smell is cloning large owned data (`String`, `Vec`, `HashMap`) reflexively.

## Borrow in function signatures

Accept the most general borrowed form. This lets callers pass borrowed data without allocating, and is the clearest single marker of experienced Rust:

| Don't take | Take |
|---|---|
| `&String` | `&str` |
| `&Vec<T>` | `&[T]` |
| `&PathBuf` | `&Path` or `impl AsRef<Path>` |
| `String` (when you only read it) | `&str` |
| `Box<T>` (when you only read it) | `&T` |

```rust
// Accent: forces caller to own a Vec and a String
fn total(items: &Vec<Item>, label: &String) -> u64 { ... }

// Idiomatic: works with arrays, slices, sub-slices, string literals
fn total(items: &[Item], label: &str) -> u64 { ... }
```

Rule of thumb: **borrow in arguments, own in storage.** A struct field that holds data long-term owns it (`String`, `Vec<T>`); a function that merely inspects data borrows it.

## Take what you need: `&T`, `&mut T`, or `T`

- `&T` — you only read.
- `&mut T` — you mutate in place.
- `T` (by value) — you consume/transform it, or store it.

Returning ownership is often cleaner than `&mut` plumbing. Don't write Java-style `void mutate(obj)` chains when a function can take a value and return the transformed value.

## Lifetimes are descriptive, not scary

A lifetime annotation doesn't *create* a constraint; it *describes* a relationship that already exists ("the output borrows from this input"). Most functions need none thanks to elision. When you do need one, it's usually because a struct holds a reference:

```rust
struct Parser<'a> {
    input: &'a str,   // Parser cannot outlive the string it borrows
    pos: usize,
}
```

If lifetimes start fighting you across a whole design, that's usually a signal to own the data instead of borrowing it — a `String` field instead of `&'a str`. Borrowing in structs is an optimization; reach for it when you've measured a need, not by default.

## `Rc`/`Arc`/`RefCell` usually signal a design to reconsider

Reaching for `Rc<RefCell<T>>` reflexively is often a Java/Python accent — recreating "everything is a shared mutable reference." Rust's grain is single ownership with borrows. Before reaching for shared ownership, ask whether a clearer ownership structure (a tree the parent owns, indices into a `Vec` instead of pointers, passing `&mut` down) would work.

Legitimate uses exist: `Arc` for sharing immutable data across threads, `Rc<RefCell<T>>` for genuine graphs/observer patterns where single ownership truly doesn't fit. But it should be a considered choice, not the first reach. Graph-shaped data is often better as an arena/index design (`Vec<Node>` + `usize` indices) than a web of `Rc`s.

## `Cow` for "usually borrowed, sometimes owned"

When a function returns borrowed data unchanged in the common case but occasionally must allocate a modified version, `Cow<str>` (clone-on-write) expresses exactly that, avoiding an unconditional allocation:

```rust
fn normalize(input: &str) -> Cow<str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input)
    }
}
```

## `unsafe` is not a performance button

C/C++ programmers sometimes scatter `unsafe` to "go fast" or to use familiar pointer patterns. Idiomatic Rust treats `unsafe` as a last resort with a documented safety justification, confined to the smallest possible block, ideally wrapped in a safe API. If you're writing `unsafe` to dodge the borrow checker rather than to do something genuinely impossible in safe Rust (FFI, certain data structures), stop and find the safe design.
