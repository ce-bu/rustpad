# The Entity Component System (ECS) Pattern in Rust

**Estimated reading time: ~10 minutes**

## What Is ECS?

Entity Component System is an architectural pattern that organises data and behaviour into three distinct concepts:

| Concept       | Role |
|---------------|------|
| **Entity**    | A lightweight identifier (usually just an integer or index). It has no data or behaviour of its own. |
| **Component** | A plain data struct attached to an entity — position, velocity, health, sprite, etc. |
| **System**    | A function that iterates over entities possessing a specific combination of components and applies logic. |

```text
 Entity 0  ──►  Position { x: 1.0, y: 2.0 }
                 Velocity { dx: 0.5, dy: -0.3 }

 Entity 1  ──►  Position { x: 4.0, y: 7.0 }
                 Health   { hp: 100 }

 System: move_system  ──  for each (Position, Velocity) pair → update position
```

The key departure from traditional OOP:
- **No inheritance hierarchies.** Behaviour is composed by attaching components.
- **Data is stored by type, not by object.** All positions live together, all velocities live together.
- **Logic lives in systems, not in the data.**

---

## Why ECS Is a Natural Fit for Rust's Borrow Checker

Rust's ownership rules are simple to state but famously difficult to satisfy in object-graph-heavy code:

1. A value has exactly **one owner**.
2. You may have **either** one `&mut T` **or** any number of `&T` — never both.
3. References must not outlive the data they point to.

These rules make traditional OOP patterns — deep inheritance, back-pointers, observer graphs — awkward or impossible without `Rc<RefCell<T>>` wrappers everywhere. ECS side-steps these problems entirely.

### 1. Columnar storage eliminates shared mutable aliasing

In an OOP design, a game object might look like this:

```rust
// OOP-style: one struct owns everything
struct GameObject {
    position: Position,
    velocity: Velocity,
    health: Health,
    // ...
}
```

A `move_system` needs `&mut position` and `&velocity`. A `damage_system` needs `&mut health`. If both systems want to operate on the same `GameObject`, you hit a borrow conflict:

```rust
fn move_system(obj: &mut GameObject)   { /* borrows all of obj */ }
fn damage_system(obj: &mut GameObject) { /* also borrows all of obj */ }

// ERROR: cannot borrow `obj` as mutable more than once
move_system(&mut obj);
// ...fine on its own, but composing systems is painful.
```

ECS stores components in **separate, homogeneous collections** — often called "tables" or "archetypes":

```rust
struct World {
    positions:  Vec<Option<Position>>,
    velocities: Vec<Option<Velocity>>,
    healths:    Vec<Option<Health>>,
}
```

Now the borrow checker is perfectly happy:

```rust
fn move_system(positions: &mut [Option<Position>],
               velocities: &[Option<Velocity>]) {
    for (pos, vel) in positions.iter_mut().zip(velocities.iter()) {
        if let (Some(p), Some(v)) = (pos, vel) {
            p.x += v.dx;
            p.y += v.dy;
        }
    }
}

fn damage_system(healths: &mut [Option<Health>]) {
    // ...
}
```

**Each system borrows only the columns it needs.** Two systems that touch disjoint sets of components can even run in parallel with zero unsafe code — the compiler proves they don't alias.

### 2. No back-pointers, no reference cycles

In OOP it's common for children to hold references to parents, or for an observer to hold a reference to the subject. These bidirectional references are a nightmare for Rust's ownership model.

ECS avoids this entirely. Entities reference each other **by ID**, not by pointer:

```rust
struct Follow {
    target: EntityId,  // just a u32 / usize — no reference, no lifetime
}
```

Looking up the target's position is an index into a `Vec`, not a pointer dereference. There are no lifetimes, no reference cycles, no need for `Rc` or `Arc`.

### 3. Ownership is flat and unambiguous

In ECS, the `World` struct **owns** all component storage. Entities are just indices. Systems are stateless functions that borrow slices from the world. The ownership graph is a shallow tree:

```text
World
 ├── positions:  Vec<...>   (owner)
 ├── velocities: Vec<...>   (owner)
 └── healths:    Vec<...>   (owner)

Systems borrow &mut / & slices temporarily
```

There is never a question of "who owns this data?" — it is always the world.

### 4. Natural parallelism from disjoint borrows

Because component storage is split by type, the compiler can statically guarantee that two systems touching different component types never alias:

```rust
use std::thread;

// These two closures borrow DIFFERENT fields of `world`.
// Rust's borrow checker allows this via disjoint field borrows.
let handle = thread::scope(|s| {
    s.spawn(|| move_system(&mut world.positions, &world.velocities));
    s.spawn(|| damage_system(&mut world.healths));
});
```

Frameworks like **Bevy** automate this: they analyse each system's component access at startup and schedule non-conflicting systems on a thread pool — all without `unsafe`.

---

## A Minimal ECS From Scratch

Below is a tiny ECS to illustrate the core ideas (no external crates):

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;

/// A unique entity identifier.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct Entity(usize);

/// Type-erased column of components.
struct ComponentVec {
    data: Vec<Option<Box<dyn Any>>>,
}

impl ComponentVec {
    fn new() -> Self {
        Self { data: Vec::new() }
    }

    fn push_none(&mut self) {
        self.data.push(None);
    }

    fn set<T: 'static>(&mut self, index: usize, component: T) {
        self.data[index] = Some(Box::new(component));
    }

    fn get<T: 'static>(&self, index: usize) -> Option<&T> {
        self.data[index]
            .as_ref()
            .and_then(|b| b.downcast_ref::<T>())
    }

    fn get_mut<T: 'static>(&mut self, index: usize) -> Option<&mut T> {
        self.data[index]
            .as_mut()
            .and_then(|b| b.downcast_mut::<T>())
    }
}

/// The world holds all entities and their components.
struct World {
    entity_count: usize,
    components: HashMap<TypeId, ComponentVec>,
}

impl World {
    fn new() -> Self {
        Self {
            entity_count: 0,
            components: HashMap::new(),
        }
    }

    /// Create a new entity (returns its ID).
    fn spawn(&mut self) -> Entity {
        let id = self.entity_count;
        self.entity_count += 1;

        // Extend every existing column with None for the new entity.
        for column in self.components.values_mut() {
            column.push_none();
        }

        Entity(id)
    }

    /// Attach a component to an entity.
    fn insert<T: 'static>(&mut self, entity: Entity, component: T) {
        let type_id = TypeId::of::<T>();

        let column = self.components.entry(type_id).or_insert_with(|| {
            let mut col = ComponentVec::new();
            // Back-fill None for all existing entities.
            for _ in 0..self.entity_count {
                col.push_none();
            }
            col
        });

        column.set(entity.0, component);
    }

    /// Read a component for an entity.
    fn get<T: 'static>(&self, entity: Entity) -> Option<&T> {
        self.components
            .get(&TypeId::of::<T>())
            .and_then(|col| col.get::<T>(entity.0))
    }

    /// Mutably access a component for an entity.
    fn get_mut<T: 'static>(&mut self, entity: Entity) -> Option<&mut T> {
        self.components
            .get_mut(&TypeId::of::<T>())
            .and_then(|col| col.get_mut::<T>(entity.0))
    }
}
```

### Using the mini-ECS

```rust
#[derive(Debug)]
struct Position { x: f32, y: f32 }

#[derive(Debug)]
struct Velocity { dx: f32, dy: f32 }

#[derive(Debug)]
struct Health { hp: i32 }

fn main() {
    let mut world = World::new();

    let player = world.spawn();
    world.insert(player, Position { x: 0.0, y: 0.0 });
    world.insert(player, Velocity { dx: 1.0, dy: 0.5 });
    world.insert(player, Health { hp: 100 });

    let tree = world.spawn();
    world.insert(tree, Position { x: 5.0, y: 3.0 });
    world.insert(tree, Health { hp: 50 });

    // A simple "move" system — operates only on Position + Velocity.
    for id in 0..world.entity_count {
        let entity = Entity(id);
        if let (Some(vel_dx), Some(vel_dy)) = (
            world.get::<Velocity>(entity).map(|v| v.dx),
            world.get::<Velocity>(entity).map(|v| v.dy),
        ) {
            if let Some(pos) = world.get_mut::<Position>(entity) {
                pos.x += vel_dx;
                pos.y += vel_dy;
            }
        }
    }

    println!("{:?}", world.get::<Position>(player)); // Position { x: 1.0, y: 0.5 }
    println!("{:?}", world.get::<Position>(tree));    // Position { x: 5.0, y: 3.0 }
}
```

---

## ECS vs. OOP: Borrow Checker Friction Comparison

| Scenario | OOP Approach | Borrow Checker Pain | ECS Approach | Borrow Checker Pain |
|----------|-------------|---------------------|--------------|---------------------|
| Two systems mutating different aspects of the same object | `&mut self` on the whole object | **High** — overlapping borrows | Borrow separate component columns | **None** |
| Parent ↔ child references | `Rc<RefCell<T>>` or raw pointers | **High** — runtime panics or `unsafe` | Entity IDs (plain integers) | **None** |
| Adding new behaviour | New trait impl or inheritance | **Medium** — trait object lifetimes | Attach new component + new system | **None** |
| Iterating heterogeneous collections | `Vec<Box<dyn Trait>>` with dynamic dispatch | **Medium** — lifetime coercion | Typed, homogeneous `Vec<T>` per component | **None** |
| Parallel execution | Manual locking (`Mutex`, `RwLock`) | **High** — deadlocks, contention | Automatic disjoint-borrow scheduling | **None** |

---

## The Archetype Model (How Real ECS Frameworks Do It)

Production ECS implementations like **Bevy** and **hecs** use an **archetype** model for performance. Instead of one `Vec` per component type with `Option` gaps, entities are grouped by their set of components:

```text
Archetype A: (Position, Velocity)
  ┌───────────────┬──────────────────┐
  │ Position[]    │ Velocity[]       │
  ├───────────────┼──────────────────┤
  │ {x:0, y:0}   │ {dx:1, dy:0.5}   │
  │ {x:3, y:1}   │ {dx:0, dy:-1}    │
  └───────────────┴──────────────────┘

Archetype B: (Position, Health)
  ┌───────────────┬───────────┐
  │ Position[]    │ Health[]  │
  ├───────────────┼───────────┤
  │ {x:5, y:3}   │ {hp:50}   │
  └───────────────┴───────────┘
```

Benefits:
- **No `Option` holes** — every slot is occupied, so iteration is dense.
- **Cache-friendly** — components accessed together are stored contiguously.
- **Fast queries** — finding all entities with `(Position, Velocity)` is a direct archetype lookup, not a per-entity filter.

The borrow checker story is the same: systems borrow archetype tables by component type, and disjoint queries run in parallel.

---

## Bevy Example: ECS in Practice

[Bevy](https://bevyengine.org/) is the most popular Rust ECS framework. Here's how the pattern looks with Bevy's API:

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Position { x: f32, y: f32 }

#[derive(Component)]
struct Velocity { dx: f32, dy: f32 }

#[derive(Component)]
struct Health { hp: i32 }

/// System: moves entities that have both Position and Velocity.
fn move_system(mut query: Query<(&mut Position, &Velocity)>) {
    for (mut pos, vel) in &mut query {
        pos.x += vel.dx;
        pos.y += vel.dy;
    }
}

/// System: prints the health of every entity that has Health.
fn health_report(query: Query<(Entity, &Health)>) {
    for (entity, health) in &query {
        println!("{entity:?} has {} HP", health.hp);
    }
}

fn setup(mut commands: Commands) {
    // Player
    commands.spawn((
        Position { x: 0.0, y: 0.0 },
        Velocity { dx: 1.0, dy: 0.5 },
        Health { hp: 100 },
    ));

    // Tree
    commands.spawn((
        Position { x: 5.0, y: 3.0 },
        Health { hp: 50 },
    ));
}

fn main() {
    App::new()
        .add_systems(Startup, setup)
        .add_systems(Update, (move_system, health_report))
        .run();
}
```

Bevy analyses the `Query` parameters at schedule-time:
- `move_system` reads `Velocity` and writes `Position`.
- `health_report` reads `Health`.

These access sets are **disjoint**, so Bevy runs them on separate threads automatically. If two systems *did* conflict, Bevy would sequence them. All of this is enforced at compile time through Rust's type system — no runtime locks needed.

---

## When to Reach for ECS

ECS is not a silver bullet. Use it when:

- You have **many entities** with **varying combinations** of data and behaviour (games, simulations, data pipelines).
- You need **parallel iteration** over large datasets and want the compiler to guarantee safety.
- Your domain is **composition-heavy** rather than hierarchy-heavy.
- You want to **avoid `Rc<RefCell<T>>`** and the associated runtime borrow-check panics.

For simpler applications — CLI tools, web servers, CRUD apps — standard structs with owned data are usually sufficient. ECS adds architectural complexity that only pays off at scale.

---

## Summary

| ECS Principle | Why It Aligns With Rust |
|--------------|------------------------|
| Data stored by type in separate collections | Enables disjoint `&mut` borrows — no aliasing conflicts |
| Entities are integer IDs, not pointers | No lifetimes, no reference cycles, no `Rc`/`RefCell` |
| Systems are stateless functions | Borrow what they need, release it when done |
| Flat ownership (World owns everything) | Single owner, clear drop order, no graph cycles |
| Composition over inheritance | No `dyn Trait` lifetime headaches, no vtable indirection |

The ECS pattern succeeds in Rust not by fighting the borrow checker, but by **structuring data so the borrow checker's rules become trivially satisfiable**. Separate storage per component type turns what would be overlapping `&mut` borrows in OOP into provably disjoint accesses — unlocking safe parallelism as a free bonus.
