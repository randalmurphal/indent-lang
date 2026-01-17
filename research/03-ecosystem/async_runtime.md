# Async Runtime Architecture for Indent: A Comprehensive Design Guide

**Indent's "colorless" async goal fundamentally shapes its runtime architecture.** Unlike Rust's explicit async/await coloring, Indent aims to let any function perform asynchronous I/O transparently—matching Go's ergonomic model. This requirement strongly favors **stackful coroutines** over stackless state machines, with significant implications for scheduler design, memory overhead, and I/O backend choices. This report provides component-by-component recommendations based on deep analysis of Tokio, Go, and .NET runtimes.

The recommended architecture combines Go's stackful coroutine model with Tokio's modern work-stealing scheduler, io_uring-first I/O (with epoll fallback), hierarchical timing wheels, and comprehensive observability instrumentation from the start.

---

## Why stackful coroutines enable colorless async

The choice between stackful and stackless coroutines is Indent's most consequential design decision. For colorless async, **stackful coroutines (like Go's goroutines) are strongly recommended** over stackless state machines.

Go achieves colorless async through stackful coroutines because each goroutine maintains its own stack, allowing suspension from any nested call frame—not just explicit await points. When a goroutine performs blocking I/O, the runtime transparently parks it and switches to another goroutine without requiring any special syntax or function coloring. The same `http.Get()` call works identically whether called from a goroutine or the main function.

Rust's stackless approach compiles async functions into state machines (enums) where each `.await` point becomes a state transition. This is incredibly memory-efficient—Tokio tasks require only **64 bytes** of base overhead versus Go's **~2.4KB minimum**—but creates the "colored function" problem. Async functions can only suspend at explicit `.await` points, and calling async code from sync contexts requires explicit bridging.

| Aspect | Stackful (Go) | Stackless (Rust) |
|--------|---------------|------------------|
| Memory per task | ~2.4KB minimum (2KB stack + 392B metadata) | 64 bytes + state machine size |
| Suspension depth | Unlimited—any call frame | Only at explicit `.await` points |
| Function coloring | None—any function can be async | Async/sync divide required |
| Context switch cost | ~200 nanoseconds | ~200 nanoseconds |
| Maximum per task | 1GB (growable stacks) | Fixed at compile time |
| Library compatibility | Seamless | Ecosystem split |

For Indent's colorless goal, stackful coroutines provide the natural path despite higher memory overhead. With 1 million tasks, expect approximately **2.65GB** of stack memory versus potentially **~100MB** with stackless—a significant but manageable tradeoff for modern servers. Go demonstrates this scales to 12+ million goroutines on sufficiently large machines.

---

## Scheduler architecture: work-stealing with message-passing optimization

For Indent's M:N threading model, a **work-stealing scheduler** provides the best balance of throughput, load balancing, and implementation complexity. Thread-per-core designs (Seastar, Glommio) offer lower latency for specific workloads but require applications to manually shard data.

### Recommended scheduler structure

The scheduler should use a three-level task queue inspired by Tokio's 2019 rewrite, which achieved **12x speedup** on chained spawns:

**LIFO Slot (per worker):** A single-task "hot slot" for message-passing optimization. When a task spawns another, the child goes in the LIFO slot and runs immediately—critical for request/response patterns. Disable after 3 consecutive uses to ensure fairness.

**Local Queue (per worker):** Fixed-size 256-task queue using the Chase-Lev work-stealing deque. This lock-free data structure enables O(1) push/pop for the owner thread and efficient stealing by other workers. When full, move half the tasks to the global queue.

**Global Queue:** Mutex-protected overflow queue checked every ~61 scheduled tasks (matching both Tokio and Go's heuristics). External task injection enters here.

```
Worker Thread Architecture:
  +-----------+
  | LIFO Slot |  ← Message passing optimization
  +-----------+
       |
  +-----------+
  | Local Q   |  ← 256 tasks, Chase-Lev deque
  +-----------+
       |
  +-----------+
  | Global Q  |  ← Overflow + injection
  +-----------+
```

### Worker thread count

Default to **one worker per logical CPU core**, following both Tokio and Go's GOMAXPROCS defaults. Unlike Go, which dynamically spawns threads when goroutines block on syscalls, Indent should use a separate blocking thread pool (discussed below) to keep the worker count fixed and predictable.

Go's container-aware GOMAXPROCS (Go 1.25+) provides a useful reference: use the minimum of available CPU cores, affinity mask, and cgroup CPU quota. This handles containerized deployments correctly.

### Chase-Lev deque implementation

The Chase-Lev work-stealing deque is foundational to efficient work-stealing. Key implementation details:

- **Single producer (owner):** O(1) push/pop with minimal atomic operations
- **Multiple consumers (thieves):** CAS-based steal from the opposite end
- **Fixed-size buffer:** Use 256 entries to avoid growth complexity; overflow to global queue
- **Batch stealing:** Steal half of the victim's queue to reduce steal frequency

The critical insight is that under load, each worker mostly processes its own queue—stealing is rare. The deque enables efficient local operation while providing load balancing when needed.

### NUMA awareness

Current major runtimes (Tokio, Go) are not NUMA-aware by default. For Indent, consider optional NUMA-aware scheduling for large multi-socket systems. Research shows **up to 5x speedup** with 99%+ local memory accesses on 192-core systems with 24 NUMA nodes. Implementation approaches include partitioning workers by NUMA node and preferring to steal from same-node workers.

---

## I/O backend: io_uring-first with epoll fallback

Indent should embrace **completion-based I/O (io_uring)** as the primary model, with epoll/kqueue fallback for older systems and cross-platform support. This represents a departure from Tokio's current readiness-based abstraction.

### Completion vs readiness models

**Readiness-based (epoll/kqueue):** The kernel notifies when a socket is ready, then the application performs the I/O operation. Simpler buffer management but requires multiple syscalls per operation.

**Completion-based (io_uring/IOCP):** The application submits an I/O operation with a buffer; the kernel notifies when complete. Better syscall amortization but requires eager buffer allocation.

For Indent's colorless async model, completion-based I/O maps more naturally—you submit an operation and get notified when it's done, matching the mental model of "just call the function."

### io_uring performance advantages

From quantitative analysis on Intel Xeon systems:

- **Syscall overhead with mitigations:** epoll ~2200ns per operation; io_uring amortizes to (2600ns / batch_size)
- **At 1000 connections with batching:** io_uring shows ~10% higher throughput
- **High IOPS storage workloads:** io_uring achieves ~40% improvement
- **Syscall reduction:** 5000 I/O operations in 23 io_uring_enter calls (~217 ops/syscall)

io_uring wins when:
1. CPU vulnerability mitigations are enabled (larger syscall overhead to amortize)
2. Batch sizes exceed ~1.3 operations
3. Connection counts are high (enabling effective batching)

### io_uring advanced features for Indent

**Registered files and buffers:** Pre-register file descriptors and buffers to eliminate per-I/O atomic reference counting. Required for SQPOLL mode.

**SQPOLL mode:** Kernel thread polls submission queue, enabling zero-syscall submission when the thread is active. Best for latency-sensitive workloads.

**Linked operations:** Chain operations (e.g., write-then-fsync) with `IOSQE_IO_LINK`. Useful for data integrity patterns.

**Kernel requirements:** Basic io_uring requires Linux 5.1+, but network operations need 5.5+, and multi-shot operations need 6.0+. For stability, target **kernel 5.10+** as minimum.

### Cross-platform strategy

For cross-platform support, implement:
1. **io_uring primary:** Full performance on modern Linux
2. **epoll fallback:** Older Linux systems
3. **kqueue:** macOS/BSD (more versatile than epoll, supports timers/signals)
4. **Future consideration:** Windows IOCP

Note that Docker Desktop 4.42.0+ blocks io_uring for security (seccomp bypass), requiring automatic fallback in containerized deployments.

---

## Blocking operations: dedicated thread pool with @blocking annotation

Indent's `@blocking` annotation should dispatch operations to a dedicated blocking thread pool, similar to Tokio's `spawn_blocking` but with potential for static analysis.

### Blocking thread pool design

**Recommended configuration:**
- **Maximum threads:** 512 (Tokio's default, works well for I/O-bound blocking)
- **Thread creation:** Lazy—spawn threads on-demand when needed
- **Keep-alive timeout:** 60 seconds before idle threads terminate
- **Queue:** Unbounded task queue (monitor for growth)

Avoid Go's approach of unlimited thread creation for blocking syscalls—it can lead to thread accumulation ("thread leak") that never recovers, as Go doesn't recycle threads.

### @blocking annotation semantics

Indent's annotation should enable both static and dynamic handling:

```indent
@blocking
fn read_file(path: String) -> String {
    std.fs.read_to_string(path)
}
```

**Static analysis benefits:**
- Compiler warns if `@blocking` function called from async context without proper dispatch
- IDE tooling can flag potential blocking calls
- Documentation generation shows blocking nature

**Runtime behavior:**
- When called from async context: automatically dispatch to blocking pool
- When called from blocking context: execute directly
- Optional timeout annotation: `@blocking(max_time = "100ms")`
- Distinguish I/O-bound (high parallelism OK) vs CPU-bound (limit parallelism)

### CPU-bound work considerations

For compute-intensive work, the 512-thread blocking pool is suboptimal—too many threads cause contention. Recommend a separate **compute pool** with thread count matching CPU cores, similar to rayon's approach. This could be triggered with a `@compute` annotation.

Go's approach is instructive: since Go 1.14, the runtime uses SIGURG-based async preemption to interrupt goroutines running >10ms without yielding. This handles compute-bound work that forgets to yield, but requires deep runtime integration.

---

## Timer implementation: hierarchical timing wheels

For efficient timer management at scale, implement **hierarchical hashed timing wheels** based on the Varghese & Lauck design used by Tokio and Kafka.

### Recommended structure

**6-level hierarchy with 64 slots per level:**
- Level 0: 64 × 1ms = 64ms range
- Level 1: 64 × 64ms = ~4 seconds range
- Level 2: 64 × ~4s = ~4 minutes range
- Level 3: 64 × ~4min = ~4 hours range
- Level 4: 64 × ~4hr = ~12 days range
- Level 5: 64 × ~12d = ~2 years range

**Time complexity:**
| Operation | Complexity |
|-----------|------------|
| Insert timer | O(1) |
| Cancel timer | O(1) |
| Fire timer | O(1) |
| Advance time | O(1) amortized |

### Timer vs heap comparison

Heap-based timers (Go's approach using 4-ary min-heap per P) provide O(log n) insert/cancel but O(1) minimum finding. For massive timer counts (tens of thousands), timing wheels win. For sparse timers with wide time ranges, heaps are more memory-efficient.

Given Indent's expected use cases (network timeouts, structured concurrency deadlines), timing wheels are recommended. Provide **1ms resolution** as default, with documentation noting that actual accuracy varies with system load.

---

## Fairness and preemption in structured concurrency

Indent's `concurrent { }` blocks for structured concurrency need fairness mechanisms to prevent task starvation.

### Cooperative budget system

Implement a budget system similar to Tokio's:
- Each task gets **128 operations** per scheduling quantum
- I/O operations, channel operations, and timer checks decrement the budget
- When budget exhausted, task yields to scheduler
- Budget resets when scheduler switches to the task

This prevents a single task from monopolizing a worker indefinitely.

### Preemption for colorless async

Since stackful coroutines enable suspension from any call frame, Indent can implement Go-style preemption:

**Synchronous preemption:** Insert checks at function prologues (can be done during stack growth checks with zero overhead)

**Async preemption (recommended):** Use SIGURG signal to interrupt long-running tasks. After 10ms without yielding, the runtime sends SIGURG, and the signal handler suspends the task.

This is critical for colorless async—without preemption, a tight compute loop could block all other tasks on that worker indefinitely.

---

## Channels and backpressure in structured concurrency

Indent's structured concurrency model should include built-in channels with backpressure.

### Recommended channel types

**Bounded MPSC (multi-producer, single-consumer):** Primary channel type for most use cases. Blocks senders when full, propagating backpressure upstream.

**Unbounded MPSC:** Available but discouraged—document as "memory leak waiting to happen."

**Broadcast:** For pub-sub patterns with drop-oldest semantics when receivers lag.

**Oneshot:** Single-use channels for task result communication (essential for structured concurrency).

### Buffer sizing guidance

- Default to **small bounded channels** (16-64 capacity)
- Size should approximately match expected burst size
- Too small: frequent blocking, reduced throughput
- Too large: wasted memory, delayed backpressure signaling

### Backpressure semantics

Match Go's blocking semantics for consistency with colorless async:
- Send on full bounded channel blocks (suspends task)
- Receive on empty channel blocks
- `try_send`/`try_recv` for non-blocking variants
- `select`-style multi-channel waiting

---

## Runtime configuration and lifecycle

### Sensible defaults

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| Worker threads | CPU cores | Standard for I/O-bound async |
| Blocking threads max | 512 | Handles diverse blocking workloads |
| Blocking keep-alive | 60s | Balance thread reuse vs cleanup |
| Local queue size | 256 tasks | Matches Go/Tokio, proven effective |
| Global queue check interval | 61 ticks | Matches Go/Tokio heuristics |
| Timer resolution | 1ms | Standard for network applications |
| Coroutine initial stack | 2KB | Go's proven minimum |
| Coroutine max stack | 1GB | Go's default for 64-bit |

### Configuration API

```indent
Runtime.configure()
    .worker_threads(8)
    .blocking_threads_max(256)
    .blocking_keep_alive(Duration.seconds(30))
    .timer_resolution(Duration.millis(1))
    .build()
```

### Environment variables

- `INDENT_WORKER_THREADS`: Override worker count
- `INDENT_BLOCKING_THREADS`: Override blocking pool limit
- Container-aware defaults (detect cgroup limits)

### Graceful shutdown

Structured concurrency simplifies shutdown—cancellation propagates through the task tree:

1. Signal shutdown (cancel root context)
2. All `concurrent { }` blocks receive cancellation
3. Wait for task completion with timeout
4. Force-terminate remaining tasks after timeout
5. Clean up resources (close I/O drivers, terminate threads)

---

## Debugging and observability from day one

Build observability into the runtime from the start—retrofitting is painful.

### Essential metrics

**Scheduler metrics:**
- `tasks_alive`: Currently running tasks
- `tasks_spawned_total`: Total tasks ever spawned
- `queue_depth_global`: Global queue size
- `queue_depth_local[worker]`: Per-worker local queue
- `steals_total[worker]`: Work stealing operations
- `busy_duration[worker]`: Time spent executing tasks
- `park_count[worker]`: Times worker went idle

**Blocking pool metrics:**
- `blocking_queue_depth`: Pending blocking tasks
- `blocking_threads_active`: Currently executing threads

**Timer metrics:**
- `timers_active`: Outstanding timers
- `timers_fired_total`: Timers that completed

### Task dumps for stackful coroutines

With stackful coroutines, Indent can provide Go-style goroutine dumps:
- On SIGQUIT: dump all task stacks to stderr
- HTTP endpoint: `/debug/tasks` returning task list with states
- Each task shows: ID, state (running/blocked/sleeping), stack trace, blocking location

This is a major advantage over stackless coroutines, which require complex instrumentation (like `async-backtrace`) to reconstruct the call tree.

### Integration with tracing/telemetry

Support structured logging with span propagation:
- Automatic trace ID injection into spawned tasks
- Context propagation through `concurrent { }` blocks
- OpenTelemetry-compatible trace export

---

## Runtime comparison: Go, Tokio, and .NET

| Feature | Go Runtime | Tokio | .NET Task Runtime | Indent (Recommended) |
|---------|-----------|-------|-------------------|---------------------|
| **Coroutine model** | Stackful (goroutines) | Stackless (futures) | Stackless (async/await) | Stackful |
| **Memory per task** | ~2.4KB | ~64 bytes + state | 24 bytes + state | ~2.4KB |
| **Function coloring** | None | Required | Required | None |
| **Scheduler** | M:N work-stealing | M:N work-stealing | M:N with hill-climbing | M:N work-stealing |
| **Worker threads** | GOMAXPROCS (dynamic M) | Fixed at startup | Dynamic injection | Fixed at startup |
| **Blocking handling** | Transparent (spawn M) | spawn_blocking pool | ThreadPool | @blocking annotation |
| **I/O model** | Readiness (netpoller) | Readiness (mio) | Completion (IOCP) | Completion (io_uring) |
| **Timer** | 4-ary heap per P | Timing wheel | Various | Timing wheel |
| **Preemption** | Async (SIGURG, 10ms) | Cooperative only | Cooperative | Async (SIGURG) |
| **Fairness** | Preemption + stealing | Budget system | Hill-climbing | Budget + preemption |
| **Max blocking threads** | 10,000 (soft) | 512 | Dynamic | 512 |
| **Task-local storage** | context.Context | task_local! macro | AsyncLocal | context-like |
| **Debugging** | SIGQUIT, pprof | tokio-console | VS debugger | SIGQUIT, HTTP endpoints |

---

## Recommended architecture summary

For Indent's colorless async with structured concurrency, the recommended architecture is:

**Task model:** Stackful coroutines with 2KB initial stacks, growable to 1GB. Accept the memory overhead for colorless async ergonomics.

**Scheduler:** Work-stealing with LIFO slot, 256-task local queues (Chase-Lev deque), and global overflow queue. One worker per core. Async preemption via SIGURG for long-running tasks.

**I/O backend:** io_uring primary on Linux 5.10+, epoll fallback for older systems, kqueue for BSD/macOS. Embrace completion-based model.

**Blocking dispatch:** Dedicated blocking thread pool (512 max) with lazy creation and 60s keep-alive. `@blocking` annotation for static analysis and automatic dispatch.

**Timers:** 6-level hierarchical timing wheel with 1ms resolution. O(1) operations.

**Channels:** Bounded MPSC as default with blocking backpressure. Match Go's blocking semantics.

**Observability:** Built-in metrics, SIGQUIT task dumps, HTTP debug endpoints, tracing integration.

**Structured concurrency:** `concurrent { }` blocks integrate with cancellation propagation, and `spawn` creates child tasks in the current scope.

This architecture prioritizes developer ergonomics (colorless async) and production reliability (observability) while achieving competitive performance through modern I/O (io_uring) and proven scheduler design (work-stealing with optimizations from both Go and Tokio).