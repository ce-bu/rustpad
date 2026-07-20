# Pinning in Rust: A Complete Guide

**Estimated reading time: ~65 minutes**

This document provides an in-depth exploration of pinning in Rust—why it exists, how it works, and how to use it correctly. We'll cover everything from first principles to advanced patterns like structural pinning and drop guarantees.

## Table of Contents

1. [The Problem: Self-Referential Types](#the-problem-self-referential-types)
2. [What Pin Actually Does](#what-pin-actually-does)
3. [The Unpin Trait](#the-unpin-trait)
4. [Creating Pinned Values](#creating-pinned-values)
5. [Why Box::pin?](#why-boxpin)
6. [Stack Pinning](#stack-pinning)
7. [Structural Pinning](#structural-pinning)
8. [Pin Drop Guarantees](#pin-drop-guarantees)
9. [The pin-project Crate](#the-pin-project-crate)
10. [Essential Pin Methods and Macros](#essential-pin-methods-and-macros)
11. [Common Patterns and Pitfalls](#common-patterns-and-pitfalls)
12. [Pin in Async Rust](#pin-in-async-rust)
13. [Pin Beyond Async](#pin-beyond-async)

---

## The Problem: Self-Referential Types

### Why Can't Rust Have Self-References?

In most languages, creating a struct that points to itself is straightforward:

```c
// C code - works fine
struct Node {
    int value;
    struct Node* self_ptr;  // Can point to this very struct
};
```

In Rust, this is problematic. Consider:

```rust
struct SelfReferential {
    data: String,
    ptr: *const String,  // Supposed to point to `data`
}

impl SelfReferential {
    fn new(data: String) -> Self {
        let mut s = SelfReferential {
            data,
            ptr: std::ptr::null(),
        };
        s.ptr = &s.data;  // Point to our own field
        s
    }
}
```

**The problem**: When this function returns, `s` is moved to the caller's stack frame. But `ptr` still points to where `data` *used to be*. Now `ptr` is a dangling pointer—undefined behavior.

```
Before return:                After return (MOVE):
┌──────────────────┐          ┌──────────────────┐
│ Stack frame of   │          │ Stack frame of   │
│ new()            │          │ caller           │
├──────────────────┤          ├──────────────────┤
│ data: "hello"    │◄─┐       │ data: "hello"    │ (moved here)
│ ptr: ────────────┼──┘       │ ptr: ────────────┼──► ??? (dangling!)
└──────────────────┘          └──────────────────┘
```

### Where Do Self-References Come From?

You might think "I'll just avoid self-referential structs." But they appear implicitly through async/await. Here's a concrete example:

```rust
async fn example() {
    let data = vec![1, 2, 3];      
    let slice = &data[..];         // <── THIS is the self-reference!
    some_async_op().await;         //     `slice` is a pointer INTO `data`
    println!("{:?}", slice);       //     Both must be stored in the future
}
```

**What actually happens:**

1. **The async fn returns immediately** - it doesn't run the body, it returns a `Future`
2. **The future is a struct** - containing all local variables that live across `.await` points
3. **The future gets moved** - from the call site to wherever it's stored (heap, executor, etc.)

The compiler transforms the async fn into a state machine struct:

```rust
// This struct IS the future. Both `data` AND `slice` live INSIDE it.
enum ExampleFuture {
    // ─── State 0: Before any code runs ───
    Start,
    
    // ─── State 1: After first .await, before println ───
    AfterAwait { 
        data: Vec<i32>,       // `data` is a FIELD of this struct
        slice: *const [i32],  // `slice` POINTS TO `data` above! 
                              //  ↑ THIS IS THE SELF-REFERENCE
    },
    
    // ─── State 2: Future completed ───
    Done,
}
```

**The problem:**

```rust
let fut = example();        // Future created on stack (State: Start)
                            // `data` and `slice` don't exist yet

let boxed = Box::new(fut);  // MOVED to heap at address 0x1000
                            // Still safe - no self-references yet

// Executor polls, future runs until .await:
// 1. Transitions to AfterAwait state  
// 2. `data` initialized at address 0x1000 + offset
// 3. `slice` set to point at 0x1000 + offset (pointing into `data`)
//    
//    Memory layout:
//    ┌─────────────────────────────────┐
//    │ AfterAwait @ 0x1000             │
//    │   data: [1, 2, 3]  @ 0x1008     │ ◄─┐
//    │   slice: 0x1008 ───────────────────┘ (points to data)
//    └─────────────────────────────────┘

// If we moved the future NOW:
let moved = *boxed;         // Future moves to stack at 0x2000
                            //
//    Memory layout AFTER move:
//    ┌─────────────────────────────────┐
//    │ AfterAwait @ 0x2000             │
//    │   data: [1, 2, 3]  @ 0x2008     │   (data moved here)
//    │   slice: 0x1008 ─────────────────► DANGLING! Points to old location!
//    └─────────────────────────────────┘
```

`slice` still contains `0x1008` but `data` now lives at `0x2008`. Using `slice` is undefined behavior.

### The Core Insight

Rust's move semantics are incompatible with self-references. The solution isn't to prevent self-references (they're too useful), but to **prevent moves after self-references are established**.

That's what `Pin` does.

---

## What Pin Actually Does

### The Pin Type

`Pin<P>` is a wrapper around a pointer type `P` (like `&mut T`, `Box<T>`, `Rc<T>`):

```rust
pub struct Pin<P> {
    pointer: P,
}
```

The key is what `Pin` **doesn't** let you do: get a `&mut T` that would allow moving the `T` (via `mem::swap`, `mem::replace`, or plain assignment—all of which copy the bytes to a new location).

### The Invariant

`Pin<P>` where `P: Deref<Target = T>` guarantees:

> The value `T` will not be moved from its current memory location for the rest of its lifetime (until drop).

This is enforced through the API:

```rust
impl<P: Deref> Pin<P> {
    // Can always get a shared reference (can't move through &T)
    pub fn as_ref(&self) -> Pin<&P::Target> { ... }
}

impl<P: DerefMut> Pin<P> {
    // Getting &mut is ONLY allowed if T: Unpin
    pub fn get_mut(self) -> &mut P::Target
    where
        P::Target: Unpin,
    { ... }
    
    // Otherwise, you need unsafe
    pub unsafe fn get_unchecked_mut(self) -> &mut P::Target { ... }
}
```

**Why `as_ref` returning `Pin<&T>` matters:**

You use `Pin<&T>` to call methods that require pinned receivers.

**Key fact: `Pin<P>` implements `Deref` when `P` implements `Deref`:**

```rust
// From the standard library:
impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    
    fn deref(&self) -> &Self::Target {
        &*self.pointer  // Just delegates to the inner pointer
    }
}

// This means:
// - Pin<&T>     implements Deref<Target = T>
// - Pin<Box<T>> implements Deref<Target = T>  
// - Pin<&mut T> implements Deref<Target = T>
```

So inside a method with `self: Pin<&Self>`, you access fields normally:

```rust
struct MyFuture {
    completed: bool,
}

impl MyFuture {
    fn is_complete(self: Pin<&Self>) -> bool {
        // self is Pin<&MyFuture>, but Deref lets us use it like &MyFuture:
        self.completed  // Works because Pin<&Self>: Deref<Target = Self>
    }
    
    // Equivalent explicit version:
    fn is_complete_explicit(self: Pin<&Self>) -> bool {
        let this: &Self = &*self;  // Deref coercion
        this.completed
    }
}

// Usage:
let future: Pin<Box<MyFuture>> = Box::pin(MyFuture::new());
let is_done = future.as_ref().is_complete();
```

Without `as_ref`, you couldn't call these methods on a `Pin<Box<T>>` because the receiver type wouldn't match.
```

### Creating a Pin

```rust
// For Unpin types, you can pin anything (it's a no-op conceptually)
let mut x = 5;
let pinned: Pin<&mut i32> = Pin::new(&mut x);  // OK because i32: Unpin

// For !Unpin types, you need a stable memory location
let future = async { /* ... */ };
// Pin::new(&mut future);  // ERROR: future is not Unpin
let pinned: Pin<Box<_>> = Box::pin(future);  // OK: Box provides stable location
```

---

## The Unpin Trait

### What Unpin Means

`Unpin` is an auto-trait that most types implement. It says:

> "This type doesn't care about being pinned. Even after pinning, moves are safe."

```rust
// These are all Unpin:
// - All primitives (i32, bool, f64, etc.)
// - References (&T, &mut T)
// - Most standard library types
// - Any struct where all fields are Unpin

struct AllUnpin {
    a: i32,
    b: String,
    c: Vec<u8>,
}
// AllUnpin: Unpin automatically
```

### Opting Out of Unpin

To create a !Unpin type, you need either:

1. **PhantomPinned marker**:
```rust
use std::marker::PhantomPinned;

struct NotUnpin {
    data: String,
    _pin: PhantomPinned,
}
// NotUnpin: !Unpin
```

2. **A !Unpin field** (like most futures):
```rust
async fn foo() {}
// The future returned by foo() is !Unpin
```

### Why Auto-Implement Unpin?

Most types don't have self-references, so pinning them is pointless. Making `Unpin` opt-out means you only pay the complexity cost when you actually need pinning guarantees.

```rust
// For Unpin types, Pin is effectively transparent:
fn takes_pin<T: Unpin>(pinned: Pin<&mut T>) {
    let regular: &mut T = Pin::into_inner(pinned);
    // Can move *regular freely
}
```

---

## Creating Pinned Values

Before discussing `Box::pin`, let's survey all the ways to create pinned values.

### Method 1: `Pin::new()` — Only for Unpin Types

Takes a **pointer** (like `&mut T`, `Box<T>`), not the value itself:

```rust
use std::pin::Pin;

let mut value = String::from("hello");
let pinned: Pin<&mut String> = Pin::new(&mut value);  // &mut value is the pointer

// This works because String: Unpin
// Pin::new requires the pointed-to type to be Unpin
```

**Signature:**
```rust
impl<P: Deref> Pin<P> 
where
    P::Target: Unpin,  // The type BEHIND the pointer must be Unpin
{
    pub fn new(pointer: P) -> Pin<P> { ... }
    //         ^^^^^^^^^ P is a pointer type: &mut T, Box<T>, Rc<T>, etc.
}
```

### Method 2: `Box::pin()` — For Any Type

```rust
use std::pin::Pin;

let future = async { 42 };  // !Unpin
let pinned: Pin<Box<_>> = Box::pin(future);

// Works for any type, Unpin or not
// The Box provides a stable heap location
```

### Method 3: `pin!()` Macro — Stack Pinning

```rust
use std::pin::pin;

let future = async { 42 };
let pinned: Pin<&mut _> = pin!(future);  // Pinned on the stack

// Or with tokio:
tokio::pin!(my_future);
```

The macro expands to something like:
```rust
let mut future = async { 42 };
let pinned = unsafe { Pin::new_unchecked(&mut future) };
// The macro ensures `future` isn't used directly after this
```

### Method 4: `Pin::new_unchecked()` — Unsafe, Manual

```rust
use std::pin::Pin;

let mut value = MyNotUnpinType::new();
// SAFETY: We promise not to move `value` after this
let pinned = unsafe { Pin::new_unchecked(&mut value) };
```

**When to use:** Almost never directly. Use `pin!()` macro instead.

### Summary Table

| Method | Works with !Unpin? | Allocation | Safety |
|--------|-------------------|------------|--------|
| `Pin::new()` | No (Unpin only) | None | Safe |
| `Box::pin()` | Yes | Heap | Safe |
| `pin!()` macro | Yes | Stack | Safe |
| `Pin::new_unchecked()` | Yes | None | Unsafe |

---

## Why Box::pin?

### The Stability Problem

`Pin` guarantees a value won't move, but where does the value live?

```rust
fn bad_attempt() {
    let future = async { /* ... */ };
    let pinned = unsafe { Pin::new_unchecked(&mut future) };
    // Now `pinned` promises `future` won't move...
    
    // ...but `future` is on the stack and will be destroyed when
    // this function returns! The promise is meaningless.
}
```

For pinning to be useful, the value needs a **stable memory location**:
- The heap (via `Box`, `Rc`, `Arc`)
- A carefully managed stack location

### Box Provides Heap Stability

`Box::pin` is the standard solution:

```rust
pub fn pin(x: T) -> Pin<Box<T>> {
    // 1. Allocate x on the heap
    // 2. Return a Pin wrapper
    // The heap location is stable for the lifetime of the Box
}
```

```rust
let future = async { /* ... */ };
let pinned: Pin<Box<dyn Future<...>>> = Box::pin(future);

// The future is now on the heap. Even if we move `pinned`,
// the Box's pointer remains valid—the actual future doesn't move.
```

### The Indirection

```
Stack                    Heap
┌─────────────┐         ┌─────────────────────┐
│ pinned:     │         │                     │
│ Pin<Box<T>> │────────►│  T (the future)     │
│             │         │  (never moves)      │
└─────────────┘         └─────────────────────┘

Moving `pinned` on the stack is fine because the Box pointer is a member of the stack-allocated `pinned` variable— the heap pointer just gets copied. The actual value `T` itself stays put.
```

### When Box::pin Is Used

1. **Type erasure**: `Pin<Box<dyn Future>>` for trait objects
2. **Returning futures**: Can't return references to stack locals
3. **Storing futures**: In collections or struct fields
4. **Any time you need a stable, owned, pinned value**

### The Cost

`Box::pin` has costs:
- Heap allocation
- Pointer indirection
- Potential cache misses

For hot paths, consider stack pinning or custom futures.

---

## Stack Pinning

### The Concept

Stack pinning pins a value to its current stack location. It's more efficient than `Box::pin` but requires care.

### The tokio::pin! Macro

```rust
use tokio::pin;

async fn example() {
    let future = some_async_fn();
    pin!(future);  // future is now Pin<&mut impl Future>
    
    // Can use pinned future
    future.await;
}
```

The macro expands to something like:

```rust
let future = some_async_fn();
// Shadow `future` with a pinned reference
let mut future = future;
#[allow(unused_mut)]
let mut future = unsafe { Pin::new_unchecked(&mut future) };
```

### Why This Is Safe

The macro creates a situation where:
1. The original variable is shadowed immediately
2. Only the pinned reference is accessible
3. The original can't be moved because it can't be named

### Manual Stack Pinning (Unsafe)

```rust
fn manual_stack_pin() {
    let mut future = some_async_fn();
    
    // SAFETY: We guarantee not to move `future` after this point
    let pinned = unsafe { Pin::new_unchecked(&mut future) };
    
    // Use pinned...
    
    // CRITICAL: Don't do anything with `future` directly anymore!
    // std::mem::swap(&mut future, &mut other);  // UNDEFINED BEHAVIOR
}
```

### Stack Pinning Pitfalls

```rust
fn bad_stack_pin() -> impl Future<Output = ()> {
    let future = async { /* ... */ };
    
    // WRONG: Can't return a reference to a local!
    // let pinned = unsafe { Pin::new_unchecked(&mut future) };
    // pinned  // Dangling reference
    
    // This is why Box::pin exists for returning futures
    Box::pin(future)
}
```

---

## Structural Pinning

**Pin projection** is the act of going from a pinned struct to a pinned reference to one of its fields: `Pin<&mut Struct>` → `Pin<&mut Field>`. It's called "projection" because you're projecting the pin guarantee from the struct down to an individual field.

The central question is: **for each field in the struct, should pinning the struct also pin that field?**

- **Yes → "structurally pinned":** You can get `Pin<&mut Field>` from `Pin<&mut Struct>`. But you must **never** move, swap, or take the field's value out.
- **No → "not structurally pinned":** You can only get `&mut Field`. But you're free to move, swap, or take it.

> **If you just want working code:** Use the `pin-project` crate. Mark fields with `#[pin]` when you need `Pin<&mut Field>` (typically nested futures you need to poll), then call `.project()`. The crate generates all the unsafe projection code and enforces correctness at compile time. The rest of this section explains *why* these rules exist.

### The Problem

You have a pinned struct. How do you access its fields?

```rust
struct MyFuture {
    inner: SomeOtherFuture,
    state: u32,
}

impl Future for MyFuture {
    type Output = ();
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // self is Pin<&mut MyFuture>
        // We need Pin<&mut SomeOtherFuture> to poll inner
        // How do we get it?
    }
}
```

### What Is Structural Pinning?

First, let's understand how fields are stored in memory:

**Fields are stored inline (embedded), not as pointers:**

```rust
struct SomeFuture {
    data: [i32; 3],
    ptr: *const i32,  // Self-reference: points to data[0]
}

struct MyStruct {
    future: SomeFuture,  // SomeFuture's bytes are stored HERE, inline
}
```

```
MyStruct in memory at address 0x1000:
┌─────────────────────────────────────────┐
│ future: SomeFuture                      │
│   ├─ data: [1, 2, 3]     @ 0x1000       │ ◄─┐
│   └─ ptr: 0x1000 ────────────────────────────┘ (points to data)
└─────────────────────────────────────────┘
```

When you pin `MyStruct`, the **memory addresses are fixed**—the struct won't move, so its fields won't move. That's guaranteed.

**But you can still move a VALUE out of a field:**

A Rust "move" copies the bytes to a new location. Even with a pinned struct, `option.take()` can copy the bytes out:

```rust
struct MyStruct {
    future: Option<SomeFuture>,
}

let pinned: Pin<&mut MyStruct> = ...;

// This is STILL possible with unsafe:
unsafe {
    let inner = pinned.get_unchecked_mut();
    // get_unchecked_mut() gives us &mut MyStruct, bypassing Pin.
    // Now we can call take() which needs &mut Option<SomeFuture>.
    // This MOVES the SomeFuture bytes out of the Option!
    let stolen = inner.future.take();  // Copies bytes to `stolen`!
}
// If `future` is meant to be structurally pinned, the above is UNSOUND.
// If `future` is NOT structurally pinned, this is fine.
```

```
BEFORE take():                          
┌─────────────────────────┐
│ Option::Some @ 0x1000   │
│   SomeFuture            │
│     data: [1,2,3]       │ ◄─┐ at 0x1008
│     ptr: 0x1008 ───────────┘ points here ✓
└─────────────────────────┘

AFTER take() — bytes copied to `stolen`:
┌─────────────────────────┐     ┌─────────────────────────┐
│ Option::None @ 0x1000   │     │ stolen @ 0x2000         │
│   (empty)               │     │   SomeFuture            │
└─────────────────────────┘     │     data: [1,2,3]       │   at 0x2008
                                │     ptr: 0x1008 ────────┼──► DANGLING!
                                └─────────────────────────┘
```

The bytes were copied, but `ptr` still contains `0x1008`. The data is now at `0x2008`. **The self-reference is broken.**

**So "structural pinning" answers: can we safely hand out `Pin<&mut Field>`?** 

Only if we promise NEVER to move the value out of that field. Here's why:

```rust
struct Wrapper<F> {
    inner: F,  // F is a self-referential future
}

impl<F: Future> Wrapper<F> {
    fn get_inner(self: Pin<&mut Self>) -> Pin<&mut F> {
        // QUESTION: Is this safe?
        unsafe { self.map_unchecked_mut(|w| &mut w.inner) }
    }
}
```

If `inner` is "structurally pinned", the above is safe—we promise never to move the value out of `inner`.

> **⚠️ This is a hard guarantee YOU must uphold—the compiler cannot check it.**
> 
> The `unsafe` block is where you assert: "I, the programmer, guarantee this field will never be moved out." If you later add a method that moves the field, you've introduced undefined behavior. The compiler won't stop you.
>
> Note: The **wrapper struct itself** can be moved freely before it's pinned. A `Wrapper<F>` can be passed between functions, stored in collections, even sent between threads/executors—all fine. The guarantee only kicks in **after** someone creates a `Pin<&mut Wrapper<F>>`. From that point on, the structurally pinned field must not be moved.

### Example: How a `reset()` Method Breaks Pinning

Here's another concrete example. A common pattern is wanting to "reset" or replace a contained future:

```rust
struct Resettable<F> {
    future: F,
}

impl<F> Resettable<F> {
    // Pin projection: gives caller Pin<&mut F>
    fn project(self: Pin<&mut Self>) -> Pin<&mut F> {
        unsafe { self.map_unchecked_mut(|r| &mut r.future) }
    }
    
    // Looks innocent—who doesn't want to reset their task?
    fn reset(&mut self, new_future: F) {
        self.future = new_future;
    }
}
```

**Having both methods is unsound.** Here's the disaster scenario:

```rust
// Create a self-referential future
let self_ref = async {
    let data = vec![1, 2, 3];
    let slice = &data[..];      // Self-reference!
    some_async_op().await;
    println!("{:?}", slice);
};

let mut resettable = Resettable { future: self_ref };

// Pin it
let mut pinned = unsafe { Pin::new_unchecked(&mut resettable) };

// Get a pinned reference to the inner future
let future_pin: Pin<&mut _> = pinned.as_mut().project();
// ↑ future_pin PROMISES the future won't move

// But wait—someone saved an &mut Resettable from earlier...
// Or we do this (which compiles!):
drop(pinned);  // Unpin
resettable.reset(async { 42 });  // Old future MOVED out and DROPPED!

// If anyone still held future_pin, it's now dangling.
// Even without holding future_pin, we violated the contract:
// we promised not to move the future, then immediately moved it.
```

**The fix:** If you want `reset()`, you CANNOT provide a pin projection. Pick one:

```rust
impl<F> Resettable<F> {
    // Option A: No reset(), provide projection
    fn project(self: Pin<&mut Self>) -> Pin<&mut F> { ... }
    // NO reset() method
    
    // Option B: Have reset(), no projection
    fn reset(&mut self, new_future: F) { ... }
    // Can only return &mut F, not Pin<&mut F>
}
```

### How to Reason About Structural Pinning in Practice

Since the compiler can't check this, **you** must audit your code. Here's how:

**Step 1: List every way to get `&mut` access to the field**

For a structurally pinned field, search your ENTIRE impl block for:
- `get_unchecked_mut()` calls
- Any method returning `&mut TheField`
- Any method with `&mut self` receiver (not `Pin<&mut Self>`)

```rust
impl<F> Wrapper<F> {
    // ✗ DANGEROUS: Returns &mut to pinned field
    fn get_inner_mut(&mut self) -> &mut F { &mut self.inner }
    
    // ✗ DANGEROUS: &mut self lets caller access inner
    fn do_something(&mut self) { /* could move self.inner */ }
    
    // ✓ SAFE: Pin<&mut Self> receiver, returns Pin<&mut F>
    fn project(self: Pin<&mut Self>) -> Pin<&mut F> { ... }
    
    // ✓ SAFE: Only shared access
    fn inspect(&self) -> &F { &self.inner }
}
```

**Step 2: Search for these dangerous operations on the field**

If any of these can happen to your pinned field, it's NOT structurally pinned:

```rust
// ALL of these MOVE the value:
std::mem::swap(&mut self.inner, &mut other);  // ✗ Swap
std::mem::replace(&mut self.inner, new);       // ✗ Replace
std::mem::take(&mut self.inner);               // ✗ Take
self.inner = new_value;                        // ✗ Assignment (drops old, moves new)
let moved = self.inner;                        // ✗ Move out
option_field.take();                           // ✗ Option::take moves inner
```

**Step 3: Check your Drop implementation**

**The danger:** `Drop::drop` receives `&mut self`, NOT `Pin<&mut Self>`. This is a backdoor that bypasses all your careful pinning work. Even if every other method takes `Pin<&mut Self>`, Rust will call `drop(&mut self)` when the value is destroyed.

```rust
impl<F> Drop for Wrapper<F> {
    fn drop(&mut self) {
        // HERE'S THE DANGER:
        // - All your methods carefully took Pin<&mut Self>
        // - But Drop::drop gives you &mut Self anyway!
        // - You now have &mut access to the "pinned" field
        
        let _ = std::mem::take(&mut self.inner);  // ✗ COMPILES! But UNSOUND
    }
}
```

**Why is this unsound?** Pin's "drop guarantee" promises TWO things:

1. The value won't move until drop
2. **Drop WILL be called** while the value is still at its pinned location

That second part is crucial. A self-referential type's `Drop` implementation might need to clean up those self-references safely. It expects to run while the data is still in place:

```rust
impl Drop for SelfReferential {
    fn drop(&mut self) {
        // I expect my self-references to still be valid here!
        // I might need to de-register a callback, unlink from a list, etc.
        // If someone moved me before my drop ran, these references are garbage.
    }
}
```

If your `Wrapper::drop` calls `std::mem::take(&mut self.inner)`, you:
1. Move `inner` to a temporary location
2. The temporary is dropped immediately (at the new location!)
3. `inner`'s drop runs with broken self-references → **undefined behavior**

**To be clear:** Implementing `Drop` is fine—just don't move the pinned field:

```rust
impl<F> Drop for Wrapper<F> {
    fn drop(&mut self) {
        // ✓ OK: Read from the pinned field
        println!("dropping wrapper with state: {:?}", self.inner.some_field);
        
        // ✓ OK: Call methods on the pinned field (they get &mut, not ownership)
        self.inner.cleanup();
        
        // ✓ OK: Modify non-pinned fields
        self.counter = 0;
        
        // ✗ DANGEROUS: Move/swap/take the pinned field
        // let _ = std::mem::take(&mut self.inner);
    }
}
```

**What if you know there are no self-references?** Then moving is fine:

```rust
impl<F> Drop for Wrapper<F> {
    fn drop(&mut self) {
        // If you can GUARANTEE that `inner` has no active self-references
        // at this point (e.g., the future completed, or F: Unpin), then
        // moving it is safe:
        
        // SAFETY: `inner` has no self-references because [your reason here]
        let _ = std::mem::take(&mut self.inner);  // OK if guarantee holds
    }
}
```

The key question is: **does `inner`'s drop implementation expect its self-references to be valid?** If yes, don't move it. If no (or there are no self-references), you're fine.

For extra safety, use `#[pinned_drop]` from `pin-project` which gives you `Pin<&mut Self>` instead of `&mut self`, making it harder to accidentally move pinned fields.

**Step 4: Ensure no `&mut self` methods exist (or are sound)**

The safest approach: **never have `&mut self` methods** on types with structurally pinned fields.

```rust
impl<F> Wrapper<F> {
    // If this method exists, ANYONE can get &mut self:
    fn bad(&mut self) { }
}

// And then do:
let mut wrapper = Wrapper::new(future);
wrapper.bad();  // Now they have &mut Wrapper
std::mem::swap(&mut wrapper.inner, &mut other);  // Moved the "pinned" field!
```

**The Rule:** If you have `&mut self` methods, users can call them BEFORE pinning. That's fine. But they could also call them after getting `&mut` through other means—ensure that's impossible or harmless.

**Checklist for a structurally pinned field:**

- [ ] No method returns `&mut Field`  
- [ ] No `&mut self` method that could move/swap/take the field
- [ ] `Drop` impl (if any) doesn't move the field
- [ ] No public access that would let users get `&mut Struct`
- [ ] Or: use `pin-project` crate which enforces these at compile time!

**This is why `pin-project` exists:** it generates the projection code AND checks these invariants at compile time, so you don't have to manually audit.

**What happens if you provide a pin projection for a field that ISN'T structurally pinned?** The projection becomes unsound. Consider a variant where `inner` is `Option<F>` and we provide both a projection *and* a way to move the value out:

```rust
struct Wrapper<F> {
    inner: Option<F>,
}

impl<F> Wrapper<F> {
    // Pin projection — promises inner won't move
    fn project_inner(self: Pin<&mut Self>) -> Option<Pin<&mut F>> {
        unsafe {
            self.get_unchecked_mut().inner.as_mut().map(|f| Pin::new_unchecked(f))
        }
    }
    
    // But this method MOVES the value out — breaking the promise!
    fn take_inner(self: Pin<&mut Self>) -> Option<F> {
        unsafe { self.get_unchecked_mut().inner.take() }
    }
}

// The disaster:
let pinned_wrapper: Pin<&mut Wrapper<SelfRefFuture>> = ...;
let inner_pin: Pin<&mut SelfRefFuture> = pinned_wrapper.as_mut().project_inner().unwrap();
// inner_pin PROMISES the future won't move

let stolen = pinned_wrapper.take_inner();  // MOVED! Future copied to new location
// inner_pin now points at None memory — undefined behavior!
```

**The lesson:** You must pick one. Either provide a pin projection (and never move the field), or allow moving (and never hand out `Pin<&mut Field>`).

**So "structural pinning" means:**
- We declare: "this field's VALUE will never be moved out"
- Therefore: it's safe to hand out `Pin<&mut Field>`
- We CANNOT also have methods that move/swap/take that field

**"NOT structurally pinned" means:**
- We might move/swap/take the value from this field
- Therefore: we can only hand out `&mut Field`, not `Pin<&mut Field>`
- But we CAN call `.take()`, `std::mem::replace()`, etc.

```rust
struct MyStruct {
    pinned_field: SomeType,      // Structurally pinned: can get Pin<&mut SomeType>
    not_pinned_field: OtherType, // NOT pinned: can only get &mut OtherType
}
```

### The Projection

"Projecting" a pin means going from `Pin<&mut Struct>` to `Pin<&mut Field>`:

```rust
impl MyStruct {
    fn project(self: Pin<&mut Self>) -> ProjectedMyStruct<'_> {
        // UNSAFE: We must uphold pinning invariants
        unsafe {
            let this = self.get_unchecked_mut();
            ProjectedMyStruct {
                pinned_field: Pin::new_unchecked(&mut this.pinned_field),
                not_pinned_field: &mut this.not_pinned_field,
            }
        }
    }
}

struct ProjectedMyStruct<'a> {
    pinned_field: Pin<&'a mut SomeType>,      // Pinned projection
    not_pinned_field: &'a mut OtherType,      // Regular reference
}
```

### Pin Projection Step by Step

Here's the complete mental model in one place:

1. **You have** `Pin<&mut MyStruct>` — the whole struct is pinned and can't move
2. **You want** `Pin<&mut SomeField>` — because you need to call a method with a pinned receiver (e.g., `Future::poll`)
3. **You call** `.project()` — which returns a projection struct where each field is either `Pin<&mut Field>` (structurally pinned) or `&mut Field` (not pinned)
4. **The safety contract**: for every field exposed as `Pin<&mut Field>`, you promise to **never** move that field's value out of the struct

```rust
// The complete picture in one example:
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

#[pin_project]                              // Step 0: Annotate the struct
struct RetryFuture<F> {
    #[pin]  inner: F,                       // Will project to Pin<&mut F>
            attempts: u32,                  // Will project to &mut u32
}

impl<F: Future> Future for RetryFuture<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<F::Output> {
        let this = self.project();          // Project: Pin<&mut Self> → fields
        //  this.inner:    Pin<&mut F>      — can pass to F::poll()
        //  this.attempts: &mut u32         — can modify directly

        *this.attempts += 1;
        this.inner.poll(cx)                 // F::poll() needs Pin<&mut F> ✓
    }
}
```

**Why can't we just use `&mut F` to poll?** Because `Future::poll` requires `Pin<&mut Self>`. A future may contain self-references (e.g., an async block borrowing a local variable across an `.await`). If you could get `&mut F`, you could `mem::swap` it to a different location, breaking those self-references. `Pin<&mut F>` prevents that.

### When Should a Field Be Structurally Pinned?

**Structurally pin a field when you need `Pin<&mut Field>` from `Pin<&mut Struct>`.**

The most common case: **nested futures that you need to poll**.

```rust
struct MyFuture {
    // STRUCTURALLY PINNED: We need Pin<&mut F> to call inner.poll()
    inner: F,
}

impl<F: Future> Future for MyFuture {
    type Output = F::Output;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<F::Output> {
        // We MUST get Pin<&mut F> because Future::poll requires it
        let inner: Pin<&mut F> = self.project().inner;  // Pin projection
        inner.poll(cx)
    }
}
```

**Requirements for structural pinning:**

1. **No `repr(packed)`**: Packed structs can silently move fields during access
2. **Don't move/swap/take the field** (except in `Drop::drop`)
3. **If you implement `Drop`**: Don't move the field out in your drop impl
4. **The whole struct must be `!Unpin`** if any field is structurally pinned

### When Should a Field NOT Be Structurally Pinned?

**Don't structurally pin a field if you need to move, swap, replace, or take it.**

#### Example 1: Option<T> where you call `.take()`

```rust
struct Task<F> {
    // NOT PINNED: We need to call .take() when the future completes
    future: Option<F>,
    result: Option<F::Output>,
}

impl<F: Future> Task<F> {
    fn poll_task(self: Pin<&mut Self>, cx: &mut Context<'_>) -> bool {
        let this = self.project();
        
        if let Some(fut) = this.future.as_mut() {  // ← Can't do this if pinned!
            // ... poll the future ...
            if ready {
                *this.future = None;  // ← Moving out! Only OK if NOT pinned
            }
        }
        // ...
    }
}
```

If `future` were structurally pinned, calling `.take()` or assigning `None` would be **unsound**—you'd be moving a pinned value.

#### Example 2: Swappable/replaceable fields

```rust
struct Timeout<F> {
    // NOT PINNED: We want to replace the timer when reset
    deadline: Instant,
    
    // PINNED: We poll this, never replace it
    inner: F,
}

impl<F> Timeout<F> {
    fn reset(self: Pin<&mut Self>, new_deadline: Instant) {
        let this = self.project();
        *this.deadline = new_deadline;  // OK: deadline is not pinned
        // Can't replace this.inner—it's pinned
    }
}
```

#### Example 3: Fields you need mutable access to without Pin

```rust
struct Counter {
    // NOT PINNED: Simple data, no reason to complicate access
    count: usize,
    
    // PINNED: Contains self-references
    async_state: AsyncStateMachine,
}

impl Counter {
    fn increment(self: Pin<&mut Self>) {
        let this = self.project();
        *this.count += 1;  // Regular &mut usize, not Pin<&mut usize>
    }
}
```

### Decision Summary

| You need to... | Structurally pin? |
|----------------|-------------------|
| Poll a nested `Future` | ✓ Yes |
| The field has self-references | ✓ Yes |
| Call `.take()` on `Option<T>` | ✗ No |
| Swap or replace the field | ✗ No |
| It's simple data (`usize`, `bool`) | ✗ No |
| The field is always `Unpin` | ✗ No (pointless) |
```

---

## Pin Drop Guarantees

### The Drop Problem

When a pinned value is dropped, `Drop::drop(&mut self)` is called. This gives you `&mut self`—enough to move fields!

```rust
impl Drop for SelfReferential {
    fn drop(&mut self) {
        // self is &mut SelfReferential
        // We can do std::mem::swap(&mut self.data, &mut other)
        // This would move data... violating pinning?
    }
}
```

### The Pin Drop Guarantee

`Pin` makes a crucial guarantee:

> A pinned value will not be moved from its memory location until it is dropped. **Drop will be called while the value is still in place.**

This means:
- During `drop()`, the value is still at its pinned location
- The value's `Drop` impl can rely on self-references still being valid
- After `drop()` returns, the memory may be deallocated

**Subtlety:** The pin contract for a `Pin<&mut Wrapper>` ends when `Wrapper::drop` starts. But if `Wrapper` has a structurally pinned field `F`, that field's *own* pin contract persists until `F::drop` runs. So you still shouldn't move `F` out in `Wrapper::drop`—doing so would call `F::drop` at the wrong location.

### Safe Drop for Pinned Types

For most cases, the default Drop is fine. But sometimes you need drop behavior that respects structural pinning:

```rust
struct HasPinnedField {
    pinned: ManuallyDrop<SomeNonUnpin>,
    _pin: PhantomPinned,
}

impl Drop for HasPinnedField {
    fn drop(&mut self) {
        // SAFETY: We're in drop, the pin guarantee is ending
        unsafe {
            ManuallyDrop::drop(&mut self.pinned);
        }
    }
}
```

### The PinnedDrop Pattern

**The problem:** Regular `Drop::drop` gives you `&mut self`, which lets you accidentally move pinned fields. You have to manually uphold the pinning invariant.

**The solution:** `pin-project` provides `#[pinned_drop]` which gives you `Pin<&mut Self>` instead:

```rust
use pin_project::{pin_project, pinned_drop};

#[pin_project(PinnedDrop)]
struct MyType {
    #[pin]
    future: SomeFuture,
    data: String,
}

#[pinned_drop]
impl PinnedDrop for MyType {
    fn drop(self: Pin<&mut Self>) {
        // self is Pin<&mut Self>, not &mut Self!
        // Can use project() to access fields safely
        let this = self.project();
        // this.future is Pin<&mut SomeFuture>
        // this.data is &mut String
        
        // Can't accidentally do this (won't compile):
        // std::mem::take(&mut this.future);  // ERROR: this.future is Pin<&mut>
        
        // Do cleanup...
    }
}
```

**Why this is safer:**
- You get `Pin<&mut Self>`, not `&mut self`
- You can call `project()` to get the proper pinned/unpinned field references
- Moving pinned fields is a compile error, not a silent bug
- The API guides you toward correct usage

### Drop Order and Structural Pinning

When a struct with structurally pinned fields is dropped:

1. `Drop::drop()` is called (if impl'd)
2. Fields are dropped in declaration order
3. Each structurally pinned field is dropped while still pinned

```rust
#[pin_project(PinnedDrop)]
struct TwoFutures {
    #[pin]
    first: FutureA,   // Dropped second (field drop order)
    #[pin]
    second: FutureB,  // Dropped third
}

#[pinned_drop]
impl PinnedDrop for TwoFutures {
    fn drop(self: Pin<&mut Self>) {
        // Dropped first — both fields are still in place here
        let this = self.project();
        // this.first: Pin<&mut FutureA> — still pinned, safe to access
        // this.second: Pin<&mut FutureB> — still pinned, safe to access
    }
}
```

> **Note:** `pin-project` rejects `impl Drop` on types with `#[pin]` fields — you must use `#[pinned_drop]` instead, which gives you `Pin<&mut Self>` and makes it a compile error to accidentally move a pinned field.

### Unsafe Drop Interactions

If you implement `Drop` for a type with structurally pinned fields, you must NOT:

```rust
impl Drop for BadDrop {
    fn drop(&mut self) {
        // DON'T: Move a structurally pinned field
        let stolen = std::mem::take(&mut self.pinned_field);  // UNSOUND
        
        // DON'T: Swap a structurally pinned field
        std::mem::swap(&mut self.pinned_field, &mut other);  // UNSOUND
        
        // OK: Read from pinned fields
        println!("{}", self.pinned_field.some_value);
        
        // OK: Call methods on pinned fields (they get &mut self, not ownership)
        self.pinned_field.some_method();
    }
}
```

### Why Drop Gets &mut self, Not Pin<&mut Self>

Rust's `Drop` trait predates `Pin` and uses `&mut self`:

```rust
trait Drop {
    fn drop(&mut self);
}
```

This is safe because:
1. The pin guarantee is about **not moving until drop**
2. Drop is permitted to move—it's the end of the value's lifetime
3. After drop returns, the memory is invalidated anyway

But for user code in drop that wants to maintain pin invariants during cleanup, `PinnedDrop` exists.

---

## The pin-project Crate

### Why pin-project Exists

Writing correct pin projections manually is error-prone:

```rust
// Manual projection - easy to mess up
unsafe {
    let this = self.get_unchecked_mut();
    Pin::new_unchecked(&mut this.field)
}
```

`pin-project` generates safe, correct projections:

```rust
use pin_project::pin_project;

#[pin_project]
struct MyFuture<F> {
    #[pin]
    inner: F,
    state: u32,
}
```

### How It Works

The macro generates:

```rust
// Generated by #[pin_project]
struct MyFuture<F> {
    inner: F,
    state: u32,
}

// Projection type
struct __MyFutureProjection<'a, F> {
    inner: Pin<&'a mut F>,   // Pinned projection
    state: &'a mut u32,      // Regular reference
}

impl<F> MyFuture<F> {
    fn project(self: Pin<&mut Self>) -> __MyFutureProjection<'_, F> {
        unsafe {
            let this = self.get_unchecked_mut();
            __MyFutureProjection {
                inner: Pin::new_unchecked(&mut this.inner),
                state: &mut this.state,
            }
        }
    }
}
```

### Usage

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

#[pin_project]
struct Timeout<F> {
    #[pin]
    future: F,
    #[pin]
    delay: tokio::time::Sleep,
    cancelled: bool,  // Not pinned
}

impl<F: Future> Future for Timeout<F> {
    type Output = Option<F::Output>;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        
        // this.future: Pin<&mut F>
        // this.delay: Pin<&mut Sleep>
        // this.cancelled: &mut bool
        
        if *this.cancelled {
            return Poll::Ready(None);
        }
        
        // Poll the inner future
        if let Poll::Ready(value) = this.future.poll(cx) {
            return Poll::Ready(Some(value));
        }
        
        // Check timeout
        if this.delay.poll(cx).is_ready() {
            *this.cancelled = true;
            return Poll::Ready(None);
        }
        
        Poll::Pending
    }
}
```

### pin-project Variants

**`#[pin_project]`** - Standard projection
```rust
#[pin_project]
struct Foo<T> {
    #[pin]
    pinned: T,
    unpinned: u32,
}
```

**`#[pin_project(PinnedDrop)]`** - Custom pinned drop
```rust
#[pin_project(PinnedDrop)]
struct Foo<T> {
    #[pin]
    pinned: T,
}

#[pinned_drop]
impl<T> PinnedDrop for Foo<T> {
    fn drop(self: Pin<&mut Self>) {
        // Custom drop with Pin access
    }
}
```

**`#[pin_project(!Unpin)]`** - Force !Unpin
```rust
#[pin_project(!Unpin)]
struct AlwaysPinned<T> {
    #[pin]
    inner: T,
}
// AlwaysPinned: !Unpin even if T: Unpin
```

**`#[pin_project(project = Name)]`** - Custom projection name
```rust
#[pin_project(project = FooProj)]
struct Foo<T> {
    #[pin]
    inner: T,
}
// Generated type is FooProj instead of __FooProjection
```

### What pin-project Checks

The macro enforces:

1. **No `repr(packed)`**: Compile error if used
2. **Consistent pinning**: A field is either always pinned or never
3. **Safe Drop**: If you implement `Drop`, must use `PinnedDrop`
4. **No `Unpin` impl**: You can't manually impl `Unpin` (would break projections)

---

## Essential Pin Methods and Macros

This section covers the key APIs you'll use constantly when working with Pin.

### The `ready!` Macro

`ready!` is used in `poll` implementations to early-return on `Pending`:

```rust
use std::task::ready;

fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<String> {
    // Without ready!:
    let value = match self.inner_future.poll(cx) {
        Poll::Ready(v) => v,
        Poll::Pending => return Poll::Pending,
    };
    
    // With ready! (equivalent):
    let value = ready!(self.project().inner_future.poll(cx));
    
    // Continue with value...
    Poll::Ready(format!("Got: {}", value))
}
```

**Key points:**
- `ready!(expr)` returns the value if `Ready`, or returns `Pending` from the enclosing function
- It's essentially `expr?` but for `Poll` instead of `Result`
- Available as `std::task::ready!` (Rust 1.64+) or `futures::ready!`

### `Pin::set()` — Replacing a Pinned Value

`set()` replaces the value behind a `Pin<&mut T>` without moving the old value out:

```rust
impl<F: Future> Future for StateMachine<F> {
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<...> {
        match self.as_mut().project() {
            StateMachineProj::Running { future } => {
                let result = ready!(future.poll(cx));
                // Transition to Done state
                self.set(StateMachine::Done { result });
                //   ^^^ Replaces *self with new value
                Poll::Ready(result)
            }
            StateMachineProj::Done { .. } => panic!("polled after done"),
        }
    }
}
```

**Signature:**
```rust
impl<P: DerefMut> Pin<P> {
    pub fn set(&mut self, value: P::Target)
    where
        P::Target: Sized;
}
```

**Why `set()` and not assignment?**
- You can't do `*self = new_value` on a `Pin<&mut T>` when `T: !Unpin`—there's no `DerefMut` impl, so you can't get `&mut T` to assign through
- `set()` provides a safe way to replace the entire pinned value
- It drops the old value in-place, then writes the new value
- Safe for state machine transitions where you're replacing the entire enum

### `as_mut()` — Reborrowing a Pin

`as_mut()` is very useful for reborrowing a `Pin<&mut T>` so you can use it multiple times:

```rust
fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
    // First use
    let result = self.as_mut().project().future.poll(cx);
    //               ^^^^^^^^ reborrow - self is still usable
    
    if result.is_pending() {
        // Second use - would fail without as_mut() above
        self.as_mut().project().counter += 1;
    }
    
    // Can still use self
    self.set(Self::Done);
    
    Poll::Ready(())
}
```

**Without `as_mut()`:**
```rust
// This won't compile:
let proj1 = self.project();  // self is moved
let proj2 = self.project();  // ERROR: self was moved
```

**Signature:**
```rust
impl<P: DerefMut> Pin<P> {
    pub fn as_mut(&mut self) -> Pin<&mut P::Target>;
}
```

### `as_ref()` — Downgrading to Shared Pin

`as_ref()` converts `Pin<&mut T>` to `Pin<&T>` (or `Pin<Box<T>>` to `Pin<&T>`):

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
    // Get a shared pinned reference
    let shared: Pin<&Self> = self.as_ref();
    
    // Call methods that only need Pin<&Self>
    if shared.is_complete() {
        return Poll::Ready(());
    }
    
    // ...
}

// Useful for calling methods on Pin<Box<T>>:
let boxed: Pin<Box<MyFuture>> = Box::pin(MyFuture::new());
let pinned_ref: Pin<&MyFuture> = boxed.as_ref();
pinned_ref.some_method();  // Method taking Pin<&Self>
```

### `map_unchecked_mut()` — Manual Pin Projection

For manual projections without `pin-project`:

```rust
impl Wrapper {
    fn project_inner(self: Pin<&mut Self>) -> Pin<&mut Inner> {
        // SAFETY: Inner is structurally pinned
        unsafe { self.map_unchecked_mut(|this| &mut this.inner) }
    }
}
```

**When to use:** Almost never—use `pin-project` instead. But useful for understanding what projection macros generate.

### `poll_fn` — Creating Futures from Closures

`std::future::poll_fn` creates a future from a poll function:

```rust
use std::future::poll_fn;
use std::task::Poll;

async fn example() {
    // Create a future that yields once then completes
    let mut yielded = false;
    let result = poll_fn(|cx| {
        //         ^^ Where does cx come from?
        if yielded {
            Poll::Ready(42)
        } else {
            yielded = true;
            cx.waker().wake_by_ref();  // Schedule re-poll
            Poll::Pending
        }
    }).await;  // ← The .await is key!
    
    println!("Got: {}", result);  // "Got: 42"
}
```

**How does the closure get `cx`?**

1. `poll_fn(|cx| ...)` returns a `Future`, it doesn't run the closure yet
2. When you `.await` it, the async runtime (tokio, async-std, etc.) polls the future
3. The runtime provides the `Context` containing the waker
4. That `Context` is passed to your closure each time it's polled

```rust
// poll_fn is roughly equivalent to:
struct PollFn<F>(F);

impl<F, T> Future for PollFn<F>
where
    F: FnMut(&mut Context<'_>) -> Poll<T>,
{
    type Output = T;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        // The runtime provides cx, we pass it to your closure
        (self.get_mut().0)(cx)
    }
}
```

**Practical example — wrapping a blocking check:**

```rust
use std::future::poll_fn;
use std::task::Poll;

async fn wait_for_file(path: &str) {
    poll_fn(|cx| {
        if std::path::Path::new(path).exists() {
            Poll::Ready(())
        } else {
            // Schedule a wake-up (in real code, use a proper watcher)
            let waker = cx.waker().clone();
            std::thread::spawn(move || {
                std::thread::sleep(std::time::Duration::from_millis(100));
                waker.wake();
            });
            Poll::Pending
        }
    }).await;
}
```

**Useful for:**
- Testing poll behavior
- Wrapping non-async APIs
- One-off futures without defining a struct

### Quick Reference Table

| Method/Macro | Purpose | Common Usage |
|--------------|---------|--------------|
| `ready!(expr)` | Early return on Pending | Inside `poll()` implementations |
| `self.set(val)` | Replace pinned value | State machine transitions |
| `self.as_mut()` | Reborrow Pin<&mut T> | Multiple uses in one poll |
| `self.as_ref()` | Pin<&mut T> → Pin<&T> | Calling shared methods |
| `pin!()` | Stack pinning | Local futures |
| `Box::pin()` | Heap pinning | Owned/returned futures |
| `project()` | Pin projection | Accessing pinned fields |
| `poll_fn()` | Closure → Future | Ad-hoc futures |

---

## Common Patterns and Pitfalls

### Pattern: Implementing Future

```rust
use pin_project::pin_project;

#[pin_project]
pub struct Map<F, Fn> {
    #[pin]
    future: F,
    f: Option<Fn>,  // NOT pinned - we need to take() it
}

impl<F: Future, Fn: FnOnce(F::Output) -> T, T> Future for Map<F, Fn> {
    type Output = T;
    
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let this = self.project();
        
        match this.future.poll(cx) {
            Poll::Ready(output) => {
                let f = this.f.take().expect("polled after completion");
                Poll::Ready(f(output))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}
```

**Note**: `f` is NOT pinned because we need `take()`.

### Pattern: State Machines

```rust
#[pin_project(project = StateProj)]
enum State<A, B> {
    First {
        #[pin]
        future: A,
    },
    Second {
        #[pin]
        future: B,
    },
    Done,
}

impl<A: Future, B: Future<Output = A::Output>> Future for State<A, B> {
    type Output = A::Output;
    
    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        match self.as_mut().project() {
            StateProj::First { future } => {
                let value = ready!(future.poll(cx));
                self.set(State::Done);
                Poll::Ready(value)
            }
            StateProj::Second { future } => {
                let value = ready!(future.poll(cx));
                self.set(State::Done);
                Poll::Ready(value)
            }
            StateProj::Done => panic!("polled after completion"),
        }
    }
}
```

### Pitfall: Double Pinning

```rust
// WRONG: Pinning something that's already pinned
#[pin_project]
struct Wrapper {
    #[pin]
    inner: Pin<Box<dyn Future<Output = ()>>>,  // Already pinned!
}

// CORRECT: Don't double-pin
#[pin_project]
struct Wrapper {
    // No #[pin] needed - Pin<Box<_>> handles its own pinning
    inner: Pin<Box<dyn Future<Output = ()>>>,
}
```

### Pitfall: Forgetting #[pin]

```rust
#[pin_project]
struct MyFuture<F> {
    inner: F,  // FORGOT #[pin]!
}

impl<F: Future> Future for MyFuture<F> {
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<...> {
        let this = self.project();
        // this.inner is &mut F, not Pin<&mut F>
        this.inner.poll(cx)  // ERROR: expected Pin<&mut F>
    }
}
```

### Pitfall: Moving Pinned Fields

```rust
#[pin_project]
struct Bad<F> {
    #[pin]
    future: F,
}

impl<F> Bad<F> {
    fn steal(self: Pin<&mut Self>) -> F {
        let this = self.project();
        // this.future is Pin<&mut F>
        
        // Can't do this - won't compile:
        // *this.future  // Can't move out of Pin
        
        // Can't do this either:
        // std::mem::replace(...)  // No &mut F access
        
        unreachable!()
    }
}
```

### Pitfall: Impl Unpin Incorrectly

```rust
#[pin_project]
struct MyType<T> {
    #[pin]
    inner: T,
}

// WRONG: pin-project will error if you try this
// unsafe impl<T> Unpin for MyType<T> {}

// CORRECT: Let pin-project handle Unpin
// MyType<T>: Unpin iff T: Unpin (automatic)
```

---

## Pin in Async Rust

### Why Futures Need Pin

The `Future` trait requires `Pin<&mut Self>`:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

This is because:
1. Futures are state machines generated by async/await
2. These state machines may contain self-references
3. Polling must not move the future

### The async/await Transformation

```rust
async fn fetch_data(url: &str) -> Data {
    let response = http::get(url).await;
    let text = response.text().await;
    parse(text)
}
```

Becomes conceptually:

```rust
enum FetchDataFuture<'a> {
    Start { url: &'a str },
    WaitingForResponse { 
        url: &'a str,
        get_future: HttpGetFuture,
    },
    WaitingForText {
        response: HttpResponse,
        text_future: TextFuture,
    },
    Done,
}

impl Future for FetchDataFuture<'_> {
    // poll() implementation that transitions between states
}
```

If `text_future` holds a reference to `response` (within the same enum), moving the enum would invalidate that reference.

### Using Pin with Futures

```rust
// Spawning a future (requires stable location)
tokio::spawn(async { ... });  // Internally boxes the future

// Selecting between futures
tokio::select! {
    result = future1 => { ... }
    result = future2 => { ... }
}
// Internally pins futures on the stack

// Manual polling
use std::future::Future;
use std::pin::Pin;

fn poll_future(future: Pin<&mut impl Future>) {
    let waker = /* ... */;
    let mut cx = Context::from_waker(&waker);
    let _ = future.poll(&mut cx);
}
```

### Boxing vs Stack Pinning in Async

**Box::pin when:**
- Returning futures from functions
- Storing in collections
- Type erasure needed (`dyn Future`)
- Lifetime/ownership complexities

```rust
fn make_future() -> Pin<Box<dyn Future<Output = i32> + Send>> {
    Box::pin(async { 42 })
}
```

**Stack pinning when:**
- Local future usage
- Performance critical
- Short-lived futures

```rust
async fn local_use() {
    let future = async { 42 };
    tokio::pin!(future);
    // Use pinned future locally
}
```

### The select! Macro and Pinning

`tokio::select!` pins futures on the stack:

```rust
tokio::select! {
    biased;  // First ready wins
    
    result = &mut future1 => {
        // future1 completed
    }
    result = &mut future2 => {
        // future2 completed
    }
}

// Expands roughly to:
// let mut future1 = ...;
// let mut future2 = ...;
// loop {
//     tokio::pin!(future1);
//     tokio::pin!(future2);
//     
//     if future1.poll(cx).is_ready() { ... }
//     if future2.poll(cx).is_ready() { ... }
// }
```

## Pin Beyond Async

While `Pin` is most commonly encountered in async Rust, it has several important non-async use cases.

### Intrusive Data Structures

Intrusive collections store pointers directly inside the elements, rather than in separate nodes. This is common in OS kernels and embedded systems for zero-allocation insertion.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;
use std::ptr::NonNull;

/// A node in an intrusive doubly-linked list.
/// The node contains pointers to siblings—moving it would break the list!
struct IntrusiveNode {
    value: i32,
    prev: Option<NonNull<IntrusiveNode>>,
    next: Option<NonNull<IntrusiveNode>>,
    _pin: PhantomPinned,  // Opt out of Unpin
}

impl IntrusiveNode {
    fn new(value: i32) -> Self {
        IntrusiveNode {
            value,
            prev: None,
            next: None,
            _pin: PhantomPinned,
        }
    }
    
    /// Link this node after `prev`. Both must be pinned!
    fn link_after(self: Pin<&mut Self>, mut prev: Pin<&mut Self>) {
        unsafe {
            let this = self.get_unchecked_mut();
            let prev_ptr = prev.as_mut().get_unchecked_mut();
            
            // Update our pointers
            this.prev = Some(NonNull::new_unchecked(prev_ptr));
            this.next = prev_ptr.next;
            
            // Update prev's next pointer
            prev_ptr.next = Some(NonNull::new_unchecked(this));
            
            // Update old next's prev pointer (if any)
            if let Some(mut old_next) = this.next {
                old_next.as_mut().prev = Some(NonNull::new_unchecked(this));
            }
        }
    }
}
```

**Why Pin?** If any node moves, the `prev`/`next` pointers in neighboring nodes become dangling. Pinning ensures nodes stay put once linked.

**Real-world examples:**
- Linux kernel's `list_head` pattern
- `tokio::sync::Notify` uses intrusive waitlists
- `parking_lot` uses intrusive queues for waiting threads

### FFI and C Interop

When passing Rust data to C code that stores pointers, you must ensure the data doesn't move:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

#[repr(C)]
struct CallbackData {
    value: i32,
    // C library stores a pointer to this struct
    _pin: PhantomPinned,
}

extern "C" {
    // C function that stores a pointer to our data
    fn register_callback(data: *mut CallbackData);
    fn unregister_callback(data: *mut CallbackData);
}

impl CallbackData {
    fn new(value: i32) -> Self {
        CallbackData { value, _pin: PhantomPinned }
    }
    
    /// Register with C library. Self must be pinned because C stores our address.
    fn register(self: Pin<&mut Self>) {
        unsafe {
            register_callback(self.get_unchecked_mut());
        }
    }
    
    /// Unregister. Still need Pin because we're still "in" the C library's view.
    fn unregister(self: Pin<&mut Self>) {
        unsafe {
            unregister_callback(self.get_unchecked_mut());
        }
    }
}

// Usage:
fn use_callback() {
    let mut data = Box::pin(CallbackData::new(42));
    data.as_mut().register();
    
    // ... do work ...
    // data MUST NOT be dropped or moved while registered!
    
    data.as_mut().unregister();
    // Now safe to drop
}
```

**Common C patterns requiring Pin:**
- Event loops storing pointers to handlers
- Async I/O libraries (io_uring, epoll with user data)
- Plugin systems with callback registration

### Generators and Coroutines

Rust's experimental generators (now called coroutines) are self-referential, just like async blocks:

```rust
#![feature(coroutines, coroutine_trait)]
use std::ops::{Coroutine, CoroutineState};
use std::pin::Pin;

fn main() {
    let mut coroutine = || {
        let x = vec![1, 2, 3];
        let slice = &x[..];  // Self-reference across yield!
        yield slice.len();
        yield slice[0] as usize;
    };
    
    // Must pin before resuming
    let mut pinned = Pin::new(&mut coroutine);
    
    match pinned.as_mut().resume(()) {
        CoroutineState::Yielded(len) => println!("Length: {}", len),
        _ => {}
    }
    match pinned.as_mut().resume(()) {
        CoroutineState::Yielded(first) => println!("First: {}", first),
        _ => {}
    }
}
```

The `Coroutine::resume` method requires `Pin<&mut Self>` for the same reason `Future::poll` does.

### Self-Referential Configuration

Sometimes you want a struct that holds both data and references into that data, for efficient access:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

/// A parsed config that maintains pointers into the original string.
struct ParsedConfig {
    raw: String,
    // These point into `raw` - classic self-reference
    name: *const str,
    values: Vec<*const str>,
    _pin: PhantomPinned,
}

impl ParsedConfig {
    /// Create unparsed config
    fn new(raw: String) -> Self {
        ParsedConfig {
            raw,
            name: std::ptr::null(),
            values: Vec::new(),
            _pin: PhantomPinned,
        }
    }
    
    /// Parse the config. Must be pinned first!
    fn parse(self: Pin<&mut Self>) {
        unsafe {
            let this = self.get_unchecked_mut();
            
            // Parse and store pointers into `raw`
            let mut lines = this.raw.lines();
            if let Some(name_line) = lines.next() {
                this.name = name_line as *const str;
            }
            for line in lines {
                this.values.push(line as *const str);
            }
        }
    }
    
    /// Access parsed name (safe after parse)
    fn name(&self) -> &str {
        unsafe { &*self.name }
    }
}

// Usage:
fn use_config() {
    let config = ParsedConfig::new("MyApp\nvalue1\nvalue2".to_string());
    let mut pinned = Box::pin(config);
    pinned.as_mut().parse();
    
    println!("Name: {}", pinned.name());  // Zero-copy access!
}
```

**Trade-off:** This pattern avoids copying parsed strings, but requires careful handling of pinning. For most cases, just clone the strings—it's simpler and fast enough.

### Memory-Mapped I/O and Fixed Addresses

When working with memory-mapped hardware or files, data must stay at specific addresses:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

/// Represents a memory region that must not move.
struct MappedRegion {
    base: *mut u8,
    len: usize,
    // Registered with hardware/kernel—address is fixed
    _pin: PhantomPinned,
}

impl MappedRegion {
    /// Access the mapped memory. Must be pinned because the mapping 
    /// is tied to this struct's address in some hardware/kernel table.
    fn write(self: Pin<&mut Self>, offset: usize, data: &[u8]) {
        assert!(offset + data.len() <= self.len);
        unsafe {
            let base = self.get_unchecked_mut().base;
            std::ptr::copy_nonoverlapping(data.as_ptr(), base.add(offset), data.len());
        }
    }
}
```

### When NOT to Use Pin for These Cases

Pin isn't always the right tool:

| Situation | Better Alternative |
|-----------|-------------------|
| Simple arena allocation | Use indices instead of pointers |
| Graph structures | Use `petgraph` or index-based graphs |
| Parent-child references | Use `Rc`/`Arc` + `Weak` |
| Cached computations | Use `OnceCell` or `lazy_static` |
| Short-lived references | Just use lifetimes normally |

**Rule of thumb:** Only reach for `Pin` when:
1. You truly need self-references or fixed addresses
2. The data lives long enough that moves are a real concern
3. Simpler patterns (indices, Rc/Weak) don't fit

---

## Summary

### Key Concepts

| Concept | Meaning |
|---------|---------|
| `Pin<P>` | Wrapper guaranteeing the pointee won't move |
| `Unpin` | Marker trait: "pinning doesn't matter for this type" |
| `!Unpin` | Type that must not be moved once pinned |
| Structural pinning | Pinning a struct implies pinning certain fields |
| Pin projection | Getting `Pin<&mut Field>` from `Pin<&mut Struct>` |
| `PhantomPinned` | Marker to make a type `!Unpin` |
| `PinnedDrop` | Drop implementation with `Pin<&mut Self>` |

### Decision Tree

```
Do I need pinning?
├── Is the type self-referential or contains futures? 
│   ├── Yes → Needs pinning
│   └── No → Probably doesn't need pinning (Unpin is fine)
│
└── Where should I pin?
    ├── Need to return/store/erase type? → Box::pin
    ├── Local usage, performance matters? → Stack pin (pin!/Pin::new_unchecked)
    └── In a struct with other fields? → Use pin-project for projections
```

### Rules of Thumb

1. **Most types are Unpin** - Don't add pinning complexity unless needed
2. **Use pin-project** - Don't write manual projections
3. **Prefer Box::pin for simplicity** - Optimize later if needed
4. **#[pin] means "can't move"** - Don't pin fields you need to take/swap
5. **Drop happens at the end** - The pin guarantee ends at drop

### Further Reading

- [std::pin documentation](https://doc.rust-lang.org/std/pin/)
- [pin-project crate](https://docs.rs/pin-project)
- [Rust Async Book - Pinning](https://rust-lang.github.io/async-book/04_pinning/01_chapter.html)
- [Jon Gjengset's Pin explanation](https://www.youtube.com/watch?v=DkMwYxfSYNQ)
