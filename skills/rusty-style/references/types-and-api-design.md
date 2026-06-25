# Types and API Design

This is where OOP accents are loudest: class hierarchies, getter/setter walls, stringly-typed data, anemic "data classes." Rust's type system wants to be used as a *design tool*. The mantra:

## Make illegal states unrepresentable

The defining instinct of idiomatic Rust. Encode invariants in types so the bad case can't be constructed, and entire categories of validation and defensive checks vanish.

```rust
// Accent: stringly-typed, validated at runtime, can drift
struct Connection {
    status: String,           // "connected"? "Connected"? "CONNECTED"? typo?
    error_message: String,    // empty string means... no error?
}

// Idiomatic: the type makes the bad states impossible
enum Connection {
    Connected { since: Instant },
    Disconnected { reason: DisconnectReason },
}
```

Two more common applications:

**Mutually exclusive options → enum.** If you have two `Option` fields and only one is ever `Some`, that's an enum:

```rust
// Accent:
struct Payment {
    card: Option<Card>,
    bank_transfer: Option<BankTransfer>,  // both Some? both None? illegal
}

// Idiomatic:
enum Payment {
    Card(Card),
    BankTransfer(BankTransfer),
}
```

**Validated values → newtype.** A function taking `(email: String, age: u32)` can be called with the arguments swapped or with an unvalidated email. Newtypes fix both:

```rust
struct Email(String);   // construct only via validating `TryFrom<String>`
struct Age(u8);

impl TryFrom<String> for Email {
    type Error = InvalidEmail;
    fn try_from(s: String) -> Result<Self, Self::Error> { /* validate once */ }
}
```

Once an `Email` exists, every function downstream can trust it — no re-validation, no defensive checks. The type *is* the proof.

## No getter/setter walls

The Java reflex of a private field plus `get_x()`/`set_x()` for everything is not idiomatic. If a field has no invariant to protect, make it `pub`. Add methods only when there's an actual invariant (a setter that must keep two fields in sync) or a computed value. And follow RFC 430 naming: the getter is `fn x(&self)`, **not** `fn get_x(&self)`.

```rust
// Accent:
pub struct Point { x: f64, y: f64 }
impl Point {
    pub fn get_x(&self) -> f64 { self.x }
    pub fn set_x(&mut self, x: f64) { self.x = x; }
    // ...repeat for y
}

// Idiomatic — no invariant, so just expose the fields:
pub struct Point { pub x: f64, pub y: f64 }
```

## Composition and small traits over inheritance

Rust has no inheritance, and trying to simulate it with trait hierarchies and `dyn` everywhere is an accent. The idiomatic toolkit:

- **Enums** for closed sets of variants (you control all the cases).
- **Generics + trait bounds** for static polymorphism (`fn f<T: Read>(r: T)`).
- **Small, focused traits** describing one capability (`Read`, `Write`, `Display`) — composed, not stacked into deep hierarchies.
- **`dyn Trait`** only when you genuinely need runtime heterogeneity (a `Vec<Box<dyn Plugin>>`).
- **Composition** (struct holds the thing) over "is-a" modeling.

If you're building an abstract base class with template-method overrides, step back: that's usually an enum (closed) or a trait with default methods (open), not a hierarchy.

## The standard trait vocabulary — use it instead of inventing names

| You want | Implement | Not |
|---|---|---|
| conversion `A → B` | `From<A> for B` (gives you `Into` free) | `fn to_b()` constructor |
| fallible conversion | `TryFrom` | `fn try_to_b()` |
| a default value | `Default` | `fn empty()` / `fn zero()` |
| human-readable text | `Display` | `fn to_string_pretty()` |
| debug text | `#[derive(Debug)]` | hand-rolled dump |
| equality / ordering | derive `PartialEq`/`Ord` | `fn equals()` |
| use in `for` loop | `IntoIterator` | `fn each()` |
| indexing `a[i]` | `Index` | `fn at()` |
| deref to inner | `Deref` (sparingly) | `fn inner()` |

Implementing `From` rather than a `to_xyz` method means your type slots into `?` conversions, `.into()` calls, and generic code that bounds on `Into`. Custom-named conversions don't compose.

## Constructors: `new`, `Default`, and builders

- A simple `fn new(...) -> Self` is fine for a few obvious args.
- Many fields, many optional → **builder pattern** (`Foo::builder().a(1).b(2).build()`), not a `new` with eight positional args.
- All-defaults-then-tweak-a-few → derive `Default` and use struct update syntax: `Foo { timeout: 30, ..Default::default() }`.

Avoid the constructor-overloading accent (multiple `new_with_x`, `new_from_y`). Name them honestly (`from_bytes`, `with_capacity`) or use a builder.

## Return `impl Trait`, accept generics

Accept the most general thing, return the most specific:

```rust
// Accept any iterable of items, return a concrete-ish iterator
fn evens(nums: impl IntoIterator<Item = u32>) -> impl Iterator<Item = u32> {
    nums.into_iter().filter(|n| n % 2 == 0)
}
```

`impl Trait` in argument position keeps callers from allocating; in return position it hides implementation types without boxing.

## Derive generously

`#[derive(Debug, Clone, PartialEq)]` and friends are nearly free and expected. A public type with no `Debug` impl is a papercut for everyone using it. Derive `Debug` on essentially everything; add `Clone`, `PartialEq`, `Eq`, `Hash`, `Default`, `Serialize`/`Deserialize` as the type's role warrants.
