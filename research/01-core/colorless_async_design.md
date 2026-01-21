# Zig's Colorless Async for Indent: A Decision Framework

Zig's new `std.Io` interface, merged in October 2025 for version 0.16.0, achieves colorless async through **runtime polymorphism** rather than compiler magic—a fundamentally different approach from what you may have envisioned. The key insight: same source code runs unchanged with blocking I/O, thread pools, green threads, or stackless coroutines. For Indent, this approach is **recommended with modifications**, offering Go's ergonomic simplicity with Rust-level performance, without requiring separate stacks for the common case.

## How Zig's Io interface actually works

The Io interface is essentially a **vtable passed explicitly** through the call chain, similar to Zig's existing Allocator pattern. This is not compiler-driven async transformation—it's a library-level abstraction:

```zig
fn saveData(io: std.Io, data: []const u8) !void {
    const file = try Io.Dir.cwd().createFile(io, "save.txt", .{});
    defer file.close(io);
    try file.writeAll(io, data);
}
```

The caller decides execution semantics. With `Io.Blocking`, this becomes direct syscalls. With `Io.Threaded`, operations dispatch to a thread pool. With `Io.Evented` (work-in-progress), it uses io_uring/kqueue with stackful coroutines. A planned stackless implementation will use compiler-generated state machines.

**The colorless property emerges because functions don't declare async-ness**—they simply accept an Io parameter. There's no `async` keyword, no special return type. The distinction between `io.async()` and `io.concurrent()` is subtle but crucial: async expresses operations *may* happen out of order (optimization hint), while concurrent *requires* parallel execution or fails with `error.ConcurrencyUnavailable`.

## Compiler mechanics for call graph analysis

Currently, Zig's implementation uses **no compiler-level async transformation**. The Io interface is pure library code with vtable dispatch. However, proposal #23446 outlines future stackless coroutine support requiring compiler involvement.

The planned approach for stackless coroutines introduces builtins:
- `@asyncFrameSize(func)` returns comptime-known frame buffer size
- `@asyncInit(frame_buf, func)` initializes the frame
- `@asyncResume(frame, arg)` resumes execution
- `@asyncSuspend(data)` yields control

**Call graph analysis** determines async-ness transitively: a function is async if it uses `@asyncSuspend` or calls another async function. The compiler performs graph traversal to apply transformations only where needed—the "colors are still there, but the compiler yields the paint."

**Indirect calls through function pointers** are handled via proposal #23367's restricted function types. The `@allowCallee` mechanism explicitly registers permitted callees for a function pointer type, enabling:
- Guaranteed devirtualization when single Io implementation exists
- Upper-bound stack size determination
- Proper async transformation even through vtables

This is the critical enabler: when your program uses only one Io implementation (the common case), the optimizer eliminates all vtable overhead, even in debug builds.

## Runtime overhead: surprisingly minimal

Direct benchmarks comparing Zig's new Io to Go goroutines don't exist yet—the feature merged only in late October 2025. However, theoretical analysis and general benchmarks provide guidance:

| Metric | Zig Io (vtable) | Go Goroutines | Rust async |
|--------|-----------------|---------------|------------|
| **Per-call overhead** | ~90ns non-devirtualized | ~0 (implicit runtime) | ~0 (monomorphized) |
| **Memory per task** | Implementation-dependent | ~2KB minimum | Size of state machine |
| **Pathological state size** | Bounded by implementation | Stack growth | Up to 400KB observed |

**Critical insight**: Zig's design guarantees **zero vtable overhead in the single-implementation case** through devirtualization. Since most applications use exactly one Io backend, production code pays no polymorphism tax. Additionally, buffering at the Reader/Writer interface level amortizes virtual calls across many bytes—you're not paying per-byte overhead.

General Zig vs Go benchmarks show Zig **1.5-10x faster** in CPU-bound tasks (mandelbrot 10x, LRU 3.6x, knucleotide 2x). Memory usage is consistently lower. These advantages would compound in I/O-heavy scenarios where Zig's zero-cost abstractions shine.

## Cancellation and structured concurrency

Zig achieves structured concurrency through **idiomatic patterns** rather than special syntax:

```zig
fn process(io: Io) !void {
    var a = io.async(taskA, .{io});
    defer a.cancel(io) catch {};  // Cancel on early exit
    
    var b = io.async(taskB, .{io});
    defer b.cancel(io) catch {};
    
    try a.await(io);
    try b.await(io);
}
```

The `Future.cancel(io)` method has identical semantics to `await` except it also requests cancellation. It's **idempotent**—canceling a completed future simply returns its result. Incomplete operations return `error.Canceled`. The defer pattern ensures cleanup regardless of error propagation path.

Error handling flows naturally through Zig's standard error unions. The anti-pattern of sequential `try` without defer risks leaking futures; the correct pattern separates await from error handling to ensure all futures are awaited before propagating errors.

## Prior art informs Indent's design

Several languages have tackled colorless async with varying tradeoffs:

**Go goroutines** remain the gold standard for simplicity: `go func()` spawns concurrent work with zero ceremony. Functions don't declare concurrency capability—the runtime handles everything. However, Go lacks structured concurrency by default (goroutines can leak), and error handling in concurrent contexts requires manual coordination via channels.

**OCaml 5's effect handlers** use algebraic effects with fiber-based runtime. The `perform` operation suspends computation; handlers receive delimited continuations and decide whether to resume. This enables implementing async/await as a library but adds cognitive overhead and currently lacks typed effects.

**Java's Project Loom** virtual threads achieve colorless async through JVM-managed lightweight threads. Write blocking code; the runtime unmounts threads during I/O. This is the closest analog to Go's model and demonstrates colorless async is viable even on managed runtimes.

**Kotlin's context receivers** provide implicit parameter passing similar to Scala's `given/using`. Functions declare required contexts; the compiler resolves them from scope. This pattern directly informs how Indent could hide Io parameters while maintaining explicitness at boundaries.

The key lesson: **colorless is achievable through either runtime fiber management (Go, Loom) or effect handler mechanisms (OCaml)**. Zig's Io interface is unique in achieving colorless semantics through explicit parameter passing with guaranteed devirtualization—no runtime tricks, no GC required.

## Compiler implementation strategy for Indent

For a Rust-based compiler targeting Cranelift/LLVM:

**Call graph analysis** uses petgraph (9M+ downloads/month) for graph representation and salsa for incremental computation. Salsa's tracked functions automatically cache and invalidate dependent queries—critical for fast recompilation when one async function changes.

**State machine generation** follows Rust's pattern:
1. Build CFG from AST with explicit suspension points
2. Perform liveness analysis: which variables are live across suspension points?
3. Build storage conflict matrix: which variables can share memory?
4. Generate enum with state variants containing only necessary fields
5. Transform function body into state-dispatch pattern

```rust
enum FutureState {
    Initial { args... },
    Suspend0 { live_vars_across_first_await... },
    Suspend1 { live_vars_across_second_await... },
    Completed,
}
```

**Cranelift lacks native coroutine intrinsics** unlike LLVM. Perform state machine transformation at your IR level before lowering to CLIF. Generate explicit switch dispatch:

```
brif state == 0, block_state0, block_state1
block_state0:
    // Resume from state 0
block_state1:
    // Resume from state 1
```

**Closure capture in async contexts** requires careful analysis. Async closures that borrow from captures need Pin semantics to prevent moving the future after references are established. Consider Rust's approach: capture by move by default, with explicit annotation for by-reference captures.

## Syntax specification for Indent

The recommended design uses **implicit Io within explicit context blocks**:

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

**Key syntax elements**:

1. **`with async_io():`** establishes Io context—alternatives include `blocking_io()`, `threaded_io()`. This is the only place where execution model is specified.

2. **`concurrent:`** block spawns all contained statements as concurrent tasks. Block exit awaits all tasks. This is structured concurrency with indentation.

3. **No `async def`**—functions don't declare async capability. They work in any context.

4. **Explicit spawn when needed**: `go fetch_user(123)` for fire-and-forget within structured block.

5. **Futures for explicit control**:
```python
with async_io():
    concurrent as tasks:
        future1 = tasks.spawn(fetch_user, 123)
        future2 = tasks.spawn(fetch_orders, 123)
        
        user = await future1
        orders = await future2
```

**Error handling** uses exception groups compatible with Python 3.11+:

```python
try:
    concurrent:
        risky_task1()
        risky_task2()
except* ValueError as errors:
    for e in errors:
        log(e)
```

This design achieves **Python's readability** (indentation-based, familiar keywords), **Go's simplicity** (no function coloring, minimal ceremony), and **Rust's performance** (zero-cost abstraction via compile-time Io resolution).

## Hybrid approach: stackless by default, stackful when needed

The recommended architecture supports both:

1. **Io-style stackless** (default, 90% of code): Compiler-generated state machines, WASM-compatible, zero overhead
2. **Stackful coroutines** (opt-in): For FFI callbacks, recursive algorithms, complex call stacks

```python
with async_io():  # Uses stackless by default
    concurrent:
        fetch_data(url)

with stackful_io():  # Uses green threads
    call_ffi_that_calls_back_into_indent()
```

Interoperation is seamless because both use the same Io interface. The runtime detects capabilities and adapts. User code doesn't change—just the context block.

## Decision recommendation

**Adopt a Zig-inspired approach with Pythonic syntax** rather than pure stackful coroutines. The rationale:

| Factor | Zig-style Io | Go-style Stackful |
|--------|--------------|-------------------|
| Memory per task | Bytes (state machine) | KB (stack) |
| WASM support | Native | Requires stack switching emulation |
| Compilation complexity | Higher (state machine gen) | Lower |
| Debugging | State visible in enum | Natural stack traces |
| FFI compatibility | Requires explicit handling | Seamless |
| Maximum concurrency | Millions of tasks | Limited by stack memory |

The Zig approach achieves colorless semantics without the memory overhead of stackful coroutines. For Indent's target (Rust's performance), this matters: **1 million concurrent tasks at 2KB each = 2GB RAM** for Go-style stacks, versus potentially **~100MB** for compact state machines.

**Implementation roadmap**:
1. Phase 1: Implement blocking Io (baseline, zero async overhead)
2. Phase 2: Add thread-pool Io (multiplexed blocking)
3. Phase 3: Add compiler-driven stackless coroutines
4. Phase 4: Optional stackful coroutines for FFI/recursive cases

The syntax proposed above hides complexity behind `with async_io():` while preserving escape hatches. Users write sequential-looking code; the compiler and runtime handle the rest. This achieves the stated goals: Python's readability, Go's simplicity, Rust's performance—with colorless async as the foundation.