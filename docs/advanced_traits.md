# Advanced Traits in Rust

**Estimated reading time: ~75 minutes**

> **Prerequisite:** Read [lifetimes.md](lifetimes.md) first — this document assumes familiarity with early/late-bound lifetimes, variance, `dyn Trait` lifetime bounds, and HRTBs.

This document is a comprehensive deep-dive into Rust's trait system beyond the basics. We cover existential types (`impl Trait`, `dyn Trait`), advanced associated types (GATs, defaults, bounds), generic traits, higher-ranked trait bounds, trait objects, auto traits, marker traits, specialization, coherence, and the many subtle patterns that emerge from their interaction.

---

## Table of Contents

1. [Trait Fundamentals Refresher](#1-trait-fundamentals-refresher)
2. [Associated Types — Beyond the Basics](#2-associated-types--beyond-the-basics)
   - 2.1 [Associated Types vs Generic Parameters — When to Use Each](#21-associated-types-vs-generic-parameters--when-to-use-each)
   - 2.2 [Associated Type Defaults](#22-associated-type-defaults)
   - 2.3 [Associated Type Bounds](#23-associated-type-bounds)
   - 2.4 [Where Clauses on Associated Types](#24-where-clauses-on-associated-types)
   - 2.5 [Associated Constants and Functions](#25-associated-constants-and-functions)
3. [Generic Associated Types (GATs)](#3-generic-associated-types-gats)
   - 3.1 [The Lending Iterator Problem](#31-the-lending-iterator-problem)
   - 3.2 [GAT Syntax and Semantics](#32-gat-syntax-and-semantics)
   - 3.3 [GATs with Lifetime Parameters](#33-gats-with-lifetime-parameters)
   - 3.4 [GATs with Type Parameters](#34-gats-with-type-parameters)
   - 3.5 [GATs in Async Traits](#35-gats-in-async-traits)
   - 3.6 [Required Bounds on GATs](#36-required-bounds-on-gats)
4. [Existential Types: `impl Trait`](#4-existential-types-impl-trait)
   - 4.1 [Argument Position `impl Trait` (APIT)](#41-argument-position-impl-trait-apit)
   - 4.2 [Return Position `impl Trait` (RPIT)](#42-return-position-impl-trait-rpit)
   - 4.3 [RPIT in Traits (RPITIT)](#43-rpit-in-traits-rpitit)
   - 4.4 [Type Alias `impl Trait` (TAIT)](#44-type-alias-impl-trait-tait)
   - 4.5 [impl Trait in Let Bindings](#45-impl-trait-in-let-bindings)
   - 4.6 [Capturing Rules and Lifetime Semantics](#46-capturing-rules-and-lifetime-semantics)
5. [Trait Objects and Dynamic Dispatch: `dyn Trait`](#5-trait-objects-and-dynamic-dispatch-dyn-trait)
   - 5.1 [Vtables and Fat Pointers](#51-vtables-and-fat-pointers)
   - 5.2 [Object Safety Rules](#52-object-safety-rules)
   - 5.3 [Multiple Trait Bounds in `dyn`](#53-multiple-trait-bounds-in-dyn)
   - 5.4 [Lifetime Elision for `dyn Trait`](#54-lifetime-elision-for-dyn-trait)
   - 5.5 [Upcasting and Trait Object Coercions](#55-upcasting-and-trait-object-coercions)
   - 5.6 [`dyn*` — Experimental Inline Trait Objects](#56-dyn--experimental-inline-trait-objects)
6. [Generic Traits (Traits Parameterized by Types)](#6-generic-traits-traits-parameterized-by-types)
   - 6.1 [Defining Generic Traits](#61-defining-generic-traits)
   - 6.2 [Multiple Implementations via Generic Params](#62-multiple-implementations-via-generic-params)
   - 6.3 [Generic Traits vs Associated Types — Decision Framework](#63-generic-traits-vs-associated-types--decision-framework)
   - 6.4 [Operator Overloading as Generic Traits](#64-operator-overloading-as-generic-traits)
7. [Higher-Ranked Trait Bounds (HRTBs)](#7-higher-ranked-trait-bounds-hrtbs)
   - 7.1 [The `for<'a>` Syntax](#71-the-fora-syntax)
   - 7.2 [HRTBs with Closures](#72-hrtbs-with-closures)
   - 7.3 [HRTBs and GATs](#73-hrtbs-and-gats)
   - 7.4 [Higher-Ranked Types (Aspirational)](#74-higher-ranked-types-aspirational)
8. [Supertraits and Trait Inheritance](#8-supertraits-and-trait-inheritance)
   - 8.1 [Supertrait Syntax and Semantics](#81-supertrait-syntax-and-semantics)
   - 8.2 [Diamond Inheritance and Ambiguity](#82-diamond-inheritance-and-ambiguity)
   - 8.3 [Sealed Traits](#83-sealed-traits)
9. [Marker Traits and Auto Traits](#9-marker-traits-and-auto-traits)
   - 9.1 [Send, Sync, Unpin, Sized](#91-send-sync-unpin-sized)
   - 9.2 [Negative Implementations](#92-negative-implementations)
   - 9.3 [Auto Trait Leakage](#93-auto-trait-leakage)
   - 9.4 [Custom Marker Traits](#94-custom-marker-traits)
10. [Coherence, Orphan Rules, and Overlap](#10-coherence-orphan-rules-and-overlap)
    - 10.1 [The Orphan Rule](#101-the-orphan-rule)
    - 10.2 [The Newtype Pattern](#102-the-newtype-pattern)
    - 10.3 [Blanket Implementations](#103-blanket-implementations)
    - 10.4 [Overlap and Specialization](#104-overlap-and-specialization)
11. [Trait Bound Patterns and Tricks](#11-trait-bound-patterns-and-tricks)
    - 11.1 [Conditional Method Availability](#111-conditional-method-availability)
    - 11.2 [Extension Traits](#112-extension-traits)
    - 11.3 [Trait Aliases](#113-trait-aliases)
    - 11.4 [The Turbofish and Fully Qualified Syntax](#114-the-turbofish-and-fully-qualified-syntax)
    - 11.5 [Implied Bounds](#115-implied-bounds)
12. [Async and Traits](#12-async-and-traits)
    - 12.1 [Async fn in Traits](#121-async-fn-in-traits)
    - 12.2 [Send Bounds on Async Trait Methods](#122-send-bounds-on-async-trait-methods)
    - 12.3 [Async Closures and Fn Traits](#123-async-closures-and-fn-traits)
13. [The Sized, ?Sized, and Unsized Universe](#13-the-sized-sized-and-unsized-universe)
14. [Summary and Mental Model](#14-summary-and-mental-model)

---

## 1. Trait Fundamentals Refresher

Before diving into advanced topics, let's establish the key points that underpin everything:

```rust
// A trait declares a set of behaviors (methods, types, constants).
trait Summary {
    fn summarize(&self) -> String;
}

// An impl block connects a type to a trait.
struct Article { title: String, body: String }

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}...", self.title, &self.body[..20])
    }
}
```

**Key rules:**
- A trait can have **required** items (no body) and **provided** items (default body).
- Implementations can override defaults.
- Traits are **not types** themselves — but `dyn Trait` (trait objects) and `impl Trait` (existential types) are types.
- Traits may have **associated types**, **associated constants**, **associated functions**, **generic parameters**, and **supertraits**.

---

## 2. Associated Types — Beyond the Basics

### 2.1 Associated Types vs Generic Parameters — When to Use Each

Traits can carry type information in two ways: **generic parameters** and **associated types**. Neither is universally better — they encode different relationships between a type and a trait.

#### The core distinction

**Generic parameter** = an **input** to the trait. The *caller* (or the context) selects the type. A single type can implement the trait multiple times — once per distinct generic argument.

**Associated type** = an **output** of the trait. The *implementer* fixes the type. A single type can implement the trait at most once, and the associated type is determined by that implementation.

```rust
// Generic trait — multiple impls possible for the same Self type:
trait Convert<T> {
    fn convert(&self) -> T;
}

impl Convert<String> for i32 {
    fn convert(&self) -> String { self.to_string() }
}
impl Convert<f64> for i32 {
    fn convert(&self) -> f64 { *self as f64 }
}

fn main() {
    let x = 100i32;
    let s: String = x.convert(); // Convert<String>
    let f: f64 = x.convert();     // Convert<f64>
    let x = <i32 as Convert<String>>::convert(&100); // fully qualified syntax
}

// Associated type — exactly one impl per Self type:
trait IntoOwned {
    type Owned;
    fn into_owned(self) -> Self::Owned;
}

impl IntoOwned for &str {
    type Owned = String;
    fn into_owned(self) -> String { self.to_string() }
}
```

#### When to use generic parameters

Use generic parameters when:

1. **A type should implement the trait multiple times with different type arguments.** For example, `From<T>` is generic because `String` implements both `From<&str>` and `From<Vec<u8>>` — the source type is an input that varies.
2. **The caller needs to select the type.** If you want callers to choose which variant of the trait to invoke (e.g., `i32::convert::<f64>()`), the type must be a generic parameter.
3. **You are modelling a family of relationships rather than a single unique relationship.** `Add<Rhs>` is generic because a matrix might support addition with both another matrix and a scalar.

#### When to use associated types

Use associated types when:

1. **Given `Self`, the type is uniquely determined.** `Iterator::Item` is an associated type because the iterator returned by `Vec<u8>::into_iter()` always yields `u8` — it wouldn't make sense to implement `Iterator` twice with different item types.
2. **You want to enforce that there's only one implementation per type.** This prevents ambiguity and simplifies type inference — the compiler can figure out what `Item` is without the caller annotating it.
3. **You want ergonomic bounds.** With associated types, downstream code writes `T: Iterator<Item = u32>` which clearly communicates "the iterator whose item is `u32`". With a generic parameter, you'd write `T: Iterator<u32>`, which the compiler cannot distinguish from other potential impls without more context.

#### Why `Iterator` uses an associated type

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

If `Iterator` were `trait Iterator<Item>`, then the iterator returned by `Vec<u8>::into_iter()` could hypothetically implement `Iterator<u8>` and `Iterator<String>` simultaneously. That makes no sense — a given iterator produces one fixed element type. The associated type encodes this *"exactly one output type per Self"* invariant at the type-system level.

Additionally, because the type is uniquely determined, the compiler can infer it without annotation:

```rust
let v = vec![1, 2, 3];
let sum: i32 = v.into_iter().sum(); // compiler knows Item = i32
```

#### Quick decision heuristic

| Question | If **yes** → | If **no** → |
|----------|-------------|------------|
| Can `Self` meaningfully implement this trait multiple times with different types? | Generic parameter | Associated type |
| Does the *caller* choose the type? | Generic parameter | Associated type |
| Is the type uniquely determined once you know `Self`? | Associated type | Generic parameter |

#### Associated types in the standard library

`Iterator` is the most cited example, but associated types appear throughout the standard library wherever a trait has a uniquely determined output type:

| Trait | Associated type | Why it's associated |
|-------|----------------|---------------------|
| `Deref` | `type Target` | A `Box<T>` always derefs to exactly one type (`T`). You'd never want `Box<T>` to deref to two different types. |
| `Future` | `type Output` | A given future always resolves to one fixed type. |
| `FromStr` | `type Err` | Parsing `"hello"` into a `SocketAddr` can only fail in one way — the error type is determined by the implementer. |
| `IntoIterator` | `type Item`, `type IntoIter` | A `Vec<u8>` always converts into one specific iterator type yielding `u8`. |
| `ToOwned` | `type Owned` | `str` always produces a `String`; `[T]` always produces a `Vec<T>`. The owned form is uniquely determined. |
| `Add`, `Sub`, `Mul`... | `type Output` | Adding two `f64`s always produces an `f64`. The *output* type is fixed once you know `Self` and `Rhs`. |
| `Index` | `type Output` | Indexing a `Vec<T>` always yields a `T`. |

Notice the pattern: in every case, once you know the implementing type (and any generic parameters on the trait), the associated type is uniquely determined — there's exactly one correct answer.

Contrast with `From<T>`, which is generic in `T` because `String` legitimately implements `From<&str>`, `From<Vec<u8>>`, `From<char>`, etc. — the source type is an *input* that varies.

### 2.2 Associated Type Defaults

Traits can provide **default** associated types (stabilized in Rust 1.0 for type aliases, but fully usable in traits):

```rust
trait Builder {
    type Output = Vec<u8>;  // default associated type

    fn build(&self) -> Self::Output;
}

struct SimpleBuilder;

// Can omit `type Output` to accept the default:
impl Builder for SimpleBuilder {
    // type Output = Vec<u8>;  — inherited
    fn build(&self) -> Vec<u8> {
        vec![1, 2, 3]
    }
}

struct CustomBuilder;

impl Builder for CustomBuilder {
    type Output = String;  // override the default
    fn build(&self) -> String {
        String::from("custom")
    }
}
```

**Caveat**: When calling methods on a generic `T: Builder`, the compiler cannot assume `T::Output` is `Vec<u8>` — it could be overridden. You must use the associated type abstractly or add bounds:

```rust
fn process<T: Builder>(b: &T)
where
    T::Output: std::fmt::Debug,
{
    println!("{:?}", b.build());
}
```

### 2.3 Associated Type Bounds

You can constrain associated types directly in trait bounds:

```rust
// Long form:
fn print_items<I>(iter: I)
where
    I: Iterator,
    I::Item: std::fmt::Display,
{
    for item in iter {
        println!("{item}");
    }
}

// Short form with associated type bound (RFC 2289):
fn print_items_short<I: Iterator<Item: Display>>(iter: I) {
    for item in iter {
        println!("{item}");
    }
}
```

This syntax is especially useful with `dyn` and `impl Trait`:

```rust
fn make_displayable_iter() -> impl Iterator<Item: Display + Clone> {
    vec![1, 2, 3].into_iter()
}

fn take_iter(iter: &mut dyn Iterator<Item: Display>) {
    // Item is constrained but erased
}
```

You can nest these bounds arbitrarily:

```rust
fn deep_bound<I>()
where
    I: Iterator<Item: Iterator<Item: Display>>,
{
    // I is an iterator of iterators of displayable things
}
```

### 2.4 Where Clauses on Associated Types

Inside a trait definition, you can place bounds directly on associated types:

```rust
trait Graph {
    type Node: Clone + Debug;
    type Edge: Clone + Debug;

    fn nodes(&self) -> Vec<Self::Node>;
    fn edges(&self) -> Vec<Self::Edge>;
    fn neighbors(&self, node: &Self::Node) -> Vec<Self::Node>;
}
```

These bounds are **required** of every implementation. They're equivalent to:

```rust
trait Graph where Self::Node: Clone + Debug, Self::Edge: Clone + Debug {
    type Node;
    type Edge;
    // ...
}
```

You can also add bounds that relate associated types to each other:

```rust
trait Collection {
    type Item;
    type Iter<'a>: Iterator<Item = &'a Self::Item> where Self: 'a;

    fn iter(&self) -> Self::Iter<'_>;
    fn add(&mut self, item: Self::Item);
}
```

### 2.5 Associated Constants and Functions

Traits can also declare associated constants:

```rust
trait Dimensioned {
    const DIMENSIONS: usize;

    fn origin() -> Self;
}

struct Point2D { x: f64, y: f64 }

impl Dimensioned for Point2D {
    const DIMENSIONS: usize = 2;

    fn origin() -> Self {
        Point2D { x: 0.0, y: 0.0 }
    }
}

fn print_dimensions<T: Dimensioned>() {
    println!("Dimensions: {}", T::DIMENSIONS);
}
```

Associated constants can have defaults:

```rust
trait Configurable {
    const MAX_RETRIES: u32 = 3;
    const TIMEOUT_MS: u64 = 5000;
}
```

---

## 3. Generic Associated Types (GATs)

*Stabilized in Rust 1.65.*

Generic Associated Types allow associated types to have their own generic parameters (lifetimes or types). This unlocks patterns that were previously impossible.

### 3.1 The Lending Iterator Problem

Consider a standard `Iterator`: `next(&mut self) -> Option<Self::Item>`. The returned `Item` cannot borrow from `self` because the associated type `Item` cannot express a lifetime relationship with the `&mut self` borrow — it is fixed at impl time. This prevents writing iterators that yield references into their own internal state.:

```rust
// This is what we WANT, but standard Iterator can't express:
trait LendingIterator {
    type Item<'a> where Self: 'a;  // <-- GAT!
    
    fn next(&mut self) -> Option<Self::Item<'_>>;
}
```

Without GATs, `Item` is fixed at impl time and cannot vary with each call to `next`. With GATs, `Item<'a>` borrows from the iterator for lifetime `'a` (so it can return references into its own internal state). As an aside this iterators must be covariant in their `Item` type because the Item belongs to the parent collection.

### 3.2 GAT Syntax and Semantics

```rust
trait Container {
    type Item<'a> where Self: 'a;
    
    fn get(&self, index: usize) -> Self::Item<'_>;
}

struct VecWrapper(Vec<String>);

impl Container for VecWrapper {
    type Item<'a> = &'a str where Self: 'a;
    
    fn get(&self, index: usize) -> &str {
        &self.0[index]
    }
}
```

The `where Self: 'a` clause is crucial — it tells the compiler that the `Self` type must live at least as long as `'a`, which is required when borrowing from `self`.

### 3.3 GATs with Lifetime Parameters

The primary use case is lifetime-parameterized associated types:

```rust
trait StreamingIterator {
    type Item<'a> where Self: 'a;
    
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// A windows iterator that yields overlapping slices
struct Windows<'s, T> {
    data: &'s [T],
    pos: usize,
    size: usize,
}

impl<'s, T> StreamingIterator for Windows<'s, T> {
    type Item<'a> = &'a [T] where Self: 'a;
    
    fn next(&mut self) -> Option<&[T]> {
        if self.pos + self.size > self.data.len() {
            return None;
        }
        let window = &self.data[self.pos..self.pos + self.size];
        self.pos += 1;
        Some(window)
    }
}
```

### 3.4 GATs with Type Parameters

GATs can also be parameterized by types:

```rust
trait Functor {
    type Mapped<B>;
    
    fn map<B, F>(self, f: F) -> Self::Mapped<B>
    where
        F: FnMut(Self::Inner) -> B;
    
    type Inner;
}

// Option as a Functor
impl<A> Functor for Option<A> {
    type Inner = A;
    type Mapped<B> = Option<B>;
    
    fn map<B, F>(self, f: F) -> Option<B>
    where
        F: FnMut(A) -> B,
    {
        self.map(f)
    }
}

// Vec as a Functor
impl<A> Functor for Vec<A> {
    type Inner = A;
    type Mapped<B> = Vec<B>;
    
    fn map<B, F>(self, mut f: F) -> Vec<B>
    where
        F: FnMut(A) -> B,
    {
        self.into_iter().map(|a| f(a)).collect()
    }
}
```

This is the key to encoding higher-kinded type patterns in Rust!

### 3.5 GATs in Async Traits

GATs are fundamental to how async methods in traits work under the hood. When you write:

```rust
trait AsyncReader {
    async fn read(&mut self, buf: &mut [u8]) -> std::io::Result<usize>;
}
```

The compiler desugars this into something like:

```rust
trait AsyncReader {
    fn read<'a>(&'a mut self, buf: &'a mut [u8]) -> impl Future<Output = std::io::Result<usize>> + 'a;
}
```

Which conceptually involves a GAT for the return future that borrows from `self`.

### 3.6 Required Bounds on GATs

The compiler may require you to add bounds on GAT definitions. The most common is:

```rust
trait MyTrait {
    // The compiler requires `Self: 'a` when the GAT borrows from Self
    type Ref<'a>: Debug where Self: 'a;
}
```

Other common required bounds:

```rust
trait Fancy {
    // GAT that must be Send
    type Output<T>: Send where T: Send;
    
    // GAT with multiple lifetime params
    type Borrowed<'a, 'b>: Display where Self: 'a, 'b: 'a;
}
```

---

## 4. Existential Types: `impl Trait`

An **existential type** says "there exists some concrete type that implements this trait, but I won't name it." In Rust, this is spelled `impl Trait`.

### 4.1 Argument Position `impl Trait` (APIT)

```rust
fn print_it(val: impl Display) {
    println!("{val}");
}
```

This is **syntactic sugar** for a generic parameter:

```rust
fn print_it<T: Display>(val: T) {
    println!("{val}");
}
```

They are nearly identical, but there are subtle differences:

| Aspect | `impl Trait` (APIT) | Generic `<T: Trait>` |
|--------|---------------------|----------------------|
| Turbofish | Cannot use `::<Type>` syntax | Can specify with turbofish |
| Multiple params | Each `impl Trait` is an independent type | Can reuse `T` for same type |
| Inference | Type is inferred per-argument | Type can be tied across arguments |

```rust
// These two signatures have DIFFERENT semantics:
fn same_type<T: Display>(a: T, b: T) {}       // a and b must be same type
fn diff_types(a: impl Display, b: impl Display) {} // a and b can differ

same_type(1u32, 2u32);   // OK
// same_type(1u32, "hi"); // ERROR: u32 ≠ &str

diff_types(1u32, "hi");  // OK: each impl Trait is independent
```

### 4.2 Return Position `impl Trait` (RPIT)

```rust
fn make_iter() -> impl Iterator<Item = u32> {
    (0..10).filter(|x| x % 2 == 0)
}
```

The caller **cannot** name the concrete type — they can only use it through the `Iterator` interface. The compiler knows the concrete type and monomorphizes everything.

**Key properties:**

1. **Single concrete type per call site**: An `impl Trait` return must always return the **same** concrete type on all code paths:

```rust
fn choose(flag: bool) -> impl Display {
    if flag {
        42i32
    } else {
        // 42i32 — must be the same type
        0i32
    }
    // Cannot return "hello" on one branch and 42 on another!
}

// WRONG — different types on different branches:
// fn bad(flag: bool) -> impl Display {
//     if flag { 42i32 } else { "hello" }  // ERROR
// }
```

2. **The concrete type is opaque, but fixed**: The caller cannot downcast or introspect, but the compiler performs static dispatch.

3. **Auto traits leak through**: If the hidden type is `Send + Sync`, callers can use it in `Send`/`Sync` contexts. This is called **auto trait leakage**.

```rust
fn make_thing() -> impl Display {
    42i32  // i32 is Send + Sync, so the return is too
}

fn needs_send(val: impl Display + Send) {}

fn example() {
    needs_send(make_thing()); // OK: auto trait leakage
}
```

### 4.3 RPIT in Traits (RPITIT)

*Stabilized in Rust 1.75.*

You can now use `impl Trait` in trait method return positions:

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = &str>;
}

struct MyContainer {
    data: Vec<String>,
}

impl Container for MyContainer {
    fn items(&self) -> impl Iterator<Item = &str> {
        self.data.iter().map(|s| s.as_str())
    }
}
```

Under the hood, each implementation defines its own opaque return type. The compiler creates an anonymous associated type for each `impl Trait` return.

**Important caveat** — dyn compatibility: A trait using RPITIT is **not** object-safe by default (each implementor may return a different type):

```rust
trait Processor {
    fn process(&self) -> impl Display;
}

// This does NOT work:
// let processors: Vec<Box<dyn Processor>> = ...; // ERROR: not object-safe
```

To use with trait objects, you need to erase the return type explicitly:

```rust
trait Processor {
    fn process(&self) -> Box<dyn Display>;  // Now object-safe
}
```

### 4.4 Type Alias `impl Trait` (TAIT)

*Nightly feature: `type_alias_impl_trait`.*

TAIT allows naming an existential type at the module level:

```rust
#![feature(type_alias_impl_trait)]

type MyIter = impl Iterator<Item = u32>;

fn make_iter() -> MyIter {
    (0..10).filter(|x| x % 2 == 0)
}

fn use_iter() -> MyIter {
    // Must return the same concrete type as make_iter!
    (0..10).filter(|x| x % 2 == 0)
    // If the closure is different, the concrete type changes → ERROR
}
```

**Where TAIT shines:**

1. **Naming the type in structs**: Without TAIT, you cannot store `impl Trait` in a struct.

```rust
#![feature(type_alias_impl_trait)]

type MyFuture = impl Future<Output = u32>;

struct Worker {
    pending: MyFuture,   // Now we can store it!
}
```

2. **Recursive types and trait impls**: TAIT enables patterns where you need to refer to an existential type in multiple locations.

3. **Associated type positions**: Useful for satisfying associated type requirements:

```rust
#![feature(type_alias_impl_trait)]

type IterType<'a> = impl Iterator<Item = &'a str>;

trait HasIter {
    type Iter<'a>: Iterator<Item = &'a str> where Self: 'a;
    fn iter(&self) -> Self::Iter<'_>;
}

struct MyType(Vec<String>);

impl HasIter for MyType {
    type Iter<'a> = IterType<'a> where Self: 'a;
    // Alternatively, in post-RPITIT Rust:
    // type Iter<'a> = impl Iterator<Item = &'a str> where Self: 'a;
    fn iter(&self) -> Self::Iter<'_> {
        self.0.iter().map(|s| s.as_str())
    }
}
```

### 4.5 impl Trait in Let Bindings

*Nightly feature, partially.*

Conceptually, you might want to write:

```rust
let x: impl Display = 42;
```

This is not yet stable, but the semantics would parallel RPIT — the type is inferred by the compiler but opaque in the surrounding scope.

### 4.6 Capturing Rules and Lifetime Semantics

In RPIT, the opaque type **captures** lifetimes from the enclosing function. The rules changed significantly:

**Edition 2021 and earlier**: RPIT only captures lifetimes that appear in the `impl Trait` bounds. Input lifetimes are not captured by default:

```rust
// Edition 2021: 'a is NOT captured → return type does not borrow from input
fn foo<'a>(s: &'a str) -> impl Display {
    String::from(s)  // Must return owned data
}
```

**Edition 2024**: RPIT captures **all** in-scope lifetimes by default (including generic lifetime parameters). Use `+ use<...>` to explicitly control captures:

```rust
// Edition 2024: 'a IS captured by default
fn foo<'a>(s: &'a str) -> impl Display {
    s  // This now works — 'a is captured
}

// Opt out of capturing 'a:
fn bar<'a>(s: &'a str) -> impl Display + use<> {
    String::from(s)  // Explicitly capture nothing
}

// Capture only specific lifetimes:
fn baz<'a, 'b>(a: &'a str, b: &'b str) -> impl Display + use<'a> {
    a  // Only 'a is captured
}
```

---

## 5. Trait Objects and Dynamic Dispatch: `dyn Trait`

While `impl Trait` is resolved at compile time (static dispatch, monomorphization), `dyn Trait` provides **runtime polymorphism** via vtables.

### 5.1 Vtables and Fat Pointers

A `dyn Trait` value is a **fat pointer** consisting of:
1. A pointer to the data
2. A pointer to the vtable

```
┌──────────────────────┐
│  dyn Trait (fat ptr)  │
├──────────────────────┤
│  data_ptr ──────────────► [ actual data ]
│  vtable_ptr ────────────► ┌─────────────────┐
│                           │ drop()          │
│                           │ size            │
│                           │ align           │
│                           │ method_1()      │
│                           │ method_2()      │
│                           │ ...             │
│                           └─────────────────┘
└──────────────────────┘
```

The vtable contains:
- A **destructor** (`drop` function pointer)
- The **size** and **alignment** of the concrete type
- **Function pointers** for each trait method

```rust
trait Animal {
    fn speak(&self) -> &str;
    fn legs(&self) -> u32;
}

struct Dog;
impl Animal for Dog {
    fn speak(&self) -> &str { "woof" }
    fn legs(&self) -> u32 { 4 }
}

struct Spider;
impl Animal for Dog {
    fn speak(&self) -> &str { "..." }
    fn legs(&self) -> u32 { 8 }
}

// Dynamic dispatch:
let animals: Vec<Box<dyn Animal>> = vec![Box::new(Dog), Box::new(Spider)];
for a in &animals {
    println!("{}: {} legs", a.speak(), a.legs()); // vtable dispatch
}
```

**Performance implications:**
- Each method call goes through an indirect function pointer (vtable lookup).
- No inlining across the dynamic dispatch boundary (though Link-Time Optimization can sometimes help).
- The fat pointer is 2×usize (16 bytes on 64-bit).

### 5.2 Object Safety Rules

Not all traits can be made into `dyn Trait`. A trait is **object-safe** (officially: "dyn-compatible") if and only if:

1. **All methods have a receiver** (`self`, `&self`, `&mut self`, `self: Box<Self>`, `self: Pin<&mut Self>`, etc.)
   — no "bare" associated functions without `self`.

2. **No method returns `Self`** (the concrete type is erased; `Self` is unsized behind `dyn`).

3. **No method has generic type parameters** (the vtable cannot have infinite entries).

4. **The trait does not require `Self: Sized`**.

```rust
// Object-safe ✓
trait Draw {
    fn draw(&self);
}

// NOT object-safe ✗ — returns Self
trait Clone {
    fn clone(&self) -> Self;
}

// NOT object-safe ✗ — generic method
trait Serialize {
    fn serialize<W: std::io::Write>(&self, writer: &mut W);
}

// NOT object-safe ✗ — no receiver
trait Factory {
    fn create() -> Self;
}
```

**Workarounds** for non-object-safe methods:

```rust
trait MyTrait {
    // Object-safe method:
    fn normal(&self);

    // Non-object-safe method — exclude from vtable with `where Self: Sized`:
    fn clone_self(&self) -> Self
    where
        Self: Sized;

    // Generic method — exclude similarly:
    fn serialize<W: std::io::Write>(&self, w: &mut W)
    where
        Self: Sized;
    
    // Or erase the generic with dynamic dispatch:
    fn serialize_dyn(&self, w: &mut dyn std::io::Write);
}
```

Methods bounded by `where Self: Sized` are **excluded** from the vtable and cannot be called on `dyn MyTrait`, but they don't break object safety.

### 5.3 Multiple Trait Bounds in `dyn`

You can combine one non-auto trait with auto traits:

```rust
// OK: one "real" trait + auto traits
let x: Box<dyn Display + Send + Sync> = Box::new(42);

// ERROR: two non-auto traits
// let y: Box<dyn Display + Debug> = ...; // not allowed
```

As of Rust 1.75+ with trait upcasting, the story is improving, but the fundamental vtable limitation remains: `dyn` uses a single vtable pointer for one trait (plus auto traits which have no methods).

**Workaround** — supertrait approach:

```rust
trait DisplayDebug: Display + Debug {}
impl<T: Display + Debug> DisplayDebug for T {}

let x: Box<dyn DisplayDebug> = Box::new(42);
// Can call both Display and Debug methods through the supertrait
```

### 5.4 Lifetime Elision for `dyn Trait`

Trait objects have an implicit lifetime bound. The elision rules are:

```rust
// Behind a reference — inherits the reference lifetime:
fn foo(x: &dyn Display)           // means &'_ (dyn Display + '_)
fn bar(x: &'a dyn Display)        // means &'a (dyn Display + 'a)

// Behind Box or other owned pointer — defaults to 'static:
fn baz(x: Box<dyn Display>)       // means Box<dyn Display + 'static>

// In a struct — defaults to 'static if no lifetime parameter:
struct Foo {
    x: Box<dyn Display>,          // 'static
}

struct Bar<'a> {
    x: Box<dyn Display + 'a>,     // explicit non-'static
}
```

This catches many beginners off-guard:

```rust
fn make_display(s: &str) -> Box<dyn Display> {
    // ERROR: s doesn't live long enough — Box<dyn Display> implies 'static!
    // Box::new(s)
    Box::new(s.to_string())  // OK: String is 'static
}

fn make_display_borrowed<'a>(s: &'a str) -> Box<dyn Display + 'a> {
    Box::new(s)  // OK: lifetime is explicit
}
```

### 5.5 Upcasting and Trait Object Coercions

*Trait upcasting stabilized in Rust 1.76.*

If `SubTrait: SuperTrait`, you can upcast:

```rust
trait Animal {
    fn name(&self) -> &str;
}

trait Dog: Animal {
    fn bark(&self);
}

struct Labrador;
impl Animal for Labrador {
    fn name(&self) -> &str { "Buddy" }
}
impl Dog for Labrador {
    fn bark(&self) { println!("Woof!"); }
}

// Upcasting: dyn Dog → dyn Animal
let dog: Box<dyn Dog> = Box::new(Labrador);
let animal: Box<dyn Animal> = dog;  // upcast!
println!("{}", animal.name());
```

Before upcasting stabilization, you had to add an explicit `as_animal()` method or use unsafe code. Now the compiler generates the necessary vtable adjustment automatically.

### 5.6 `dyn*` — Experimental Inline Trait Objects

*Nightly feature: `dyn_star`.*

`dyn* Trait` stores the value **inline** (up to pointer-size) instead of behind an indirection:

```rust
#![feature(dyn_star)]

fn use_display(x: dyn* Display) {
    println!("{x}");
}

use_display(42u32);  // No heap allocation — u32 fits inline
```

This is primarily motivated by async traits where futures need to be trait objects but many are small enough to inline.

---

## 6. Generic Traits (Traits Parameterized by Types)

### 6.1 Defining Generic Traits

A trait can have type parameters, allowing multiple implementations for different type arguments:

```rust
trait From<T> {
    fn from(val: T) -> Self;
}

// A single type can implement From for many source types:
impl From<i32> for String {
    fn from(val: i32) -> String { val.to_string() }
}

impl From<bool> for String {
    fn from(val: bool) -> String { val.to_string() }
}
```

The generic parameter is an **input** — the caller (or inference) decides which `T` is used.

### 6.2 Multiple Implementations via Generic Params

This is the key distinction from associated types: one type can implement a generic trait **multiple times**:

```rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

// String implements AsRef for multiple target types:
// impl AsRef<str> for String { ... }
// impl AsRef<[u8]> for String { ... }
// impl AsRef<Path> for String { ... }
```

This fan-out pattern is impossible with associated types (which fix the output to a single type per impl).

### 6.3 Generic Traits vs Associated Types — Decision Framework

| Question | Generic Param | Associated Type |
|----------|--------------|-----------------|
| Can a type have multiple impls? | Yes | No (one impl per type) |
| Who chooses the type? | Caller / inference | Implementor |
| Is the type an "input" or "output"? | Input | Output |
| Does ambiguity arise? | More often (need turbofish) | Rarely |
| Better for conversion traits? | Yes (`From<T>`, `Into<T>`) | No |
| Better for collection iteration? | No | Yes (`Iterator::Item`) |

**Hybrid approach** — both at once:

```rust
trait Handler<Request> {           // Generic: one handler can handle many request types
    type Response;                 // Associated: response is determined by impl
    type Error;                    // Associated: error is determined by impl

    fn handle(&self, req: Request) -> Result<Self::Response, Self::Error>;
}
```

### 6.4 Operator Overloading as Generic Traits

Rust's operators are defined as generic traits:

```rust
// std::ops::Add is generic over the RHS:
pub trait Add<Rhs = Self> {   // default type parameter!
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```

This enables a type to define addition with multiple other types:

```rust
use std::ops::Add;

#[derive(Clone, Copy)]
struct Meters(f64);

#[derive(Clone, Copy)]
struct Centimeters(f64);

impl Add for Meters {  // Meters + Meters
    type Output = Meters;
    fn add(self, rhs: Meters) -> Meters {
        Meters(self.0 + rhs.0)
    }
}

impl Add<Centimeters> for Meters {  // Meters + Centimeters
    type Output = Meters;
    fn add(self, rhs: Centimeters) -> Meters {
        Meters(self.0 + rhs.0 / 100.0)
    }
}
```

Notice: `Rhs = Self` is a **default type parameter**. If you write `impl Add for Meters`, the `Rhs` defaults to `Meters`. Default type parameters are extensively used in the standard library for ergonomics.

---

## 7. Higher-Ranked Trait Bounds (HRTBs)

### 7.1 The `for<'a>` Syntax

Sometimes you need a bound that says "this works for **any** lifetime, not just some specific one." This is what `for<'a>` expresses:

```rust
fn apply<F>(f: F)
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let owned = String::from("hello");
    let result = f(&owned);
    println!("{result}");
}
```

The bound `for<'a> Fn(&'a str) -> &'a str` means: "for **every possible** lifetime `'a`, `F` implements `Fn(&'a str) -> &'a str`."

Without `for<'a>`, you'd need a specific lifetime, which limits usage:

```rust
// This is MORE restrictive — ties 'a to the caller's choice:
fn apply_limited<'a, F: Fn(&'a str) -> &'a str>(f: F, input: &'a str) -> &'a str {
    f(input)
}
```

### 7.2 HRTBs with Closures

Closures that take references almost always need HRTBs. Rust inserts them automatically in most cases:

```rust
fn takes_closure(f: impl Fn(&str) -> usize) {
    // Desugars to: impl for<'a> Fn(&'a str) -> usize
    println!("{}", f("hello"));
}

takes_closure(|s| s.len());
```

You need explicit `for<'a>` syntax when:

1. The return type depends on the input lifetime:

```rust
fn takes_closure<F>(f: F)
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let s = String::from("hello world");
    println!("{}", f(&s));
}

takes_closure(|s| &s[..5]);
```

2. Building complex trait bounds that Rust's sugar can't express:

```rust
fn complex<F>(f: F)
where
    F: for<'a, 'b> Fn(&'a str, &'b str) -> &'a str,
{
    let a = String::from("hello");
    let b = String::from("world");
    println!("{}", f(&a, &b));
}
```

### 7.3 HRTBs and GATs

HRTBs are particularly important with GATs:

```rust
trait LendingIterator {
    type Item<'a> where Self: 'a;
    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// Process any lending iterator whose items are always Debug:
fn process<I>(mut iter: I)
where
    I: LendingIterator,
    for<'a> I::Item<'a>: std::fmt::Debug,
{
    while let Some(item) = iter.next() {
        println!("{:?}", item);
    }
}
```

### 7.4 Higher-Ranked Types (Aspirational)

Rust does not yet support higher-ranked types beyond lifetimes:

```rust
// NOT valid Rust — hypothetical higher-ranked type:
// fn foo<F: for<T> Fn(T) -> T>(f: F) { ... }
```

`for<'a>` only works with lifetimes. Fully general higher-ranked polymorphism (System F style) is not part of Rust's type system, though GATs and `impl Trait` provide many of the same capabilities.

---

## 8. Supertraits and Trait Inheritance

### 8.1 Supertrait Syntax and Semantics

A trait can **require** other traits as prerequisites:

```rust
trait Debuggable: Debug {
    fn debug_print(&self) {
        println!("{:?}", self);
    }
}

// Any type implementing Debuggable must also implement Debug.
// The compiler automatically knows that Self: Debug inside Debuggable methods.
```

Multiple supertraits:

```rust
trait Storable: Serialize + DeserializeOwned + Clone + Send + Sync + 'static {
    fn key(&self) -> String;
}
```

Supertraits are **not inheritance** in the OOP sense. They're bounds: "to implement `Storable`, you must also implement all of these traits." There's no virtual dispatch chain or method overriding across the supertrait hierarchy.

### 8.2 Diamond Inheritance and Ambiguity

```rust
trait A { fn method(&self) -> &str; }
trait B: A { }
trait C: A { }
trait D: B + C { }  // "Diamond" — D requires A via both B and C

// No ambiguity! There's only ONE impl of A for any type.
struct Concrete;
impl A for Concrete { fn method(&self) -> &str { "A" } }
impl B for Concrete { }
impl C for Concrete { }
impl D for Concrete { }

let d: &dyn D = &Concrete;
d.method(); // Unambiguous: calls A::method
```

Rust avoids the diamond problem because trait implementations are flat — there's no inheritance of method bodies across supertraits (only bounds).

### 8.3 Sealed Traits

A sealed trait cannot be implemented outside its defining crate:

```rust
// In your crate:
mod private {
    pub trait Sealed {}
}

pub trait MyPublicTrait: private::Sealed {
    fn do_thing(&self);
}

// External crates can USE MyPublicTrait but not IMPLEMENT it,
// because they can't access private::Sealed.

// Internal implementations:
impl private::Sealed for String {}
impl MyPublicTrait for String {
    fn do_thing(&self) { println!("{self}"); }
}
```

**Why seal a trait?**
- You can add methods to the trait without breaking downstream code.
- You can rely on a closed set of implementations for exhaustive matching.
- Used extensively in `std` (e.g., the `Pattern` trait, `Termination` trait).

---

## 9. Marker Traits and Auto Traits

### 9.1 Send, Sync, Unpin, Sized

Marker traits have no methods — they purely classify types:

| Trait | Meaning |
|-------|---------|
| `Send` | Can be transferred to another thread |
| `Sync` | Can be shared between threads (`&T` is `Send`) |
| `Unpin` | Can be moved after being pinned |
| `Sized` | Has a known size at compile time |

`Send` and `Sync` are **auto traits** — the compiler implements them automatically based on a type's fields:

```rust
struct Foo {
    a: String,  // Send + Sync
    b: Vec<u8>, // Send + Sync
}
// Foo is automatically Send + Sync

struct Bar {
    a: Rc<u32>, // NOT Send, NOT Sync
}
// Bar is NOT Send, NOT Sync (Rc prevents it)
```

### 9.2 Negative Implementations

You can explicitly opt OUT of an auto trait:

```rust
// In std:
impl<T: ?Sized> !Send for Rc<T> {}
impl<T: ?Sized> !Sync for Rc<T> {}

// Custom type that is deliberately !Send:
struct NotSend {
    _marker: PhantomData<*const ()>,  // *const T is !Send
}
```

Using `PhantomData<*const ()>` is the idiomatic way to make a type `!Send + !Sync` on stable Rust (explicit negative impls require nightly).

### 9.3 Auto Trait Leakage

Auto trait bounds "leak" through `impl Trait` returns:

```rust
fn make_thing() -> impl Debug {
    vec![1, 2, 3]  // Vec<i32> is Send + Sync
}

// Callers can rely on auto traits even though they aren't declared:
fn use_thing() {
    let thing = make_thing();
    std::thread::spawn(move || {
        println!("{thing:?}");  // Works! Auto trait leakage says it's Send
    });
}
```

**Danger**: If you change the hidden type to something non-Send, downstream code breaks:

```rust
fn make_thing() -> impl Debug {
    Rc::new(42)  // RC is NOT Send — this breaks the spawn() above!
}
```

This is a form of **semver hazard**. To prevent leakage, explicitly bound:

```rust
fn make_thing() -> impl Debug + Send + Sync {
    // Now it's part of the API contract
    vec![1, 2, 3]
}
```

### 9.4 Custom Marker Traits

You can define marker traits for type-level tagging:

```rust
/// Marker: types that are safe to cache indefinitely.
trait Cacheable: Send + Sync + 'static {}

/// Marker: types representing valid database entities.
trait Entity: Sized {
    const TABLE_NAME: &'static str;
}

struct User { id: u64, name: String }

impl Cacheable for User {}
impl Entity for User {
    const TABLE_NAME: &'static str = "users";
}
```

A powerful pattern: **typestate** using marker traits:

```rust
trait State {}
struct Draft;
struct Published;
impl State for Draft {}
impl State for Published {}

struct Article<S: State> {
    title: String,
    body: String,
    _state: PhantomData<S>,
}

impl Article<Draft> {
    fn publish(self) -> Article<Published> {
        Article {
            title: self.title,
            body: self.body,
            _state: PhantomData,
        }
    }
}

impl Article<Published> {
    fn url(&self) -> String {
        format!("/articles/{}", self.title.to_lowercase().replace(' ', "-"))
    }
    // Cannot call publish() — not defined for Published state
}
```

---

## 10. Coherence, Orphan Rules, and Overlap

### 10.1 The Orphan Rule

Rust's coherence rules prevent conflicting trait implementations:

> You can implement trait `T` for type `S` only if **at least one of `T` or `S` is defined in your crate**.

```rust
// In YOUR crate:

// ✓ Your trait, foreign type:
trait MyTrait {}
impl MyTrait for String {}

// ✓ Foreign trait, your type:
struct MyType;
impl Display for MyType { /* ... */ }

// ✗ Foreign trait, foreign type:
// impl Display for Vec<u8> { }  // ERROR: orphan rule violation
```

**Covered types exception**: You can implement a foreign trait for a foreign type if your local type "covers" a type parameter:

```rust
struct Wrapper<T>(T);

// OK: Wrapper is local, even though Display and Vec are foreign
impl<T: Display> Display for Wrapper<Vec<T>> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Wrapper({:?})", self.0)
    }
}
```

### 10.2 The Newtype Pattern

The standard workaround for the orphan rule:

```rust
// Want to impl Display for Vec<String>? Wrap it:
struct PrettyVec(Vec<String>);

impl Display for PrettyVec {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        for (i, s) in self.0.iter().enumerate() {
            if i > 0 { write!(f, ", ")?; }
            write!(f, "{s}")?;
        }
        Ok(())
    }
}
```

To transparently access inner methods, implement `Deref`:

```rust
use std::ops::Deref;

impl Deref for PrettyVec {
    type Target = Vec<String>;
    fn deref(&self) -> &Vec<String> {
        &self.0
    }
}

let pv = PrettyVec(vec!["a".into(), "b".into()]);
println!("Length: {}", pv.len());  // Deref to Vec
println!("Display: {pv}");         // Our Display impl
```

### 10.3 Blanket Implementations

A blanket impl implements a trait for all types satisfying a bound:

```rust
// From std — every Display type gets ToString for free:
impl<T: Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{self}")
    }
}

// Every &T where T: Trait also implements Trait (common pattern):
impl<T: Read + ?Sized> Read for &mut T {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        (**self).read(buf)
    }
}
```

**Blanket impl hazards**: Once a blanket impl exists, it prevents more specific implementations due to potential overlap:

```rust
trait MyTrait {
    fn greet(&self) -> &str;
}

// Blanket impl:
impl<T: Display> MyTrait for T {
    fn greet(&self) -> &str { "hello from Display" }
}

// Now you CANNOT do:
// impl MyTrait for String { ... }  // ERROR: conflicts with blanket impl
// (String: Display, so it's covered by the blanket)
```

### 10.4 Overlap and Specialization

*Nightly feature: `specialization` / `min_specialization`.*

Specialization allows a more specific impl to override a blanket impl:

```rust
#![feature(specialization)]

trait FastHash {
    fn fast_hash(&self) -> u64;
}

// Default blanket impl:
impl<T: Hash> FastHash for T {
    default fn fast_hash(&self) -> u64 {
        let mut hasher = DefaultHasher::new();
        self.hash(&mut hasher);
        hasher.finish()
    }
}

// Specialized impl for u64 — override the blanket:
impl FastHash for u64 {
    fn fast_hash(&self) -> u64 {
        *self  // Identity hash for u64
    }
}
```

**Why `default`?** The `default` keyword marks a method as overridable by more specialized impls. Without specialization, Rust conservatively rejects all overlapping impls.

**`min_specialization`** is a more conservative version used internally by `std`, allowing specialization only in limited, sound cases. Full specialization has unresolved soundness issues (interaction with lifetimes).

---

## 11. Trait Bound Patterns and Tricks

### 11.1 Conditional Method Availability

Methods can be available only when certain trait bounds are satisfied:

```rust
struct Wrapper<T>(T);

impl<T> Wrapper<T> {
    fn new(val: T) -> Self {
        Wrapper(val)
    }
}

// These methods only exist when T: Display
impl<T: Display> Wrapper<T> {
    fn print(&self) {
        println!("{}", self.0);
    }
}

// These methods only exist when T: Default + Clone
impl<T: Default + Clone> Wrapper<T> {
    fn reset(&mut self) {
        self.0 = T::default();
    }

    fn duplicate(&self) -> Self {
        Wrapper(self.0.clone())
    }
}
```

This is how `Vec` provides `sort()` only when `T: Ord`, or `join()` only when the elements are strings.

### 11.2 Extension Traits

Add methods to foreign types through a trait:

```rust
trait StringExt {
    fn truncate_ellipsis(&self, max_len: usize) -> String;
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn truncate_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len.saturating_sub(3)])
        }
    }

    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}

// Usage:
let s = "Hello, World!";
println!("{}", s.truncate_ellipsis(8));  // "Hello..."
println!("{}", "   ".is_blank());        // true
```

Convention: name extension traits `ThingExt` (e.g., `FutureExt`, `StreamExt`, `IteratorExt`). The `futures` and `tokio` crates use this pattern extensively.

### 11.3 Trait Aliases

*Nightly feature: `trait_alias`.*

Define shorthand for complex trait bound combinations:

```rust
#![feature(trait_alias)]

trait Storable = Serialize + DeserializeOwned + Clone + Send + Sync + 'static;

fn store<T: Storable>(val: &T) {
    // ...
}
```

**Stable workaround** using blanket impls:

```rust
trait Storable: Serialize + DeserializeOwned + Clone + Send + Sync + 'static {}
impl<T> Storable for T
where
    T: Serialize + DeserializeOwned + Clone + Send + Sync + 'static,
{}
```

Or using a type alias with a where clause (less common):

```rust
// Can't directly alias traits on stable, but can use this in signatures:
fn store<T>(val: &T)
where
    T: Serialize + DeserializeOwned + Clone + Send + Sync + 'static,
{
    // ...
}
```

### 11.4 The Turbofish and Fully Qualified Syntax

When methods from multiple traits conflict, use **fully qualified syntax** (also called Universal Function Call Syntax, UFCS):

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) { println!("Captain speaking"); }
}

impl Wizard for Human {
    fn fly(&self) { println!("Expelliarmus!"); }
}

impl Human {
    fn fly(&self) { println!("*flaps arms*"); }
}

let person = Human;

// Ambiguous — calls the inherent method:
person.fly();               // "*flaps arms*"

// Fully qualified — disambiguate:
Pilot::fly(&person);        // "Captain speaking"
Wizard::fly(&person);       // "Expelliarmus!"

// Fully fully qualified (with angle brackets):
<Human as Pilot>::fly(&person);
<Human as Wizard>::fly(&person);
```

For associated functions (no `self`):

```rust
trait Animal {
    fn name() -> String;
}

struct Dog;

impl Animal for Dog {
    fn name() -> String { "dog".into() }
}

impl Dog {
    fn name() -> String { "puppy".into() }
}

// Must use fully qualified syntax — no self to dispatch on:
let n1 = <Dog as Animal>::name(); // "dog"
let n2 = Dog::name();             // "puppy"
```

### 11.5 Implied Bounds

Rust infers some bounds from the structure of types. This reduces boilerplate:

```rust
// You DON'T need to write `T: Sized` — it's implied:
fn foo<T>(val: T) { }  // T: Sized is implicit

// To opt out:
fn bar<T: ?Sized>(val: &T) { }  // T may be unsized

// Similarly, lifetime bounds from struct definitions propagate:
struct Ref<'a, T: 'a>(&'a T);

// In functions using Ref, you don't need to re-state T: 'a:
fn process<'a, T>(r: Ref<'a, T>) {  // T: 'a is implied by Ref's definition
    // ...
}
```

---

## 12. Async and Traits

### 12.1 Async fn in Traits

*Stabilized in Rust 1.75.*

You can now write async methods directly in traits:

```rust
trait AsyncDatabase {
    async fn get(&self, key: &str) -> Option<Vec<u8>>;
    async fn put(&self, key: &str, value: Vec<u8>) -> Result<(), Error>;
}

struct RedisDb { /* ... */ }

impl AsyncDatabase for RedisDb {
    async fn get(&self, key: &str) -> Option<Vec<u8>> {
        // ... await redis operations ...
        Some(vec![1, 2, 3])
    }

    async fn put(&self, key: &str, value: Vec<u8>) -> Result<(), Error> {
        // ... await redis operations ...
        Ok(())
    }
}
```

Under the hood, each `async fn` in a trait desugars to a method returning an opaque `impl Future`. Each implementation generates its own future type.

### 12.2 Send Bounds on Async Trait Methods

The hidden future type might not be `Send`. If you need to spawn the future on a multi-threaded runtime:

```rust
// The returned future is not guaranteed Send:
trait MyService {
    async fn call(&self) -> Response;
}

// To require Send, use the desugared form:
trait MyServiceSend {
    fn call(&self) -> impl Future<Output = Response> + Send;
}

// Or with the return_type_notation (experimental):
// fn use_service<T: MyService<call(..): Send>>(svc: T) { ... }
```

If you're writing library code for general consumption, consider whether callers need `Send` futures. The `async-trait` crate was the pre-stabilization solution, boxing futures into `Pin<Box<dyn Future + Send>>`.

### 12.3 Async Closures and Fn Traits

*Stabilized in Rust 1.85.*

Rust now supports `async` closures directly:

```rust
let fetch = async |url: &str| -> String {
    // ... perform async fetch ...
    format!("response from {url}")
};
```

Async closures implement the `AsyncFn`, `AsyncFnMut`, and `AsyncFnOnce` traits (analogous to `Fn`, `FnMut`, `FnOnce`):

```rust
async fn apply<F>(f: F) -> String
where
    F: AsyncFn(&str) -> String,
{
    f("https://example.com").await
}
```

Before async closures, the standard workaround was a regular closure returning a future:

```rust
fn apply<F, Fut>(f: F) -> impl Future<Output = String>
where
    F: Fn(&str) -> Fut,
    Fut: Future<Output = String>,
{
    f("https://example.com")
}
```

The problem with this approach was that the closure couldn't borrow across `.await` points, and the returned future couldn't borrow from the closure's captures. Async closures solve this.

---

## 13. The Sized, ?Sized, and Unsized Universe

`Sized` is the implicit bound on all type parameters. Understanding when and how to relax it is crucial for advanced generic programming.

### Sized as Default

```rust
fn foo<T>(val: T) { }
// Equivalent to:
fn foo<T: Sized>(val: T) { }
```

### Opting out with ?Sized

```rust
// Accept unsized types (str, [T], dyn Trait):
fn describe<T: Display + ?Sized>(val: &T) {
    println!("{val}");
}

describe("hello");               // T = str (unsized)
describe(&42);                    // T = i32 (sized)
describe(&*Box::new(42) as &dyn Display); // T = dyn Display (unsized)
```

### When ?Sized Matters

**Trait definitions** — if your trait should be implementable for `dyn` types:

```rust
// Without ?Sized, this trait cannot be used as `dyn MyTrait`:
trait MyTrait: Sized {
    fn method(&self);
}
// let x: &dyn MyTrait = ...;  // ERROR: Sized is not object-safe!

// With ?Sized (default for traits — Self is implicitly ?Sized):
trait MyTrait {
    fn method(&self);
}
// let x: &dyn MyTrait = ...;  // OK
```

Note: `Self` in a trait is `?Sized` by default. If you add `: Sized` as a supertrait, it prevents the trait from being object-safe.

### Unsized Coercions

```rust
let boxed: Box<[i32; 3]> = Box::new([1, 2, 3]);
let unsized_box: Box<[i32]> = boxed;  // Coercion: sized → unsized

let boxed_fn: Box<dyn Fn()> = Box::new(|| println!("hi"));
// Coercion: concrete closure type → dyn Fn()
```

### The Unsize Trait and CoerceUnsized

Behind the scenes, unsized coercions are governed by unstable traits:

```rust
// In std (simplified):
trait Unsize<T: ?Sized> { }
// Implemented automatically: [i32; 3]: Unsize<[i32]>, etc.

trait CoerceUnsized<T: ?Sized> { }
// Box<[i32; 3]>: CoerceUnsized<Box<[i32]>>, etc.
```

These enable smart pointers like `Arc`, `Rc`, `Box` to support unsized coercions.

---

## 14. Summary and Mental Model

### The Trait System at a Glance

```
                        ┌──────────────────────────────────┐
                        │         TRAIT DEFINITION          │
                        │  trait Foo<GenericParam> {        │
                        │      type Assoc;                 │
                        │      type Gat<'a>;               │
                        │      const C: usize;             │
                        │      fn method(&self);           │
                        │      async fn async_m(&self);    │
                        │  }                               │
                        └──────────┬───────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
             ┌────────────┐ ┌──────────┐ ┌──────────────┐
             │   Static   │ │ Dynamic  │ │  Existential │
             │  Dispatch  │ │ Dispatch │ │    Types     │
             │ T: Trait   │ │dyn Trait │ │ impl Trait   │
             │ monomorph. │ │  vtable  │ │ opaque type  │
             └────────────┘ └──────────┘ └──────────────┘
```

### Key Decision Points

| I need... | Use... |
|-----------|--------|
| One impl per type (output type) | Associated type |
| Many impls per type (input type) | Generic parameter |
| Type family (varies by lifetime/type) | GAT |
| Caller doesn't care about concrete type | `impl Trait` (RPIT) |
| Runtime polymorphism / heterogeneous collection | `dyn Trait` |
| Bound that must hold for all lifetimes | `for<'a>` HRTB |
| Prevent external impls | Sealed trait |
| Zero-method classification | Marker trait |
| Add methods to foreign types | Extension trait |
| Override blanket impls | Specialization (nightly) |
| Name an `impl Trait` type | TAIT (nightly) |

### The Spectrum from Static to Dynamic

```
More static                                              More dynamic
◄────────────────────────────────────────────────────────────────────►
  Generics       impl Trait       enum dispatch      dyn Trait     Any
  (monomorph)    (opaque type)    (match arms)       (vtable)    (TypeId)
  
  Zero cost      Zero cost        Branch cost        Indirect     Box +
  per call       per call         per dispatch       call cost    downcast
```

### Trait Coherence Mental Model

```
Can I implement Trait for Type?

1. Is Trait or Type local to my crate?
   NO  → ✗ Orphan rule violation
   YES → Continue

2. Does an existing impl overlap?
   YES, same specificity → ✗ Conflicting impls
   YES, more specific    → Only with specialization (nightly)
   NO  → ✓ OK

3. Does a blanket impl cover this case?
   YES → ✗ Cannot add a more specific impl (without specialization)
   NO  → ✓ OK
```

### Feature Stability Cheat Sheet (as of Rust 1.85)

| Feature | Status |
|---------|--------|
| Associated types | Stable |
| Associated type defaults | Stable |
| Associated type bounds (`Item: Debug`) | Stable |
| GATs | Stable (1.65) |
| `impl Trait` in arg position (APIT) | Stable |
| `impl Trait` in return position (RPIT) | Stable |
| RPITIT (impl Trait in trait returns) | Stable (1.75) |
| Async fn in traits | Stable (1.75) |
| Trait upcasting | Stable (1.76) |
| Async closures (`AsyncFn` traits) | Stable (1.85) |
| TAIT (type_alias_impl_trait) | Nightly |
| Trait aliases | Nightly |
| Specialization | Nightly (soundness issues) |
| Negative impls | Nightly |
| `dyn*` | Nightly |
| `impl Trait` in let bindings | Nightly / Partial |

---

## Appendix A: Complete Example — A Plugin System

Bringing many concepts together in a realistic example:

```rust
use std::any::Any;
use std::collections::HashMap;
use std::fmt::Debug;

// --- Sealed trait for plugin categories ---
mod private {
    pub trait Sealed {}
    impl Sealed for super::InputPlugin {}
    impl Sealed for super::OutputPlugin {}
}

// --- Marker types for plugin categories ---
struct InputPlugin;
struct OutputPlugin;

// --- Core trait with GATs, associated types, and supertraits ---
trait Plugin: Debug + Send + Sync + 'static {
    /// Name of the plugin, used for registration.
    const NAME: &'static str;

    /// Configuration type — each plugin defines its own.
    type Config: Debug + Clone + Send + 'static;

    /// Error type — each plugin defines its own.
    type Error: std::error::Error + Send + 'static;

    /// Create from configuration.
    fn from_config(config: Self::Config) -> Result<Self, Self::Error>
    where
        Self: Sized;

    /// Process a unit of work.
    fn process(&self, input: &[u8]) -> Result<Vec<u8>, Self::Error>;

    /// Health check — default implementation.
    fn health_check(&self) -> bool {
        true
    }
}

// --- Extension trait for Display-capable plugins ---
trait PluginExt: Plugin {
    fn describe(&self) -> String {
        format!("Plugin '{}': {:?}", Self::NAME, self)
    }
}

// Blanket impl: every Plugin gets PluginExt for free.
impl<T: Plugin> PluginExt for T {}

// --- Dynamic dispatch wrapper ---
// Since Plugin has associated types and Sized requirements,
// we create an object-safe wrapper:
trait DynPlugin: Debug + Send + Sync {
    fn name(&self) -> &str;
    fn process(&self, input: &[u8]) -> Result<Vec<u8>, Box<dyn std::error::Error + Send>>;
    fn health_check(&self) -> bool;
    fn as_any(&self) -> &dyn Any;
}

impl<T: Plugin> DynPlugin for T {
    fn name(&self) -> &str {
        T::NAME
    }

    fn process(&self, input: &[u8]) -> Result<Vec<u8>, Box<dyn std::error::Error + Send>> {
        Plugin::process(self, input).map_err(|e| Box::new(e) as _)
    }

    fn health_check(&self) -> bool {
        Plugin::health_check(self)
    }

    fn as_any(&self) -> &dyn Any {
        self
    }
}

// --- Registry using trait objects ---
struct PluginRegistry {
    plugins: HashMap<String, Box<dyn DynPlugin>>,
}

impl PluginRegistry {
    fn new() -> Self {
        Self { plugins: HashMap::new() }
    }

    fn register<P: Plugin>(&mut self, plugin: P) {
        self.plugins.insert(P::NAME.to_string(), Box::new(plugin));
    }

    fn get(&self, name: &str) -> Option<&dyn DynPlugin> {
        self.plugins.get(name).map(|b| b.as_ref())
    }

    /// Downcast back to concrete type if needed:
    fn get_concrete<P: Plugin>(&self) -> Option<&P> {
        self.plugins
            .get(P::NAME)
            .and_then(|b| b.as_any().downcast_ref::<P>())
    }

    fn health_check_all(&self) -> bool {
        self.plugins.values().all(|p| p.health_check())
    }
}
```

This example demonstrates:
- **Associated types** (`Config`, `Error`)
- **Associated constants** (`NAME`)
- **Supertraits** (`Debug + Send + Sync + 'static`)
- **Blanket implementations** (`PluginExt for all Plugin`)
- **Extension traits** (`PluginExt`)
- **Sealed traits** (the `private::Sealed` module)
- **Object safety workaround** (`DynPlugin` wraps the non-object-safe `Plugin`)
- **Trait objects** (`Box<dyn DynPlugin>`)
- **Downcasting** (`as_any().downcast_ref()`)
- **Default methods** (`health_check`)

---

## Appendix B: Trait Resolution — How the Compiler Picks an Impl

When you write `x.method()`, the compiler:

1. **Collects candidates**: inherent methods, trait methods in scope.
2. **Auto-ref/auto-deref**: tries `self`, `&self`, `&mut self`, then derefs and repeats.
3. **Filters by bounds**: checks which candidates satisfy all trait bounds.
4. **Disambiguation**: if multiple candidates remain, it's an error (use UFCS).

For generic code `T: Trait`, the compiler uses the **trait solver**:
- Checks whether there's a unique `impl Trait for ConcreteType`.
- For blanket impls, partially unifies type parameters.
- With where clauses, assumes the bound holds and defers resolution.

The new trait solver (Chalk-inspired, shipping incrementally starting in Rust 2024 Edition as `-Znext-solver`) handles more complex cases including:
- Better GAT support
- Coinductive trait resolution (important for auto traits)
- Negative reasoning
- Better error messages for complex bounds

---

*This document covers Rust traits as of stable 1.85 and nightly features current as of early 2026. For the latest on experimental features, check the [Rust RFC repository](https://github.com/rust-lang/rfcs) and [tracking issues](https://github.com/rust-lang/rust/labels/C-tracking-issue).*
