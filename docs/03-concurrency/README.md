# Concurrency Model Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Backend work is inherently concurrent
**Key Question**: How do we make concurrency simple, safe, and efficient?

## The Problem Space

We want:
- ✅ Easy to write concurrent code (Go-like simplicity)
- ✅ Hard to write buggy concurrent code (Rust-like safety)
- ✅ Efficient (thousands of concurrent operations)
- ✅ Good for servers (HTTP handlers, background jobs)
- ✅ Good for scripting (parallel file operations, API calls)
- ✅ Predictable behavior (no surprise race conditions)

The tension: **Simplicity vs Safety vs Performance** - pick two?

## Concurrency vs Parallelism

Important distinction:

- **Concurrency**: Dealing with multiple things at once (structure)
- **Parallelism**: Doing multiple things at once (execution)

A single-threaded async runtime is concurrent but not parallel.
Our language should handle both well.

## Existing Models

### 1. Threads + Shared Memory + Locks

**How it works**: OS threads share memory. Locks/mutexes prevent data races.

**Used by**: C, C++, Java, Python (with GIL complications)

```python
# Pseudocode
lock = Mutex()
counter = 0

def increment():
    lock.acquire()
    counter += 1
    lock.release()

# Spawn threads
for i in range(10):
    Thread(increment).start()
```

**Pros**:
- Simple mental model (initially)
- True parallelism
- Direct control

**Cons**:
- Deadlocks
- Race conditions (forget a lock = undefined behavior)
- Hard to compose
- Lock granularity is tricky
- Priority inversion
- Debugging nightmares

**Verdict**: Too error-prone. Not the default model.

---

### 2. Goroutines + Channels (CSP)

**How it works**: Lightweight "goroutines" communicate via channels. Don't share memory.

**Used by**: Go, Clojure (core.async)

```go
func worker(jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Start 3 workers
    for w := 1; w <= 3; w++ {
        go worker(jobs, results)
    }

    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // Collect results
    for a := 1; a <= 5; a++ {
        <-results
    }
}
```

**Pros**:
- Simple to write
- Lightweight (thousands of goroutines)
- Channels provide synchronization
- Good for I/O-bound work

**Cons**:
- Can still share memory unsafely (Go allows it)
- Goroutine leaks (no structured concurrency)
- Channel semantics can be subtle
- No compile-time race prevention

**Go's reality**: The `go` keyword is easy but leads to:
- Leaked goroutines
- Orphaned work
- Difficult cancellation
- Context threading everywhere

---

### 3. Async/Await

**How it works**: Functions can suspend execution, allowing other work. Single-threaded by default.

**Used by**: JavaScript, Python (asyncio), Rust, C#, Swift, Kotlin

```python
async def fetch_data(url):
    response = await http.get(url)
    return response.json()

async def main():
    # Concurrent, not parallel (by default)
    results = await gather(
        fetch_data("url1"),
        fetch_data("url2"),
    )
```

**Pros**:
- Explicit suspension points (await)
- No race conditions on single-threaded runtime
- Good for I/O-bound work
- Composable (async functions compose)

**Cons**:
- "Colored functions" problem (sync can't call async easily)
- Viral (async spreads through codebase)
- Can block the runtime if misused
- Complexity with multi-threaded runtimes

**The colored function problem**:
```python
def sync_function():
    # Can't call async_function() here easily
    # Need run_sync() or similar
    pass

async def async_function():
    # Can call sync_function() fine
    pass
```

This is why Go avoided async/await.

---

### 4. Actors

**How it works**: Isolated actors with mailboxes. No shared state. Communicate by message passing.

**Used by**: Erlang/Elixir, Akka (Scala), Swift (actors)

```elixir
defmodule Counter do
  use GenServer

  def start_link(initial) do
    GenServer.start_link(__MODULE__, initial)
  end

  def increment(pid) do
    GenServer.call(pid, :increment)
  end

  def handle_call(:increment, _from, count) do
    {:reply, count + 1, count + 1}
  end
end
```

**Pros**:
- True isolation (no shared state)
- Location transparent (can be networked)
- Fault tolerant (actors can crash independently)
- Good for distributed systems

**Cons**:
- Verbose for simple tasks
- Performance overhead (copying messages)
- Not intuitive for everyone
- Overkill for many backend tasks

---

### 5. Structured Concurrency

**How it works**: Concurrency is scoped. Child tasks cannot outlive their parent scope.

**Used by**: Kotlin (coroutines), Swift (TaskGroup), Trio (Python), Java (Project Loom)

```kotlin
suspend fun fetchAll(): List<Data> = coroutineScope {
    val a = async { fetchFromA() }
    val b = async { fetchFromB() }

    // Both complete or both are cancelled
    // Cannot leak
    listOf(a.await(), b.await())
}
// When this returns, all child coroutines are GUARANTEED done
```

**Key principle**: Tasks form a tree. Parent waits for children. Cancellation propagates down.

**Pros**:
- No leaked tasks (impossible by construction)
- Automatic cancellation propagation
- Easy reasoning about lifetimes
- Composable
- Error handling is clearer

**Cons**:
- Less flexible than raw goroutines (by design)
- Newer, less familiar
- Some patterns harder to express

**This solves Go's goroutine leaks**:
```go
// Go: Easy to leak
go func() {
    // What if nobody's listening on resultChan?
    // This goroutine leaks forever
    resultChan <- doWork()
}()

// Structured: Impossible to leak
scope.spawn(func() {
    // When scope exits, this is guaranteed complete or cancelled
    return doWork()
})
```

---

### 6. Rust's Model (Ownership + Threads/Async)

**How it works**: Ownership system prevents data races at compile time.

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];

    // data is MOVED into the thread
    let handle = thread::spawn(move || {
        println!("{:?}", data);
    });

    // println!("{:?}", data);  // COMPILE ERROR: data was moved

    handle.join().unwrap();
}
```

**Pros**:
- Data races impossible (compile-time guarantee)
- Zero-cost abstractions
- Works with both threads and async

**Cons**:
- Complexity of ownership in concurrent contexts
- `Send` and `Sync` traits add cognitive load
- Fighting the borrow checker in complex concurrent code

---

## Analysis for Our Language

### What Our Target Users Need

Backend developers, DevOps, scripting:

| Use Case | Concurrency Need |
|----------|-----------------|
| HTTP server handling | Many concurrent requests |
| API calls in scripts | Parallel fetches |
| File processing | Parallel I/O |
| Background jobs | Long-running tasks |
| Database operations | Connection pooling |
| WebSocket handling | Many persistent connections |

**Common pattern**: Lots of I/O-bound work, occasional CPU-bound work.

### What We Should Avoid

1. **Goroutine leaks** - Go's biggest concurrency problem
2. **Colored functions** - async/await's biggest problem
3. **Race conditions** - Threads + shared memory problem
4. **Boilerplate** - Actors for simple tasks
5. **Hidden complexity** - Magic that breaks under load

## Proposed Model: Structured Concurrency + Channels

**Core principles**:
1. All concurrency is structured (scoped)
2. Communication via channels (no shared mutable state by default)
3. No "colored functions" (any function can be concurrent)
4. Compile-time race prevention (via ownership/regions)

### Syntax Proposal

```
# Concurrent scope - all spawned work completes before scope exits
fn fetch_all() -> list[Data]:
    concurrent:
        a = spawn fetch_from_a()
        b = spawn fetch_from_b()
        # Both must complete before concurrent block exits
    return [a, b]

# With channels
fn pipeline():
    ch = channel[int](buffer=10)

    concurrent:
        # Producer
        spawn:
            for i in range(100):
                ch.send(i)
            ch.close()

        # Consumer
        spawn:
            for item in ch:
                process(item)

# Fire-and-forget (explicit, rare)
fn background_job():
    detach:
        # This runs independently
        # Must not reference parent scope
        long_running_cleanup()
```

### Key Features

#### 1. Structured by Default

```
concurrent:
    spawn task_a()
    spawn task_b()
# GUARANTEE: Both complete (or are cancelled) before this line
```

No `go` keyword that creates orphaned goroutines.

#### 2. No Colored Functions

```
# Any function can be used concurrently
fn fetch(url: str) -> Response:
    return http.get(url)  # This might suspend, caller doesn't care

# Caller decides concurrency
fn main():
    # Sequential
    a = fetch("url1")
    b = fetch("url2")

    # Concurrent
    concurrent:
        a = spawn fetch("url1")
        b = spawn fetch("url2")
```

The runtime handles suspension transparently. No `async` keyword needed.

#### 3. Channels for Communication

```
fn worker_pool(jobs: channel[Job], results: channel[Result]):
    concurrent:
        for _ in range(num_workers):
            spawn:
                for job in jobs:
                    results.send(process(job))
```

Channels are typed and provide synchronization.

#### 4. Compile-Time Safety

```
fn bad_example():
    data = mutable_list()

    concurrent:
        spawn:
            data.append(1)  # COMPILE ERROR: can't mutate shared data

        spawn:
            data.append(2)  # COMPILE ERROR: can't mutate shared data

# Safe version
fn good_example():
    ch = channel[int]()

    concurrent:
        spawn:
            ch.send(1)  # OK: channel is concurrent-safe

        spawn:
            ch.send(2)  # OK: channel is concurrent-safe
```

Ownership/regions prevent data races.

#### 5. Cancellation

```
fn fetch_with_timeout(url: str, timeout: duration) -> Result[Response, TimeoutError]:
    concurrent timeout(timeout):
        return spawn fetch(url)
    # If timeout, spawned task is cancelled automatically
```

Cancellation propagates through the task tree.

### Runtime Design

**Hybrid approach**:
- M:N threading (many tasks, few OS threads)
- Work-stealing scheduler
- I/O based on io_uring (Linux) / kqueue (macOS)
- Transparent suspension for I/O operations

```
┌─────────────────────────────────────────────────────┐
│                    User Code                        │
│                                                     │
│   concurrent:                                       │
│       spawn task_a()                                │
│       spawn task_b()                                │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                  Task Scheduler                     │
│                                                     │
│   Task Queue    Task Queue    Task Queue            │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐          │
│   │ task_a  │   │ task_b  │   │  ...    │          │
│   └─────────┘   └─────────┘   └─────────┘          │
└─────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  OS Thread  │  │  OS Thread  │  │  OS Thread  │
│   Worker 1  │  │   Worker 2  │  │   Worker 3  │
└─────────────┘  └─────────────┘  └─────────────┘
```

## Comparison with Other Models

| Feature | Go | Rust async | Our Proposal |
|---------|-----|------------|--------------|
| Spawn syntax | `go f()` | `spawn(f())` | `spawn f()` |
| Structured | ❌ No | Library (tokio) | ✅ Built-in |
| Colored functions | ❌ No | ✅ Yes | ❌ No |
| Race prevention | ❌ Runtime | ✅ Compile | ✅ Compile |
| Channels | ✅ Built-in | Library | ✅ Built-in |
| Cancellation | Manual (context) | Manual | ✅ Automatic |
| Learning curve | Low | High | Medium |

## Open Questions

### 1. Detached tasks?

Sometimes you want fire-and-forget. Options:
- Disallow entirely (too restrictive?)
- Require explicit `detach` with restrictions
- Allow but warn

**Recommendation**: Allow `detach` but require explicit opt-in and no references to parent scope.

### 2. Thread-local vs task-local storage?

- Go has neither (pass context explicitly)
- Thread-local breaks with M:N threading
- Task-local needs runtime support

**Recommendation**: Task-local storage with explicit `tasklocal` keyword.

### 3. Blocking operations?

What if someone calls a blocking C function?

**Options**:
- Dedicated thread pool for blocking work
- Explicit `blocking { }` scope
- Automatic detection (hard)

**Recommendation**: Explicit `blocking { }` that runs on dedicated thread.

### 4. CPU-bound parallelism?

Structured concurrency is great for I/O. What about CPU work?

```
# Parallel map over data
results = parallel_map(data, expensive_computation)

# Parallel for
parallel for item in data:
    compute(item)
```

**Recommendation**: Explicit `parallel` constructs for CPU-bound work.

### 5. Select/multiplexing?

Go's `select` is powerful:
```go
select {
case msg := <-ch1:
    handle(msg)
case msg := <-ch2:
    handle(msg)
case <-timeout:
    return
}
```

**Recommendation**: Similar `select` construct for multiple channels.

```
select:
    case msg from ch1:
        handle(msg)
    case msg from ch2:
        handle(msg)
    case timeout(5.seconds):
        return
```

## Integration with Memory Model

The concurrency model must work with our memory model (see doc 01):

| Memory Model | Concurrency Integration |
|--------------|------------------------|
| Regions | Request region = task lifetime, natural fit |
| ARC | Atomic RC for shared data across tasks |
| Ownership | Moved into spawned tasks, no sharing |

**Key**: Spawned tasks own their data or use channels. No shared mutable state.

```
fn example():
    # data moved into task
    data = compute_data()
    concurrent:
        spawn:
            process(data)  # data moved here
        # Can't use data here anymore

    # Channel: safe sharing
    ch = channel[Data]()
    concurrent:
        spawn:
            ch.send(compute())
        spawn:
            result = ch.receive()
```

## Recommended Approach

### Core Model
**Structured concurrency with channels**

- `concurrent` blocks for scoped concurrency
- `spawn` for tasks within scope
- Channels for communication
- Compile-time race prevention

### Runtime
**M:N with work-stealing**

- Few OS threads, many tasks
- Efficient I/O (io_uring/kqueue)
- Automatic load balancing

### Syntax
**Go-inspired but safer**

- No colored functions
- Explicit spawn points
- Automatic cancellation
- Built-in timeout/deadline support

## Next Steps

1. [ ] Design channel semantics in detail (buffering, closing, iteration)
2. [ ] Define cancellation propagation rules
3. [ ] Prototype scheduler in Rust
4. [ ] Explore interaction with memory regions
5. [ ] Design `select` construct
6. [ ] Define thread pool for blocking work

---

## Research Links

- [Notes on Structured Concurrency](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/)
- [Go Concurrency Patterns](https://go.dev/blog/pipelines)
- [Kotlin Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [Trio (Python structured concurrency)](https://trio.readthedocs.io/)
- [What Color is Your Function?](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)

---

*Last updated: 2024-01-17*
*Decision: Structured concurrency + channels (tentative)*
