# Rust Tower: A Comprehensive Tutorial

**Estimated reading time: ~2.5 hours**

Tower is a library of modular and reusable components for building robust networking clients and servers with Rust. It's the backbone of many high-performance systems. This tutorial explores Tower's core concepts with practical examples.

## Table of Contents

1. [Introduction to Tower](#introduction-to-tower)
2. [The Service Trait Deep Dive](#the-service-trait-deep-dive)
3. [The Layer Stack](#the-layer-stack)
4. [Understanding Pin and Pinning](#understanding-pin-and-pinning)
5. [The NewService Pattern](#the-newservice-pattern)
6. [Advanced Routing Patterns](#advanced-routing-patterns)
7. [Backpressure Mechanisms](#backpressure-mechanisms)
8. [Zero-Copy Techniques](#zero-copy-techniques)
9. [Error Handling and Resilience](#error-handling-and-resilience)
10. [Practical Examples](#practical-examples)
11. [High Performance Tower Service Optimization](#high-performance-tower-service-optimization)

## Introduction to Tower

Tower is built around a few core abstractions:

- **Service**: The fundamental trait representing an asynchronous function from a request to a response
- **Layer**: A higher-order component that transforms one service into another
- **MakeService/NewService**: A factory for creating services

These abstractions enable powerful composition patterns, allowing developers to build complex systems from simple, reusable components.

Tower's design is heavily influenced by futures and async/await in Rust, making it a natural fit for high-performance networking applications.

### The Mental Model

Think of a Tower service like a function with superpowers. A regular function takes an input and returns an output:

```rust
fn process(request: Request) -> Response
```

A Tower service is similar, but:

1. **Asynchronous**: It returns a `Future` instead of blocking, allowing efficient use of resources
2. **Fallible**: It can return errors through the `Result` type
3. **Stateful**: It can maintain internal state and signal when it's ready to accept requests
4. **Composable**: Multiple services can be stacked using layers to add behavior

The key insight is that by making services uniform in shape (request in, future of response out), Tower enables powerful middleware patterns similar to what you'd find in web frameworks—but at a more fundamental level that works with any protocol.

### Tower in High-Performance Networking

Tower plays a critical role in modern high-performance networking applications in Rust for several reasons:

1. **Composable Middleware**: Tower's layer system allows for clean separation of concerns. Each layer handles one aspect of request processing (logging, rate limiting, authentication) without knowledge of other layers.

2. **Backpressure Propagation**: Tower's `poll_ready` mechanism provides built-in backpressure handling, essential for preventing resource exhaustion in high-load systems.

3. **Protocol Agnosticism**: Tower abstractions work with any protocol (HTTP, gRPC, custom protocols) since the `Service` trait is generic over the request and response types.

4. **Resource Management**: Tower helps manage scarce resources like connections, file handles, and memory through proper backpressure.

5. **Concurrency Control**: Tower services can limit concurrency, ensuring systems don't become overwhelmed during traffic spikes.

In production systems like Tower enables:

```rust
// A typical Tower stack in a production service mesh
let proxy_service = ServiceBuilder::new()
    .layer(TraceLayer::new_for_http()) // Distributed tracing
    .layer(TimeoutLayer::new(Duration::from_secs(30))) // Request timeouts
    .layer(ConcurrencyLimitLayer::new(100)) // Limit concurrent requests
    .layer(RateLimitLayer::new(50, Duration::from_secs(1))) // Rate limiting
    .layer(RetryLayer::new(retry_policy)) // Automatic retries
    .layer(LoadBalancerLayer::new(discover_backends)) // Load balancing
    .service(HttpService::new()); // Base HTTP service
```

This stack provides a complete set of reliability features while maintaining clean separation between components. Each layer can be tested, configured, and reasoned about independently.

## The Service Trait Deep Dive

The `Service` trait is the cornerstone of Tower. Before we look at the code, let's understand what problem it solves.

### The Problem Service Solves

In async Rust, you might write request handling like this:

```rust
async fn handle(request: Request) -> Result<Response, Error> {
    // process request
}
```

But this simple signature lacks important capabilities:
- **No backpressure**: How does the caller know if you're overwhelmed with requests?
- **No state management**: How do you track connections, rate limits, or resources?
- **No composition**: How do you add logging, timeouts, or retries uniformly?

The `Service` trait addresses all of these by formalizing the request-response pattern:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

Let's examine each component:

- **`type Response`**: The successful response type. Generic to support HTTP responses, TCP streams, or any protocol.
- **`type Error`**: The error type. Allows services to define their own error semantics.
- **`type Future`**: The future type returned by `call()`. This allows efficient, non-allocating futures when possible.
- **`poll_ready()`**: The backpressure mechanism. Returns `Poll::Ready(Ok(()))` when the service can accept requests.
- **`call()`**: The actual request processing. Takes `&mut self` allowing stateful services.

### Why `poll_ready` is the Secret Sauce

The `poll_ready` method is what makes Tower particularly powerful for building reliable distributed systems. It serves as a backpressure mechanism, allowing services to signal when they're ready to accept new requests.

#### The Critical Role in Reliability

In a microservice app `poll_ready` enables several critical reliability features:

1. **Load Balancing**: Before sending a request to a backend, a proxy checks if that backend is ready to receive requests
2. **Circuit Breaking**: If a backend is failing, `poll_ready` will return `Poll::Pending` or an error
3. **Rate Limiting**: Services can use `poll_ready` to implement rate limiting by only becoming ready at a certain cadence
4. **Connection Management**: Ensures connections are established before sending requests

#### The Failure Mode: Calling `call()` Before `poll_ready` Returns Ready

A critical mistake developers can make is calling `call()` before `poll_ready` returns `Poll::Ready`. This can lead to several serious failure modes:

```rust
// INCORRECT USAGE - DO NOT DO THIS
let response = service.call(request).await?; // Dangerous!

// CORRECT USAGE
service.ready().await?; // This calls poll_ready internally
let response = service.call(request).await?;
```

If you call `call()` before the service is ready:

1. **Resource Exhaustion**: The service might not have the resources to handle the request, leading to memory leaks or OOM errors
2. **Connection Failures**: If the connection isn't established yet, the request will fail
3. **Cascading Failures**: In a distributed system, this can lead to cascading failures as services become overwhelmed
4. **Inconsistent State**: The service might be in an inconsistent state, leading to corrupt responses

In a production system specifically, this could mean:
- Sending requests to unhealthy backends
- Overwhelming backends that are already struggling
- Breaking circuit breaker patterns
- Bypassing rate limits

The `ServiceExt` trait provides the `.ready()` method which safely waits for `poll_ready` to return `Poll::Ready` before allowing `call()` to be invoked, making it much easier to use services correctly.

## The Layer Stack

Layers are Tower's mechanism for middleware composition. A `Layer` transforms one service into another, adding behavior like logging, timeouts, or rate limiting.

### Understanding Layers Conceptually

The key insight is that layers don't directly handle requests—they wrap services. Think of it as the decorator pattern:

```
     Layer wraps Service
┌─────────────────────────────┐
│  Layer (e.g., Timeout)      │
│  ┌────────────────────────┐ │
│  │  Layer (e.g., Logging) │ │
│  │  ┌───────────────────┐ │ │
│  │  │  Your Service     │ │ │
│  │  └───────────────────┘ │ │
│  └────────────────────────┘ │
└─────────────────────────────┘
```

When a request arrives:
1. The outer layer (Timeout) receives it first
2. It passes to the next layer (Logging)
3. Which finally passes to your service
4. The response travels back through the layers in reverse

The `Layer` trait is simple:

```rust
pub trait Layer<S> {
    type Service;
    fn layer(&self, inner: S) -> Self::Service;
}
```

It takes a service `S` and returns a new service (of type `Self::Service`) that wraps it.

### Composing Layers

Here's how to compose multiple layers using Tower's `ServiceBuilder`:

```rust
let service = ServiceBuilder::new()
    .layer(RequestTimingLayer::new())
    .layer(ProtocolDetectionLayer::new())
    .service(MainService::new());
```

**Important**: Layer order matters! Layers are applied inside-out. In the example above:
- `ProtocolDetectionLayer` wraps `MainService`
- `RequestTimingLayer` wraps the result

So requests flow: Timing → Protocol → Main → Protocol → Timing

### Custom Layer Examples

Let's implement two custom layers: one for request timing and one for protocol detection. We'll walk through the concepts as we build them.

#### Request Timing Layer

A timing layer measures how long requests take. It needs to:
1. Record the start time before calling the inner service
2. Wait for the response
3. Calculate and log the duration

The implementation has two parts: the Layer (configuration) and the Service (behavior):

```rust
// A service that measures request timing
struct RequestTimingService<S> {
    inner: S,
}

impl<S, R> Service<R> for RequestTimingService<S>
where
    S: Service<R>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<Output = Result<S::Response, S::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, request: R) -> Self::Future {
        let start = Instant::now();
        let future = self.inner.call(request);

        Box::pin(async move {
            let response = future.await?;
            let duration = start.elapsed();
            println!("Request took {:?}", duration);
            Ok(response)
        })
    }
}

// The layer that wraps services with RequestTimingService
struct RequestTimingLayer;

impl RequestTimingLayer {
    fn new() -> Self {
        RequestTimingLayer
    }
}

impl<S> Layer<S> for RequestTimingLayer {
    type Service = RequestTimingService<S>;

    fn layer(&self, service: S) -> Self::Service {
        RequestTimingService { inner: service }
    }
}
```

#### Protocol Detection Layer

```rust
// A service that detects the protocol of incoming requests
struct ProtocolDetectionService<S> {
    inner: S,
}

impl<S> Service<http::Request<hyper::Body>> for ProtocolDetectionService<S>
where
    S: Service<http::Request<hyper::Body>>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, mut request: http::Request<hyper::Body>) -> Self::Future {
        // Detect protocol from headers or other request properties
        let protocol = if request.headers().contains_key("upgrade") {
            "websocket"
        } else if request.uri().path().ends_with(".grpc") {
            "grpc"
        } else {
            "http"
        };

        // Add protocol information to request extensions
        request.extensions_mut().insert(Protocol(protocol.to_string()));
        
        self.inner.call(request)
    }
}

// Protocol information to store in request extensions
#[derive(Clone, Debug)]
struct Protocol(String);

// The layer that wraps services with ProtocolDetectionService
struct ProtocolDetectionLayer;

impl ProtocolDetectionLayer {
    fn new() -> Self {
        ProtocolDetectionLayer
    }
}

impl<S> Layer<S> for ProtocolDetectionLayer {
    type Service = ProtocolDetectionService<S>;

    fn layer(&self, service: S) -> Self::Service {
        ProtocolDetectionService { inner: service }
    }
}
```

### Combining Layers with a Main Service

```rust
// A simple main service that handles requests
struct MainService;

impl MainService {
    fn new() -> Self {
        MainService
    }
}

impl Service<http::Request<hyper::Body>> for MainService {
    type Response = http::Response<hyper::Body>;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, request: http::Request<hyper::Body>) -> Self::Future {
        // Extract protocol information if available
        let protocol = request.extensions()
            .get::<Protocol>()
            .map(|p| p.0.clone())
            .unwrap_or_else(|| "unknown".to_string());

        Box::pin(async move {
            // Process the request based on protocol or other factors
            let response = http::Response::builder()
                .status(200)
                .body(hyper::Body::from(format!("Processed {} request", protocol)))
                .unwrap();
            
            Ok(response)
        })
    }
}

// Putting it all together
fn create_service_stack() -> impl Service<http::Request<hyper::Body>> {
    ServiceBuilder::new()
        .layer(RequestTimingLayer::new())
        .layer(ProtocolDetectionLayer::new())
        .service(MainService::new())
}
```

## Understanding Pin and Pinning

Pinning is a fundamental concept in Rust's async ecosystem that's essential for writing Tower services correctly. While it may seem abstract at first, understanding pinning is crucial for:

- Implementing custom futures
- Creating services with complex async state
- Debugging lifetime and borrow errors in async code
- Working with self-referential data structures

### Why Pinning Exists

To understand pinning, we first need to understand why it's necessary. Consider what happens when Rust compiles async functions:

```rust
async fn example() -> String {
    let data = String::from("hello");
    some_async_operation().await;
    data.to_uppercase() // Uses `data` after the await point
}
```

The compiler transforms this into a state machine—a struct that stores all local variables that live across `.await` points. Conceptually, it looks something like:

```rust
enum ExampleFuture {
    State0 { data: String },  // Before the await
    State1 { data: String },  // After the await, before returning
    Completed,
}
```

The problem arises when a future contains **self-references**—pointers from one field to another within the same struct. Consider this example:

```rust
async fn self_referential() -> &str {
    let data = String::from("hello");
    let slice: &str = &data;    // `slice` points to `data`
    some_async_operation().await;
    slice  // Return the slice
}
```

The generated state machine would need to store both `data` and `slice`, where `slice` is a reference pointing into `data`:

```rust
struct SelfReferentialFuture {
    data: String,
    slice: &str,  // Points into `data` above!
}
```

**The core problem**: If this struct is moved in memory, the `slice` pointer would become invalid because it still points to the old memory location. This is undefined behavior in Rust.

### What Pin Guarantees

`Pin<P>` is a wrapper that provides a crucial guarantee: **the pointee of `P` will not be moved**. This means:

```rust
// Once you have a Pin<&mut T>, you cannot get a &mut T
// (which would allow moving T) without unsafe code
let pinned: Pin<&mut MyFuture> = ...;

// This is NOT allowed without T: Unpin
let mutable_ref: &mut MyFuture = pinned.get_mut(); // Error!
```

The `Unpin` trait is an auto-trait that most types implement. It says "this type doesn't care about being pinned—it's safe to move even after pinning." Most types in Rust are `Unpin` because they don't contain self-references.

### The Pin API

Here's how to work with `Pin`:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

// Creating pinned values on the heap (common pattern)
let boxed: Pin<Box<MyFuture>> = Box::pin(MyFuture::new());

// Creating pinned references (requires the value is already pinned)
let pinned_ref: Pin<&MyFuture> = boxed.as_ref();
let pinned_mut: Pin<&mut MyFuture> = boxed.as_mut();

// For Unpin types, you can freely convert between Pin<&mut T> and &mut T
fn work_with_unpin<T: Unpin>(pinned: Pin<&mut T>) {
    let regular_ref: &mut T = Pin::into_inner(pinned);
    // Can now move *regular_ref freely
}

// For !Unpin types, you must use projection (see pin-project below)
```

### Pinning in the Service Trait

Look at the `Service` trait's `Future` type:

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

The `call` method returns a `Future`. When implementing services, you'll often see this pattern:

```rust
type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

fn call(&mut self, request: Request) -> Self::Future {
    Box::pin(async move {
        // Your async logic here
        Ok(response)
    })
}
```

`Box::pin` creates a `Pin<Box<...>>` which:
1. Allocates the future on the heap (so it has a stable address)
2. Wraps it in `Pin` (preventing moves)
3. Allows the future to be safely polled

### Why Pin<Box<...>> is Common

You'll see `Pin<Box<dyn Future + Send>>` frequently because:

1. **Type erasure**: Different async blocks produce different opaque types. Boxing allows storing them as trait objects.
2. **Stable address**: Box provides heap allocation, giving the future a fixed memory location.
3. **Send requirement**: Most Tower services need to be Send for use with async runtimes like Tokio.

However, this pattern has a cost—heap allocation. For performance-critical code, you might want to avoid boxing.

### Implementing Custom Futures Without Boxing

For performance-critical paths, you can implement futures directly:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MyCustomFuture<F> {
    inner: F,
    state: u32,
}

impl<F: Future> Future for MyCustomFuture<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // PROBLEM: How do we access `inner` and `state`?
        // We have Pin<&mut Self>, but we need Pin<&mut F> to poll inner
        
        // UNSAFE approach (error-prone):
        unsafe {
            let this = self.get_unchecked_mut();
            let inner = Pin::new_unchecked(&mut this.inner);
            inner.poll(cx)
        }
    }
}
```

The unsafe code above is tricky to get right. This is where `pin-project` helps.

### The pin-project Crate

`pin-project` is a procedural macro that generates safe projection code:

```rust
use pin_project::pin_project;

#[pin_project]
struct MyCustomFuture<F> {
    #[pin]  // This field needs to be pinned
    inner: F,
    state: u32,  // This field doesn't need pinning
}

impl<F: Future> Future for MyCustomFuture<F> {
    type Output = F::Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // project() gives us access to fields with correct pinning
        let this = self.project();
        
        // `this.inner` is Pin<&mut F> - safe to poll
        // `this.state` is &mut u32 - can be used normally
        
        this.inner.poll(cx)
    }
}
```

The `#[pin]` attribute marks fields that need to remain pinned. The generated `project()` method provides:
- `Pin<&mut T>` for `#[pin]` fields
- `&mut T` for regular fields

### Practical Pinning Patterns in Tower

Here's a realistic Tower middleware that properly handles pinning:

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use tower::Service;

// A timing middleware that measures request duration
pub struct TimingService<S> {
    inner: S,
}

// The custom future returned by TimingService
#[pin_project]
pub struct TimingFuture<F> {
    #[pin]
    inner: F,  // The wrapped service's future
    start: Instant,  // When the request started (doesn't need pinning)
}

impl<F: Future> Future for TimingFuture<F> {
    type Output = (F::Output, Duration);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        
        match this.inner.poll(cx) {
            Poll::Ready(output) => {
                let duration = this.start.elapsed();
                Poll::Ready((output, duration))
            }
            Poll::Pending => Poll::Pending,
        }
    }
}

impl<S, Request> Service<Request> for TimingService<S>
where
    S: Service<Request>,
{
    type Response = (S::Response, Duration);
    type Error = S::Error;
    type Future = TimingFuture<S::Future>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let inner = self.inner.call(request);
        TimingFuture {
            inner,
            start: Instant::now(),
        }
    }
}
```

This approach:
1. Avoids heap allocation (no `Box::pin`)
2. Uses `pin_project` for safe field access
3. Returns a concrete future type instead of a trait object

### Self-Referential Structs in Practice

Sometimes you genuinely need self-referential data. Here's how to handle it safely:

```rust
use std::marker::PhantomPinned;
use std::pin::Pin;
use std::ptr::NonNull;

struct SelfReferential {
    data: String,
    // Pointer to `data` - creates a self-reference
    data_ptr: Option<NonNull<String>>,
    // Prevents this type from implementing Unpin
    _pin: PhantomPinned,
}

impl SelfReferential {
    fn new(data: String) -> Pin<Box<Self>> {
        let res = Self {
            data,
            data_ptr: None,
            _pin: PhantomPinned,
        };
        
        let mut boxed = Box::pin(res);
        
        // Initialize the self-reference (must be done after pinning)
        let data_ptr = NonNull::from(&boxed.data);
        
        // SAFETY: We're not moving the struct, just initializing a field
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).data_ptr = Some(data_ptr);
        }
        
        boxed
    }
    
    fn get_data(self: Pin<&Self>) -> &str {
        &self.data
    }
    
    fn get_data_via_ptr(self: Pin<&Self>) -> &str {
        // SAFETY: The struct is pinned, so data_ptr is still valid
        unsafe {
            self.data_ptr.unwrap().as_ref()
        }
    }
}
```

Key points:
- `PhantomPinned` marker prevents auto-implementing `Unpin`
- Self-references are only initialized **after** the struct is pinned
- All access goes through `Pin<&Self>` or `Pin<&mut Self>`

### Common Pinning Mistakes and Solutions

**Mistake 1: Trying to move a pinned value**

```rust
let mut pinned = Box::pin(my_future);
let moved = *pinned;  // ERROR: Cannot move out of Pin
```

**Solution**: Work with the pinned value in place:

```rust
let mut pinned = Box::pin(my_future);
let result = pinned.as_mut().poll(cx);
```

**Mistake 2: Creating Pin from a local variable**

```rust
fn bad_example() {
    let mut future = MyFuture::new();
    let pinned = Pin::new(&mut future);  // Only works if MyFuture: Unpin
    // If MyFuture is !Unpin, this won't compile
}
```

**Solution**: Use `Box::pin` or stack pinning macros:

```rust
use tokio::pin;

fn good_example() {
    let future = MyFuture::new();
    tokio::pin!(future);  // Creates a Pin<&mut MyFuture> on the stack
    // Now `future` is pinned and can be polled
}
```

**Mistake 3: Forgetting to mark fields with `#[pin]`**

```rust
#[pin_project]
struct MyFuture<F> {
    inner: F,  // Forgot #[pin]!
}

impl<F: Future> Future for MyFuture<F> {
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<...> {
        let this = self.project();
        this.inner.poll(cx)  // ERROR: expected Pin<&mut F>, got &mut F
    }
}
```

**Solution**: Add `#[pin]` to fields that need pinning:

```rust
#[pin_project]
struct MyFuture<F> {
    #[pin]
    inner: F,
}
```

### When to Use Each Pattern

| Situation | Recommended Approach |
|-----------|---------------------|
| Quick prototyping | `Box::pin(async { ... })` |
| Public API with flexibility | `Pin<Box<dyn Future + Send>>` |
| Performance-critical middleware | `pin_project` with concrete futures |
| Self-referential data | Manual pinning with `PhantomPinned` |
| Wrapping other futures | `pin_project` |

### Summary

Pinning is essential for safe async Rust because:

1. **Futures can be self-referential**: The state machines generated by async/await may contain internal references.

2. **Pin prevents moves**: Once pinned, a value cannot be moved to a different memory location.

3. **Unpin is an opt-out**: Most types are `Unpin` and don't need special handling. Only self-referential types or types that explicitly opt out need pinning guarantees.

4. **pin-project makes it ergonomic**: The `pin_project` crate handles the tricky unsafe code for projecting pins to struct fields.

For Tower services, you'll most commonly:
- Use `Box::pin` for simple cases
- Use `pin_project` when implementing custom futures for performance
- Understand pinning errors when they arise in complex middleware

## The NewService Pattern

The `NewService` (or `MakeService`) pattern is Tower's solution for handling multiple connections, each requiring its own service instance.

### Service vs. NewService

- **Service**: Processes individual requests
- **NewService**: Creates new Service instances, typically one per connection

This pattern is crucial for handling thousands of concurrent TCP connections efficiently.

### The MakeService Trait

```rust
pub trait MakeService<Target, Request> {
    type Response;
    type Error;
    type Service: Service<Request, Response = Self::Response, Error = Self::Error>;
    type MakeError;
    type Future: Future<Output = Result<Self::Service, Self::MakeError>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::MakeError>>;
    fn make_service(&mut self, target: Target) -> Self::Future;
}
```

### Why NewService Matters for Concurrent Connections

When handling thousands of concurrent TCP connections:

1. **Connection-Specific State**: Each connection needs its own state
2. **Resource Isolation**: Failures in one connection shouldn't affect others
3. **Per-Connection Configuration**: Different connections might need different configurations
4. **Lifecycle Management**: Connections have independent lifecycles

### Example: A TCP Connection Pool

The `MakeService` pattern is commonly used for connection-oriented protocols. Here's an example that creates per-connection services:

```rust
use std::collections::HashMap;
use std::future::Future;
use std::net::SocketAddr;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll};
use tower::MakeService;

/// A factory that creates TcpConnection services for incoming connections
struct TcpConnectionFactory {
    /// Backend addresses to connect to
    backend_addresses: Vec<SocketAddr>,
    /// Shared state tracking connection health
    connection_health: Arc<Mutex<HashMap<SocketAddr, bool>>>,
}

impl TcpConnectionFactory {
    fn new(addresses: Vec<SocketAddr>) -> Self {
        let health = addresses.iter()
            .map(|addr| (*addr, true)) // Initially healthy
            .collect();
        
        TcpConnectionFactory {
            backend_addresses: addresses,
            connection_health: Arc::new(Mutex::new(health)),
        }
    }
}

/// Target information for creating a new service
/// (e.g., the remote address of an incoming connection)
struct ConnectionTarget {
    remote_addr: SocketAddr,
}

// Implement MakeService to create per-connection services
impl MakeService<ConnectionTarget, Vec<u8>> for TcpConnectionFactory {
    type Response = Vec<u8>;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Service = TcpConnection;
    type MakeError = Box<dyn std::error::Error + Send + Sync>;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Service, Self::MakeError>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::MakeError>> {
        // Check if we can create new services (e.g., not at connection limit)
        Poll::Ready(Ok(()))
    }

    fn make_service(&mut self, target: ConnectionTarget) -> Self::Future {
        // Create a new service for this specific connection
        let backend = self.backend_addresses.first().cloned();
        let health = Arc::clone(&self.connection_health);
        
        Box::pin(async move {
            match backend {
                Some(backend_addr) => {
                    // In reality, you'd establish a TCP connection here
                    println!("Creating service for connection from {}", target.remote_addr);
                    
                    Ok(TcpConnection {
                        backend_addr,
                        connection_health: health,
                    })
                }
                None => Err("No backend addresses configured".into()),
            }
        })
    }
}

/// The per-connection service that handles requests
struct TcpConnection {
    backend_addr: SocketAddr,
    connection_health: Arc<Mutex<HashMap<SocketAddr, bool>>>,
}

impl tower::Service<Vec<u8>> for TcpConnection {
    type Response = Vec<u8>;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        let health = self.connection_health.lock().unwrap();
        match health.get(&self.backend_addr) {
            Some(true) => Poll::Ready(Ok(())),
            _ => Poll::Ready(Err("Backend unhealthy".into())),
        }
    }

    fn call(&mut self, request: Vec<u8>) -> Self::Future {
        let addr = self.backend_addr;
        
        Box::pin(async move {
            // In a real implementation, send data over the TCP connection
            println!("Forwarding {} bytes to backend {}", request.len(), addr);
            
            // Simulate a response
            Ok(vec![1, 2, 3, 4])
        })
    }
}

// Usage with a TCP listener
async fn serve_connections(mut factory: TcpConnectionFactory) {
    use tower::MakeService;
    
    // Simulating incoming connections
    let incoming = vec![
        ConnectionTarget { remote_addr: "192.168.1.10:12345".parse().unwrap() },
        ConnectionTarget { remote_addr: "192.168.1.11:12346".parse().unwrap() },
    ];
    
    for target in incoming {
        // Wait until the factory is ready to create a new service
        futures::future::poll_fn(|cx| factory.poll_ready(cx)).await.unwrap();
        
        // Create a new service for this connection
        let mut connection_service = factory.make_service(target).await.unwrap();
        
        // Spawn a task to handle this connection
        tokio::spawn(async move {
            // Handle requests on this connection
            let request = vec![1, 2, 3];
            
            // Wait for service to be ready
            use tower::ServiceExt;
            let response = connection_service.ready().await.unwrap()
                .call(request).await.unwrap();
            
            println!("Got response: {:?}", response);
        });
    }
}
```

**Key differences from `Service`:**
- `make_service()` creates a *new service* for each connection, not just a response
- Each created service has its own isolated state
- The `Target` parameter provides context about the connection being established
- `poll_ready` on `MakeService` indicates readiness to *create services*, not handle requests

## Advanced Routing Patterns

Routing is a fundamental aspect of networked applications, and Tower provides powerful abstractions for implementing sophisticated routing patterns. This section explores practical routing implementations using Tower's composable services.

### Path-Based Routing

Path-based routing directs requests to different services based on the URL path. Here's a practical implementation:

```rust
use std::collections::HashMap;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use tower::{Service, ServiceExt};
use http::{Request, Response, StatusCode};
use bytes::Bytes;
use futures::future::{self, BoxFuture, FutureExt};

// A router that dispatches requests based on path
struct PathRouter<B> {
    // Map paths to services
    routes: HashMap<String, Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>>,
    // Default service for unmatched paths
    fallback: Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>,
}

type BoxError = Box<dyn std::error::Error + Send + Sync>;
type BoxFuture<T> = Pin<Box<dyn Future<Output = T> + Send>>;

impl<B> PathRouter<B>
where
    B: Send + 'static,
{
    fn new(fallback: impl Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static) -> Self {
        PathRouter {
            routes: HashMap::new(),
            fallback: Box::new(fallback),
        }
    }
    
    // Register a service for a specific path
    fn add_route<S>(&mut self, path: &str, service: S)
    where
        S: Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static,
    {
        self.routes.insert(path.to_string(), Box::new(service));
    }
}

impl<B> Service<Request<B>> for PathRouter<B>
where
    B: Send + 'static,
{
    type Response = Response<B>;
    type Error = BoxError;
    type Future = BoxFuture<Result<Self::Response, Self::Error>>;
    
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // A router is ready if any route is ready
        // In a production implementation, you might want more sophisticated readiness tracking
        for (_, service) in &mut self.routes {
            if service.poll_ready(cx).is_ready() {
                return Poll::Ready(Ok(()));
            }
        }
        
        // If no routes are ready, check the fallback
        self.fallback.poll_ready(cx)
    }
    
    fn call(&mut self, req: Request<B>) -> Self::Future {
        // Extract the path from the request
        let path = req.uri().path().to_string();
        
        // Find a matching service or use the fallback
        let service = self.routes.get_mut(&path)
            .unwrap_or(&mut self.fallback);
            
        // Clone the request since we need to move it into the future
        let req_clone = req.map(|b| b);
        
        // Call the selected service
        Box::pin(async move {
            // In a real implementation, you'd want to handle the case where the service
            // isn't ready yet by calling ready() first
            service.call(req_clone).await
        })
    }
}

// Example usage
async fn router_example() {
    // Create services for different paths
    let users_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::OK)
            .body(Bytes::from("Users API"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let products_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::OK)
            .body(Bytes::from("Products API"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let fallback_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::NOT_FOUND)
            .body(Bytes::from("Not Found"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    // Create the router
    let mut router = PathRouter::new(fallback_service);
    router.add_route("/users", users_service);
    router.add_route("/products", products_service);
    
    // Use the router
    let request = Request::builder()
        .uri("/users")
        .body(Bytes::from(""))
        .unwrap();
        
    let response = router.oneshot(request).await.unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    
    // The body would be "Users API"
}
```

### Header-Based Routing

Header-based routing allows for content negotiation, versioning, and other protocol-specific behaviors:

```rust
struct HeaderRouter<B> {
    routes: HashMap<String, HashMap<String, Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>>>,
    fallback: Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>,
}

impl<B> HeaderRouter<B>
where
    B: Send + 'static,
{
    fn new(fallback: impl Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static) -> Self {
        HeaderRouter {
            routes: HashMap::new(),
            fallback: Box::new(fallback),
        }
    }
    
    // Register a service for a specific header value
    fn add_route<S>(&mut self, header: &str, value: &str, service: S)
    where
        S: Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static,
    {
        self.routes
            .entry(header.to_string())
            .or_insert_with(HashMap::new)
            .insert(value.to_string(), Box::new(service));
    }
}

impl<B> Service<Request<B>> for HeaderRouter<B>
where
    B: Send + 'static,
{
    type Response = Response<B>;
    type Error = BoxError;
    type Future = BoxFuture<Result<Self::Response, Self::Error>>;
    
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Similar to PathRouter, we're ready if any route is ready
        for (_, values) in &mut self.routes {
            for (_, service) in values {
                if service.poll_ready(cx).is_ready() {
                    return Poll::Ready(Ok(()));
                }
            }
        }
        
        self.fallback.poll_ready(cx)
    }
    
    fn call(&mut self, req: Request<B>) -> Self::Future {
        // Try to find a service based on headers
        let mut matched_service = None;
        
        for (header_name, values) in &mut self.routes {
            if let Some(header_value) = req.headers().get(header_name) {
                if let Ok(value_str) = header_value.to_str() {
                    if let Some(service) = values.get_mut(value_str) {
                        matched_service = Some(service);
                        break;
                    }
                }
            }
        }
        
        // Use the matched service or fallback
        let service = matched_service.unwrap_or(&mut self.fallback);
        let req_clone = req.map(|b| b);
        
        Box::pin(async move {
            service.call(req_clone).await
        })
    }
}

// Example usage for content negotiation
async fn content_negotiation_example() {
    // Create services for different content types
    let json_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .header("Content-Type", "application/json")
            .status(StatusCode::OK)
            .body(Bytes::from(r#"{"message":"Hello, World!"}"#))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let xml_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .header("Content-Type", "application/xml")
            .status(StatusCode::OK)
            .body(Bytes::from("<message>Hello, World!</message>"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let fallback_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .header("Content-Type", "text/plain")
            .status(StatusCode::OK)
            .body(Bytes::from("Hello, World!"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    // Create the router
    let mut router = HeaderRouter::new(fallback_service);
    router.add_route("Accept", "application/json", json_service);
    router.add_route("Accept", "application/xml", xml_service);
    
    // Use the router with JSON request
    let json_request = Request::builder()
        .uri("/api")
        .header("Accept", "application/json")
        .body(Bytes::from(""))
        .unwrap();
        
    let response = router.oneshot(json_request).await.unwrap();
    assert_eq!(response.headers()["Content-Type"], "application/json");
}
```

### Dynamic Routing Based on Request Properties

For more complex routing decisions, you can implement dynamic routing based on request properties:

```rust
// A predicate that decides whether a service should handle a request
trait RoutePredicate<B>: Send + Sync {
    fn matches(&self, request: &Request<B>) -> bool;
}

// A path prefix predicate
struct PathPrefixPredicate {
    prefix: String,
}

impl<B> RoutePredicate<B> for PathPrefixPredicate {
    fn matches(&self, request: &Request<B>) -> bool {
        request.uri().path().starts_with(&self.prefix)
    }
}

// A header predicate
struct HeaderPredicate {
    name: String,
    value: String,
}

impl<B> RoutePredicate<B> for HeaderPredicate {
    fn matches(&self, request: &Request<B>) -> bool {
        if let Some(header) = request.headers().get(&self.name) {
            if let Ok(value) = header.to_str() {
                return value == self.value;
            }
        }
        false
    }
}

// A route with a predicate and service
struct Route<B> {
    predicate: Box<dyn RoutePredicate<B>>,
    service: Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>,
}

// A dynamic router
struct DynamicRouter<B> {
    routes: Vec<Route<B>>,
    fallback: Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>,
}

impl<B> DynamicRouter<B>
where
    B: Send + 'static,
{
    fn new(fallback: impl Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static) -> Self {
        DynamicRouter {
            routes: Vec::new(),
            fallback: Box::new(fallback),
        }
    }
    
    // Add a route with a predicate
    fn add_route<P, S>(&mut self, predicate: P, service: S)
    where
        P: RoutePredicate<B> + 'static,
        S: Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static,
    {
        self.routes.push(Route {
            predicate: Box::new(predicate),
            service: Box::new(service),
        });
    }
}

impl<B> Service<Request<B>> for DynamicRouter<B>
where
    B: Send + 'static,
{
    type Response = Response<B>;
    type Error = BoxError;
    type Future = BoxFuture<Result<Self::Response, Self::Error>>;
    
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Check if any route is ready
        for route in &mut self.routes {
            if route.service.poll_ready(cx).is_ready() {
                return Poll::Ready(Ok(()));
            }
        }
        
        self.fallback.poll_ready(cx)
    }
    
    fn call(&mut self, req: Request<B>) -> Self::Future {
        // Find the first matching route
        for route in &mut self.routes {
            if route.predicate.matches(&req) {
                let service = &mut route.service;
                let req_clone = req.map(|b| b);
                
                return Box::pin(async move {
                    service.call(req_clone).await
                });
            }
        }
        
        // Use fallback if no route matches
        let fallback = &mut self.fallback;
        let req_clone = req.map(|b| b);
        
        Box::pin(async move {
            fallback.call(req_clone).await
        })
    }
}

// Example usage with dynamic routing
async fn dynamic_routing_example() {
    // Create services
    let api_v1_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::OK)
            .body(Bytes::from("API v1"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let api_v2_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::OK)
            .body(Bytes::from("API v2"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let admin_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::OK)
            .body(Bytes::from("Admin Panel"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    let fallback_service = tower::service_fn(|_: Request<Bytes>| async {
        let response = Response::builder()
            .status(StatusCode::NOT_FOUND)
            .body(Bytes::from("Not Found"))
            .unwrap();
        Ok::<_, BoxError>(response)
    });
    
    // Create the router
    let mut router = DynamicRouter::new(fallback_service);
    
    // Add routes with predicates
    router.add_route(
        PathPrefixPredicate { prefix: "/api/v1".to_string() },
        api_v1_service
    );
    
    router.add_route(
        PathPrefixPredicate { prefix: "/api/v2".to_string() },
        api_v2_service
    );
    
    router.add_route(
        HeaderPredicate {
            name: "Authorization".to_string(),
            value: "Bearer admin-token".to_string()
        },
        admin_service
    );
    
    // Use the router
    let request = Request::builder()
        .uri("/api/v1/users")
        .body(Bytes::from(""))
        .unwrap();
        
    let response = router.oneshot(request).await.unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    // Body would be "API v1"
}
```

### gRPC-Aware Router Example

For gRPC services, we need a router that understands gRPC's path format and streaming semantics:

```rust
// A simplified gRPC router
struct GrpcRouter<B> {
    // Map service+method to handlers
    routes: HashMap<String, Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>>,
    fallback: Box<dyn Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync>,
}

impl<B> GrpcRouter<B>
where
    B: Send + 'static,
{
    fn new(fallback: impl Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static) -> Self {
        GrpcRouter {
            routes: HashMap::new(),
            fallback: Box::new(fallback),
        }
    }
    
    // Register a service for a gRPC method
    // Format: "/package.Service/Method"
    fn add_route<S>(&mut self, method: &str, service: S)
    where
        S: Service<Request<B>, Response = Response<B>, Error = BoxError> + Send + Sync + 'static,
    {
        self.routes.insert(method.to_string(), Box::new(service));
    }
}

impl<B> Service<Request<B>> for GrpcRouter<B>
where
    B: Send + 'static,
{
    type Response = Response<B>;
    type Error = BoxError;
    type Future = BoxFuture<Result<Self::Response, Self::Error>>;
    
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Check if any route is ready
        for (_, service) in &mut self.routes {
            if service.poll_ready(cx).is_ready() {
                return Poll::Ready(Ok(()));
            }
        }
        
        self.fallback.poll_ready(cx)
    }
    
    fn call(&mut self, req: Request<B>) -> Self::Future {
        // Extract the gRPC method from the path
        let path = req.uri().path().to_string();
        
        // Find a matching service or use the fallback
        let service = self.routes.get_mut(&path)
            .unwrap_or(&mut self.fallback);
            
        let req_clone = req.map(|b| b);
        
        Box::pin(async move {
            service.call(req_clone).await
        })
    }
}

// Example usage with gRPC
async fn grpc_router_example() {
    // Create gRPC service handlers
    let get_user_service = tower::service_fn(|req: Request<Bytes>| async {
        // In a real implementation, this would decode the protobuf message
        // and call the actual service implementation
        
        // Create a gRPC response
        let response = Response::builder()
            .header("content-type", "application/grpc")
            .status(StatusCode::OK)
            .body(Bytes::from(vec![0, 0, 0, 0, 1, 10, 5, 65, 108, 105, 99, 101])) // Simulated protobuf response
            .unwrap();
            
        Ok::<_, BoxError>(response)
    });
    
    let list_users_service = tower::service_fn(|req: Request<Bytes>| async {
        // Simulated streaming response
        let response = Response::builder()
            .header("content-type", "application/grpc")
            .status(StatusCode::OK)
            .body(Bytes::from(vec![
                // First message: "Alice"
                0, 0, 0, 0, 1, 10, 5, 65, 108, 105, 99, 101,
                // Second message: "Bob"
                0, 0, 0, 0, 1, 10, 3, 66, 111, 98
            ]))
            .unwrap();
            
        Ok::<_, BoxError>(response)
    });
    
    let fallback_service = tower::service_fn(|_: Request<Bytes>| async {
        // gRPC error response - method not found
        let response = Response::builder()
            .header("content-type", "application/grpc")
            .header("grpc-status", "12") // UNIMPLEMENTED
            .status(StatusCode::OK) // gRPC always uses 200 OK at HTTP level
            .body(Bytes::from("Method not found"))
            .unwrap();
            
        Ok::<_, BoxError>(response)
    });
    
    // Create the router
    let mut router = GrpcRouter::new(fallback_service);
    router.add_route("/users.UserService/GetUser", get_user_service);
    router.add_route("/users.UserService/ListUsers", list_users_service);
    
    // Use the router
    let request = Request::builder()
        .uri("/users.UserService/GetUser")
        .header("content-type", "application/grpc")
        .body(Bytes::from(vec![0, 0, 0, 0, 1, 10, 1, 49])) // Simulated protobuf request
        .unwrap();
        
    let response = router.oneshot(request).await.unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    assert_eq!(response.headers()["content-type"], "application/grpc");
}
```

These routing patterns demonstrate Tower's flexibility in handling different routing requirements. By composing services and leveraging Tower's abstractions, you can build sophisticated routing systems that are both maintainable and performant.

## Backpressure Mechanisms

Backpressure is one of Tower's most powerful features for building reliable distributed systems. This section explores practical implementations of backpressure mechanisms using Tower's `poll_ready` pattern.

### Understanding Backpressure Beyond the Basics

Backpressure is a flow control mechanism that prevents a fast producer from overwhelming a slow consumer. Without backpressure, a system can experience:

- **Memory exhaustion**: Unbounded queues grow until the system runs out of RAM
- **Cascading failures**: One slow service causes upstream services to queue requests, eventually failing
- **Resource starvation**: Connections, file descriptors, and threads get consumed without release
- **Degraded latency**: As queues grow, tail latencies explode

**The key insight**: A well-designed system must be able to say "slow down" when it's overwhelmed.

In Tower, backpressure is implemented through the `poll_ready` method:

```rust
fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
```

This method returns:
- `Poll::Ready(Ok(()))` — "I'm ready, send me a request"
- `Poll::Ready(Err(e))` — "I've failed permanently"
- `Poll::Pending` — "I'm busy, please wait"

The critical contract is: **always call `poll_ready()` before `call()`**. This is typically done via the `ServiceExt::ready()` helper:

```rust
// The safe pattern
let mut service = MyService::new();
service.ready().await?;  // Waits for poll_ready to return Ready
service.call(request).await?;
```

### How Wakers Enable Efficient Waiting

When `poll_ready` returns `Poll::Pending`, the service must eventually wake the caller. This is done through the `Waker` in the `Context`:

```rust
fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    if self.is_busy() {
        // Store the waker to notify later when we're ready
        self.pending_waker = Some(cx.waker().clone());
        Poll::Pending
    } else {
        Poll::Ready(Ok(()))
    }
}

// Later, when the service is ready again:
fn become_ready(&mut self) {
    if let Some(waker) = self.pending_waker.take() {
        waker.wake();  // This schedules the waiting task to be polled again
    }
}
```

This cooperative model is what makes async Rust efficient—tasks sleep when waiting and are woken precisely when something changes.

### Practical Backpressure Examples

Let's explore how this works in practice with concrete examples:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use tower::{Service, ServiceExt};
use futures::future::{self, BoxFuture, FutureExt};

// A service with a limited buffer capacity
struct BufferedService<S> {
    // The inner service
    inner: S,
    // Current buffer usage
    buffer_used: usize,
    // Maximum buffer capacity
    max_buffer: usize,
}

impl<S, Request> BufferedService<S>
where
    S: Service<Request>,
{
    fn new(inner: S, max_buffer: usize) -> Self {
        BufferedService {
            inner,
            buffer_used: 0,
            max_buffer,
        }
    }
}

impl<S, Request> Service<Request> for BufferedService<S>
where
    S: Service<Request>,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // If our buffer is full, we're not ready
        if self.buffer_used >= self.max_buffer {
            // We need to check if the inner service is ready to process
            // requests, which might free up our buffer
            match self.inner.poll_ready(cx) {
                Poll::Ready(Ok(())) => {
                    // Inner service is ready, we can reduce our buffer count
                    // In a real implementation, this would happen when futures complete
                    self.buffer_used = self.buffer_used.saturating_sub(1);
                    
                    // If we're still at capacity, we're not ready
                    if self.buffer_used >= self.max_buffer {
                        Poll::Pending
                    } else {
                        Poll::Ready(Ok(()))
                    }
                }
                other => other, // Propagate errors or Pending
            }
        } else {
            // We have buffer space, but we still need to ensure the inner service is ready
            self.inner.poll_ready(cx)
        }
    }

    fn call(&mut self, request: Request) -> Self::Future {
        // Increment buffer usage
        self.buffer_used += 1;
        
        // Call the inner service
        let future = self.inner.call(request);
        
        // Create a future that decrements the buffer when it completes
        let buffer_used = Arc::new(Mutex::new(&mut self.buffer_used));
        
        Box::pin(async move {
            let result = future.await;
            
            // Decrement buffer usage when the future completes
            if let Ok(buffer) = buffer_used.lock() {
                **buffer = buffer.saturating_sub(1);
            }
            
            result
        })
    }
}

// Example usage
async fn buffered_service_example() {
    // Create a slow service that takes time to process requests
    let slow_service = tower::service_fn(|request: &'static str| async move {
        // Simulate slow processing
        tokio::time::sleep(Duration::from_millis(100)).await;
        Ok::<_, std::io::Error>(format!("Processed: {}", request))
    });
    
    // Wrap it with a buffered service that limits concurrent requests
    let mut buffered = BufferedService::new(slow_service, 5);
    
    // Process requests with backpressure
    for i in 0..10 {
        // Wait until the service is ready to accept a request
        match buffered.ready().await {
            Ok(mut service) => {
                // Service is ready, send the request
                let request = format!("Request {}", i);
                tokio::spawn(async move {
                    match service.call(request.as_str()).await {
                        Ok(response) => println!("Got response: {}", response),
                        Err(e) => eprintln!("Error: {}", e),
                    }
                });
            }
            Err(e) => {
                eprintln!("Service not ready: {}", e);
                break;
            }
        }
    }
}
```

This example demonstrates how to implement a buffered service that limits the number of in-flight requests, preventing resource exhaustion.

### Rate Limiting with Proper Backpressure

Rate limiting is a common requirement in networked applications. Here's how to implement it with proper backpressure:

```rust
// A rate-limited service that allows a certain number of requests per time period
struct RateLimitedService<S> {
    // The inner service
    inner: S,
    // Maximum requests per time window
    max_requests: usize,
    // Current request count in the window
    request_count: usize,
    // Time window duration
    time_window: Duration,
    // When the current window started
    window_start: Instant,
}

impl<S, Request> RateLimitedService<S>
where
    S: Service<Request>,
{
    fn new(inner: S, max_requests: usize, time_window: Duration) -> Self {
        RateLimitedService {
            inner,
            max_requests,
            request_count: 0,
            time_window,
            window_start: Instant::now(),
        }
    }
    
    // Check if we should reset the window
    fn check_window_reset(&mut self) {
        let now = Instant::now();
        if now.duration_since(self.window_start) >= self.time_window {
            // Reset the window
            self.window_start = now;
            self.request_count = 0;
        }
    }
}

impl<S, Request> Service<Request> for RateLimitedService<S>
where
    S: Service<Request>,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // First check if the inner service is ready
        match self.inner.poll_ready(cx) {
            Poll::Ready(Ok(())) => {
                // Inner service is ready, now check rate limit
                self.check_window_reset();
                
                if self.request_count < self.max_requests {
                    // We're under the rate limit, so we're ready
                    Poll::Ready(Ok(()))
                } else {
                    // We've hit the rate limit
                    // In a real implementation, we would register a waker to be notified
                    // when the window resets
                    let waker = cx.waker().clone();
                    let time_until_reset = self.time_window
                        .checked_sub(Instant::now().duration_since(self.window_start))
                        .unwrap_or(Duration::from_secs(0));
                    
                    // Schedule a wakeup when the window resets
                    tokio::spawn(async move {
                        tokio::time::sleep(time_until_reset).await;
                        waker.wake();
                    });
                    
                    Poll::Pending
                }
            }
            other => other, // Propagate errors or Pending
        }
    }

    fn call(&mut self, request: Request) -> Self::Future {
        // Increment the request count
        self.request_count += 1;
        
        // Call the inner service
        self.inner.call(request)
    }
}

// Example usage
async fn rate_limited_service_example() {
    // Create a service that processes requests
    let base_service = tower::service_fn(|request: &'static str| async move {
        Ok::<_, std::io::Error>(format!("Processed: {}", request))
    });
    
    // Wrap it with a rate limiter: 5 requests per second
    let mut rate_limited = RateLimitedService::new(
        base_service,
        5,
        Duration::from_secs(1)
    );
    
    // Process requests with rate limiting
    for i in 0..20 {
        // Wait until the service is ready (respects rate limit)
        match rate_limited.ready().await {
            Ok(mut service) => {
                // Service is ready, send the request
                let request = format!("Request {}", i);
                let response = service.call(request.as_str()).await.unwrap();
                println!("Got response: {}", response);
            }
            Err(e) => {
                eprintln!("Service not ready: {}", e);
                break;
            }
        }
    }
}
```

This implementation properly propagates backpressure when the rate limit is exceeded, ensuring that clients don't overwhelm the service.

### Connection Pooling with Backpressure Propagation

Connection pools are essential for efficient resource utilization. Here's how to implement one with proper backpressure:

```rust
use std::collections::VecDeque;
use std::net::SocketAddr;
use std::sync::{Arc, Mutex};

// A simplified connection type
struct Connection {
    addr: SocketAddr,
    is_busy: bool,
}

impl Connection {
    fn new(addr: SocketAddr) -> Self {
        Connection {
            addr,
            is_busy: false,
        }
    }
}

// A connection pool service
struct ConnectionPool {
    // Available connections
    connections: Arc<Mutex<VecDeque<Connection>>>,
    // Maximum number of connections
    max_connections: usize,
    // Current number of connections
    current_connections: usize,
}

impl ConnectionPool {
    fn new(max_connections: usize) -> Self {
        ConnectionPool {
            connections: Arc::new(Mutex::new(VecDeque::new())),
            max_connections,
            current_connections: 0,
        }
    }
    
    // Add a connection to the pool
    fn add_connection(&mut self, addr: SocketAddr) {
        if self.current_connections < self.max_connections {
            let mut connections = self.connections.lock().unwrap();
            connections.push_back(Connection::new(addr));
            self.current_connections += 1;
        }
    }
}

// Request type for the connection pool
struct ConnectionRequest {
    timeout: Duration,
}

// Response type is a connection handle
struct ConnectionHandle {
    connection: Connection,
    pool: Arc<Mutex<VecDeque<Connection>>>,
}

impl Drop for ConnectionHandle {
    fn drop(&mut self) {
        // Return the connection to the pool when the handle is dropped
        let mut pool = self.pool.lock().unwrap();
        self.connection.is_busy = false;
        pool.push_back(std::mem::replace(&mut self.connection, Connection::new("0.0.0.0:0".parse().unwrap())));
    }
}

impl Service<ConnectionRequest> for ConnectionPool {
    type Response = ConnectionHandle;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Check if we have any available connections
        let connections = self.connections.lock().unwrap();
        
        if connections.iter().any(|conn| !conn.is_busy) || self.current_connections < self.max_connections {
            // We either have an available connection or can create a new one
            Poll::Ready(Ok(()))
        } else {
            // No connections available and at capacity
            // In a real implementation, we would register the waker to be notified
            // when a connection becomes available
            Poll::Pending
        }
    }

    fn call(&mut self, request: ConnectionRequest) -> Self::Future {
        let pool = Arc::clone(&self.connections);
        let max_connections = self.max_connections;
        let current_connections = &mut self.current_connections;
        
        Box::pin(async move {
            // Try to get a connection with timeout
            let start = Instant::now();
            
            loop {
                // Check if we've timed out
                if start.elapsed() > request.timeout {
                    return Err("Connection timeout".into());
                }
                
                // Try to get an available connection
                let mut connections = pool.lock().unwrap();
                
                // Find an available connection
                for i in 0..connections.len() {
                    if !connections[i].is_busy {
                        // Found an available connection
                        let mut connection = connections.remove(i).unwrap();
                        connection.is_busy = true;
                        
                        return Ok(ConnectionHandle {
                            connection,
                            pool: Arc::clone(&pool),
                        });
                    }
                }
                
                // No available connection, can we create a new one?
                if *current_connections < max_connections {
                    // Create a new connection
                    let connection = Connection::new("127.0.0.1:8080".parse().unwrap());
                    *current_connections += 1;
                    
                    return Ok(ConnectionHandle {
                        connection,
                        pool: Arc::clone(&pool),
                    });
                }
                
                // No connections available and at capacity, wait a bit and retry
                tokio::time::sleep(Duration::from_millis(10)).await;
            }
        })
    }
}

// Example usage
async fn connection_pool_example() {
    // Create a connection pool with max 10 connections
    let mut pool = ConnectionPool::new(10);
    
    // Add some initial connections
    for i in 0..5 {
        let addr: SocketAddr = format!("127.0.0.1:{}", 8080 + i).parse().unwrap();
        pool.add_connection(addr);
    }
    
    // Use the pool with proper backpressure
    for i in 0..20 {
        // Wait until the pool is ready to provide a connection
        match pool.ready().await {
            Ok(mut service) => {
                // Pool is ready, get a connection
                let request = ConnectionRequest {
                    timeout: Duration::from_secs(1),
                };
                
                match service.call(request).await {
                    Ok(conn) => {
                        println!("Got connection to {}", conn.connection.addr);
                        // Use the connection...
                        
                        // Connection will be returned to the pool when `conn` is dropped
                    }
                    Err(e) => {
                        eprintln!("Failed to get connection: {}", e);
                    }
                }
            }
            Err(e) => {
                eprintln!("Pool not ready: {}", e);
                break;
            }
        }
    }
}
```

This connection pool properly propagates backpressure when all connections are in use, ensuring that clients wait for resources to become available.

### Handling Backpressure in Streaming Contexts

Streaming applications present unique backpressure challenges. Here's how to handle them with Tower:

```rust
use futures::stream::{Stream, StreamExt};
use std::pin::Pin;
use std::task::{Context, Poll};

// A service that processes streams of data
struct StreamProcessorService<S> {
    inner: S,
    max_in_flight: usize,
    current_in_flight: usize,
}

impl<S> StreamProcessorService<S> {
    fn new(inner: S, max_in_flight: usize) -> Self {
        StreamProcessorService {
            inner,
            max_in_flight,
            current_in_flight: 0,
        }
    }
}

// A stream item with a response channel
struct StreamItem<T> {
    item: T,
    response_tx: tokio::sync::oneshot::Sender<Result<(), Box<dyn std::error::Error + Send + Sync>>>,
}

impl<S, T> Service<StreamItem<T>> for StreamProcessorService<S>
where
    S: Service<T>,
    S::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    type Response = ();
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Check if we're at capacity
        if self.current_in_flight >= self.max_in_flight {
            // We're at capacity, not ready
            Poll::Pending
        } else {
            // We have capacity, but check if the inner service is ready
            match self.inner.poll_ready(cx) {
                Poll::Ready(Ok(())) => Poll::Ready(Ok(())),
                Poll::Ready(Err(e)) => Poll::Ready(Err(e.into())),
                Poll::Pending => Poll::Pending,
            }
        }
    }

    fn call(&mut self, req: StreamItem<T>) -> Self::Future {
        // Increment in-flight count
        self.current_in_flight += 1;
        
        // Call the inner service
        let future = self.inner.call(req.item);
        let response_tx = req.response_tx;
        let current_in_flight = &mut self.current_in_flight;
        
        Box::pin(async move {
            // Process the item
            let result = future.await;
            
            // Decrement in-flight count
            *current_in_flight -= 1;
            
            // Send the result back through the response channel
            let _ = response_tx.send(result.map_err(|e| e.into()));
            
            Ok(())
        })
    }
}

// A stream processor that applies backpressure
struct StreamProcessor<S, T> {
    service: S,
    _phantom: std::marker::PhantomData<T>,
}

impl<S, T> StreamProcessor<S, T>
where
    S: Service<StreamItem<T>>,
    S::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    fn new(service: S) -> Self {
        StreamProcessor {
            service,
            _phantom: std::marker::PhantomData,
        }
    }
    
    // Process a stream with backpressure
    async fn process<St>(&mut self, stream: St) -> Result<(), Box<dyn std::error::Error + Send + Sync>>
    where
        St: Stream<Item = T> + Unpin,
    {
        let mut stream = stream;
        
        while let Some(item) = stream.next().await {
            // Create a channel for the response
            let (tx, rx) = tokio::sync::oneshot::channel();
            
            // Wait until the service is ready
            self.service.ready().await?;
            
            // Send the item to the service
            let stream_item = StreamItem {
                item,
                response_tx: tx,
            };
            
            self.service.call(stream_item).await?;
            
            // Wait for the response
            rx.await??;
        }
        
        Ok(())
    }
}

// Example usage
async fn stream_processor_example() {
    // Create a service that processes stream items
    let processor_service = tower::service_fn(|item: String| async move {
        // Process the item
        println!("Processing: {}", item);
        
        // Simulate processing time
        tokio::time::sleep(Duration::from_millis(50)).await;
        
        Ok::<_, std::io::Error>(())
    });
    
    // Wrap it with a stream processor service that limits concurrency
    let limited_service = StreamProcessorService::new(processor_service, 5);
    
    // Create a stream processor
    let mut processor = StreamProcessor::new(limited_service);
    
    // Create a stream of items
    let items = futures::stream::iter((0..100).map(|i| format!("Item {}", i)));
    
    // Process the stream with backpressure
    processor.process(items).await.unwrap();
}
```

This example demonstrates how to process a stream of items with controlled concurrency, applying backpressure when the processor is at capacity.

Backpressure is a critical aspect of building reliable distributed systems. By properly implementing and respecting the `poll_ready` contract, Tower services can gracefully handle load spikes, resource constraints, and other challenges that arise in high-performance networking applications.

## Zero-Copy Techniques

Zero-copy techniques are essential for high-performance networking applications, allowing data to be processed without unnecessary copying. Tower's design makes it particularly well-suited for implementing zero-copy patterns.

### Why Zero-Copy Matters

Every memory copy has a cost:
- **CPU cycles**: Moving bytes takes time
- **Cache pollution**: Data gets copied into CPU cache, evicting useful data
- **Memory bandwidth**: Systems have limited bandwidth between CPU and RAM
- **Latency**: Each copy adds microseconds that compound at scale

In high-throughput systems handling millions of requests per second, unnecessary copies can consume significant resources. Consider a proxy processing 1GB/s of traffic—copying data even once doubles the memory bandwidth requirement.

### Understanding Zero-Copy in Tower

Zero-copy refers to processing data without making redundant copies in memory. In networking applications, this typically means:

1. **Kernel bypass or minimal kernel copies**: Using techniques like `mmap` or io_uring where possible
2. **Shared ownership**: Multiple parts of your application reference the same buffer
3. **Slicing without copying**: Creating views into buffers instead of copying subsets
4. **Pooled buffers**: Reusing allocations instead of allocating and freeing repeatedly

Tower's `Service` trait works with any request and response types, making it easy to implement zero-copy patterns by using appropriate buffer types and careful memory management.

### The `bytes` Crate

The `bytes` crate provides two key types for zero-copy patterns:

- **`Bytes`**: An immutable, reference-counted byte buffer that can be sliced without copying
- **`BytesMut`**: A mutable buffer that can be grown, shrunk, and split efficiently

Key operations:
```rust
let data = Bytes::from("hello world");

// Slicing creates a new view without copying
let hello = data.slice(0..5);  // "hello"
let world = data.slice(6..);   // "world"

// Both slices share the same underlying memory
// Reference count ensures memory is freed when all slices are dropped
```

### Implementing Zero-Copy Buffer Handling

Here's how to implement a service that processes data with zero-copy techniques:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use bytes::{Bytes, BytesMut, Buf, BufMut};
use tower::Service;
use futures::future::{self, BoxFuture, FutureExt};

// A service that processes data without unnecessary copies
struct ZeroCopyProcessor {
    // Configuration or state would go here
}

impl ZeroCopyProcessor {
    fn new() -> Self {
        ZeroCopyProcessor {}
    }
}

impl Service<Bytes> for ZeroCopyProcessor {
    type Response = Bytes;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // This service is always ready
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, data: Bytes) -> Self::Future {
        // Process the data without copying it
        // Bytes is an immutable reference-counted buffer type
        
        Box::pin(async move {
            // Example: Extract a header (first 8 bytes) and payload without copying
            if data.len() < 8 {
                return Err("Data too short".into());
            }
            
            // Split the data into header and payload views (no copying)
            let header = data.slice(0..8);
            let payload = data.slice(8..);
            
            // Process the header (example: check a magic number)
            if header[0..4] != [0x89, b'P', b'N', b'G'] {
                return Err("Invalid header magic".into());
            }
            
            // Process the payload (example: compute a checksum)
            let mut checksum: u32 = 0;
            for byte in payload.iter() {
                checksum = checksum.wrapping_add(*byte as u32);
            }
            
            // Create a response with the processed data
            // This allocates a new buffer, but only once at the end of processing
            let mut response = BytesMut::with_capacity(payload.len() + 4);
            response.put_u32(checksum);
            response.put(payload);
            
            Ok(response.freeze())
        })
    }
}

// Example usage
async fn zero_copy_example() {
    // Create some test data
    let mut data = BytesMut::with_capacity(1024);
    data.put_slice(&[0x89, b'P', b'N', b'G', 0x0D, 0x0A, 0x1A, 0x0A]); // PNG header
    data.put_slice(&[0; 100]); // Payload
    
    let input = data.freeze();
    
    // Create the processor
    let mut processor = ZeroCopyProcessor::new();
    
    // Process the data
    match processor.ready().await {
        Ok(mut service) => {
            match service.call(input).await {
                Ok(output) => {
                    println!("Processed {} bytes, checksum: {:?}",
                             output.len() - 4,
                             &output[0..4]);
                }
                Err(e) => {
                    eprintln!("Processing error: {}", e);
                }
            }
        }
        Err(e) => {
            eprintln!("Service not ready: {}", e);
        }
    }
}
```

This example demonstrates how to process data with minimal copying by using the `Bytes` type, which provides an immutable view into a buffer that can be sliced and shared without copying the underlying data.

### Integration with Bytes and BytesMut

The `bytes` crate provides efficient buffer types that work well with Tower services:

```rust
use bytes::{Bytes, BytesMut, Buf, BufMut};
use http::{Request, Response, StatusCode};
use tower::Service;

// A service that transforms HTTP requests with zero-copy techniques
struct HttpTransformer {
    // Configuration would go here
}

impl Service<Request<Bytes>> for HttpTransformer {
    type Response = Response<Bytes>;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: Request<Bytes>) -> Self::Future {
        // Extract parts without copying
        let (parts, body) = req.into_parts();
        
        Box::pin(async move {
            // Process based on content type
            let content_type = parts.headers.get("content-type")
                .and_then(|h| h.to_str().ok())
                .unwrap_or("application/octet-stream");
            
            let processed_body = match content_type {
                "application/json" => {
                    // Process JSON data
                    // In a real implementation, you might use serde_json with zero-copy techniques
                    // Here we'll just add a field to demonstrate transformation
                    
                    // This does require a copy for the transformation
                    let mut json_data = String::from_utf8(body.to_vec())?;
                    
                    // Simple transformation (in reality, use a proper JSON parser)
                    if json_data.ends_with("}") {
                        json_data.truncate(json_data.len() - 1);
                        json_data.push_str(",\"processed\":true}");
                    }
                    
                    Bytes::from(json_data)
                },
                "text/plain" => {
                    // For text, we'll just uppercase it
                    let text = String::from_utf8(body.to_vec())?;
                    Bytes::from(text.to_uppercase())
                },
                _ => {
                    // For other types, pass through unchanged (zero copy)
                    body
                }
            };
            
            // Build the response
            let response = Response::builder()
                .status(StatusCode::OK)
                .header("content-type", content_type)
                .header("content-length", processed_body.len())
                .body(processed_body)
                .unwrap();
                
            Ok(response)
        })
    }
}

// Example usage
async fn http_transformer_example() {
    // Create a JSON request
    let json_request = Request::builder()
        .uri("/api/data")
        .header("content-type", "application/json")
        .body(Bytes::from(r#"{"name":"test","value":42}"#))
        .unwrap();
        
    // Create the transformer
    let mut transformer = HttpTransformer {};
    
    // Process the request
    match transformer.ready().await {
        Ok(mut service) => {
            match service.call(json_request).await {
                Ok(response) => {
                    println!("Status: {}", response.status());
                    println!("Body: {:?}", String::from_utf8_lossy(response.body()));
                }
                Err(e) => {
                    eprintln!("Processing error: {}", e);
                }
            }
        }
        Err(e) => {
            eprintln!("Service not ready: {}", e);
        }
    }
}
```

### Buffer Management in a Service Stack

Proper buffer management is crucial for zero-copy processing in a service stack. Here's an example of a pipeline that processes data with minimal copying:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use bytes::{Bytes, BytesMut, Buf, BufMut};
use tower::{Service, ServiceBuilder, ServiceExt, Layer};
use futures::future::{self, BoxFuture, FutureExt};

// A layer that adds a header to the data
struct HeaderLayer {
    header: Bytes,
}

impl HeaderLayer {
    fn new(header: Bytes) -> Self {
        HeaderLayer { header }
    }
}

impl<S> Layer<S> for HeaderLayer {
    type Service = HeaderService<S>;

    fn layer(&self, service: S) -> Self::Service {
        HeaderService {
            inner: service,
            header: self.header.clone(),
        }
    }
}

struct HeaderService<S> {
    inner: S,
    header: Bytes,
}

impl<S> Service<Bytes> for HeaderService<S>
where
    S: Service<Bytes>,
    S::Future: Send + 'static,
    S::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    type Response = S::Response;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx).map_err(Into::into)
    }

    fn call(&mut self, data: Bytes) -> Self::Future {
        // Combine header and data without copying the data
        let mut buffer = BytesMut::with_capacity(self.header.len() + data.len());
        buffer.put(self.header.clone());
        buffer.put(data);
        
        let combined = buffer.freeze();
        let future = self.inner.call(combined);
        
        Box::pin(async move {
            future.await.map_err(Into::into)
        })
    }
}

// A layer that compresses data
struct CompressionLayer {
    level: i32,
}

impl CompressionLayer {
    fn new(level: i32) -> Self {
        CompressionLayer { level }
    }
}

impl<S> Layer<S> for CompressionLayer {
    type Service = CompressionService<S>;

    fn layer(&self, service: S) -> Self::Service {
        CompressionService {
            inner: service,
            level: self.level,
        }
    }
}

struct CompressionService<S> {
    inner: S,
    level: i32,
}

impl<S> Service<Bytes> for CompressionService<S>
where
    S: Service<Bytes>,
    S::Future: Send + 'static,
    S::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    type Response = S::Response;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx).map_err(Into::into)
    }

    fn call(&mut self, data: Bytes) -> Self::Future {
        // In a real implementation, this would use a compression library
        // For this example, we'll just simulate compression
        
        // This does require a copy for the transformation
        let mut compressed = BytesMut::with_capacity(data.len());
        compressed.put_u32(self.level as u32); // Compression level
        compressed.put_u32(data.len() as u32); // Original size
        
        // Simulate compression (just copy the data in this example)
        compressed.put(data);
        
        let future = self.inner.call(compressed.freeze());
        
        Box::pin(async move {
            future.await.map_err(Into::into)
        })
    }
}

// A base service that processes the data
struct DataProcessor;

impl Service<Bytes> for DataProcessor {
    type Response = Bytes;
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Future = BoxFuture<'static, Result<Self::Response, Self::Error>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, data: Bytes) -> Self::Future {
        Box::pin(async move {
            // Process the data (in this example, just echo it back)
            Ok(data)
        })
    }
}

// Example usage
async fn buffer_management_example() {
    // Create a service stack with proper buffer management
    let mut service = ServiceBuilder::new()
        .layer(HeaderLayer::new(Bytes::from("HEADER:")))
        .layer(CompressionLayer::new(9))
        .service(DataProcessor);
    
    // Create some test data
    let data = Bytes::from("Hello, world!");
    
    // Process the data
    match service.ready().await {
        Ok(mut svc) => {
            match svc.call(data).await {
                Ok(result) => {
                    // In a real implementation, we would decompress and parse the result
                    println!("Processed data: {:?}", result);
                }
                Err(e) => {
                    eprintln!("Processing error: {}", e);
                }
            }
        }
        Err(e) => {
            eprintln!("Service not ready: {}", e);
        }
    }
}
```

This example demonstrates a service stack that processes data with minimal copying, using `Bytes` and `BytesMut` for efficient buffer management.

Zero-copy techniques are essential for high-performance networking applications, and Tower's flexible design makes it easy to implement these patterns. By using appropriate buffer types and careful memory management, you can build Tower services that process data efficiently with minimal overhead.

## Error Handling and Resilience

Error handling in Tower services goes beyond simple `Result` types. Building resilient systems requires thinking about failure modes, recovery strategies, and graceful degradation.

### Error Types in Tower

Tower services use associated `Error` types, allowing each service to define appropriate error semantics:

```rust
impl Service<Request> for MyService {
    type Response = Response;
    type Error = MyError;  // Your custom error type
    // ...
}
```

A common pattern is using boxed trait objects for flexibility:

```rust
type BoxError = Box<dyn std::error::Error + Send + Sync + 'static>;
```

This allows different layers in a stack to produce different error types while maintaining a uniform interface.

### Error Propagation Through Layers

When composing layers, errors flow through the stack. Each layer can:
1. **Pass through**: Let the error propagate unchanged
2. **Transform**: Convert the error to a different type
3. **Handle**: Catch and recover from specific errors
4. **Wrap**: Add context while preserving the original error

```rust
impl<S, Request> Service<Request> for MyLayer<S>
where
    S: Service<Request>,
    S::Error: Into<BoxError>,  // Accept any error type
{
    type Error = BoxError;
    
    fn call(&mut self, request: Request) -> Self::Future {
        let future = self.inner.call(request);
        Box::pin(async move {
            future.await.map_err(|e| {
                // Transform the error, adding context
                let boxed: BoxError = e.into();
                format!("MyLayer error: {}", boxed).into()
            })
        })
    }
}
```

### Retry Strategies

Retries are fundamental to building resilient services. The key considerations are:

1. **What to retry**: Not all errors are retryable (e.g., 4xx HTTP errors shouldn't be retried)
2. **How many times**: Limit retries to prevent infinite loops
3. **When to retry**: Use backoff strategies to avoid hammering failing services
4. **Budget management**: Limit total retry attempts across all requests

A simple retry service might look like:

```rust
pub struct RetryService<S, P> {
    inner: S,
    policy: P,
    max_retries: usize,
}

/// A policy that decides whether to retry a failed request
pub trait RetryPolicy<E> {
    /// Returns true if the error is retryable
    fn is_retryable(&self, error: &E) -> bool;
    
    /// Returns how long to wait before retrying
    fn backoff(&self, attempt: usize) -> Duration;
}
```

### Circuit Breakers

Circuit breakers prevent cascading failures by failing fast when a service is unhealthy:

```
     ┌─────────┐      success      ┌────────┐
     │         │ ─────────────────→│        │
     │ CLOSED  │                   │  OPEN  │──┐
     │         │←────────────────  │        │  │ timeout
     └─────────┘   failures > N    └────────┘  │
          ↑                             │      │
          │         success             ▼      │
          │        ┌──────────────────────┐   │
          └────────│     HALF-OPEN       │←──┘
                   │  (test requests)    │
                   └──────────────────────┘
```

States:
- **Closed**: Normal operation, requests flow through
- **Open**: Service is failing, immediately reject requests
- **Half-Open**: Cautiously allow some requests to test recovery

### Timeouts

Timeouts prevent requests from hanging indefinitely:

```rust
pub struct TimeoutService<S> {
    inner: S,
    timeout: Duration,
}

impl<S, Request> Service<Request> for TimeoutService<S>
where
    S: Service<Request>,
{
    type Error = TimeoutError<S::Error>;
    
    fn call(&mut self, request: Request) -> Self::Future {
        let future = self.inner.call(request);
        let timeout = self.timeout;
        
        Box::pin(async move {
            match tokio::time::timeout(timeout, future).await {
                Ok(result) => result.map_err(TimeoutError::Inner),
                Err(_) => Err(TimeoutError::Elapsed),
            }
        })
    }
}
```

**Important**: Timeouts should be placed early in the layer stack (wrapped around other layers) so they apply to the total request time, not just the inner service.

### Bulkheads

Bulkheads isolate failures by limiting resource usage per service or operation:

```rust
pub struct BulkheadService<S> {
    inner: S,
    semaphore: Arc<Semaphore>,
}

impl<S, Request> Service<Request> for BulkheadService<S>
where
    S: Service<Request>,
{
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // Check if we have an available permit
        if self.semaphore.available_permits() == 0 {
            // No permits available - apply backpressure
            Poll::Pending
        } else {
            self.inner.poll_ready(cx)
        }
    }
    
    fn call(&mut self, request: Request) -> Self::Future {
        let permit = self.semaphore.clone().acquire_owned();
        let future = self.inner.call(request);
        
        Box::pin(async move {
            let _permit = permit.await.unwrap();  // Hold permit during request
            future.await
            // Permit is released when dropped
        })
    }
}
```

### Hedged Requests

For latency-sensitive applications, hedged requests send the same request to multiple backends and use the first response:

```rust
// Send request to primary, if no response in 50ms, also send to backup
pub async fn hedged_call<S, Request>(
    services: &mut [S],
    request: Request,
    hedge_delay: Duration,
) -> Result<S::Response, S::Error>
where
    S: Service<Request>,
    Request: Clone,
{
    // Start with primary
    let primary = services[0].call(request.clone());
    
    tokio::select! {
        result = primary => result,
        _ = tokio::time::sleep(hedge_delay) => {
            // Primary is slow, try backup too
            let backup = services[1].call(request);
            tokio::select! {
                result = primary => result,
                result = backup => result,
            }
        }
    }
}
```

### Combining Resilience Patterns

In production, you typically combine multiple patterns:

```rust
let service = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(30)))    // Overall timeout
    .layer(CircuitBreakerLayer::new(5, Duration::from_secs(60)))  // Fail fast
    .layer(RetryLayer::new(3, ExponentialBackoff::new()))  // Retry transient errors
    .layer(BulkheadLayer::new(100))  // Limit concurrency
    .service(BackendService::new());
```

The order matters:
1. **Timeout** (outermost): Limits total time including retries
2. **Circuit Breaker**: Prevents retrying when service is down
3. **Retry**: Handles transient failures
4. **Bulkhead** (innermost): Limits concurrent requests to backend

### Key Resilience Principles

1. **Fail fast**: Don't wait forever; use timeouts
2. **Fail gracefully**: Return sensible defaults or cached data when possible
3. **Isolate failures**: Use bulkheads to prevent one failure from affecting everything
4. **Recover automatically**: Circuit breakers allow systems to heal
5. **Be transparent**: Log failures and expose metrics for debugging
6. **Test failures**: Use chaos engineering to verify resilience

## Practical Examples

The `towerdemo` crate included with this tutorial contains comprehensive exercises covering all the concepts discussed here. The exercises are designed to reinforce your understanding through hands-on implementation.

### Running the Exercises

```bash
cd crates/towerdemo
cargo build
```

### Exercise Overview

The exercises progress from fundamentals to advanced topics:

| Exercise | Topic | Key Concepts |
|----------|-------|--------------|
| ex01 | Service Basics | Implementing `Service` trait, `poll_ready`, `call` |
| ex02 | Service Combinators | Composing and transforming services |
| ex03 | Basic Layer | Creating custom layers, the `Layer` trait |
| ex04 | Middleware Stack | Composing multiple layers with `ServiceBuilder` |
| ex05 | Backpressure Basics | Using `poll_ready` for flow control |
| ex06 | Rate Limiting | Token bucket, fixed window algorithms |
| ex07 | Load Balancer | Round-robin, least-connections, P2C |
| ex08 | Discovery Integration | Dynamic service discovery |
| ex09 | Buffer Management | Buffer pools, efficient memory usage |
| ex10 | Zero-Copy Processing | `Bytes`, `BytesMut`, avoiding allocations |
| ex11 | HTTP Proxy | Complete HTTP proxy implementation |
| ex12 | TCP Proxy | Low-level TCP forwarding |

### Recommended Learning Path

1. **Start with basics** (ex01-ex04): Understand the core abstractions
2. **Learn reliability patterns** (ex05-ex06): Critical for production systems
3. **Master distribution** (ex07-ex08): Load balancing and discovery
4. **Optimize performance** (ex09-ex10): Buffer management and zero-copy
5. **Build real systems** (ex11-ex12): Apply everything together

Each exercise includes detailed comments explaining the concepts and what needs to be implemented. Work through them at your own pace—they're designed to build on each other.

## Conclusion

Tower's abstractions provide a powerful foundation for building robust, composable networking applications in Rust. By understanding the `Service` trait, `Layer` composition, pinning, and resilience patterns, you can build systems that are both flexible and reliable.

The key takeaways:

1. **Always respect the `poll_ready` contract**: Call `ready().await` before `call()` to ensure system reliability
2. **Use layers for middleware**: Compose behavior in a clean, modular way without modifying services
3. **Understand pinning**: Use `Pin` and `pin-project` when implementing custom futures; know when `Box::pin` is acceptable
4. **Leverage the `NewService` pattern**: Create per-connection services for proper resource isolation
5. **Implement proper backpressure**: Use `poll_ready` to signal capacity and prevent cascading failures
6. **Build resilient systems**: Combine timeouts, retries, circuit breakers, and bulkheads appropriately
7. **Use zero-copy techniques**: Leverage `Bytes` and `BytesMut` for efficient buffer management
8. **Design for composition**: Create small, focused services that can be combined into larger systems
9. **Handle errors explicitly**: Use the type system to make error handling clear and composable

### Performance Optimization with Tower

When optimizing Tower-based applications for performance:

1. **Minimize Allocations**: Use buffer pooling and zero-copy techniques to reduce memory allocations in hot paths.

2. **Respect Backpressure**: Properly implement and respect the `poll_ready` contract to prevent resource exhaustion.

3. **Batch Processing**: Where appropriate, batch multiple small requests into larger ones to amortize overhead.

4. **Concurrency Control**: Use Tower's concurrency limiting layers to prevent overwhelming downstream services.

5. **Buffer Management**: Use appropriate buffer types like `Bytes` and `BytesMut` for efficient memory usage.

6. **Layer Ordering**: The order of layers matters for performance. Put frequently rejecting layers (like rate limiters) earlier in the stack.

7. **Avoid Blocking**: Keep all service implementations non-blocking to maintain high throughput.

8. **Metrics and Monitoring**: Instrument your services to identify bottlenecks and optimize the right components.

By following these principles, you can build Tower-based systems that are not only robust and maintainable but also highly performant.

For more information, check out the [Tower documentation](https://docs.rs/tower).

## High Performance Tower Service Optimization

Building a correct Tower service is the first milestone; making it fast enough for production traffic is the second. This section covers concrete, measurable optimizations organized by impact area.

### 1. Use Named Future Types Instead of `Box::pin`

`Box::pin` is the easiest way to return a `Future` from `Service::call`, but it heap-allocates on **every single call**. In a service handling 100 000 RPS, that is 100 000 allocations per second just for the future wrapper.

Replace `Box<dyn Future>` with a named struct that implements `Future` directly, using `pin-project`:

```rust
use pin_project::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// ── Before (allocates on every call) ─────────────────────────────────────────
pub struct BoxedService<S> { inner: S }

impl<S, Req> tower::Service<Req> for BoxedService<S>
where
    S: tower::Service<Req>,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error    = S::Error;
    type Future   = Pin<Box<dyn Future<Output = Result<S::Response, S::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Req) -> Self::Future {
        Box::pin(self.inner.call(req)) // 👎 heap allocation every call
    }
}

// ── After (zero-allocation wrapper) ─────────────────────────────────────────
#[pin_project]
pub struct NamedFuture<F> {
    #[pin]
    inner: F,
}

impl<F: Future> Future for NamedFuture<F> {
    type Output = F::Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.project().inner.poll(cx)
    }
}

pub struct ZeroAllocService<S> { inner: S }

impl<S, Req> tower::Service<Req> for ZeroAllocService<S>
where
    S: tower::Service<Req>,
{
    type Response = S::Response;
    type Error    = S::Error;
    type Future   = NamedFuture<S::Future>; // concrete type — no allocation

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Req) -> Self::Future {
        NamedFuture { inner: self.inner.call(req) } // 👍 stack-allocated
    }
}
```

**When `Box::pin` is acceptable**: prototyping, rarely-called paths, or when the wrapped future is inherently large and heap-allocating it saves stack pressure.

### 2. Minimize Clones in `Service::call`

Because `Service::call` takes `&mut self`, you can't hold a reference across an `await` point—Rust's borrow checker enforces this. The typical fix is to clone shared handles:

```rust
// ── Anti-pattern: cloning an Arc on every call ────────────────────────────
pub struct ExpensiveClonerService {
    db: Arc<DatabasePool>, // cloned every call
}

impl tower::Service<Request> for ExpensiveClonerService {
    // …
    fn call(&mut self, req: Request) -> Self::Future {
        let db = self.db.clone(); // 👎 atomic inc/dec on every call
        Box::pin(async move { db.query(req).await })
    }
}
```

Prefer calling `ready_and` / `poll_ready` on the *inner* service and keeping only cheap clones (or none at all) in the future:

```rust
// ── Better: pre-clone once at service construction time ─────────────────
pub struct EfficientService {
    db:     Arc<DatabasePool>,
    // If the service is called from a single task, avoid Arc entirely:
    // db: DatabasePool,
    // A channel sender is cheap to clone (atomics, no data copy):
    metrics_tx: tokio::sync::mpsc::UnboundedSender<Metric>,
}

impl tower::Service<Request> for EfficientService {
    type Response = Response;
    type Error    = Error;
    type Future   = NamedFuture<impl Future<Output = Result<Response, Error>>>;

    fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: Request) -> Self::Future {
        // Borrow `db` without cloning by using `Arc::clone` only lazily:
        let db = Arc::clone(&self.db);
        let tx = self.metrics_tx.clone(); // cheap — just clones the Sender end
        NamedFuture {
            inner: async move {
                let result = db.query(&req).await?;
                let _ = tx.send(Metric::Request);
                Ok(result)
            }
        }
    }
}
```

### 3. Buffer Pooling to Eliminate Hot-Path Allocations

For services that serialize/deserialize on every request, a thread-local buffer pool avoids per-request `Vec`/`Bytes` allocations:

```rust
use bytes::{Bytes, BytesMut};
use std::cell::RefCell;

// One encode buffer per Tokio worker thread — zero contention.
thread_local! {
    static ENCODE_BUF: RefCell<BytesMut> =
        RefCell::new(BytesMut::with_capacity(8 * 1024)); // 8 KiB initial
}

/// Encode `payload` without allocating a new buffer each time.
pub fn encode_pooled(payload: &impl serde::Serialize) -> Bytes {
    ENCODE_BUF.with(|cell| {
        let mut buf = cell.borrow_mut();
        buf.clear();
        // serde_json writes directly into the BytesMut
        serde_json::to_writer(&mut *buf, payload).expect("serialization failed");
        buf.clone().freeze() // Arc-backed, O(1) clone into caller
    })
}
```

For larger buffer pools shared across tasks, use `object_pool` or `crossbeam`'s `SegQueue`:

```rust
use crossbeam::queue::ArrayQueue;
use bytes::BytesMut;
use std::sync::Arc;

/// A fixed-size pool of BytesMut buffers.
#[derive(Clone)]
pub struct BufferPool {
    pool: Arc<ArrayQueue<BytesMut>>,
    buf_capacity: usize,
}

impl BufferPool {
    pub fn new(pool_size: usize, buf_capacity: usize) -> Self {
        let pool = Arc::new(ArrayQueue::new(pool_size));
        for _ in 0..pool_size {
            let _ = pool.push(BytesMut::with_capacity(buf_capacity));
        }
        Self { pool, buf_capacity }
    }

    /// Acquire a buffer; falls back to fresh allocation if pool is empty.
    pub fn acquire(&self) -> BytesMut {
        self.pool
            .pop()
            .unwrap_or_else(|| BytesMut::with_capacity(self.buf_capacity))
    }

    /// Return a buffer to the pool; drops it if pool is full.
    pub fn release(&self, mut buf: BytesMut) {
        buf.clear();
        let _ = self.pool.push(buf); // silently drops if at capacity
    }
}
```

### 4. Concurrency-Aware Layer Ordering

The order of layers in your `ServiceBuilder` directly affects throughput and latency. A frequently-firing rejection layer placed *after* an expensive layer wastes work:

```rust
// ── Suboptimal order ─────────────────────────────────────────────────────
let svc = ServiceBuilder::new()
    .layer(ExpensiveAuthLayer::new())   // 👎 auth runs before rate limit check
    .layer(RateLimitLayer::new(1000, Duration::from_secs(1)))
    .layer(TimeoutLayer::new(Duration::from_secs(10)))
    .service(backend);

// ── Optimized order ──────────────────────────────────────────────────────
let svc = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(10)))  // outermost: bounds total work
    .layer(RateLimitLayer::new(1000, Duration::from_secs(1))) // cheap, rejects early
    .layer(ExpensiveAuthLayer::new())  // 👍 only runs if under rate limit
    .service(backend);
```

General ordering heuristic, from **outermost to innermost**:

| Position | Layer type | Rationale |
|----------|------------|----------|
| 1st | Metrics / Tracing | Must see every request, including rejected ones |
| 2nd | Timeout | Bounds total time including all inner rejections |
| 3rd | Rate limiter | Cheap check; rejects overload before any work is done |
| 4th | Circuit breaker | Skips auth + serialization when backend is down |
| 5th | Authentication | Relatively expensive—only runs for non-rate-limited requests |
| 6th | Retry | Close to the backend; doesn't re-auth on retry |
| 7th | Load balancer | Selects backend after all policy decisions are made |
| Last | Backend service | Actual work |

### 5. Avoiding Unnecessary Waker Registrations

Every call to `Poll::Pending` from `poll_ready` or a future schedules a wakeup. Avoid spurious wakeups with careful use of `AtomicBool` guards:

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::task::{Context, Poll, Waker};

pub struct ThrottledService<S> {
    inner:    S,
    open:     Arc<AtomicBool>,     // true = accepting requests
    waker_tx: tokio::sync::watch::Sender<()>,
    waker_rx: tokio::sync::watch::Receiver<()>,
}

impl<S, Req> tower::Service<Req> for ThrottledService<S>
where
    S: tower::Service<Req>,
{
    type Response = S::Response;
    type Error    = S::Error;
    type Future   = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), S::Error>> {
        if self.open.load(Ordering::Acquire) {
            // Fast path: gate is open, forward immediately.
            return self.inner.poll_ready(cx);
        }
        // Gate is closed: register waker on the watch channel and return Pending.
        // When the gate opens, the watch channel notifies all waiting tasks.
        let mut watch = self.waker_rx.clone();
        let waker = cx.waker().clone();
        tokio::spawn(async move {
            let _ = watch.changed().await;
            waker.wake();
        });
        Poll::Pending
    }

    fn call(&mut self, req: Req) -> Self::Future {
        self.inner.call(req)
    }
}
```

### 6. Shared State Without Global Locking — Using `tokio::sync::RwLock`

For read-heavy shared state (e.g. routing tables, config snapshots), prefer
`tokio::sync::RwLock` over `std::sync::Mutex`. Unlike the `std` version, it
yields to the async executor instead of blocking the OS thread:

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
pub struct RoutingTable {
    routes: Arc<RwLock<HashMap<String, String>>>,
}

impl RoutingTable {
    pub async fn lookup(&self, key: &str) -> Option<String> {
        // Multiple readers can proceed concurrently.
        self.routes.read().await.get(key).cloned()
    }

    pub async fn update(&self, key: String, value: String) {
        // Exclusive write — all readers yield until this completes.
        self.routes.write().await.insert(key, value);
    }
}

// For read-mostly tables that change infrequently, prefer a watch channel:
// the latest snapshot is cloned atomically with no lock at all.
use tokio::sync::watch;

pub struct WatchedRoutes {
    sender:   watch::Sender<Arc<HashMap<String, String>>>,
    receiver: watch::Receiver<Arc<HashMap<String, String>>>,
}

impl WatchedRoutes {
    pub fn current(&self) -> Arc<HashMap<String, String>> {
        Arc::clone(&*self.receiver.borrow()) // zero-copy snapshot
    }
}
```

### 7. Profiling a Tower Service in Production

Before optimizing, measure. The `tracing` ecosystem integrates directly with
Tower via `tower-http`'s `TraceLayer`. Use it alongside `tokio-console` for
async task profiling and `flamegraph` for CPU profiling:

```rust
// Add to your binary to enable tokio-console instrumentation
// (requires RUSTFLAGS="--cfg tokio_unstable")
#[cfg(feature = "tokio-console")]
console_subscriber::init();

// Structured latency spans with fine-grained field capture
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::Level;

let svc = ServiceBuilder::new()
    .layer(
        TraceLayer::new_for_http()
            .make_span_with(
                DefaultMakeSpan::new()
                    .level(Level::INFO)
                    .include_headers(true), // capture headers for debugging
            )
            .on_response(
                DefaultOnResponse::new()
                    .level(Level::INFO)
                    .latency_unit(tower_http::LatencyUnit::Micros), // µs precision
            ),
    )
    .service(your_service);
```

Generate a flamegraph:

```bash
# Install cargo-flamegraph
cargo install flamegraph

# Profile a 30-second window of your server
FRAMEGRAPH=1 cargo flamegraph --bin my_tower_server -- --duration 30
# Open flamegraph.svg in a browser to find hot functions
```

### 8. Benchmarking Tower Services with Criterion

Benchmark individual layers in isolation to identify regressions early:

```rust
// benches/tower_bench.rs
use criterion::{criterion_group, criterion_main, BenchmarkId, Criterion};
use tower::{Service, ServiceExt};

async fn make_service() -> impl Service<String, Response = String, Error = std::convert::Infallible> {
    tower::ServiceBuilder::new()
        .layer(tower::limit::ConcurrencyLimitLayer::new(1000))
        .layer(tower::load_shed::LoadShedLayer::new())
        .service_fn(|req: String| async move { Ok::<_, std::convert::Infallible>(req) })
}

fn bench_stack_throughput(c: &mut Criterion) {
    let rt = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(4)
        .build()
        .unwrap();

    let mut svc = rt.block_on(make_service());

    let mut group = c.benchmark_group("tower_stack");
    for payload_len in [64usize, 256, 1024, 4096] {
        group.bench_with_input(
            BenchmarkId::from_parameter(payload_len),
            &payload_len,
            |b, &len| {
                let payload = "x".repeat(len);
                b.to_async(&rt).iter(|| async {
                    svc.ready().await.unwrap().call(payload.clone()).await.unwrap()
                });
            },
        );
    }
    group.finish();
}

criterion_group!(benches, bench_stack_throughput);
criterion_main!(benches);
```

Run with:

```bash
cargo bench --bench tower_bench
# Results are saved to target/criterion/tower_stack/
# Open target/criterion/report/index.html for charts
```

### 9. The Optimize–Measure–Repeat Loop

High performance is iterative. Follow this cycle:

```
1. Establish a baseline benchmark (Criterion + ghz)
2. Profile to identify the bottleneck (flamegraph + tokio-console)
3. Apply the most impactful optimization
4. Re-run benchmarks to confirm improvement
5. Commit only if there is a measurable, reproducible gain
6. Repeat
```

Common Tower-specific bottlenecks and their fixes:

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| High CPU, low throughput | `Box::pin` allocation in hot path | Named future types |
| GC-like pauses in `perf` | `Arc` clone storms | Pre-clone handles; use `watch` channels |
| `poll_ready` high latency | Lock contention on shared state | `DashMap`, atomics, or `RwLock` |
| Memory growth under load | Unbounded channel backlog | Size channels; use `try_send` + drop + metric |
| Layer stack 50th% fine, 99th% bad | Expensive layer after cheap reject layer | Reorder: cheap rejects first |
| Spurious wakeups in profiler | Busy-wait pollers | Replace with `watch` / `Notify` |

### 10. Production-Ready High-Performance Stack Template

Combine all the above into a starting template:

```rust
use std::time::Duration;
use tower::ServiceBuilder;
use tower_http::{
    classify::ServerErrorsAsFailures,
    compression::CompressionLayer,
    timeout::TimeoutLayer,
    trace::TraceLayer,
};

pub fn build_high_perf_stack<S>(backend: S) -> impl tower::Service<http::Request<bytes::Bytes>>
where
    S: tower::Service<http::Request<bytes::Bytes>> + Clone + Send + 'static,
    S::Future: Send + 'static,
    S::Response: Into<http::Response<bytes::Bytes>>,
    S::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    ServiceBuilder::new()
        // 1. Metrics / tracing — outermost, sees everything
        .layer(TraceLayer::new(ServerErrorsAsFailures::new_for_grpc()))
        // 2. Hard timeout — bounds total time including retries
        .layer(TimeoutLayer::new(Duration::from_secs(30)))
        // 3. Rate limit — cheap, rejects overload before encoding
        .layer(tower::limit::RateLimitLayer::new(10_000, Duration::from_secs(1)))
        // 4. Concurrency limit — prevents thundering-herd on the backend
        .layer(tower::limit::ConcurrencyLimitLayer::new(512))
        // 5. Load shedding — returns 503 immediately when at capacity
        .layer(tower::load_shed::LoadShedLayer::new())
        // 6. Response compression — reduces egress bytes
        .layer(CompressionLayer::new())
        // 7. The actual backend
        .service(backend)
}
```