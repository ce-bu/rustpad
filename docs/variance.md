# Variance in Rust

**Estimated reading time: ~20 minutes**

---

## What Is Variance?

Variance determines whether a generic type `Wrapper<T>` can be **substituted** when `T` is substituted by a related type (a subtype or supertype).

In Rust, the main source of subtyping is **lifetime relationships**: if `'long: 'short` ("`'long` outlives `'short`"), then `&'long T` is a subtype of `&'short T`. You can use a longer-lived reference wherever a shorter-lived one is expected.

There are three kinds of variance:

| Variance | Meaning for `Wrapper<T>` |
|---|---|
| **Covariant** | `T` is subtype of `U` → `Wrapper<T>` is subtype of `Wrapper<U>` (direction preserved) |
| **Contravariant** | `T` is subtype of `U` → `Wrapper<U>` is subtype of `Wrapper<T>` (direction flipped) |
| **Invariant** | No subtyping relationship between `Wrapper<T>` and `Wrapper<U>`, ever |

For lifetimes: "subtype" means "longer-lived". So `'long` is a subtype of `'short`.

---

## Quick Reference: Variance of Built-in Types

| Type | Variance in `T` / `'a` |
|---|---|
| `&'a T` | Covariant in both `'a` and `T` |
| `&'a mut T` | Covariant in `'a`, **invariant** in `T` |
| `*const T` | Covariant in `T` |
| `*mut T` | **Invariant** in `T` |
| `Box<T>` | Covariant in `T` |
| `Vec<T>` | Covariant in `T` |
| `Option<T>` | Covariant in `T` |
| `Cell<T>` | **Invariant** in `T` |
| `UnsafeCell<T>` | **Invariant** in `T` |
| `fn(T) -> U` | **Contravariant** in `T`, covariant in `U` |
| `fn() -> T` | Covariant in `T` |
| `dyn Trait + 'a` | **Invariant** in `Trait`, covariant in `'a` |

For structs: variance is **inferred** from field types. If all uses of `T` in fields are covariant → the struct is covariant. If any use is invariant → the struct is invariant.

---

## 1. Covariance: The Common Case

### 1.1 References and lifetime shrinking

```rust
fn takes_short<'s>(s: &'s str) {}

fn example() {
    let long_lived = String::from("hello");
    let r: &'static str = "world";

    // &'static str is a subtype of &'s str for any 's
    // because 'static outlives everything.
    takes_short(r); // fine: 'static shrinks to 's
}
```

### 1.2 Vec is covariant — you can coerce element lifetimes

```rust
fn take_vec<'a>(v: Vec<&'a str>) {}

fn example() {
    let v: Vec<&'static str> = vec!["hello", "world"];
    take_vec(v); // Vec<&'static str> coerces to Vec<&'a str>
}
```

This is safe because `Vec` only lets you **read** elements (or produce owned values by moving them out). It never lets you write a shorter-lived reference back in through just a `Vec<&'a str>` — there's no `set` that takes a `&'a str`. So widening is impossible.

### 1.3 Box is covariant

```rust
fn takes_box<'a>(b: Box<&'a str>) {}

fn example() {
    let b: Box<&'static str> = Box::new("hello");
    takes_box(b); // Box<&'static> → Box<&'a>: fine
}
```

### 1.4 Struct inherits covariance from fields

```rust
struct Wrapper<'a> {
    data: &'a str, // covariant in 'a
}

fn takes_wrapper<'a>(w: Wrapper<'a>) {}

fn example() {
    let s = String::from("hi");
    let w: Wrapper<'_> = Wrapper { data: &s };
    takes_wrapper(w); // fine
}
```

Rust infers `Wrapper<'a>` is covariant in `'a` because `&'a str` is covariant and there's nothing else.

---

## 2. Invariance: When You Can Read and Write

### 2.1 `&mut T` is invariant in `T`

```rust
fn assign<'a>(r: &mut &'a str, s: &'a str) {
    *r = s;
}

fn example() {
    let mut r: &'static str = "hello";
    {
        let local = String::from("local");
        // assign(&mut r, local.as_str()); // ERROR
        // This would write a &'local str into r: &'static str 
        // if we were to convert &mut &'static str to &mut &'local str.
        // After 'local ends, r would dangle.
    }
}
```

If `&mut &'a str` were covariant in `'a`, you could pass a `&mut &'static str` where `&mut &'short str` is expected, write a short-lived reference through it, and then read a dangling reference from the original `&'static str`. Invariance blocks this.

### 2.2 Cell is invariant — the canonical example

`Cell<T>` is invariant in `T` because it allows both read (`.get()`) and write (`.set()`). This is the same reasoning as `&mut T`.

```rust
use std::cell::Cell;

fn smuggle<'a>(cell: &Cell<&'a str>, s: &'a str) {
    cell.set(s);
}

fn example() {
    let cell: Cell<&'static str> = Cell::new("hello");
    {
        let local = String::from("local");
        // smuggle(&cell, local.as_str()); // ERROR: local does not live long enough
        //
        // If Cell were covariant:
        //   Cell<&'static str> <: Cell<&'local str>
        //   smuggle would accept &cell and set it to &local
        //   after the block, cell holds a dangling reference
    }
    println!("{}", cell.get()); // would be dangling
}
```

Because `Cell<T>` is **invariant**, the compiler refuses to treat `&Cell<&'static str>` as `&Cell<&'local str>`. The types are unrelated — no substitution is allowed.

### 2.3 `*mut T` is invariant

```rust
unsafe fn write_ptr<T>(p: *mut T, val: T) {
    *p = val;
}

// *mut T is invariant in T for the same reason as &mut T:
// you can both read and write through a raw mutable pointer.
```

### 2.4 Struct becomes invariant if any field is invariant

```rust
use std::cell::Cell;

struct MixedVariance<'a> {
    readable: &'a str,      // covariant in 'a
    mutable:  Cell<&'a str>, // invariant in 'a
}

// The struct is INVARIANT in 'a because Cell<&'a str> is invariant.
// The covariant field doesn't help — invariance wins.
```

---

## 3. Contravariance: Function Arguments

Contravariance arises only in **function argument position**. A function that accepts a *less specific* input is usable wherever a function accepting a *more specific* input is required.

### 3.1 The intuition

If `'Lion: 'Mamal`, then:
- `fn(&'Lion str)` can only be called with long-lived references.
- `fn(&'Mamal str)` can be called with short-lived **or** long-lived references.

So `fn(&'Mamal str)` is *more capable* (accepts more inputs) and is a **subtype** of `fn(&'Lion str)`. If I can handle a `'Mamal` reference, I can handle a `'Lion` reference because a `'Lion` is also a `'Mamal` . The direction flips: longer lifetime → smaller function subtype.

### 3.2 Concrete example

```rust
fn accept_short<'s>(_s: &'s str) {}   // can accept anything
fn accept_static(_s: &'static str) {} // can only accept 'static

fn needs_fn_with_short<'s>(f: fn(&'s str)) {
    let local = String::from("local");
    f(&local); // calls f with a short-lived reference
}

fn example() {
    // fn(&'short str) is a subtype of fn(&'long str)
    // so fn(&'short str) is acceptable where fn(&'static str) is expected? NO.
    // The other way: fn(&'static str) cannot be used where fn(&'short str) is needed.

    needs_fn_with_short(accept_short);  // fine: accept_short handles any lifetime
    // needs_fn_with_short(accept_static); // ERROR: accept_static can't handle &local
}
```

### 3.3 Callbacks and contravariance in practice

```rust
// A registry that calls its callback with a short-lived &str
struct Registry {
    callback: Box<dyn Fn(&str)>,
}

impl Registry {
    fn trigger(&self, s: &str) {
        (self.callback)(s);
    }
}

fn example() {
    // Works: the closure accepts any &str
    let r = Registry {
        callback: Box::new(|s: &str| println!("{s}")),
    };
    r.trigger("hello");
}
```

We know that `Fn(&'a str) : Fn(&'static str)` . Since `Box<T> is covariant in T`, we have `Box<dyn Fn(&'a str)> : Box<dyn Fn(&'static str)>` . So the closure that accepts any `&str` is a valid subtype (more capable/lives longer than) of the callback type that requires a `&'static str`.

### 3.4 `fn(T) -> U` — contravariant in T, covariant in U

```rust
type Handler<'a> = fn(&'a str) -> &'a str;

// Handler<'short> is a subtype of Handler<'long>
// (argument is contravariant, return is covariant — net: invariant for this
// particular type alias because 'a appears in both positions)
```

When `'a` appears in both argument and return position, the variance contributions cancel and the result is **invariant**. This is why many higher-order function types end up invariant.

---

## 4. PhantomData: Manually Controlling Variance

When you use raw pointers or have a field that doesn't mention `T` directly, Rust can't infer variance. You must use `PhantomData<X>` to tell the compiler what variance you intend. The variance of `PhantomData<X>` mirrors the variance of `X`.

### 4.1 PhantomData cheat sheet

| `PhantomData<…>` | Variance in `T` | Typical meaning |
|---|---|---|
| `PhantomData<T>` | Covariant | "I own a `T` (or could produce one)" |
| `PhantomData<*mut T>` | Invariant | "I can read and write `T`" |
| `PhantomData<fn(T)>` | Contravariant | "I consume `T` as input" |
| `PhantomData<fn(T) -> T>` | Invariant | "Both — net invariant" |
| `PhantomData<*const T>` | Covariant | "I can only read; no ownership claim" |
| `PhantomData<&'a T>` | Covariant in `'a` and `T` | "I borrow `T` for `'a`" |
| `PhantomData<&'a mut T>` | Covariant in `'a`, invariant in `T` | "I mutably borrow `T`" |

### 4.2 Covariant PhantomData — owning a T

Standard pattern for a raw-pointer container that *owns* `T`:

```rust
use std::marker::PhantomData;

struct MyBox<T> {
    ptr: *mut T,
    _marker: PhantomData<T>, // covariant: "I own a T"
}

// MyBox<&'static str> is a subtype of MyBox<&'a str> — safe because
// we only ever read out the T we own; nobody can write a shorter lifetime in.

impl<T> MyBox<T> {
    fn new(val: T) -> Self {
        let ptr = Box::into_raw(Box::new(val));
        MyBox { ptr, _marker: PhantomData }
    }

    fn get(&self) -> &T {
        unsafe { &*self.ptr }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe { drop(Box::from_raw(self.ptr)); }
    }
}
```

### 4.3 Invariant PhantomData — reading and writing a T

Use when callers can both read and write through your type, like a pool or arena:

```rust
use std::marker::PhantomData;

struct Pool<T> {
    ptr: *mut T,
    len: usize,
    _marker: PhantomData<*mut T>, // invariant: "I read AND write T"
}

// Pool<&'static str> is NOT a subtype of Pool<&'a str>.
// Without invariance, a caller could store a short-lived &str
// into the pool and then read it out as a long-lived reference.
```

### 4.4 Covariant-but-no-ownership — PhantomData<*const T>

When your struct uses raw pointers but does **not** own `T` (e.g. it holds a pointer into memory owned elsewhere):

```rust
use std::marker::PhantomData;

struct Slice<T> {
    ptr: *const T,
    len: usize,
    _marker: PhantomData<*const T>, // covariant, no ownership claim
}

// Analogous to &[T]: covariant in T, doesn't claim to drop T.
```

This tells the compiler: "I can read `T` but I don't own it, so drop check doesn't apply".

### 4.5 Contravariant PhantomData — a sink

Useful for types that model a *consumer* or *sink* of `T`:

```rust
use std::marker::PhantomData;

struct Sink<T> {
    write: unsafe fn(*mut (), T),
    data:  *mut (),
    _marker: PhantomData<fn(T)>, // contravariant in T
}

// Sink<&'short str> is a subtype of Sink<&'long str>:
// a sink that can consume short-lived strings can surely also
// consume long-lived ones.
```

### 4.6 Lifetime PhantomData for borrow tracking

```rust
use std::marker::PhantomData;

// Marks that this handle borrows something for lifetime 'a.
struct Handle<'a, T> {
    ptr: *mut T,
    _marker: PhantomData<&'a T>, // covariant in 'a and T
}

// The borrow checker treats Handle<'a, T> as if it contains &'a T,
// so it enforces that the source lives at least as long as 'a.
```

If you need a mutable borrow instead:

```rust
struct HandleMut<'a, T> {
    ptr: *mut T,
    _marker: PhantomData<&'a mut T>, // covariant in 'a, INVARIANT in T
}
```

---

## 5. Drop Check Interaction with Variance

Variance compounds with drop check. See [dropcheck.md](dropcheck.md) for the full treatment; the key interaction is:

- **Covariant + `Drop`**: the compiler must enforce that `'a` strictly outlives the struct, because covariance could otherwise shorten `'a` to before `drop()` runs — making the reference dangle inside `drop()`.
- **Invariant + `Drop`**: invariance already prevents lifetime shrinkage, so the strict-outlive requirement from drop check is less surprising (the lifetime was locked anyway).
- **`PhantomData<T>` (covariant) + `Drop`**: behaves like owning a `T`. Drop check applies.
- **`PhantomData<*const T>` (covariant, no ownership) + `Drop`**: drop check does NOT apply because you're not claiming ownership.

```rust
struct CovariantWithDrop<'a> {
    data: &'a str,
}

impl<'a> Drop for CovariantWithDrop<'a> {
    fn drop(&mut self) { println!("{}", self.data); }
}

fn broken() {
    let v: CovariantWithDrop<'_>;
    let s = String::from("hi");
    v = CovariantWithDrop { data: &s };
    // ERROR: `s` doesn't strictly outlive `v` (both end in the same scope,
    // and v is dropped first — it would read s inside drop()).
}
```

---

## 6. Trait Objects and Variance

`dyn Trait + 'a` is **covariant** in `'a` (the lifetime bound), but the trait's associated types and method signatures determine variance there.

```rust
trait Reader {
    fn read(&self) -> &str;
}

fn use_reader<'a>(r: &dyn Reader) {}

fn example<'long: 'short, 'short>(r: &'long dyn Reader) {
    use_reader(r); // &'long dyn Reader coerces to &'short dyn Reader: fine
}
```

`dyn Trait` is invariant in `Trait` itself — you cannot substitute one trait for another through subtyping (there is no trait subtyping at the type level in Rust, unlike Java).

---

## 7. Practical Pitfall Catalogue

### 7.1 Storing a `&mut` inside a struct — accidentally invariant

```rust
struct Holder<'a, T> {
    r: &'a mut T, // INVARIANT in T because &mut T is invariant in T
}

// You CANNOT coerce Holder<'_, &'static str> to Holder<'_, &'short str>.
// This matters if you wanted a Holder to accept different string lifetimes.
```

**Fix:** If you only need to read `T`, use `&'a T` instead.

### 7.2 Wrapping a `Cell` kills covariance

```rust
use std::cell::Cell;

struct Cache<'a> {
    value: &'a str,        // covariant
    flag:  Cell<bool>,     // invariant in bool, but bool has no lifetime → OK
    alias: Cell<&'a str>,  // INVARIANT in 'a — this kills covariance for the whole struct
}

// Cache<'long> is NOT a subtype of Cache<'short> because of alias.
```

### 7.3 Returning a shorter lifetime from a covariant position — fine

```rust
fn longest<'a>(x: &'a str, _y: &'a str) -> &'a str { x }

fn example() {
    let result;
    let s1 = String::from("long");
    {
        let s2 = String::from("short");
        result = longest(s1.as_str(), s2.as_str());
        // result's lifetime is tied to the shorter of s1 and s2 — fine inside this block
    }
    // result cannot be used here — 's2' doesn't live long enough
}
```

### 7.4 Invariance prevents a useful coercion — manual workaround

```rust
use std::cell::Cell;

fn requires_cell_of_short<'s>(c: &Cell<&'s str>) {}

fn example() {
    let cell: Cell<&'static str> = Cell::new("hello");
    // requires_cell_of_short(&cell); // ERROR: &'static str ≠ &'s str due to invariance

    // Workaround: reconstruct with the desired lifetime explicitly.
    let cell2: Cell<&str> = Cell::new("hello"); // 'a inferred as short
    requires_cell_of_short(&cell2);
}
```

### 7.5 PhantomData missing — drop check fires unexpectedly

```rust
use std::marker::PhantomData;

struct IterMut<'a, T: 'a> {
    ptr: *mut T,
    end: *mut T,
    // Without PhantomData, the compiler has no variance info for 'a or T.
    // Adding PhantomData<&'a mut T> makes it invariant in T and covariant in 'a,
    // matching the semantics of a mutable slice iterator.
    _marker: PhantomData<&'a mut T>,
}
```

Without `_marker`, `rustc` would not know that `IterMut` borrows `T` for `'a`, so it would accept code that produces dangling iterators.

### 7.6 Contravariance in closures and callbacks

```rust
// A scheduler that stores a callback to execute with a short-lived context.
struct Scheduler<'ctx> {
    job: Box<dyn Fn(&'ctx str)>,
}

impl<'ctx> Scheduler<'ctx> {
    fn run(&self, ctx: &'ctx str) {
        (self.job)(ctx);
    }
}

fn example() {
    let sched = Scheduler {
        // This closure accepts any &str — more permissive than required.
        // Contravariance means it's a valid subtype of Fn(&'ctx str).
        job: Box::new(|s: &str| println!("{s}")),
    };
    let s = String::from("hello");
    sched.run(&s);
}
```

---

## 8. How the Compiler Infers Variance

For a struct, Rust walks every field:

1. Each generic parameter starts as **bivariant** (no constraint).
2. For each use of the parameter in a field type, the compiler applies the variance of that position.
3. Variance contributions are **merged**: covariant + covariant = covariant; covariant + contravariant = invariant; anything + invariant = invariant.

```
struct Foo<T> {
    a: Vec<T>,          // covariant in T
    b: fn(T) -> (),     // contravariant in T
}
// covariant + contravariant = INVARIANT in T
```

```
struct Bar<T> {
    a: *const T,        // covariant in T
    b: *const T,        // covariant in T
}
// covariant + covariant = COVARIANT in T
```

```
struct Baz<T> {
    a: *const T,        // covariant
    b: *mut T,          // invariant
}
// covariant + invariant = INVARIANT in T
```

---

## 9. Summary

- **Covariance** is the default for "read-only" containers. You can shrink lifetimes safely because nobody writes back.
- **Invariance** arises when you can **write** (mutation, interior mutability, raw mutable pointers). Substituting a shorter lifetime would allow dangling reads.
- **Contravariance** only occurs in **function argument** position. More permissive inputs equal a more capable (sub)type.
- **PhantomData** is the tool for expressing variance in raw-pointer structs. Match it to your actual semantics:
  - Own `T`? → `PhantomData<T>`
  - Borrow `T` for `'a`? → `PhantomData<&'a T>`
  - Mutably borrow? → `PhantomData<&'a mut T>`
  - Read/write `T` without ownership? → `PhantomData<*mut T>`
  - Consume `T` as input? → `PhantomData<fn(T)>`
- **Drop check** amplifies variance requirements: covariant + `Drop` forces strict-outlive.
- When variance is wrong, errors manifest as **"does not live long enough"** or **"lifetime mismatch"** that feel unrelated to the actual code.
