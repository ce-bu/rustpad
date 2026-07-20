# Architecture Blueprint: Type-Dispatched Strategy Pattern in Rust

Type-Dispatched Architecture (also known as the Typestate or Phantom Type pattern) shifts your system's business rules, states, and strategies out of runtime execution and directly into the compile-time type checker. 

Instead of checking runtime conditions, you pass the strategy as a generic type parameter. The compiler uses monomorphization to generate entirely separate, highly optimized machine code paths for each variation, resulting in zero runtime overhead and absolute compile-time safety.

---

## Architectural Core

A pure type-dispatched architecture consists of three decoupled components:

1. Strategy Markers (Zero-Sized Types): Stateless structures that act as compiler flags. They occupy exactly 0 bytes of memory.
2. The Phantomed Container: Your data layout (e.g., NodeRef), which binds these markers using std::marker::PhantomData without inflating its physical memory footprint.
3. The Static Dispatcher (Trait): An interface that resolves into specific behavior based strictly on the bound type parameter.

---

## 💻 Implementation Blueprint

Here is a complete implementation of a compile-time strategy pattern designed for custom data structures (like augmented search trees).
```
use anyhow::Result;
use std::{fmt::Debug, marker::PhantomData};

trait AugmentationStrategy {}

struct NullAugment;
struct FullAugment;
impl AugmentationStrategy for NullAugment {}
impl AugmentationStrategy for FullAugment {}

struct NodeRef<T, AS> {
    value: T,
    _marker: PhantomData<AS>,
}

impl<T: Clone, AS> Clone for NodeRef<T, AS> {
    fn clone(&self) -> Self {
        NodeRef {
            value: self.value.clone(),
            _marker: PhantomData,
        }
    }
}

impl<T, AS> NodeRef<T, AS> {
    fn new(value: T) -> Self {
        NodeRef {
            value,
            _marker: PhantomData,
        }
    }
}

trait ExecuteAugmentation {
    fn execute(self);
}

impl<T> ExecuteAugmentation for NodeRef<T, FullAugment> {
    fn execute(self) {
        println!("Executing FullAugment");
    }
}

impl<T> ExecuteAugmentation for NodeRef<T, NullAugment> {
    fn execute(self) {
        println!("Executing NullAugment");
    }
}

trait PerformAugmentation {
    fn augment(self);
}

impl<T, AS> PerformAugmentation for NodeRef<T, AS>
where
    Self: ExecuteAugmentation,
    AS: AugmentationStrategy,
{
    fn augment(self) {
        self.execute();
    }
}

struct MyCollection<T, AS: AugmentationStrategy> {
    items: Vec<NodeRef<T, AS>>,
    _marker: PhantomData<AS>,
}

// Added the ExecuteAugmentation bound here
impl<T: Clone, AS: AugmentationStrategy> MyCollection<T, AS>
where
    NodeRef<T, AS>: ExecuteAugmentation,
{
    fn new() -> Self {
        MyCollection {
            items: Vec::new(),
            _marker: PhantomData,
        }
    }

    fn add(&mut self, item: T) {
        self.items.push(NodeRef::new(item));
    }

    fn traverse(&self) {
        for item in &self.items {
            item.clone().augment();
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let mut coll = MyCollection::<i32, NullAugment>::new();
    coll.add(1);
    coll.add(2);
    coll.traverse();
    Ok(())
}
```

---

## Strategic Benefits

* No Performance Tax: Zero runtime match blocks, zero dynamic trait objects, and absolutely zero branch mispredictions for your CPU.
* Hermetic Compile Safety: It is structurally impossible to accidentally insert a LazyAugment node into an EagerAugment tree layout; the compiler will reject the code immediately.
* Algorithmic Cleanliness: Your balancing and rotation functions can simply invoke node.augment_upstream() without tracking state variables or passing execution toggles.
