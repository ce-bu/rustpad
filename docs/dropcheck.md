# Drop Check in Rust: Generics, Lifetimes & the Eyepatch

**Estimated reading time: ~15 minutes**

## The Problem Drop Check Solves

When a value is dropped, its `Drop::drop` implementation runs. If a struct holds references or generic types, the compiler must guarantee that any data accessible through those types is **still valid** when `drop` runs. This analysis is called **drop check**.

```rust
struct Inspector<'a>(&'a u32);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        // We can read self.0 here — the referent MUST still be alive.
        println!("inspecting: {}", self.0);
    }
}
```

Without drop check, a dangling reference could be read inside `drop`.

---

## The Core Rule

> **Drop check rule (simplified):**  
> For a value of type `T` that implements `Drop`, the compiler requires that **all lifetimes/types reachable through `T` must strictly outlive `T` itself**.

"Strictly outlive" means the referent must be alive *at least one scope longer* than the value being dropped. This is more conservative than the usual borrowing rules.

---

## Generic Structs and Drop Check

Consider a generic struct with a `Drop` impl:

```rust
struct Wrapper<T> {
    value: T,
}

impl<T> Drop for Wrapper<T> {
    fn drop(&mut self) {
        // Could potentially access self.value here,
        // so T must be valid when drop runs.
        println!("dropping wrapper");
    }
}
```

### What the compiler assumes

Because `Wrapper<T>` implements `Drop`, the compiler conservatively assumes `drop` **might access `T`**. Therefore:

- If `T = &'a u32`, the lifetime `'a` must strictly outlive the `Wrapper`.
- If `T = Vec<&'a str>`, the lifetime `'a` embedded inside `Vec` must strictly outlive the `Wrapper`.

```rust
fn main() {
    let x = 42;
    let w = Wrapper { value: &x }; // OK: x outlives w (x dropped after w)

    // This would fail:
    // let w;
    // let x = 42;
    // w = Wrapper { value: &x }; // ERROR: x does NOT outlive w
}
```

### Without a `Drop` impl — no extra constraint

If `Wrapper<T>` does **not** implement `Drop`, the compiler does not add the strict-outlive requirement from drop check. The normal borrow-checking rules still apply, but they are less restrictive:

```rust
struct NoDrop<T> {
    value: T,
}

// No Drop impl — so no drop-check requirement on T.

fn main() {
    let w;
    let x = 42;
    w = NoDrop { value: &x };
    // This can be fine because there is no Drop impl that could
    // observe the reference after x is destroyed.
}
```

---

## Variance and Drop Check

Variance determines how subtyping of a generic parameter relates to subtyping of the containing type.

| Variance      | Meaning for `Struct<'a>` |
|---------------|--------------------------|
| **Covariant** | If `'long: 'short`, then `Struct<'long>: Struct<'short>` — the lifetime can shrink. |
| **Contravariant** | The lifetime can grow (rare, mainly in function argument position). |
| **Invariant** | The lifetime cannot change at all. |

### Why it matters for drop check

Drop check interacts with variance because variance determines which lifetime substitutions the compiler considers valid.

```rust
struct Covariant<'a> {
    data: &'a str, // covariant in 'a
}

impl<'a> Drop for Covariant<'a> {
    fn drop(&mut self) {
        println!("{}", self.data);
    }
}
```

Because `Covariant<'a>` is **covariant** in `'a` *and* has a `Drop` impl, the compiler enforces that `'a` strictly outlives the struct. Without this, covariance would allow the compiler to shorten `'a` — potentially making the reference dangle before `drop` reads it.

### Example: how covariance + `Drop` can cause a use-after-free (without drop check)

Imagine the compiler did **not** enforce drop check. Here's what could go wrong:

```rust
struct CovariantDrop<'a> {
    data: &'a str,  // covariant in 'a
}

impl<'a> Drop for CovariantDrop<'a> {
    fn drop(&mut self) {
        println!("dropping: {}", self.data);  // reads the reference!
    }
}

fn would_be_unsound() {
    let cd: CovariantDrop<'_>;
    let s = String::from("hello");
    cd = CovariantDrop { data: &s };

    // Drop order (reverse declaration): cd is dropped BEFORE s.
    //
    // WITHOUT drop check:
    //   Covariance allows the compiler to shrink 'a to a very short lifetime.
    //   The compiler could reason: "cd only needs &s to be valid during
    //   construction, not at drop time." It would shorten 'a so that it
    //   ends before cd is dropped.
    //
    //   Then cd.drop() runs → reads self.data → but s is already being
    //   dropped or its memory is logically dead → USE-AFTER-FREE.
    //
    // WITH drop check:
    //   The compiler requires 'a to strictly outlive cd.
    //   Since s and cd are dropped in reverse order and s doesn't
    //   strictly outlive cd → COMPILE ERROR. Crisis averted.
}
```

The compiler actually rejects this:
```
error[E0597]: `s` does not live long enough
 --> src/main.rs
  |
  |     cd = CovariantDrop { data: &s };
  |                                 ^^ borrowed value does not live long enough
  |     // ...
  | }
  | - `s` dropped here while still borrowed
```

### The same scenario without `Drop` — no problem

```rust
struct CovariantNoDrop<'a> {
    data: &'a str,  // still covariant
}

// No Drop impl!

fn this_is_fine() {
    let cd: CovariantNoDrop<'_>;
    let s = String::from("hello");
    cd = CovariantNoDrop { data: &s };

    // cd has no Drop impl, so no code runs when cd is destroyed.
    // The compiler doesn't care that s might die first — nobody reads
    // the reference after its last use. Covariance can safely shrink 'a.
    // This compiles fine.
}
```

This demonstrates the key interaction: **covariance is safe when there's no `Drop`**, because nobody observes the reference at destruction time. But **covariance + `Drop` is dangerous**, because `drop()` can read the reference after it dangles. Drop check exists to prevent exactly this.

### A more subtle example: covariance hiding inside a generic

```rust
struct Smuggle<T> {
    value: T,  // covariant in T
}

impl<T: std::fmt::Debug> Drop for Smuggle<T> {
    fn drop(&mut self) {
        println!("smuggled: {:?}", self.value);  // reads T!
    }
}

fn subtle_break() {
    let s: Smuggle<&str>;
    let data = String::from("boom");
    s = Smuggle { value: &*data };

    // T = &'a str. Smuggle<T> is covariant in T, which means
    // covariant in 'a. Drop reads T. Without drop check, the
    // compiler could shorten 'a and allow this.
    //
    // Drop check catches it: 'a (from &'a str) must strictly
    // outlive s. Since data doesn't strictly outlive s → ERROR.
}
```

This is the pattern that makes drop check essential for **any generic struct with `Drop`** — you don't even need explicit lifetime parameters. The lifetime is hidden inside `T = &'a str`, and covariance on `T` means covariance on `'a`.

### Invariant types relax the concern (somewhat)

If the struct is **invariant** in a lifetime (e.g., it holds `&'a mut T` or uses `Cell<&'a T>`), the lifetime is already "locked" and cannot be shortened. Drop check still applies its rule, but invariance already prevents the most dangerous shrinking.

```rust
use std::cell::Cell;

struct Invariant<'a> {
    data: Cell<&'a str>, // invariant in 'a
}

impl<'a> Drop for Invariant<'a> {
    fn drop(&mut self) {
        println!("{}", self.data.get());
    }
}
// Drop check still requires 'a to strictly outlive the struct.
```

---

## When the Type Parameter Is a Reference

When `T` is instantiated as a reference (e.g., `T = &'a u32`), the lifetime `'a` becomes "reachable through" the struct:

```rust
struct Holder<T> {
    inner: T,
}

impl<T> Drop for Holder<T> {
    fn drop(&mut self) {
        // Could in theory inspect self.inner.
        // If T = &'a u32, then 'a must be alive here.
    }
}

fn bad_example() {
    let holder;
    let data = 42u32;
    holder = Holder { inner: &data };
    // ERROR: `data` does not strictly outlive `holder`
    // because they are dropped in reverse declaration order,
    // and drop-check requires strict outliving.
}
```

The key insight: **the compiler does not look at what `drop` actually does** — it only looks at what it *could* do based on the type signature. Since `Holder<T>` has `fn drop(&mut self)` and `self` contains a `T`, the compiler assumes `T` (and all lifetimes inside it) must be alive.

---

## The Dropcheck Eyepatch: `#[may_dangle]`

### The problem it solves

Sometimes your `Drop` impl genuinely **does not** access the generic parameter. The conservative drop-check rule then over-restricts your users. The **eyepatch** `#[may_dangle]` tells the compiler:

> "I promise my `drop` does not access this type/lifetime parameter (except possibly to drop it in place)."

### Syntax

```rust
#![feature(dropck_eyepatch)]

unsafe impl<#[may_dangle] T> Drop for Wrapper<T> {
    fn drop(&mut self) {
        // We promise NOT to read/use T here.
        // We may still drop T (i.e., the compiler will drop fields),
        // but we won't dereference or inspect it.
        println!("dropping wrapper, ignoring T");
    }
}
```

### Key details

| Aspect | Without eyepatch | With `#[may_dangle]` |
|--------|------------------|----------------------|
| `impl` keyword | `impl<T> Drop for S<T>` | `unsafe impl<#[may_dangle] T> Drop for S<T>` |
| Compiler assumption | `drop` may access `T` | `drop` does **not** access `T` |
| Lifetime requirement | `T` (and its lifetimes) must strictly outlive `S` | No extra strict-outlive requirement on `T` |
| Safety | Safe — compiler is conservative | **Unsafe** — you manually guarantee the promise |
| Stability | Stable Rust | **Nightly only** (`#![feature(dropck_eyepatch)]`) |

### Applying it to a lifetime parameter

You can also mark a *lifetime* as `#[may_dangle]`:

```rust
#![feature(dropck_eyepatch)]

struct Inspector<'a>(&'a u32);

unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        // We promise NOT to read self.0.
        // The referent of 'a can already be dead at this point.
    }
}
```

This is exactly how the standard library implements `Drop` for `Vec<T>`, `Box<T>`, `HashMap<K, V>`, etc. — they don't inspect the elements in `drop`, they just deallocate memory or drop elements in place.

### A real-world example from `Vec`

```rust
// Simplified from the standard library:
unsafe impl<#[may_dangle] T, A: Allocator> Drop for Vec<T, A> {
    fn drop(&mut self) {
        // Drops elements via ptr::drop_in_place (not "using" T).
        // Deallocates buffer (only uses the allocator A).
    }
}
```

Without `#[may_dangle]`, you could not do this:

```rust
let mut v;
let x = String::from("hello");
v = vec![&x];
// Without the eyepatch on Vec's Drop, this would fail drop check
// because &x's lifetime doesn't strictly outlive v.
// With #[may_dangle], it compiles — Vec promises not to inspect T in drop.
```

---

## `PhantomData` and Drop Check

When a struct doesn't actually *store* a `T` but logically owns one (e.g., raw pointers), `PhantomData` controls what drop check sees:

```rust
use std::marker::PhantomData;

struct RawVec<T> {
    ptr: *mut T,               // raw pointer — no ownership/lifetime info
    len: usize,
    _marker: PhantomData<T>,   // tells the compiler "I logically own T"
}
```

| `PhantomData` type | Variance in `T` | Owns `T`? | Drop check effect |
|---------------------|-----------------|-----------|-------------------|
| `PhantomData<T>` | **Covariant** | Yes | Drop check applies to `T` (if struct has `Drop` impl) |
| `PhantomData<*const T>` | **Covariant** | No | Drop check does **not** constrain `T` |
| `PhantomData<*mut T>` | **Invariant** | No | Drop check does **not** constrain `T` |
| `PhantomData<fn(T)>` | **Contravariant** | No | Drop check does **not** constrain `T` |
| `PhantomData<fn() -> T>` | **Covariant** | No | Drop check does **not** constrain `T` |
| `PhantomData<fn(T) -> T>` | **Invariant** | No | Drop check does **not** constrain `T` |

> **Key distinction:** `PhantomData<T>` and `PhantomData<*const T>` are **both covariant** in `T`. The difference is **ownership**: `PhantomData<T>` tells the compiler the struct logically owns a `T`, so the compiler will consider `T`'s destructor when the struct is dropped. `PhantomData<*const T>` opts out of that ownership claim.

---

## Stable Workarounds for `#[may_dangle]`

`#[may_dangle]` requires nightly. Here are the practical techniques to achieve the same relaxed drop-check behavior on **stable Rust**.

### 1. Remove the `Drop` impl entirely

The simplest fix: if your struct doesn't need custom drop logic, **don't implement `Drop`**. Without a `Drop` impl, drop check adds no extra constraints on your generic parameters at all.

```rust
struct Wrapper<T> {
    value: T,
    // No Drop impl → no drop-check tightening.
    // Rust will still drop `value` automatically.
}

fn works_fine() {
    let w;
    let x = String::from("hello");
    w = Wrapper { value: &x }; // OK — no Drop impl, no strict-outlive rule
}
```

If you need cleanup for *parts* of the struct but not the generic field, put the cleanup-needing part into a separate, non-generic inner struct:

```rust
struct Inner {
    handle: std::fs::File,
}

impl Drop for Inner {
    fn drop(&mut self) {
        // Custom cleanup — does not touch T at all.
        println!("closing handle");
    }
}

struct Outer<T> {
    inner: Inner,   // has its own Drop
    data: T,        // Outer has NO Drop impl, so T is unconstrained by drop check
}
```

This is the **recommended** approach on stable and the most idiomatic.

### 2. Move the `Drop` impl to a non-generic inner type

Generalization of the pattern above. Extract everything that needs custom destruction into a private helper that doesn't mention `T`:

```rust
// Non-generic — holds the resource.
struct RawBuffer {
    ptr: *mut u8,
    cap: usize,
}

impl Drop for RawBuffer {
    fn drop(&mut self) {
        if self.cap > 0 {
            unsafe {
                std::alloc::dealloc(
                    self.ptr,
                    std::alloc::Layout::from_size_align(self.cap, 1).unwrap(),
                );
            }
        }
    }
}

// Generic — but no Drop impl on this struct.
struct MyVec<T> {
    buf: RawBuffer,
    len: usize,
    _marker: std::marker::PhantomData<T>,
}

// Because MyVec<T> has no Drop impl, T is NOT subject to drop check.
// RawBuffer's Drop only cleans up the allocation, never touches T.
```

#### Drop order for `MyVec<T>`

When a `MyVec<T>` goes out of scope, Rust drops its fields **in declaration order**. Here's a concrete example showing exactly when the deallocation happens:

```rust
fn main() {
    {
        let v = MyVec::<String> {
            buf: RawBuffer { ptr: alloc_some_memory(), cap: 128 },
            len: 3,
            _marker: std::marker::PhantomData,
        };

        // ... use v ...

    }   // <-- v goes out of scope here. Rust drops fields in order:
        //
        //  1. v.buf   (RawBuffer)  → RawBuffer::drop() runs
        //     ↳ calls std::alloc::dealloc(ptr, ...) — memory freed HERE
        //
        //  2. v.len   (usize)      → no destructor (Copy type, nothing happens)
        //
        //  3. v._marker (PhantomData<String>) → no destructor (ZST, nothing happens)
        //
        // Note: the String *values* that were logically stored behind buf.ptr
        // are NOT dropped automatically — MyVec would need a manual impl
        // (e.g., iterating and calling ptr::drop_in_place on each element)
        // BEFORE the RawBuffer is dropped. A real implementation would do
        // this by dropping elements in a Drop impl on a wrapper, or by
        // calling a cleanup method.
}
```

The critical point: **`RawBuffer::drop()` runs as part of dropping `MyVec`'s fields, not via a `Drop` impl on `MyVec` itself.** Because `MyVec<T>` has no `Drop` impl, the compiler never adds drop-check constraints on `T` — yet the allocation is still freed through `RawBuffer`'s own destructor.

In a full implementation you would drop the elements before the buffer:

```rust
impl<T> MyVec<T> {
    /// # Safety: must only be called once, before the buffer is freed.
    unsafe fn drop_elements(&mut self) {
        if self.len > 0 {
            let slice = std::ptr::slice_from_raw_parts_mut(self.buf.ptr as *mut T, self.len);
            unsafe { std::ptr::drop_in_place(slice); }
        }
    }
}

// Still no Drop on MyVec<T> — instead, wrap it so elements are dropped first:
struct SafeVec<T> {
    inner: MyVec<T>,
}

impl<T> Drop for SafeVec<T> {
    fn drop(&mut self) {
        // 1. Drop all T elements.
        unsafe { self.inner.drop_elements(); }
        // 2. inner.buf (RawBuffer) is dropped automatically AFTER this
        //    Drop impl returns → memory is freed.
    }
}
```

This is exactly the layered design the standard library uses internally (`RawVec` + `Vec`).

### 3. Use `ManuallyDrop<T>` to hide `T` from drop check

If you wrap the generic field in `ManuallyDrop<T>`, the compiler knows it will **not** be automatically dropped, so drop check does not apply the strict-outlive rule to it:

```rust
use std::mem::ManuallyDrop;

struct Wrapper<T> {
    value: ManuallyDrop<T>,
}

impl<T> Drop for Wrapper<T> {
    fn drop(&mut self) {
        // You can have a Drop impl here!
        // Because `value` is ManuallyDrop, the compiler knows it
        // won't be auto-dropped, so T is not constrained.
        
        // If you still need to actually drop T, do it explicitly:
        unsafe { ManuallyDrop::drop(&mut self.value); }
    }
}
```

**Warning:** You take on the responsibility of dropping `T` yourself. If you forget, you leak. If you drop twice, you get UB. This is still `unsafe`, but it compiles on stable.

### 4. Adjust `PhantomData` to opt out of ownership

If your struct uses raw pointers and `PhantomData`, you can switch from `PhantomData<T>` (which implies ownership) to `PhantomData<*const T>` (which does not):

```rust
use std::marker::PhantomData;

struct RawContainer<T> {
    ptr: *mut T,
    len: usize,
    _marker: PhantomData<*const T>, // covariant, but NO ownership claim
}

impl<T> Drop for RawContainer<T> {
    fn drop(&mut self) {
        // cleanup that doesn't read T values
    }
}
// T is NOT constrained by drop check because PhantomData<*const T>
// does not imply the struct owns a T.
```

**Caveat:** This tells the compiler you do *not* own `T`, which affects auto-trait inference (e.g., `Send`/`Sync` bounds may need manual impls).

### Comparison of stable workarounds

| Technique | Safe? | Stable? | Ergonomics | Runtime cost | Caveat |
|-----------|-------|---------|------------|-------------|--------|
| Remove `Drop` / split into non-generic inner | Yes | Yes | Best | None | Requires architectural split |
| `ManuallyDrop<T>` | No (`unsafe` to drop) | Yes | Good | None | Must manually drop; leak/double-drop risk |
| `PhantomData<*const T>` | Yes* | Yes | Good | None | Only for raw-pointer structs; affects `Send`/`Sync` |

\* Safe at the type level, but raw-pointer structs typically involve `unsafe` elsewhere.

---

## Summary: Decision Flowchart

```
Does your struct implement Drop?
├── NO  → Normal borrow check only. No extra drop-check constraints.
└── YES → Does drop() actually access the generic param T / lifetime 'a?
    ├── YES → Keep the default. T/'a must strictly outlive your struct.
    └── NO  → Are you on nightly?
        ├── YES → Use #[may_dangle] (unsafe) to relax the constraint.
        └── NO  → Use a stable workaround:
                   • Split into non-generic inner type (preferred)
                   • ManuallyDrop<T> (unsafe but zero-cost)
                   • PhantomData<*const T> (raw-pointer structs only)
```

### Rules of thumb

1. **Adding `Drop` to a generic struct tightens lifetime requirements** for users of that struct.
2. **Variance matters** — covariant types are especially affected because the compiler could otherwise shorten lifetimes past the `drop` call.
3. **`#[may_dangle]`** is the escape hatch, but it is `unsafe` and nightly-only — you must manually guarantee your `drop` doesn't read the marked parameter.
4. **On stable, prefer splitting your type** so the `Drop` impl lives on a non-generic inner struct that never touches `T`.
5. **`PhantomData`** shapes what the compiler believes your struct owns, directly influencing drop check.
