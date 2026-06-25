# Iterators and Control Flow

The accent here is the imperative index-and-accumulate loop carried over from C, Java, or pre-comprehension Python. Idiomatic Rust expresses transformations as iterator adapter chains and uses the `Option`/`Result` combinators instead of manual unwrapping.

## Iterate over elements, not indices

```rust
// Accent (C/Java-style index loop):
let mut sum = 0;
for i in 0..nums.len() {
    sum += nums[i];
}

// Idiomatic:
let sum: i32 = nums.iter().sum();
```

Index loops cost you bounds checks, risk off-by-one and out-of-range bugs, and obscure intent. You almost never need the index. When you genuinely do, use `.enumerate()`:

```rust
for (i, item) in items.iter().enumerate() { ... }
```

## Adapter chains over manual accumulation

The pattern "make an empty `Vec`, loop, conditionally push" is almost always a `filter`/`map`/`collect`:

```rust
// Accent:
let mut names = Vec::new();
for user in &users {
    if user.active {
        names.push(user.name.to_uppercase());
    }
}

// Idiomatic:
let names: Vec<String> = users
    .iter()
    .filter(|u| u.active)
    .map(|u| u.name.to_uppercase())
    .collect();
```

The chain reads as a sentence: filter to active, transform to uppercase names, gather. The accumulator loop makes the reader reconstruct that intent from mechanism.

Core vocabulary worth knowing cold: `map`, `filter`, `filter_map`, `flat_map`, `fold`, `sum`, `product`, `count`, `any`, `all`, `find`, `position`, `take`, `skip`, `take_while`, `skip_while`, `chain`, `zip`, `rev`, `enumerate`, `collect`, `partition`, `min`/`max`, `min_by_key`/`max_by_key`, `sorted` (via `sort` on the collected vec). When a loop is doing one of these, name it with the adapter.

## `collect` into more than `Vec`

`collect` is polymorphic via `FromIterator`. It builds `HashMap`, `HashSet`, `String`, and — crucially — `Result<Vec<T>, E>` from an iterator of `Result`s, short-circuiting on the first error:

```rust
// Parse every line, fail on the first bad one:
let nums: Result<Vec<i32>, _> = lines.iter().map(|l| l.parse::<i32>()).collect();
```

This `collect::<Result<_,_>>()` trick replaces a manual loop with early-return error handling and is deeply idiomatic.

## `Option`/`Result` combinators over manual matching

`match` on an `Option`/`Result` just to transform it is verbose. The combinators say it in one line:

| Manual `match`/`if` | Combinator |
|---|---|
| `match o { Some(x) => f(x), None => None }` | `o.map(f)` |
| `match o { Some(x) => f(x), None => default }` | `o.map_or(default, f)` |
| `match o { Some(x) => x, None => default }` | `o.unwrap_or(default)` |
| `match o { Some(x) => x, None => expensive() }` | `o.unwrap_or_else(expensive)` |
| `match o { Some(x) => f(x), None => None }` (f returns Option) | `o.and_then(f)` |
| `if o.is_some() { o.unwrap() }` | `if let Some(x) = o` |
| `o.ok_or(E)?` to turn None into an error | `o.ok_or(E)?` |

Use `unwrap_or_else` (closure, lazy) when the default is expensive to compute; `unwrap_or` (eager) when it's a cheap literal.

## `if let`, `let else`, `match`

- **`if let`** — handle one variant, ignore the rest: `if let Some(x) = opt { ... }`.
- **`let else`** — bind in the happy path, diverge (return/break/panic) otherwise. Great for stripping leading guards without rightward drift:

```rust
let Some(user) = lookup(id) else {
    return Err(Error::NotFound);
};
// `user` is in scope for the rest of the function, no nesting
```

- **`match`** — multiple variants, or exhaustiveness matters. Prefer `match` over chained `if/else if` on an enum; the compiler then forces you to handle new variants, which is a feature.

Avoid the `is_some()` + `unwrap()` pair and the `is_ok()` + `unwrap()` pair entirely — they're the dynamic-language "check then access" habit, and `if let`/`let else` do it without the redundant unwrap.

## Don't `collect` just to loop again

Collecting into a `Vec` only to immediately `for`-loop over it allocates for nothing. Iterators are lazy — chain the work and consume once. Collect when you need the materialized collection (to return it, to index it, to iterate twice), not as a reflex between steps.

## Ranges, slicing, and `windows`/`chunks`

For "look at pairs/triples/neighbors," reach for `.windows(n)` and `.chunks(n)` instead of index arithmetic:

```rust
// Accent: for i in 1..v.len() { compare v[i-1], v[i] }
// Idiomatic:
for pair in v.windows(2) {
    let [a, b] = pair else { continue };
    compare(a, b);
}
```
