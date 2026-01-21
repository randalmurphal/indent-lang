# Structured Concurrency for a Safety-First Backend Language

A new backend language targeting Go's simplicity with Rust's safety must make careful tradeoffs in its concurrency model. The central tension: **Go's unstructured `go func()` is ergonomically ideal but enables goroutine leaks and race conditions, while Rust's explicit async/await provides safety at the cost of function coloring**. This report examines how modern implementations navigate these tradeoffs and identifies design patterns that could achieve "the right thing easy, the wrong thing hard."

## The core design decision: function coloring vs runtime complexity

The "colored function problem" (Bob Nystrom, 2015) describes languages where async functions can only be called from async contexts, creating a **viral propagation** through codebases. Go uniquely avoids this by making all I/O appear synchronous while the runtime handles async operations transparently through its netpoller. This design requires a sophisticated scheduler (the **GMP model**) but delivers remarkable developer ergonomics.

The tradeoff is stark: Rust's tokio runtime consumes just **64 bytes per task** with ~10ns scheduling overhead, while Go goroutines require **2KB initial stacks** and ~1-5µs spawn time. At one million concurrent tasks, benchmarks show Rust using ~100MB versus Go's ~1.2GB—a **12× memory difference**. For backend services handling millions of connections, this matters. Discord's migration from Go to Rust eliminated predictable 10-40ms GC latency spikes caused by goroutine memory pressure.

However, Go's colorless model has profound ergonomic benefits. Any function can block on I/O without the caller needing to know—database queries, HTTP calls, and file operations share identical syntax. This achieves your goal of "95% of code shouldn't need special knowledge." The question is whether a new language can capture this simplicity while adding structured concurrency guarantees.

## How structured concurrency actually works inside the runtime

Kotlin, Swift, and Java Loom each implement structured concurrency differently, but share core mechanics: **parent-child hierarchies, automatic cancellation propagation, and scope-based lifetime enforcement**.

**Kotlin's approach** uses a Job hierarchy embedded in CoroutineContext. Every coroutine builder (`launch`, `async`) creates a child Job linked to its parent. The critical insight: coroutines enter a "Completing" state after their body finishes but wait for all children to complete before transitioning to "Completed." Cancellation propagates via `CancellationException`, which is specially handled—ignored by error handlers but causes the coroutine to exit cleanly at suspension points.

```kotlin
suspend fun processOrders() = coroutineScope {
    val user = async { fetchUser() }     // Child 1
    val orders = async { fetchOrders() } // Child 2
    UserWithPosts(user.await(), orders.await())
}  // Block waits for BOTH children; exception in either cancels sibling
```

**Swift's runtime** maintains a task tree where each Task contains a reference to its parent. The `withTaskGroup` function creates a scope that **cannot exit until all children complete**, even if you return early from the closure body. Swift's cooperative thread pool is fixed at CPU core count, preventing thread explosion—a deliberate constraint that trades potential throughput for predictability.

**Java Loom's StructuredTaskScope** (JEP 453) uses try-with-resources to enforce scope completion. The `fork()` method returns `Subtask<T>` rather than `Future<T>` to discourage unstructured patterns. Built-in policies like `ShutdownOnFailure` (all-or-nothing semantics) and `ShutdownOnSuccess` (racing pattern) provide common patterns with strong guarantees.

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Supplier<User> user = scope.fork(() -> fetchUser());
    Supplier<List<Order>> orders = scope.fork(() -> fetchOrders());
    scope.join().throwIfFailed();
    return new Response(user.get(), orders.get());
}  // Scope guarantees cleanup; exception cancels all subtasks
```

## Go's scheduler mechanics reveal both brilliance and limitations

The GMP model separates **G (goroutines)**, **M (OS threads)**, and **P (processors/logical CPUs)**. This three-way split enables efficient scheduling: when a goroutine blocks on I/O, its P detaches and attaches to another M, allowing work to continue. The netpoller converts async I/O (via epoll/kqueue/iocp) into goroutine-level blocking semantics—the magic that enables colorless functions.

Work stealing operates on per-P local run queues of **256 entries**. When a P's queue is empty, it steals half of another P's queue. The scheduler checks the global run queue every 1/61 iterations (61 is prime, reducing hash collisions) to prevent starvation. Spinning threads—idle Ms that actively look for work rather than sleeping—minimize wake-up latency at the cost of some CPU cycles.

**Async preemption** (Go 1.14+) solved the historical problem of tight loops starving other goroutines. The sysmon thread detects goroutines running >10ms and sends SIGURG signals to trigger preemption. The runtime captures CPU state and, if at a GC-safe point, yields the goroutine. This works because the compiler generates stack/register maps at nearly every instruction.

Key failure modes to avoid in a new language:
- **Container GOMAXPROCS mismatch**: Go defaults to host CPU count, not container limits, causing excessive context switching
- **runnext starvation** (current issue #73964): Short-sleeping goroutines repeatedly landing in the priority slot can starve others
- **Thread explosion from blocking FFI**: Each blocking cgo call occupies an OS thread, risking ulimit exhaustion

## The FFI problem constrains all concurrency designs

All green thread implementations face a fundamental constraint: **native code is opaque to the runtime**. When a goroutine calls blocking C code via cgo, the M becomes stuck, the P detaches, and potentially a new OS thread is created. Each cgo call incurs ~40ns overhead—**100× slower than pure Go calls**—due to scheduler coordination and stack switching.

Rust's approach makes this explicit. The `tokio::task::spawn_blocking` function offloads blocking work to a dedicated thread pool (default ~500 threads). This is ergonomically heavier but prevents the silent performance degradation that occurs when developers accidentally block tokio's executor threads.

```rust
let result = tokio::task::spawn_blocking(|| {
    unsafe { blocking_ffi_call() }  // Runs on blocking pool, not executor
}).await?;
```

Java Loom introduces **pinning**: when a virtual thread makes a JNI call, it becomes pinned to its carrier thread and cannot unmount. If all carrier threads are pinned, throughput collapses. JDK 24 fixes synchronized-block pinning, but **native frame pinning is permanent**—the JVM cannot save/restore native stack frames.

For a new language, the implications are clear: **FFI interaction must be explicit** in the type system or API design. Silent blocking that degrades the scheduler is "magic" that violates your "explicit over magic" principle.

## Leak prevention requires structured lifetimes, not just syntax

Nathaniel J. Smith's insight that "go statements are a form of goto" crystallizes the problem. Both violate the **black box rule**: when you call a function, you should be able to treat it as sequential flow without knowing whether it spawns background work. Go's `go func()` breaks this—any function might leak goroutines that outlive their caller.

Trio's nursery model provides the strictest guarantees:

```python
async with trio.open_nursery() as nursery:
    nursery.start_soon(task_a)
    nursery.start_soon(task_b)
# Execution reaches here ONLY when both tasks complete
# If either throws, the other is cancelled and exceptions are grouped
```

The nursery cannot exit until all children complete. Cancellation and errors propagate automatically. Resources opened before the nursery are guaranteed to still exist while children run. Smith's comparison is compelling: the "Happy Eyeballs" TCP connection algorithm is ~600 lines in Twisted (unstructured) with logic bugs, versus ~40 lines in Trio that's correct on first try.

Kotlin balances structure with pragmatism. `coroutineScope` enforces structured concurrency, but `GlobalScope` exists as an explicit escape hatch marked `@DelicateCoroutinesApi`. The pattern of injecting scopes makes testing easier while maintaining safety:

```kotlin
class ArticlesRepository(
    private val scope: CoroutineScope = GlobalScope  // Injected for testing
) {
    suspend fun bookmark(article: Article) {
        scope.launch { saveToDatabase(article) }.join()
    }
}
```

## Performance realities favor explicit async under high load

Real-world benchmarks reveal meaningful differences. At one million concurrent sleeping tasks:

| Runtime | Memory Usage | Memory/Task |
|---------|--------------|-------------|
| Rust tokio | ~100MB | 64 bytes |
| Java Loom virtual threads | ~160MB | ~160 bytes |
| Go goroutines | ~1.2GB | 2KB initial |

Discord's Go→Rust migration showed p99 read latencies dropping from 40-125ms to ~15ms by eliminating GC pressure from goroutine stacks. Grab achieved **5× CPU efficiency improvement** (20 cores → 4.5 cores for 1,000 req/s) with 70% infrastructure cost reduction.

However, Kotlin's dispatcher benchmarks reveal a crucial nuance: **for suspending (I/O-bound) tasks, thread count doesn't matter**—performance is identical across 1-100 threads. The scheduler overhead only matters for blocking/CPU-bound work. This suggests a new language could offer colorless syntax for the common I/O case while requiring explicit constructs for CPU-bound work.

## Design recommendations for your new language

Given your principles ("obvious over clever," "explicit over magic," "progressive complexity"), here's a synthesis of the research:

**1. Structured by default with explicit escape hatches.** Follow Swift's hierarchy: `async let` for fixed concurrency → task groups for dynamic spawning → explicit `detached` for fire-and-forget. Make the unsafe path syntactically heavier.

**2. Colorless functions for I/O, but visible scheduling points.** Go's netpoller approach could be adopted, but with syntax that hints at potential yields—perhaps `await db.query()` that compiles to synchronous-looking code but marks suspension points in the source. This preserves readability while aiding reasoning about concurrent access.

**3. Make FFI blocking explicit in the type system.** A function calling C code that might block should require annotation or return a special type. This prevents silent scheduler starvation without requiring full function coloring.

**4. Lightweight tasks with bounded memory.** Target Rust's 64-byte overhead rather than Go's 2KB. This enables the million-connection use case without memory pressure. Use heap-allocated continuations (like Loom) rather than growable stacks.

**5. Supervisor scopes for fault tolerance.** Kotlin's distinction between `coroutineScope` (failure cancels siblings) and `supervisorScope` (failures isolated) maps to real backend patterns. A language should support both: transactional all-or-nothing semantics and supervisor trees for service isolation.

**6. Built-in leak detection.** Runtime tracking of task lifetimes with warnings when tasks outlive their logical scope catches bugs that static analysis misses. Go's goroutine profiling via pprof is a minimum; active detection during development is better.

The theoretical ceiling for your language: **Go's syntax with Kotlin's structured guarantees and Rust's memory efficiency**. Loom's continuation-based virtual threads suggest this is implementable—Java achieved it within JVM constraints. A language designed from scratch can go further, encoding structured concurrency in the type system while maintaining the readability of synchronous code.

The key insight from this research: structured concurrency isn't about restricting developers—it's about making the runtime's behavior match the code's visual structure. When a function returns, all work it spawned should be complete. This is the "obvious" behavior that Go's `go func()` violates and that your language should enforce by default.

---

## DECISION: Zig-Style Colorless Async via Io Interface

**See also:** `research/01-core/colorless_async_design.md` for detailed implementation research.

After extensive research, **Indent adopts a Zig-inspired Io interface approach** rather than pure stackful coroutines. This achieves colorless semantics with Rust-level memory efficiency.

### Architecture Overview

```
Indent Source Code (no async keyword)
              ↓
    Compiler Call Graph Analysis
              ↓
    ┌─────────────────────────────────┐
    │  Functions touching Io get      │
    │  transformed to state machines  │
    │  (stackless by default)         │
    └─────────────────────────────────┘
              ↓
    Optional stackful coroutines for
    FFI/complex cases via explicit context
```

### Recommended Syntax

```python
# Io context established at boundary
def fetch_user(id: int) -> User:
    return http.get(f"/users/{id}").json()  # Io implicitly available

def main():
    with async_io():  # Context block establishes Io
        concurrent:   # Structured concurrency block
            user = fetch_user(123)
            orders = fetch_orders(123)

        # Both complete here
        print(f"{user.name} has {len(orders)} orders")
```

### Key Design Decisions

| Factor | Zig-style Io | Go-style Stackful |
|--------|--------------|-------------------|
| Memory per task | Bytes (state machine) | KB (stack) |
| WASM support | Native | Requires stack switching emulation |
| FFI compatibility | Requires explicit handling | Seamless |
| Maximum concurrency | Millions of tasks | Limited by stack memory |

### Implementation Roadmap

1. **Phase 1:** Implement blocking Io (baseline, zero async overhead)
2. **Phase 2:** Add thread-pool Io (multiplexed blocking)
3. **Phase 3:** Add compiler-driven stackless coroutines
4. **Phase 4:** Optional stackful coroutines for FFI/recursive cases