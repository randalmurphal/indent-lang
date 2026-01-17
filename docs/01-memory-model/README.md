# Memory Model Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: CRITICAL - This decision affects everything else
**Key Question**: How do we get memory safety without GC pauses AND without manual management?

## The Problem Space

We want:
- ✅ No garbage collection pauses (Go has them, we don't want them)
- ✅ No manual memory management (Rust requires this in complex cases)
- ✅ Memory safety (no use-after-free, no data races)
- ✅ Predictable performance (no surprise allocations)
- ✅ Simple mental model (Python-like "it just works")

This is the holy grail that no language has fully achieved.

## Existing Approaches Analysis

### 1. Tracing Garbage Collection

**How it works**: Runtime periodically scans memory to find unreachable objects and frees them.

**Used by**: Java, Go, Python, JavaScript, C#

| Variant | Pause Behavior | Throughput | Memory Overhead |
|---------|---------------|------------|-----------------|
| Stop-the-world | Long pauses (10-100ms) | Good | Low |
| Concurrent (Go) | Short pauses (< 1ms typical) | Good | Medium |
| Generational | Very short young-gen pauses | Excellent | Higher |
| ZGC/Shenandoah | Sub-millisecond | Good | High |

**Go's GC specifically**:
- Concurrent, tri-color mark-and-sweep
- Target: < 1ms pause times (usually achieves this)
- Trade-off: ~25% throughput overhead, uses more memory
- GOGC tunable but defaults are reasonable

**Pros**:
- Simple mental model (allocate freely, forget about it)
- Handles cycles automatically
- Well-understood, mature implementations

**Cons**:
- Pause times (even Go's "short" pauses can spike under pressure)
- Memory overhead (need ~2x working set for concurrent GC)
- Throughput cost (Go: ~25%, Java: 10-20%)
- Unpredictable latency spikes
- Runtime required (can't compile to minimal binary)

**Verdict for us**: Go's GC is actually quite good for most backend work. The "no GC" requirement needs examination - do we mean "no pauses" or "no runtime overhead"?

---

### 2. Reference Counting (ARC)

**How it works**: Each object tracks how many references point to it. When count hits zero, immediately freed.

**Used by**: Swift, Objective-C, Python (as primary, with cycle collector), Rust (Rc/Arc)

**Variants**:

| Variant | Cycles? | Thread-safe? | Overhead |
|---------|---------|--------------|----------|
| Basic RC | ❌ Leaks | ❌ No | Low |
| RC + cycle collector | ✅ Yes | ❌ No | Medium |
| Atomic RC (ARC) | ❌ Leaks | ✅ Yes | Higher |
| ARC + weak refs | ✅ Manual | ✅ Yes | Higher |

**Swift's approach**:
- ARC by default (atomic reference counting)
- `weak` and `unowned` for breaking cycles
- Compiler inserts retain/release automatically
- No runtime GC, deterministic destruction

**Pros**:
- Deterministic destruction (resources freed immediately)
- No pause times
- Lower memory overhead than tracing GC
- Incremental cost (no "stop the world")

**Cons**:
- Cycles require manual intervention (`weak` references)
- Atomic operations have overhead (cache line bouncing)
- Frequent small allocations can be slow
- Reference count overhead on every copy

**Verdict for us**: Strong contender. Swift proves this can work well. Cycle problem is manageable with `weak` refs + static analysis to warn about likely cycles.

---

### 3. Ownership + Borrowing (Rust Model)

**How it works**: Compiler tracks ownership of every value. One owner at a time. Borrowing allows temporary access with compile-time lifetime checks.

**Used by**: Rust

**Key concepts**:
- Every value has exactly one owner
- Ownership can be moved or borrowed
- Borrows have lifetimes (how long the borrow is valid)
- Compiler rejects programs that would have memory errors

**Pros**:
- Zero runtime overhead
- No GC, no RC overhead
- Memory safety guaranteed at compile time
- Deterministic destruction
- Enables fearless concurrency

**Cons**:
- Significant learning curve
- "Fighting the borrow checker"
- Lifetime annotations required in complex cases
- Some valid programs rejected
- Refactoring can be painful

**Verdict for us**: The performance is what we want, the complexity is what we want to avoid. Can we get 80% of the benefit with 20% of the complexity?

---

### 4. Region-Based Memory Management

**How it works**: Memory is allocated in regions (arenas). Entire regions freed at once.

**Used by**: Cyclone (research), MLKit, some game engines

**Variants**:
- Lexical regions (tied to scopes)
- Dynamic regions (explicit open/close)
- Arena allocators (common in systems code)

**Pros**:
- Very fast allocation (bump pointer)
- No individual frees
- Predictable performance
- Good for request-scoped work (HTTP handlers!)

**Cons**:
- Memory not freed until region ends
- Objects can't outlive their region
- Complex region inference in research languages
- Not general-purpose

**Verdict for us**: Excellent for server handlers (request comes in, allocate in request region, free everything at end). Could be part of our solution but not the whole solution.

---

### 5. Linear/Affine Types

**How it works**: Type system ensures values are used exactly once (linear) or at most once (affine).

**Used by**: Rust (affine), Clean, some research languages

**Concept**:
```
x = Thing()
y = x        # x is MOVED, no longer valid
use(x)       # COMPILE ERROR: x was moved
```

**Pros**:
- Enables safe resource management
- No runtime overhead
- Foundation for ownership systems

**Cons**:
- Requires explicit copying when you want copies
- Can feel restrictive
- Need escape hatches for complex sharing

**Verdict for us**: This is likely part of our solution. The question is how much we can infer vs require explicit annotation.

---

### 6. Hybrid Approaches

#### Lobster (Game scripting language)
- Reference counting by default
- Compiler optimizes away most RC operations through static analysis
- Claims 95% of RC operations eliminated in typical code

#### Vale
- "Generational references" - each allocation has a generation number
- References include generation, checked at runtime
- Compile-time optimization eliminates most checks
- Still experimental

#### Mojo
- Value semantics by default (copies)
- Explicit `borrowed` and `owned` when needed
- MLIR-based, designed for ML workloads
- Claims Python syntax with C++ performance

#### Koka
- "Perceus" - precise reference counting with reuse analysis
- Functional language with effects
- Compiler often achieves in-place mutation through analysis

---

## Potential Approaches for Our Language

### Option A: "Smart ARC"

Reference counting with extensive compile-time optimization.

```
# Default: ARC managed
x = Thing()
y = x          # Refcount bumped

# Compiler optimization opportunities:
fn process(item):
    # Compiler sees item doesn't escape
    # Eliminates RC overhead entirely
    return item.value * 2

# Explicit for cycles
type Node:
    value: int
    weak next: Node?   # Weak reference, explicit
```

**How it would work**:
1. All heap objects are reference counted
2. Compiler performs escape analysis
3. Objects that don't escape → stack allocated, no RC
4. Short-lived objects → optimized RC (elide inc/dec pairs)
5. Weak references for cycles (explicit)
6. Potential: Cycle detector as optional runtime check for debugging

**Estimated complexity for users**: Low (just need to understand `weak`)

---

### Option B: "Inferred Ownership"

Rust-like ownership but with maximum inference.

```
# Compiler infers moves vs copies
x = Thing()
y = x          # Move (Thing is large) or Copy (if small/trivial)

# Borrowing is automatic for function args
fn process(item: Thing):   # Borrowed by default
    return item.value * 2

# Explicit only when needed
fn take_ownership(owned item: Thing):
    # item is moved in, we own it
    ...

# Lifetimes inferred except in complex cases
fn complex<'a>(x: &'a Thing, y: &'a Thing) -> &'a Thing:
    ...
```

**How it would work**:
1. Small types (≤ 2 words) → Copy semantics automatically
2. Large types → Move semantics automatically
3. Function arguments → Borrowed by default
4. Return values → Owned by default
5. Lifetimes inferred in most cases
6. Explicit annotations only for complex cases

**Estimated complexity for users**: Medium (need to understand moves, but rarely annotate)

---

### Option C: "Regions + Escape Analysis"

Leverage the request-oriented nature of backend work.

```
# Each handler gets an implicit region
@http.handler("/users")
fn get_users(req):
    # Everything allocated here lives in request region
    users = db.query("SELECT * FROM users")
    return json(users)
    # Region freed at end, all memory reclaimed instantly

# For long-lived data
global_cache = shared Cache()   # Explicit ARC for globals

# For performance-critical code
fn hot_path():
    region arena:
        # Explicit arena for bulk allocation
        data = arena.alloc_many(1000)
        process(data)
    # arena freed here
```

**How it would work**:
1. Request handlers automatically get a region/arena
2. Most allocations go in the region
3. Data that escapes the region → promoted to ARC
4. Compiler warns when data escapes (performance hint)
5. Global/shared data explicitly marked

**Estimated complexity for users**: Low for handlers, medium for libraries

---

### Option D: "Go-like with Escape Analysis"

Accept a GC, but make it really good.

```
# Just write code, don't think about memory
x = Thing()
y = x          # Copy or reference, compiler decides

# Escape analysis determines stack vs heap
fn process():
    local = Thing()   # Doesn't escape → stack allocated
    return local.value
    # local never hits the heap

# GC only for truly shared/escaping data
```

**How it would work**:
1. Aggressive escape analysis (like Go but better)
2. Stack allocate everything possible
3. GC for the rest, but it's a small working set
4. Concurrent GC with target < 500µs pauses
5. Optional: "no-gc" mode for real-time code sections

**Estimated complexity for users**: Very low (Go-like, just works)

**Trade-off**: We said "no GC" but Go's GC is actually fine for most backend work. Is the real requirement "no unpredictable pauses"?

---

## Comparison Matrix

| Approach | User Complexity | Performance | Pause Times | Memory Overhead | Cycles | Backend Fit |
|----------|-----------------|-------------|-------------|-----------------|--------|-------------|
| Smart ARC | Low | Good | None | Low-Med | Manual weak | ⭐⭐⭐⭐ |
| Inferred Ownership | Medium | Excellent | None | Very Low | N/A | ⭐⭐⭐ |
| Regions + Escape | Low-Med | Excellent | None | Low | Mixed | ⭐⭐⭐⭐⭐ |
| Go-like GC | Very Low | Good | < 1ms | Medium | Auto | ⭐⭐⭐⭐ |

---

## Questions to Resolve

### 1. What's the actual requirement for "no GC"?

- No pauses at all? → ARC or ownership
- No unpredictable pauses? → ARC or tuned GC acceptable
- No runtime? → Ownership required
- Just "better than Go"? → Many options

### 2. How important are cycles?

Backend code patterns:
- Trees (no cycles) - common
- Graphs with cycles - rare
- Doubly-linked structures - rare
- Event listeners (potential cycles) - moderate

If cycles are rare, ARC + manual weak refs is fine.

### 3. What about interior mutability?

Rust requires `RefCell`, `Mutex`, etc. for interior mutability.
Do we want:
- Everything immutable by default? (functional style)
- Mutable by default like Python? (familiar)
- Something in between?

### 4. How do we handle concurrency?

Memory model and concurrency model are deeply connected:
- ARC works with threads if atomic
- Ownership enables "fearless concurrency"
- Regions can be thread-local

---

## Recommendation

**Primary: Option C (Regions + Escape Analysis) with Option A (Smart ARC) as fallback**

Rationale:
1. Backend workloads are naturally request-scoped → regions are perfect
2. Long-lived shared data is the minority → ARC handles it
3. No GC pauses ever
4. Mental model: "allocate freely in handlers, be explicit for globals"
5. Escape analysis makes the common case fast

**Syntax might look like**:
```
# Most code - just works, regional allocation
fn handle_request(req):
    data = fetch_data(req.id)    # In request region
    result = process(data)        # In request region
    return json(result)           # Region freed after return

# Shared state - explicit ARC
cache: shared Dict[str, Data] = {}

fn lookup(key: str) -> Data?:
    return cache.get(key)

# Performance critical - explicit control
fn hot_path():
    region work:
        buffer = work.alloc(1024)
        # ... fast path code
    # freed instantly, no GC
```

---

## Next Steps

1. [ ] Prototype the region + ARC hybrid
2. [ ] Research Perceus (Koka's RC optimization) more deeply
3. [ ] Benchmark ARC overhead in realistic backend scenarios
4. [ ] Define the exact semantics of `shared` keyword
5. [ ] Explore how this interacts with concurrency (crucial)

---

## Research Links

- [Perceus: Garbage Free Reference Counting](https://www.microsoft.com/en-us/research/publication/perceus-garbage-free-reference-counting-with-reuse/)
- [Vale's Generational References](https://verdagon.dev/blog/generational-references)
- [Swift ARC Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- [Lobster Memory Management](https://aardappel.github.io/lobster/memory_management.html)
- [Go GC Guide](https://tip.golang.org/doc/gc-guide)
- [Region-Based Memory Management (Wikipedia)](https://en.wikipedia.org/wiki/Region-based_memory_management)

---

*Last updated: 2024-01-17*
*Decision: PENDING - Needs prototype validation*
