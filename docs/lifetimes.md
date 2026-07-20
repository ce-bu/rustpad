# Advanced Lifetimes in Rust

**Estimated reading time: ~90 minutes**

> **Reading order:** This doc should be read before [advanced_traits.md](advanced_traits.md), which builds on lifetime concepts covered here.

This document is a comprehensive deep-dive into Rust's lifetime system. We go well beyond the basics to cover elision rules, early vs late-bound lifetimes, variance, lifetime interactions with trait objects, HRTBs, async, and the many subtle patterns that emerge.

---

## Table of Contents

1. [Lifetime Refresher](#1-lifetime-refresher)
2. [Lifetime Elision Rules](#2-lifetime-elision-rules)
3. [Early-Bound vs Late-Bound Lifetimes](#3-early-bound-vs-late-bound-lifetimes)
4. [Subtyping and Variance](#4-subtyping-and-variance)
5. [Lifetimes in `dyn Trait`](#5-lifetimes-in-dyn-trait)
6. [Higher-Ranked Lifetimes (`for<'a>`)](#6-higher-ranked-lifetimes-fora)
7. [Lifetime Bounds on Types](#7-lifetime-bounds-on-types)
8. [Common Lifetime Puzzles](#8-common-lifetime-puzzles)
9. [Lifetimes and Closures](#9-lifetimes-and-closures)
10. [Lifetimes in Async](#10-lifetimes-in-async)
11. [Lifetimes in Serde](#11-lifetimes-in-serde)

---

## 1. Lifetime Refresher

Lifetimes are Rust's compile-time mechanism for tracking how long references are valid. They prevent dangling references without runtime overhead.

### 1.1 What a lifetime actually is

A lifetime is a **region of code** during which a reference is guaranteed to be valid. The compiler infers these regions and checks that no reference outlives the data it points to.

```rust
fn main() {
    let r;                  // ---------+-- 'a
                            //          |
    {                       //          |
        let x = 5;         // -+-- 'b  |
        r = &x;            //  |       |  ERROR: `x` does not live long enough
    }                       // -+       |
                            //          |
    println!("{r}");        // ---------+
}
```

`'b` is strictly shorter than `'a` (equivalently, `'a` outlives `'b` — written `'a: 'b`), so the reference `r` would dangle after `x` is dropped.

### 1.2 Named lifetimes

When the compiler can't figure out lifetimes on its own, you annotate them:

```rust
// "Given some lifetime 'a, both input and output live for at least 'a"
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

The annotations don't *change* how long things live — they help the compiler *verify* that the relationships are sound.

### 1.3 `'static`

`'static` means "valid for the entire program duration." Two common sources:

```rust
// String literals are &'static str — they're baked into the binary.
let s: &'static str = "hello";

// Owned data can satisfy 'static because it has no borrows:
let owned: String = String::from("hello");
// String: 'static — it owns its data, so it's valid "forever" (until dropped).
```

**Important:** `T: 'static` does NOT mean "lives forever." It means "T contains no non-`'static` references." Owned types like `String`, `Vec<u8>`, `i32` are all `'static`.

---

## 2. Lifetime Elision Rules

Rust applies three rules to infer lifetimes in function signatures so you don't have to write them everywhere.

### 2.1 The three rules

1. **Each reference parameter gets its own lifetime:**
   ```rust
   fn foo(x: &str, y: &str)
   // becomes:
   fn foo<'a, 'b>(x: &'a str, y: &'b str)
   ```

2. **If there's exactly one input lifetime, it's assigned to all output lifetimes:**
   ```rust
   fn foo(x: &str) -> &str
   // becomes:
   fn foo<'a>(x: &'a str) -> &'a str
   ```

3. **If one of the parameters is `&self` or `&mut self`, its lifetime is assigned to all output lifetimes:**
   ```rust
   fn foo(&self, x: &str) -> &str
   // becomes:
   fn foo<'a, 'b>(&'a self, x: &'b str) -> &'a str
   ```

### 2.2 When elision fails

When there are multiple input lifetimes and no `&self`, the compiler cannot decide which one to assign to the output:

```rust
// ERROR: missing lifetime specifier
fn longest(x: &str, y: &str) -> &str { ... }

// Fix: annotate explicitly
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }
```

### 2.3 Elision in `impl` blocks

```rust
struct Context<'a>(&'a str);

impl<'a> Context<'a> {
    // Rule 3 applies: output gets the lifetime of &self
    fn get(&self) -> &str {
        self.0
    }
    // Desugars to:
    // fn get<'b>(&'b self) -> &'b str
    //
    // But wait — self.0 has lifetime 'a, and we're returning it as 'b.
    // This works because 'a: 'b is implied by &'b Self where Self = Context<'a>.
}
```

---

## 3. Early-Bound vs Late-Bound Lifetimes

This is one of the least documented but most confusing aspects of Rust's lifetime system.

### 3.1 The distinction

Think of it by analogy with type parameters:

```rust
fn generic<T>(x: T) -> T { x }

// When you USE this function, you pick T:
generic::<i32>(5);      // T = i32 — chosen at use site
generic::<String>(s);   // T = String — chosen at use site
```

Lifetimes can work the same way — **but they don't have to**. There are two modes:

- **Late-bound lifetime** (the default, common case): The lifetime is chosen **fresh at each call site**, automatically. You never have to think about it — the compiler picks the right lifetime for each call independently. This is how most lifetimes work.

- **Early-bound lifetime** (rare, triggered by `where` clauses): The lifetime behaves like a type parameter — it becomes part of the function's identity, fixed *before* the function is called. Think of it as: the lifetime is "baked in" to which version of the function you're referring to.

A concrete example to build intuition:

```rust
// Late-bound 'a — the normal case.
// Each call to `foo` independently picks whatever 'a works:
fn foo<'a>(x: &'a str) -> &'a str { x }

fn main() {
    let s1 = String::from("hello");
    let s2 = String::from("world");
    
    foo(&s1);  // 'a = lifetime of s1
    foo(&s2);  // 'a = lifetime of s2 (different! each call picks its own)
    
    // There's just ONE `foo` — it works for any lifetime.
}
```

```rust
// Early-bound 'a — the lifetime is fixed ahead of time.
fn bar<'a>(x: &'a str) -> &'a str
where
    'a: 'static  // this where clause forces 'a to be early-bound
{ x }

fn main() {
    // Conceptually, bar is like bar::<'some_lifetime> — the lifetime
    // is pre-selected, not flexible at each call site.
    // In practice: bar can ONLY be called with 'static references now.
    bar("hello");  // OK: &'static str
    
    let owned = String::from("hi");
    // bar(&owned);  // ERROR: 'a must be 'static
}
```

**Why does this distinction exist?** It affects whether a function can be used with `for<'a>` (universal quantification). Late-bound = flexible, can satisfy `for<'a>`. Early-bound = fixed, cannot.

### 3.2 When is a lifetime early-bound?

A lifetime parameter on a function becomes **early-bound** when it appears in a `where` clause bound that isn't just part of the signature. Specifically:

```rust
// Late-bound: 'a only appears in the signature
fn foo<'a>(x: &'a str) -> &'a str { x }

// Early-bound: 'a appears in a where clause
fn bar<'a>(x: &'a str) -> &'a str
where
    'a: 'static  // any bound involving 'a makes it early-bound
{ x }
```

The key trigger: **any `where` bound that mentions the lifetime** (like `Self: 'a`, `'a: 'b`, `T: 'a`) promotes it to early-bound.

### 3.3 Why does it matter?

The practical consequence: **can this function work for any lifetime the caller throws at it, or is the lifetime pre-committed?**

Consider a function that accepts another function as an argument and wants to call it with a *local* reference — one that only lives inside the function body:

```rust
fn apply(f: impl for<'a> Fn(&'a str) -> &'a str) -> String {
    let owned = String::from("hello");  // lives only inside `apply`
    f(&owned).to_string()               // calls f with a short-lived reference
}
```

The `for<'a>` here means: "`f` must work for **any** lifetime `'a` I give it — including short-lived ones I create on the spot." This is called **universal quantification** — the function promises to handle *all possible* lifetimes, not just one specific one.

> **Quick primer on `for<'a>`:** You can read `for<'a> Fn(&'a str) -> &'a str` as "for all lifetimes 'a, this implements Fn(&'a str) -> &'a str." It's covered in depth in [Section 6](#6-higher-ranked-lifetimes-fora). For now, just understand it as "works for any lifetime."

**Late-bound lifetimes satisfy this requirement** — since the lifetime isn't pre-committed, the function *can* work for any lifetime:

```rust
fn late_bound<'a>(x: &'a str) -> &'a str { x }

fn main() {
    apply(late_bound); // OK — late_bound works for any 'a
}
```

**Early-bound lifetimes cannot** — the lifetime is already fixed, so it can't promise to work for *all* lifetimes:

```rust
fn early_bound<'a>(x: &'a str) -> &'a str
where
    'a: 'a,  // trivial bound, but makes 'a early-bound!
{ x }

fn main() {
    // apply(early_bound); // ERROR: expected `for<'a> Fn(...)`, found specific lifetime
    // The compiler says: "early_bound has a specific 'a baked in,
    // but I need something that works for ALL lifetimes."
}
```

**Summary:** In both cases the caller picks the lifetime. The difference is how the *type system* views the function:
- Late-bound: the function is *one entity* that inherently works for all lifetimes → can satisfy `for<'a>` ("works for any lifetime you give me").
- Early-bound: the function is *parameterized* by the lifetime — the caller picks one (that satisfies the constraints), producing a specific instantiation → cannot satisfy `for<'a>` ("I'm configured for a particular lifetime, not all of them at once").

### 3.4 The `where Self: 'a` case

This is why it matters in traits:

```rust
trait Produce {
    fn produce<'a>(&'a self) -> &'a str
    where
        Self: 'a;  // makes 'a early-bound
}

impl Produce for Borrowed<'_> {
    fn produce<'a>(&'a self) -> &'a str
    where
        Self: 'a,  // MUST be here — trait says 'a is early-bound
    {
        self.0
    }
}
```

If the trait declares `where Self: 'a`, the impl **must** include it too. Removing it from the impl causes E0195 because:
- The trait declares `'a` as early-bound (the `where` clause forces it)
- The impl without the clause has `'a` as late-bound
- These are incompatible function signatures

### 3.5 When do you actually need `where Self: 'a`?

For regular methods with `&'a self`, the bound is **implied** — you technically don't need it. It becomes essential with **GATs** (Generic Associated Types):

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;  // REQUIRED here
    fn next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
}
```

Without `where Self: 'a` on the GAT, you could write `<T as LendingIterator>::Item<'static>` even when `T` holds short-lived borrows — potentially creating dangling references. The bound prevents naming the associated type with a lifetime that `Self` can't satisfy.

### 3.6 Practical rule of thumb

| Scenario | Bound needed? |
|----------|--------------|
| Regular method `fn foo<'a>(&'a self) -> &'a T` | No (implied by `&'a self`) |
| GAT definition `type Item<'a>` | Yes: `where Self: 'a` |
| Trait method that returns a GAT | Depends on GAT bounds |
| You want the lifetime to be late-bound (for HRTB use) | Don't add extra where bounds |

---

## 4. Subtyping and Variance

Rust has a limited form of subtyping: it applies **only to lifetimes**. A longer lifetime is a subtype of a shorter one.

### 4.1 Lifetime subtyping

If `'long: 'short` ("`'long` outlives `'short`"), then `&'long T` can be used wherever `&'short T` is expected:

```rust
fn example<'long, 'short>(s: &'long str)
where
    'long: 'short,
{
    let _: &'short str = s; // OK: &'long str is a subtype of &'short str
}
```

This makes intuitive sense: if data lives longer, a reference to it is valid in any shorter context.

### 4.2 Variance

Variance describes how subtyping of a parameter translates to subtyping of the containing type:

| Variance | Meaning | Example |
|----------|---------|---------|
| **Covariant** | If `'a: 'b`, then `T<'a>` is a subtype of `T<'b>` | `&'a T` is covariant in `'a` |
| **Contravariant** | If `'a: 'b`, then `T<'b>` is a subtype of `T<'a>` | `fn(&'a str)` is contravariant in `'a` |
| **Invariant** | No subtyping relationship | `&'a mut T` is invariant in `T` |

### 4.3 Variance of common types

| Type | Variance in `'a` | Variance in `T` |
|------|-------------------|-----------------|
| `&'a T` | covariant | covariant |
| `&'a mut T` | covariant | **invariant** |
| `Box<T>` | — | covariant |
| `Vec<T>` | — | covariant |
| `Cell<T>` | — | **invariant** |
| `fn(T) -> U` | — | **contra** in `T`, covariant in `U` |
| `*const T` | — | covariant |
| `*mut T` | — | **invariant** |

### 4.4 Why `&mut T` is invariant in `T`

```rust
fn evil(s: &mut &'static str, short: &str) {
    // If &mut T were covariant in T, we could do:
    // *s = short;  // write a short-lived reference into a &'static str slot because we can convert &'short str to &'static str!
}

fn main() {
    let mut s: &'static str = "hello";
    let owned = String::from("temporary");
    evil(&mut s, &owned);
    // s would now point to freed memory after `owned` is dropped!
}
```

Invariance prevents this: `&mut &'static str` cannot be treated as `&mut &'short str`.

### 4.5 Variance in structs

A struct's variance is determined by how it *uses* its parameters:

```rust
struct Covariant<'a> {
    data: &'a str,  // &'a T → covariant in 'a
}

struct Invariant<'a> {
    data: &'a mut String,  // &'a mut T → invariant in T (but covariant in 'a!)
    // Actually this struct is covariant in 'a since &'a mut T is covariant in 'a
}

struct AlsoInvariant<'a> {
    data: Cell<&'a str>,  // Cell<T> → invariant in T → invariant in 'a
}
```

If you need to *force* invariance (e.g., for safety in unsafe code), use `PhantomData`:

```rust
use std::marker::PhantomData;

struct Forced<'a, T> {
    _marker: PhantomData<Cell<&'a T>>,  // forces invariance in both 'a and T
}
```

### 4.6 Practical impact

Variance issues most commonly surface as confusing "lifetime too short" errors when:
- You store a reference in a `&mut` context
- You have a struct with interior mutability (`Cell`, `RefCell`, `Mutex`)
- You try to shorten a lifetime through a mutable reference

Understanding variance helps decode these errors: ask "is this position covariant, contravariant, or invariant?" to understand why the compiler rejects a coercion.

---

## 5. Lifetimes in `dyn Trait`

Every trait object has a hidden lifetime bound. This is one of the most frequently misunderstood parts of Rust.

### 5.1 The hidden lifetime

`dyn Trait` is actually `dyn Trait + 'lifetime`. The lifetime represents how long the underlying data is valid. When you write `Box<dyn Display>`, the compiler infers a lifetime based on context.

### 5.2 Default object lifetime bounds

The rules depend on the context:

| Context | Default lifetime |
|---------|-----------------|
| `Box<dyn Trait>` (no reference) | `'static` |
| `&'a dyn Trait` | `'a` |
| `&'a mut dyn Trait` | `'a` |
| `dyn Trait + 'a` (explicit) | `'a` |
| Struct field `Box<dyn Trait>` (struct has no lifetime param) | `'static` |
| Struct field `Box<dyn Trait>` (struct has lifetime `'a`) | `'a` |

```rust
use std::fmt::Display;

// These are equivalent:
fn foo(x: Box<dyn Display>) {}           // means Box<dyn Display + 'static>
fn bar(x: Box<dyn Display + 'static>) {} // explicit

// These are equivalent:
fn baz(x: &dyn Display) {}               // means &'a (dyn Display + 'a)
fn qux<'a>(x: &'a (dyn Display + 'a)) {} // explicit
```

### 5.2.1 Example: struct field `Box<dyn Trait>` with lifetime `'a`

When a struct has a lifetime parameter, a `Box<dyn Trait>` field *implicitly* gets that lifetime as its bound (not `'static`). This is the last row in the table above.

```rust
use std::fmt::Display;

trait Renderer {
    fn render(&self) -> String;
}

// ┌── The struct has lifetime 'a
// │
struct Page<'a> {
    title: &'a str,
    //        ┌── Compiler sees this as Box<dyn Renderer + 'a>  (NOT 'static!)
    //        │   because the struct has a lifetime parameter.
    body: Box<dyn Renderer>,
}

// A renderer that borrows data:
struct MarkdownBody<'a> {
    source: &'a str,
}

impl<'a> Renderer for MarkdownBody<'a> {
    fn render(&self) -> String {
        format!("**{}**", self.source)
    }
}

fn build_page<'a>(title: &'a str, md: &'a str) -> Page<'a> {
    // This compiles because body is Box<dyn Renderer + 'a>,
    // and MarkdownBody<'a> satisfies that bound.
    Page {
        title,
        body: Box::new(MarkdownBody { source: md }),
    }
}
```

**Why this matters:** if the struct had *no* lifetime parameter, `body` would default to `Box<dyn Renderer + 'static>`, and storing a `MarkdownBody<'a>` inside it would fail.

#### What does NOT compile (and how to read the error)

```rust
// Suppose we accidentally remove the lifetime from the struct:
struct StaticPage {
    title: String,
    body: Box<dyn Renderer>,  // Now means Box<dyn Renderer + 'static>
}

fn try_build(md: &str) -> StaticPage {
    // ERROR[E0759]: `md` has lifetime `'_` but it needs to satisfy a `'static` lifetime requirement
    //
    // The fix: either make body: Box<dyn Renderer + '_> / add a lifetime param,
    // or ensure the data you put in the Box is owned ('static).
    StaticPage {
        title: String::from("oops"),
        body: Box::new(MarkdownBody { source: md }),
        //                                    ^^ borrowed data with non-'static lifetime
    }
}
```

**Debugging checklist for trait-object lifetime errors:**

1. **Read the error code** — `E0759` / `E0621` usually means "this reference doesn't live long enough for the trait object."
2. **Check the default** — Is your `Box<dyn Trait>` in a struct with a lifetime param? Then the default is that lifetime. No lifetime param? Default is `'static`.
3. **Make it explicit** — Write `Box<dyn Trait + 'a>` or `Box<dyn Trait + 'static>` to confirm what the compiler expects. If the code still fails, the mismatch is clear.
4. **Use `#[deny(elided_lifetimes_in_paths)]`** — Adding this lint forces you to write out lifetimes, making hidden defaults visible.

### 5.3 The common surprise: returning `Box<dyn Trait>`

```rust
struct MyStruct<'a> {
    data: &'a str,
}

impl<'a> MyStruct<'a> {
    fn as_display(&self) -> Box<dyn Display + '_> {
        // Must use + '_ because the default would be 'static,
        // but we're returning something that borrows 'a data
        Box::new(self.data)
    }
}
```

Without `+ '_`, the compiler defaults to `'static` which fails because the result borrows from `self`.

### 5.3.1 The lifetime of `&self` vs the struct's `'a`

This is a common source of confusion. When you see:

```rust
impl<'a> MyStruct<'a> {
    fn as_display(&self) -> Box<dyn Display + '_> { ... }
}
```

…what is `'_` here? Is it `'a`? **No.** It's the lifetime of `&self` — a *separate, potentially shorter* lifetime. Fully desugared:

```rust
impl<'a> MyStruct<'a> {
    // 'b is the lifetime of the borrow of self.
    // 'a is the lifetime of the data *inside* self.
    // The compiler knows 'a: 'b (data outlives the borrow).
    fn as_display<'b>(&'b self) -> Box<dyn Display + 'b> {
        Box::new(self.data)  // &'a str coerces to where 'b is needed, since 'a: 'b
    }
}
```

**Two distinct lifetimes are in play — both come from "callers," but at different moments:**

Think of it as two separate steps happening at two different places in the code:

```rust
use std::fmt::Display;

struct MyStruct<'a> {
    data: &'a str,
}

impl<'a> MyStruct<'a> {
    fn as_display<'b>(&'b self) -> Box<dyn Display + 'b> {
        Box::new(self.data)
    }
}

fn main() {
    let text = String::from("hello");

    // ═══ STEP 1: Construction ═══════════════════════════════════════════
    // Right here, 'a gets "locked in" — it's the lifetime of `text`.
    // 'a = the scope of `text` (line 15 until end of main).
    let s = MyStruct { data: &text };

    // ═══ STEP 2: Method call ════════════════════════════════════════════
    // Right here, 'b gets determined — it's how long we borrow `s`.
    // 'b ≤ 'a (always, because you can't borrow the struct past its data's validity)
    let d = s.as_display();  // 'b = however long the compiler needs for `d`

    println!("{d}");
}
```

The key insight: **`'a` is fixed once, at construction. `'b` is created fresh every time someone calls a `&self` method.** They are both "caller-determined," but by *different* callers at *different* times:

| Lifetime | Fixed when? | By what code? |
|----------|-------------|---------------|
| `'a` | When the struct is **constructed** | The code that writes `MyStruct { data: &something }` — this pins `'a` to `something`'s scope |
| `'b` | Each time a **method is called** | The code that writes `s.as_display()` — this starts a borrow of `s` |

**The invariant:** `'a: 'b` — the struct's data must outlive any borrow of the struct. The compiler enforces this automatically (you can't borrow `s` after `text` is dropped).

#### Showing they're *different* — `'b` can be shorter than `'a`

```rust
fn main() {
    let text = String::from("hello");
    let s = MyStruct { data: &text };   // 'a locked in here (scope of `text`)

    {
        let d = s.as_display();         // 'b starts here...
        println!("{d}");
    }                                   // ...'b ends here (d is dropped)

    // 'a is still alive! `text` hasn't been dropped yet.
    // We can borrow `s` again with a fresh 'b:
    let d2 = s.as_display();            // new 'b, same 'a
    println!("{d2}");
}
```

Each call to `as_display()` produces its own `'b`. The struct's `'a` is the same throughout — it was fixed once at construction.

**The relationship:** `'a: 'b` — the data inside must outlive any borrow of the struct. This is always true (you can't borrow a struct after its contents become invalid).

#### Why they're different — a concrete demonstration

```rust
struct MyStruct<'a> {
    data: &'a str,
}

impl<'a> MyStruct<'a> {
    fn as_display(&self) -> Box<dyn Display + '_> {
        Box::new(self.data)
    }
}

fn main() {
    let long_lived: &'static str = "hello";     // 'a = 'static
    let s = MyStruct { data: long_lived };

    let display;
    {
        // Here we borrow `s` — this borrow ('b) lasts only inside this block.
        display = s.as_display();  // 'b = this block's scope
    }
    // `display` is valid only for 'b, NOT for 'a.
    // Even though the inner data is 'static, the returned Box is tied
    // to how long we borrowed `s`, because the signature says so.
}
```

#### Why not just use `'a` directly?

You *could* write `Box<dyn Display + 'a>` instead:

```rust
impl<'a> MyStruct<'a> {
    fn as_display_long(&self) -> Box<dyn Display + 'a> {
        Box::new(self.data)
    }
}
```

This compiles too, and returns a Box valid for the *data's* full lifetime — even after you stop borrowing the struct. The trade-off:

- **`+ '_`** (i.e. `+ 'b`): More restrictive for the caller (can't use the Box after the borrow ends), but easier to reason about. This is what elision gives you.
- **`+ 'a`**: More flexible for the caller (Box lives as long as the original data), but the signature *explicitly* promises the returned value doesn't depend on anything shorter than `'a`.

**Rule of thumb:** Use `+ '_` (let elision do its job) unless you have a reason to promise the longer lifetime. If users need to keep the result after dropping the borrow, switch to `+ 'a` and verify it compiles.

### 5.4 Trait objects in structs

```rust
// This requires the inner value to be 'static:
struct Static {
    handler: Box<dyn Fn()>,  // means Box<dyn Fn() + 'static>
}

// This allows borrowed data inside the trait object:
struct Borrowed<'a> {
    handler: Box<dyn Fn() + 'a>,
}

fn example() {
    let data = String::from("hello");
    
    // This fails:
    // let s = Static { handler: Box::new(|| println!("{data}")) };
    // ERROR: `data` does not live long enough (closure captures &data, needs 'static)
    
    // This works:
    let b = Borrowed { handler: Box::new(|| println!("{data}")) };
}
```

### 5.5 Multiple lifetime bounds

A trait object can only have one lifetime bound:

```rust
// OK:
fn foo(x: Box<dyn Display + 'static>) {}
fn bar<'a>(x: Box<dyn Display + 'a>) {}

// ERROR: only a single explicit lifetime bound is permitted
// fn baz<'a, 'b>(x: Box<dyn Display + 'a + 'b>) {}
```

If you need the trait object to be valid for multiple lifetimes, use a bound on the lifetimes themselves:

```rust
fn combined<'a, 'b>(x: Box<dyn Display + 'a>)
where
    'b: 'a,  // if you need 'b to outlive 'a
{ }
```

---

## 6. Higher-Ranked Lifetimes (`for<'a>`)

### 6.1 The problem

Sometimes you need a function/closure that works for *any* lifetime, not a specific one:

```rust
fn apply_to_ref(f: ???, data: &str) -> usize {
    f(data)
}
```

What type is `f`? It should accept `&str` with *any* lifetime. You can't write `Fn(&'a str)` because `'a` would need to be declared somewhere — and tying it to a specific lifetime is too restrictive.

### 6.2 `for<'a>` syntax

Higher-ranked trait bounds (HRTBs) solve this:

```rust
fn apply_to_ref(f: impl for<'a> Fn(&'a str) -> usize, data: &str) -> usize {
    f(data)
}

// Or equivalently with sugar (Fn(&str) desugars to for<'a> Fn(&'a str)):
fn apply_to_ref_sugar(f: impl Fn(&str) -> usize, data: &str) -> usize {
    f(data)
}
```

`for<'a> Fn(&'a str)` means "implements `Fn(&'a str)` for ALL possible lifetimes `'a`."

### 6.3 When you see `for<'a>` explicitly

The sugar handles most cases, but you need explicit `for<'a>` when:

```rust
// 1. Returning a reference with a different lifetime than input:
fn apply<F>(f: F) -> String
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let owned = String::from("hello world");
    f(&owned).to_string()
}

// 2. In trait bounds with associated lifetimes:
trait Parser {
    fn parse<'i>(&self, input: &'i str) -> &'i str;
}

fn use_parser(p: impl for<'i> Parser) { /* ... */ }
// Note: this specific syntax doesn't work — for<> on traits is limited.
// Instead you'd use: impl Parser (since parse is already generic over 'i)

// 3. With closures stored in structs:
struct Callback {
    f: Box<dyn for<'a> Fn(&'a str) -> &'a str>,
}
```

### 6.4 Limitations and the "late-bound" connection

`for<'a>` quantification only works with late-bound lifetimes. If a lifetime is early-bound (due to `where` clauses), it can't be universally quantified:

```rust
// This function has a late-bound 'a — can be used with for<'a>:
fn identity<'a>(x: &'a str) -> &'a str { x }

// This function has an early-bound 'a — CANNOT be used with for<'a>:
fn constrained<'a: 'static>(x: &'a str) -> &'a str { x }

// Works:
let f: for<'a> fn(&'a str) -> &'a str = identity;

// Fails:
// let g: for<'a> fn(&'a str) -> &'a str = constrained;
```

This connects back to Section 3: early-bound lifetimes are "fixed" and can't be universally quantified.

---

## 7. Lifetime Bounds on Types

### 7.1 `T: 'a` — "T outlives 'a"

This bound means "all references inside `T` live at least as long as `'a`":

```rust
fn store_ref<'a, T: 'a>(val: &'a T) -> &'a T {
    val
}
```

For owned types (`String`, `Vec<u8>`, `i32`), `T: 'a` is always satisfied because they contain no references. The bound only becomes restrictive for types that borrow data:

```rust
struct Wrapper<'a, T: 'a> {
    value: &'a T,
}
// T: 'a is required here — you can't have &'a T unless T is valid for 'a.
// (The compiler actually infers this bound automatically since Rust 1.31.)
```

### 7.2 `T: 'static`

A very common bound meaning "T contains no non-`'static` borrows":

```rust
fn spawn_thread<T: Send + 'static>(val: T) {
    std::thread::spawn(move || {
        // val is used here — it must not contain borrowed data
        // because the thread might outlive the caller
        drop(val);
    });
}

// OK: String owns its data
spawn_thread(String::from("hello"));

// ERROR: &str has a non-'static lifetime
// spawn_thread(&some_local_string);
```

### 7.3 `'a: 'b` — lifetime outlives another

```rust
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str
where
    'b: 'a,  // 'b outlives 'a — so we could return y as &'a str too
{
    if x.len() > y.len() { x } else { y }
}
```

In practice, you more often just unify lifetimes (`fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`) and let the compiler figure out coercions via variance.

### 7.4 Implied bounds

Since Rust 1.31, many lifetime bounds are **implied** and don't need explicit annotation:

```rust
// Before 1.31, you needed:
struct Ref<'a, T: 'a> {
    value: &'a T,
}

// After 1.31, the compiler infers T: 'a from the &'a T field:
struct Ref<'a, T> {
    value: &'a T,  // implies T: 'a automatically
}
```

Similarly, `&'a self` implies `Self: 'a` without needing to write it.

### 7.5 `Self` and Lifetimes

When a struct has a lifetime parameter, `Self` carries that lifetime. This creates subtle interactions worth understanding in one place.

#### What `Self: 'a` really means

If `Self` is `MyStruct<'x>`, then `Self: 'a` requires `'x: 'a` — all borrows inside the struct must outlive `'a`:

```rust
struct Reader<'x> {
    data: &'x [u8],
}

impl<'x> Reader<'x> {
    // &'a self means Self: 'a, which means Reader<'x>: 'a, which means 'x: 'a.
    // So 'a can be at most as long as 'x.
    fn read(&self) -> &[u8] {
        self.data
    }
}
```

#### Two different lifetimes in methods

Methods on lifetime-carrying structs involve **two** lifetimes that beginners often conflate:

```rust
struct Buffer<'buf> {
    data: &'buf str,
}

impl<'buf> Buffer<'buf> {
    // Lifetime of the BORROW of self (how long we borrow the Buffer value)
    fn len(&self) -> usize {
        //      ^--- this is &'short self, a short-lived borrow
        self.data.len()
    }

    // Returns data tied to 'buf (the struct's inner lifetime) — NOT to &self
    fn contents(&self) -> &'buf str {
        self.data
        // The returned reference outlives the borrow of self!
        // This is fine because 'buf: 'borrow is implied by &'borrow self.
    }

    // Returns data tied to the borrow of self — shorter lived
    fn as_bytes(&self) -> &[u8] {
        // Elision: returns &'self_borrow [u8]
        self.data.as_bytes()
        // Actually this also works with 'buf, but elision ties it to &self
    }
}

fn demo() {
    let text = String::from("hello");
    let contents: &str;
    {
        let buf = Buffer { data: &text };
        contents = buf.contents(); // OK! Tied to 'buf (text's lifetime), not buf's borrow
    }
    // buf is dropped, but contents is still valid — it borrows text, not buf
    println!("{contents}");
}
```

#### Returning `&'a str` vs `&str` in impl blocks

```rust
struct Config<'a> {
    name: &'a str,
    version: u32,
}

impl<'a> Config<'a> {
    // Returns data from INSIDE the struct — lifetime is 'a
    fn name(&self) -> &'a str {
        self.name
    }

    // Returns data derived from the borrow of self — lifetime is tied to &self
    fn description(&self) -> String {
        // Owned return — no lifetime issues
        format!("{} v{}", self.name, self.version)
    }

    // WRONG approach — fighting the borrow checker:
    // fn name_bad(&self) -> &str {
    //     self.name
    //     // This works too! Elision gives &'self_borrow str.
    //     // But it's MORE restrictive than necessary — callers can't use
    //     // the result after the borrow of self ends.
    //     // Prefer returning &'a str to give callers maximum flexibility.
    // }
}
```

**Rule of thumb:** If a method returns data from inside the struct (not computed from `&self`), annotate the return lifetime as the struct's lifetime parameter — don't rely on elision, which would over-restrict it.

#### `Self: 'static` — when your struct has no borrows

For structs without lifetime parameters, `Self: 'static` is trivially satisfied:

```rust
struct Owned {
    data: String,
}

impl Owned {
    fn spawn_with_self(self) {
        // Works! Owned: 'static because it has no references
        std::thread::spawn(move || {
            println!("{}", self.data);
        });
    }
}

struct Borrowed<'a> {
    data: &'a str,
}

impl<'a> Borrowed<'a> {
    fn spawn_with_self(self) {
        // ERROR: Borrowed<'a>: 'static is NOT satisfied (unless 'a = 'static)
        // std::thread::spawn(move || {
        //     println!("{}", self.data);
        // });
    }
}
```

---

## 8. Common Lifetime Puzzles

### 8.1 Self-referential structs

You **cannot** create a struct that borrows from its own fields:

```rust
// This DOES NOT work:
struct SelfRef {
    data: String,
    slice: &str,  // supposed to reference `data` — but what lifetime?
}
```

Why? When the struct moves, `data` changes address, but `slice` would still point to the old location. There is no lifetime that can express "borrows from a sibling field."

**Solutions:**
- Use indices instead of references (`slice_start: usize, slice_end: usize`)
- Use `Pin` + unsafe (what `tokio` does internally for futures)
- Use crates like `ouroboros` or `self_cell`
- Restructure to avoid the self-reference

### 8.2 Returning references to local data

```rust
// ERROR: cannot return reference to local variable
fn create() -> &str {
    let s = String::from("hello");
    &s  // s is dropped at end of function!
}
```

**Solutions:**
- Return the owned type (`-> String`)
- Take the data as a parameter and return a reference to it
- Use `'static` data (string literals, `Box::leak`, `lazy_static`)

### 8.3 Multiple mutable borrows confusion

```rust
struct Data {
    first: String,
    second: String,
}

impl Data {
    fn get_both(&mut self) -> (&mut String, &mut String) {
        (&mut self.first, &mut self.second)  // OK! Disjoint borrows.
    }
}
```

This works because the compiler can see the borrows are disjoint (different fields). But this fails:

```rust
impl Data {
    fn get_first(&mut self) -> &mut String { &mut self.first }
    fn get_second(&mut self) -> &mut String { &mut self.second }
    
    fn get_both_via_methods(&mut self) -> (&mut String, &mut String) {
        // ERROR: cannot borrow `*self` as mutable more than once
        (self.get_first(), self.get_second())
    }
}
```

Each method takes `&mut self` — the compiler doesn't track which fields each method actually touches. Solution: access fields directly, or restructure.

### 8.4 The "temporary value dropped" pattern

```rust
fn get_string() -> String {
    String::from("hello")
}

// ERROR: temporary value dropped while borrowed
// let r: &str = &get_string();

// Fix: bind the temporary to extend its lifetime
let owned = get_string();
let r: &str = &owned;
```

A reference to a temporary only lives to the end of the statement (with some exceptions for `let` bindings and `match` scrutinees).

---

## 9. Lifetimes and Closures

### 9.1 Closures capture by reference

```rust
fn main() {
    let data = String::from("hello");
    
    // Captures &data — closure is valid as long as `data` lives
    let closure = || println!("{data}");
    
    closure();
    drop(data);
    // closure(); // ERROR: borrow of moved value
}
```

### 9.2 Closure return types and lifetimes

Closures that return references are tricky:

```rust
fn make_closure<'a>(data: &'a str) -> impl Fn() -> &'a str {
    move || data  // captures the reference, returns it
}
```

But you can't return a reference to captured data:

```rust
// This does NOT compile:
fn bad_closure() -> impl Fn() -> &str {
    let data = String::from("hello");
    move || &data  // data is moved into closure, but what's the lifetime of &data?
    // There's no named lifetime for "data owned by the closure"
}
```

### 9.3 Closures and HRTBs

When you write `Fn(&str) -> &str`, it desugars to `for<'a> Fn(&'a str) -> &'a str`. This means the closure must work for *any* input lifetime:

```rust
// This works:
let closure: Box<dyn Fn(&str) -> &str> = Box::new(|s| s);

// This also works — returning a prefix:
let closure: Box<dyn Fn(&str) -> &str> = Box::new(|s| &s[..1]);

// This does NOT work — returning something with a different lifetime:
let static_str: &'static str = "hello";
let closure: Box<dyn Fn(&str) -> &str> = Box::new(|_s| static_str);
// Actually this does work! 'static: 'a for any 'a, so coercion applies.
```

### 9.4 The `move` keyword and lifetimes

`move` transfers ownership of captured variables into the closure. This changes lifetime semantics:

```rust
fn make_fn(data: String) -> impl Fn() {
    // Without move: ERROR — closure captures &data, which doesn't live long enough
    // With move: OK — closure owns the String
    move || println!("{data}")
}
```

`move` is essential when returning closures or sending them to other threads.

---

## 10. Lifetimes in Async

Async introduces unique lifetime challenges because futures are state machines that hold references across `.await` points.

### 10.1 Async functions and lifetimes

An async function returns an opaque `Future` that captures all its arguments:

```rust
async fn process(data: &str) -> usize {
    data.len()
}

// Desugars (roughly) to:
fn process<'a>(data: &'a str) -> impl Future<Output = usize> + 'a {
    async move { data.len() }
}
```

The future **borrows** `data` — it must remain valid until the future completes.

### 10.2 The `Send` + `'static` requirement

`tokio::spawn` requires `T: Future + Send + 'static`:

```rust
async fn example() {
    let data = String::from("hello");
    
    // OK: move owned data into the spawned task
    tokio::spawn(async move {
        println!("{data}");
    });
    
    // ERROR: borrowed data doesn't satisfy 'static
    // let r = &data;
    // tokio::spawn(async { println!("{r}") });
}
```

### 10.3 References held across `.await`

```rust
async fn problematic() {
    let data = String::from("hello");
    let r = &data;
    
    some_async_fn().await;  // r is held across this await point
    
    println!("{r}");  // r is still used after the await
}
```

This is fine *within one task*, but the borrow must not outlive `data`. The future's state machine internally stores `r` between the await point.

### 10.4 Async trait methods and lifetimes

```rust
trait Service {
    async fn call(&self, req: &str) -> String;
    // Desugars to returning impl Future + '_ where '_ captures both &self and &req
}
```

If you need the future to be `Send` (for spawning), you add bounds:

```rust
trait Service: Send + Sync {
    fn call<'a>(&'a self, req: &'a str) -> impl Future<Output = String> + Send + 'a;
}
```

### 10.5 The `Pin` connection

Futures that hold references across await points become self-referential once polled. `Pin` ensures they don't move after being polled, which would invalidate internal references:

```rust
async fn holds_ref() {
    let data = vec![1, 2, 3];
    let r = &data[0];      // <-- reference to local
    yield_now().await;      // future pauses here, holding `r`
    println!("{r}");        // uses `r` after resuming
}
// After first poll, the future's state contains both `data` and `r`.
// Moving the future would invalidate `r` → Pin prevents this.
```

This is why you can't do `let fut = holds_ref(); mem::swap(&mut fut, ...)` after polling.

---

## 11. Lifetimes in Serde

Serde's `Deserialize` trait has a lifetime parameter `'de` that is one of the most practical examples of non-trivial lifetimes. Understanding it clarifies the difference between owned vs borrowed output and when zero-copy deserialization is possible.

### 11.1 What `'de` means

`'de` is the lifetime of the **input data** being deserialized (e.g., a `&'de str` JSON buffer). It tells the compiler: "the deserialized struct may borrow directly from the input."

```rust
use serde::Deserialize;

// 'de = lifetime of the input buffer (e.g., the JSON bytes)
#[derive(Deserialize)]
struct Request<'de> {
    // This borrows directly from the input — zero copy!
    #[serde(borrow)]
    path: &'de str,

    // This is owned — serde allocates a new String regardless
    body: String,
}
```

Desugared, the derived impl looks like:

```rust
impl<'de> Deserialize<'de> for Request<'de> {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: serde::Deserializer<'de>,
    {
        // ...
        todo!()
    }
}
```

The `'de` on both `Request` and `Deserializer` ties them together: the deserializer lends data from the input buffer, and the output struct borrows it.

### 11.2 Zero-copy deserialization

Zero-copy means the output struct contains `&'de str` fields that point directly into the input buffer — no allocation for those fields.

```rust
use serde::Deserialize;
use serde_json;

#[derive(Deserialize, Debug)]
struct Config<'a> {
    #[serde(borrow)]
    name: &'a str,
    #[serde(borrow)]
    version: &'a str,
    debug: bool,
}

fn main() {
    let json_input = r#"{"name": "myapp", "version": "1.0", "debug": true}"#;

    // `config` borrows from `json_input` — 'de = lifetime of json_input
    let config: Config = serde_json::from_str(json_input).unwrap();
    //                                        ^^^^^^^^^^
    //                   This buffer must outlive `config`

    println!("{:?}", config);
    // config.name points into json_input's memory — no String allocation!
}
```

**Critical rule:** the input buffer must outlive the deserialized struct. If you drop or overwrite the buffer, the borrowed fields dangle.

### 11.3 `DeserializeOwned` vs `Deserialize<'de>`

This is the most common decision point:

| Trait bound | Meaning | When to use |
|-------------|---------|-------------|
| `Deserialize<'de>` | Output may borrow from input | You control the input buffer's lifetime; want zero-copy |
| `DeserializeOwned` | Output owns all its data | You need the result to be `'static` / outlive the input |

`DeserializeOwned` is sugar for:

```rust
// DeserializeOwned means: "for ANY input lifetime, I can still deserialize"
// i.e., I don't borrow from the input at all.
use serde::de::DeserializeOwned;

// These are equivalent:
fn parse_owned<T: DeserializeOwned>(input: &str) -> T { todo!() }
fn parse_owned2<T: for<'de> Deserialize<'de>>(input: &str) -> T { todo!() }
//                  ^^^^^^^^^^^^^^^^^^^^^^^^^
//   HRTB! "for any 'de" = doesn't depend on input lifetime = owns everything
```

**Example — when you're forced into `DeserializeOwned`:**

```rust
use serde::de::DeserializeOwned;

// The input is consumed/dropped inside this function, so the output
// can't borrow from it. Must use DeserializeOwned.
fn fetch_and_parse<T: DeserializeOwned>(url: &str) -> T {
    let response_body: String = http_get(url);  // owned, temporary
    serde_json::from_str(&response_body).unwrap()
    // response_body is dropped here — T must not borrow from it
}
# fn http_get(_: &str) -> String { String::new() }
```

If `T` had `&'de str` fields, this would fail — the data it borrows from (`response_body`) is gone.

### 11.4 `Cow<'de, str>` — the flexible middle ground

When you want zero-copy *when possible* but owned data as a fallback (e.g., when the JSON string contains escape sequences that need unescaping):

```rust
use serde::Deserialize;
use std::borrow::Cow;

#[derive(Deserialize, Debug)]
struct Message<'a> {
    #[serde(borrow)]
    text: Cow<'a, str>,
    //    ^^^^^^^^^^^
    //    Borrowed(&'a str)  → zero-copy, points into input (no escapes)
    //    Owned(String)      → allocated, when serde had to unescape
}

fn main() {
    // No escapes → Cow::Borrowed (zero-copy)
    let input1 = r#"{"text": "hello"}"#;
    let m1: Message = serde_json::from_str(input1).unwrap();
    assert!(matches!(m1.text, Cow::Borrowed(_)));

    // Has escape sequence → Cow::Owned (must allocate to unescape)
    let input2 = r#"{"text": "hello\nworld"}"#;
    let m2: Message = serde_json::from_str(input2).unwrap();
    assert!(matches!(m2.text, Cow::Owned(_)));
}
```

### 11.5 Common mistake: dropping the buffer too early

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Entry<'a> {
    #[serde(borrow)]
    key: &'a str,
}

fn load_entry() -> Entry<'static> {
    let json = std::fs::read_to_string("data.json").unwrap();
    let entry: Entry = serde_json::from_str(&json).unwrap();
    entry
    // ERROR[E0515]: cannot return value referencing local variable `json`
    //
    // entry.key borrows from `json`, but `json` is dropped when this
    // function returns. 'de = scope of `json`, and you're trying to
    // return something that outlives it.
    //
    // Fix: use String instead of &'a str, or return the buffer alongside the struct.
}
```

**Debugging serde lifetime errors:**

1. If the error says "borrowed value does not live long enough" near a `from_str` / `from_slice` call, your struct borrows from the input and you're dropping the input too early.
2. If you can't change the struct, switch from `&'a str` to `String` (or `Cow<'a, str>`).
3. If your function must return the deserialized value and you don't control the input's lifetime, use `DeserializeOwned`.

---

## Summary

| Concept | Key takeaway |
|---------|-------------|
| Lifetime annotations | Describe relationships; don't change how long things live |
| Elision | 3 rules that cover ~90% of cases |
| Early vs late-bound | `where` bounds on lifetimes make them early-bound; affects HRTB compatibility |
| Variance | `&T` covariant, `&mut T` invariant in `T`; determines what coercions are valid |
| `dyn Trait` lifetimes | Always has a hidden lifetime; defaults depend on context |
| `for<'a>` | Universal quantification over lifetimes; only works with late-bound lifetimes |
| `T: 'a` | All references in T outlive 'a; implied for `&'a T` since Rust 1.31 |
| Self-referential structs | Not expressible with safe Rust lifetimes; use Pin or indices |
| Async lifetimes | Futures capture borrows; `spawn` needs `'static`; `Pin` prevents invalidation |
| Serde `'de` | The input buffer lifetime; enables zero-copy deserialization |
