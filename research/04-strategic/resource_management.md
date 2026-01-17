# Resource management design for Indent with regions+ARC

A regions+ARC memory model creates unique opportunities for non-memory resource management. By binding resource lifetimes to regions while using ARC for shared resources, Indent can achieve **deterministic cleanup without garbage collection** while supporting complex ownership patterns. The key insight: regions provide automatic scope-based cleanup for most resources, while ARC handles cases requiring shared ownership—but both need explicit support for async operations, cleanup ordering, and leak detection.

## The Resource trait: core abstraction for cleanup

Indent should define a `Resource` trait as the central abstraction for non-memory resource management. Based on analysis of Rust's Drop, Java's AutoCloseable, and C#'s IAsyncDisposable patterns, a **dual-mode cleanup design** handles both synchronous and asynchronous resources:

```indent
trait Resource {
    fn close(&mut self) -> Result<(), CloseError>
}

trait AsyncResource: Resource {
    async fn async_close(&mut self) -> Result<(), CloseError>
}
```

This design differs from Rust's approach in three critical ways. First, **close returns a Result** rather than silently failing—Rust's Drop cannot propagate errors, forcing developers to either ignore cleanup failures or panic. Second, **sync and async are separate traits**—the async drop problem has plagued Rust for years because their single Drop trait cannot await. Indent's explicit separation avoids this. Third, **close takes `&mut self`** rather than consuming self—this allows resources to be partially cleaned up and potentially reused in pool scenarios.

The compiler should generate "resource glue" that calls appropriate cleanup methods when values go out of scope. For types implementing only `Resource`, the synchronous `close()` runs automatically. For types implementing `AsyncResource`, the compiler inserts implicit await points when dropping in async contexts, or falls back to synchronous cleanup in sync contexts.

## Region-scoped resources enable bulk cleanup

Regions in Indent naturally align with resource lifetimes for request-scoped patterns. When a region is deallocated, all resources allocated within it should be cleaned up automatically in **reverse allocation order**—matching C++ destruction ordering:

```indent
region request {
    let conn = db_pool.get()      // Cleaned up third
    let file = File.open("log")   // Cleaned up second  
    let lock = mutex.lock()       // Cleaned up first
    // ... use resources ...
}  // Region exit triggers cleanup cascade
```

This eliminates the need for explicit `defer` statements (Go) or nested `with` blocks (Python) for most use cases. The region acts as an implicit scope guard, ensuring all resources are released when control flow exits—whether through normal return, early return, or panic.

However, resources can **escape their region** via ARC references. When a resource is stored in an ARC-managed container, cleanup defers until the reference count reaches zero. The compiler should track "region-local" vs "escaped" resources statically where possible, warning when resources accidentally escape.

## Cleanup ordering: reverse construction by default, explicit when needed

Research shows cleanup ordering failures cause subtle bugs. Database statements must close before connections; file buffers must flush before handles close. Indent should provide three ordering mechanisms:

**Automatic reverse order** covers most cases. Resources declared earlier in a region close later, matching intuition that later resources may depend on earlier ones. This aligns with C++'s destructor ordering and Java's try-with-resources.

**Explicit dependency declarations** handle complex cases where automatic ordering is insufficient:

```indent
let pool = ConnectionPool.new()
let conn = pool.get() depends_on pool  // Compiler ensures conn closes before pool
```

The `depends_on` annotation creates a cleanup dependency graph. The compiler topologically sorts cleanup operations, warning on cycles. This addresses the circular dependency problem that plagues finalizers—by making dependencies explicit, Indent can detect cycles at compile time.

**ManuallyManaged wrapper** for expert control disables automatic cleanup entirely:

```indent
let raw = ManuallyManaged.new(file_handle)
// ... transfer ownership to FFI ...
raw.defuse()  // Prevent cleanup
```

This mirrors Rust's `ManuallyDrop` for FFI scenarios where Indent shouldn't manage a resource's lifecycle.

## The `with` statement for explicit scoping

While regions handle most cleanup, an explicit `with` statement provides fine-grained control:

```indent
with file = File.create("output.txt") {
    file.write("data")
}  // file.close() called here, errors visible

// Async variant
async with conn = pool.async_get().await {
    conn.query("SELECT...").await
}  // conn.async_close().await called here
```

Unlike Python's context managers, Indent's `with` should return cleanup errors rather than suppressing them. This follows the principle that **resource cleanup failures are often as important as operation failures**—a file that fails to sync may have lost data.

Multiple resources in a single `with` statement execute cleanup in reverse order with proper exception aggregation (like Java's suppressed exceptions):

```indent
with conn = get_conn(), stmt = conn.prepare(sql) {
    stmt.execute()
}  // stmt closes first, then conn; errors aggregated
```

## Connection pooling as a language-supported pattern

Connection pools represent a special resource lifecycle pattern. Based on HikariCP's design principles, Indent should provide built-in pool semantics:

```indent
struct Pool<T: Resource> {
    fn get(&self) -> PoolGuard<T>
    async fn async_get(&self) -> PoolGuard<T>  // Backpressure-aware
    fn shutdown(&self, timeout: Duration)
}
```

**PoolGuard** is a wrapper that returns resources to the pool instead of closing them:

```indent
{
    let conn = pool.get()  // Borrows from pool
    conn.query(...)
}  // conn returns to pool, not closed
```

Pool configuration should follow HikariCP's research-backed formula: **pool_size = (cpu_cores * 2) + effective_spindle_count**. For SSDs, this simplifies to approximately **cpu_cores * 2 + 1**. Indent's standard library should default to this calculation.

Health checking should occur during idle housekeeping, not on acquisition—validation on every `get()` adds latency to the critical path. The pool maintains a `keepalive_interval` that pings idle connections and a `max_lifetime` that forces periodic reconnection:

| Configuration | Default | Purpose |
|--------------|---------|---------|
| max_pool_size | cores*2+1 | Maximum connections |
| min_idle | max_pool_size | Minimum maintained idle connections |
| connection_timeout | 30s | Max wait for available connection |
| idle_timeout | 10min | Close idle connections after |
| max_lifetime | 30min | Force reconnect older connections |
| keepalive_time | 2min | Ping idle connections |

## Async resource cleanup requires explicit support

The async drop problem—cleaning up resources that require I/O during destruction—is one of the hardest challenges in language design. Rust has struggled with this for years. Indent should address it directly:

**In async contexts**, resources implementing `AsyncResource` have their `async_close()` awaited automatically when leaving scope:

```indent
async fn process() {
    let conn = pool.async_get().await
    // ...
}  // Compiler inserts: conn.async_close().await
```

**In sync contexts**, async resources fall back to synchronous cleanup. The `AsyncResource` trait requires `Resource` as a supertrait, ensuring a sync fallback always exists. The sync cleanup may block or spawn a background task, depending on implementation.

**Cancellation and cleanup** follow structured concurrency principles from Kotlin. When an async task is cancelled, cleanup code runs in a `NonCancellable` context that cannot be interrupted:

```indent
async fn fetch_data() {
    let conn = pool.async_get().await
    // Even if task is cancelled here...
    process(conn).await
}  // ...conn.async_close() still completes
```

The runtime guarantees that async cleanup operations complete even when parent tasks are cancelled. This prevents resource leaks from task cancellation, which is a common source of connection pool exhaustion in production systems.

## Panic behavior: cleanup runs but errors are suppressed

When a panic occurs during normal execution, Indent should:

1. Mark the thread as "unwinding"
2. Run cleanup for all in-scope resources in reverse order
3. Suppress cleanup errors during unwinding (like Rust)
4. If cleanup itself panics, abort the process (double-panic = abort)

Resources can detect unwinding via `is_panicking()` and adjust behavior:

```indent
impl Resource for Reporter {
    fn close(&mut self) -> Result<(), CloseError> {
        if !is_panicking() {
            self.report_metrics()?  // Skip during panic
        }
        Ok(())
    }
}
```

This matches Rust's semantics where destructors run during stack unwinding but errors are suppressed to avoid masking the original panic.

## Static analysis for leak detection

Indent's compiler should perform accumulation analysis to detect resource leaks statically. For any type implementing `Resource`, the compiler tracks:

1. **Creation points** where resources are allocated
2. **Consumption points** where resources are closed (explicitly or via scope exit)
3. **Escape points** where resources leave tracked scope (returned, stored in globals)

A resource that reaches an escape point without being consumed or explicitly transferred triggers a warning:

```indent
fn leaky() -> File {
    let f1 = File.open("a")  // Warning: f1 never closed
    let f2 = File.open("b")
    f2  // OK: ownership transferred to caller
}
```

For stricter enforcement, Indent could support a **linear mode** where resources must be explicitly consumed:

```indent
fn strict() {
    let file: linear File = File.open("data")
    // Compile error: file must be consumed
}

fn correct() {
    let file: linear File = File.open("data")
    file.close().unwrap()  // OK: explicitly consumed
}
```

Linear types guarantee that `close()` is called, preventing leaks entirely at the cost of ergonomics. This should be opt-in per resource type.

## Runtime leak detection and diagnostics

Static analysis cannot catch all leaks, especially with dynamic resource patterns. Indent's runtime should provide:

**Leak detection threshold** for pools and handles:

```indent
let pool = Pool.new(config)
    .leak_detection_threshold(Duration::seconds(30))
    .build()
```

When a resource is held longer than the threshold, the runtime logs a warning with the acquisition stack trace—mirroring HikariCP's leak detection.

**Resource metrics** exposed via standard interface:

```indent
struct ResourceMetrics {
    active_count: u64,      // Currently held resources
    idle_count: u64,        // Available in pool
    wait_time_p99: Duration, // Acquisition latency
    leak_events: u64,       // Leak detection triggers
}
```

**File descriptor monitoring** for system limits:

```indent
let limits = ResourceLimits.current()
if limits.open_files.current > limits.open_files.soft * 0.8 {
    warn!("Approaching file descriptor limit")
}
```

## Resource categories and recommended patterns

Different resource types require different patterns:

| Resource Type | Cleanup Timing | Recommended Pattern |
|--------------|---------------|---------------------|
| File handles | Sync, fast | Region-scoped with auto-cleanup |
| Network sockets | Sync or async | `with` statement for explicit control |
| Database connections | Async preferred | Pool with health checking |
| Locks/mutexes | Must be sync | Guard pattern (auto-release) |
| GPU resources | Often async | Explicit `with` + dependency ordering |
| Temp files | Sync, may fail | `with` + error handling for cleanup |
| Process handles | Sync | Guard pattern + wait semantics |

For **locks**, Indent should use the guard pattern where the lock returns a guard that releases on drop:

```indent
{
    let guard = mutex.lock()  // Acquires lock
    *guard += 1               // Access via deref
}  // Lock released automatically
```

The guard's lifetime is tied to the lock, preventing use-after-unlock. This is Rust's MutexGuard pattern, which has proven highly effective.

For **GPU resources** and other complex subsystems, explicit dependency ordering ensures correct cleanup:

```indent
region render {
    let device = gpu.create_device()
    let allocator = device.create_allocator() depends_on device
    let buffers = allocator.alloc_buffers() depends_on allocator
}  // buffers → allocator → device cleanup order guaranteed
```

## System limits and graceful degradation

Indent programs should respect system resource limits. The standard library should:

1. **Query limits at startup** and warn if configured pools exceed them
2. **Fail fast** when limits are reached rather than queuing indefinitely
3. **Provide backpressure signals** for connection pools

```indent
// Pool returns error immediately when exhausted, rather than blocking
match pool.try_get() {
    Ok(conn) => process(conn),
    Err(PoolExhausted) => return_503_service_unavailable(),
}
```

For containers, Indent should read cgroup limits and adjust defaults accordingly—the pool_size formula should use container CPU, not host CPU.

## Summary of design recommendations

Indent's resource management should combine the best patterns from existing languages:

- **Resource/AsyncResource traits** with Result-returning cleanup (addresses Rust's silent Drop failures)
- **Region-scoped automatic cleanup** with reverse-order guarantees (simpler than Go's defer)
- **Explicit dependency ordering** via `depends_on` (solves circular dependency detection)
- **Built-in pool semantics** with research-backed defaults (HikariCP-informed)
- **Async-aware cleanup** with NonCancellable semantics (solves async drop problem)
- **Static leak detection** via accumulation analysis with optional linear types
- **Runtime leak detection** with stack traces and metrics (production debugging)
- **Guard pattern for locks** tying lock lifetime to guard lifetime

This design leverages Indent's regions+ARC foundation: regions provide the scope boundaries for automatic cleanup, while ARC handles shared ownership cases. The combination achieves deterministic resource management without sacrificing flexibility for complex ownership patterns.