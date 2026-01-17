# Memory Management Models for a Simpler Systems Language

A new programming language targeting backend developers with "Python's readability, Go's simplicity, Rust's performance" faces a fundamental tension: memory safety historically requires either runtime overhead (GC) or cognitive overhead (ownership annotations). This research reveals that **recent advances make a middle path viable**—combining automatic reference counting, scope-based regions, and targeted escape analysis to achieve near-Rust performance with Go-like simplicity.

The core insight from surveying Koka, Vale, Mojo, Swift, Rust, and Go is that **no single model wins across all dimensions**, but strategic composition of techniques can eliminate 80-95% of memory management overhead while maintaining a simple mental model. The most promising path for a "stupid simple" backend language: **affine ownership with second-class references, automatic regions for request-scoped allocation, and optimized reference counting as the escape hatch**.

## The landscape of memory management options

Modern memory management exists on a spectrum trading cognitive overhead against runtime cost. Traditional GC (Java, Go) achieves zero cognitive overhead but consumes 5-15% throughput and creates latency unpredictability. Rust's borrow checker eliminates runtime cost but creates significant learning curves—developers report "fighting the borrow checker" as their primary friction point. Swift's ARC sits between, with **42% of client app execution time** spent on reference counting in some benchmarks.

Recent research languages have explored novel points in this design space. Koka's Perceus system achieves "garbage-free reference counting" through compile-time analysis that eliminates most RC operations and enables in-place mutation of functional data structures. Vale's generational references trade a **10.84% runtime overhead** (compared to 25% for traditional RC) for full mutable aliasing support. Lobster's ownership analysis eliminates **95% of RC operations** at compile time through flow-sensitive type specialization.

| Approach | Runtime Cost | Cognitive Load | Cycles | Backend Fit |
|----------|-------------|----------------|--------|-------------|
| Tracing GC (Go) | 5-15% | Very Low | ✓ | Good |
| Reference Counting (Swift) | 10-42% | Low-Medium | Manual | Good |
| Borrow Checking (Rust) | 0% | High | N/A | Excellent |
| Perceus (Koka) | ~GC competitive | Medium | No | Unknown |
| Generational Refs (Vale) | ~11% | Low | N/A | Promising |

For backend applications specifically, Go's designers discovered that **escape analysis can substitute for generational collection**—short-lived objects naturally stack-allocate when the compiler can prove they don't escape. This insight suggests a language can achieve good performance without complex generational GC machinery if designed with escape analysis in mind.

## Novel reference counting approaches reveal surprising possibilities

Koka's Perceus system demonstrates that reference counting need not imply either cycle leaks or high overhead. The key innovation is **Functional But In-Place (FBIP)**: when the compiler detects that a value being destructured has a reference count of one, it can reuse that memory directly for a newly constructed value of the same size. This transforms purely functional code into efficient imperative code automatically.

The Perceus algorithm generates precise `dup` (increment) and `drop` (decrement) operations using a linear resource calculus formalization. Variables are tracked as multisets; when destruction leaves a unique reference, the memory becomes available for in-place reuse. The **fip and fbip keywords** (added in Koka v2.4.2) allow programmers to verify statically that functions perform no allocation—a powerful guarantee for latency-sensitive code.

However, Perceus has limitations that emerged in practice. Benchmarks show optimizations are "less effective" with "lots of sharing"—programs that create many references to the same data see less benefit from reuse analysis. The system also cannot collect cycles, though Koka addresses this by targeting functional programs where cycles are structurally impossible (all data types are inductive or coinductive).

Vale takes a radically different approach with **generational references**. Each allocation gets a random 64-bit generation number, and references store the expected generation. Before any dereference, a single check validates the generation matches—if an object was freed and the memory reused, the generations won't match. The probability of false negatives is astronomically low: approximately **1 in 2^64 per failure**, meaning unsafety would require running for thousands of years.

Vale's design enables patterns that Rust's borrow checker struggles with: observers, callbacks, and dependency injection all work naturally because mutable aliasing is permitted. For performance-critical code, Vale's planned **region system** can skip generation checks entirely for groups of objects with statically-known lifetimes. The measured overhead of 10.84% compares favorably to traditional RC at 25.29%, though development has slowed significantly (last major commits in May 2024).

Lobster achieves remarkable RC elimination through **ownership specialization integrated with type inference**. The compiler picks a single owner for each allocation, makes all other uses borrows, and only inserts RC operations when a borrow must become an owner. Functions are specialized not just by type but by ownership requirements, enabling different call sites to generate different code paths. Nim's `--gc:arc` mode reportedly implements "effectively Lobster's algorithm."

## Regions work best as explicit programmer tools, not inference targets

MLKit's region inference—the seminal academic work by Tofte and Talpin—attempted fully automatic region management for Standard ML. The system inferred which objects could share regions based on type-and-effect analysis, then allocated and deallocated regions in LIFO order. The theoretical foundations were sound, but practical adoption failed.

The problems were fundamental. Small source changes caused "drastic, unintuitive effects on object lifetimes." Early prototypes showed performance varying between "10 times faster and four times slower" depending on how "region-friendly" the program was. The **LIFO constraint** proved too restrictive—any data structure outliving its creating context required workarounds. Region space leaks occurred when popular patterns like tail recursion were used "in the straightforward way."

Cyclone, developed at Cornell and AT&T Research, adapted regions for a type-safe C dialect with more success. Rather than fully automatic inference, Cyclone required programmers to declare regions explicitly while providing **default annotations and local inference** to reduce boilerplate. In 110,000 lines of Cyclone code, only one non-default effect annotation was needed. Porting C code required altering ~8% of lines, with only 6% being region annotations.

Cyclone's key innovation was **region subtyping**: if region ρ1 outlives ρ2, then pointers to ρ1 can be coerced to pointers to ρ2. Combined with the `regions_of(τ)` type operator, this eliminated the complex effect variables that plagued earlier systems. Cyclone directly influenced Rust—lifetime annotations in Rust (`'a`) are essentially Cyclone's region variables with different syntax.

Modern arena allocators prove that **explicit, manually-managed regions are highly practical**. Game engines universally use per-frame arenas: allocate linearly during the frame, reset the entire arena with a single integer assignment (~358,000x faster than individual frees). Web servers use per-request pools (Apache's "pools," PostgreSQL's "memory contexts"). Zig makes allocators first-class, requiring explicit allocator parameters for any function that allocates.

Go's experimental arena package (1.20+) achieved **8% CPU reduction** in already-optimized services and cut GC overhead in half. However, the proposal is on hold due to "serious API concerns"—demonstrating the difficulty of safely exposing manual memory management in a GC'd language. The lesson: regions work well as opt-in tools for specific patterns, not as general-purpose automatic memory management.

## Simplified ownership gives 80% of Rust's benefits

Rust's borrow checker provides maximum memory safety but imposes significant cognitive overhead. The core rules are deceptively simple—single ownership, borrowing with either one mutable or many immutable references—but the implications create friction. Self-referential structs require complex `Pin` APIs. Conditional borrowing patterns that are obviously safe get rejected. Graph structures require escape hatches (`Rc<RefCell<T>>` or arena indices).

Research into simplified ownership systems suggests a **sweet spot exists**. Austral uses strict linear types (values must be consumed exactly once) with lexical-only borrowing—simpler than Rust's non-lexical lifetimes but more restrictive. Val (now Hylo) explores "mutable value semantics" with **second-class references** that cannot be stored in data structures, eliminating lifetime annotations entirely.

The most promising direction for a simple language is **affine ownership with second-class borrows**:

- Single ownership with move semantics (like Rust's `Box` or C++'s `unique_ptr`)
- References that exist only within lexical scopes (cannot be stored in structs)
- No lifetime parameters in type signatures
- Escape hatches to reference counting or arenas for complex patterns

This eliminates the vast majority of Rust's annotation burden while preserving memory safety. The key insight from Mojo's design: **borrow by default on function arguments** (not move) dramatically improves ergonomics. Rust requires explicit `&` for borrowing; Mojo borrows automatically and requires explicit `^` for ownership transfer.

Mojo further simplifies with **ASAP destruction**: values are destroyed immediately after their last use, not at scope end. This eliminates the need for drop flags (runtime tracking of whether a value has been dropped) and makes lifetimes more intuitive—the compiler determines optimal destruction points rather than requiring programmer reasoning about scope boundaries.

Ownership inference research indicates that **~87% of lifetime annotations** can be eliminated through elision rules (Rust already does this for common patterns). The remaining cases typically occur at function boundaries with multiple reference parameters. A language with second-class references avoids even these—since references can't be stored, function signatures need only indicate borrowed vs. owned arguments.

## Escape analysis complements ownership, enabling automatic stack allocation

Go's escape analysis demonstrates that compile-time inference can significantly reduce heap allocation without programmer annotation. The Go compiler constructs a directed graph of assignments and tracks which values flow to the heap, function returns, or goroutine captures. Values proven to stay local are stack-allocated automatically.

However, Go's analysis is **flow-insensitive and field-insensitive**: it can't distinguish between branches and treats all fields of a struct equivalently. Common patterns defeat it—storing into interfaces, capturing in goroutines, or simply returning an address forces heap allocation. Interface boxing is particularly problematic: `fmt.Println(v)` causes `v` to escape regardless of what `Println` actually does.

The JVM's HotSpot C2 compiler performs more sophisticated escape analysis with **scalar replacement**: non-escaping objects are decomposed into individual fields that may reside in registers or stack slots. The object itself is never allocated. However, flow-insensitivity limits effectiveness—only ~13% of candidate methods actually have objects scalar-replaced.

GraalVM's **Partial Escape Analysis (PEA)** represents the state of the art. It tracks escape state along each control flow path independently, allocating objects only on paths where they actually escape. This enables transformations like moving allocation into the branch where escape occurs, achieving **up to 58.5% reduction** in allocated memory and 33% performance improvement in some benchmarks.

For a new language, escape analysis can complement ownership in several ways:

1. **Automatic stack promotion**: Even with ownership, knowing when objects don't escape enables stack allocation without programmer annotation
2. **Region inference assist**: Escape analysis can suggest which objects could share regions
3. **Optimization hints**: Non-escaping objects can skip generation checks (Vale's approach) or RC operations (Lobster's approach)

The combination of ownership (programmer-specified) and escape analysis (compiler-inferred) could provide both control and convenience: explicit ownership for API boundaries, automatic stack allocation for local values.

## Modern GCs prove sub-millisecond pauses are achievable

ZGC and Shenandoah demonstrate that garbage collection need not mean unpredictable latency. Both achieve **sub-millisecond pauses** even on multi-terabyte heaps through nearly-fully-concurrent algorithms. The techniques are instructive even for non-GC languages.

ZGC uses **colored pointers**: 64-bit pointers with 4 metadata bits indicating whether the pointer is valid or potentially stale. Load barriers check pointer color on every heap object load—the fast path is optimized for valid pointers (common case). When a "bad" pointer is detected, the barrier self-heals by updating to the object's new location or triggering relocation.

Shenandoah uses **Brooks forwarding pointers**: each object contains an extra pointer field pointing to its current location. When objects are relocated, only this forwarding pointer is atomically updated. Application threads automatically follow the indirection, enabling concurrent compaction without stopping the world.

The key innovation enabling sub-millisecond pauses is **incremental stack scanning**. JDK 17's "stack watermarks" separate the stack into "safe to scan" and "needs coordination" regions. GC threads scan from the bottom up to the watermark while application threads handle returns via barriers, lowering the watermark incrementally. This eliminates thread stack scanning from pause times entirely.

For a non-GC language, these techniques suggest possibilities for **hybrid designs**:

- **GC-free zones** (D language's `@nogc`): Compile-time enforcement that certain code paths never trigger collection
- **Region + GC fallback**: Static analysis assigns most objects to regions; escaping objects fall back to traced collection
- **Concurrent cycle collection**: Use RC for most objects, trigger concurrent cycle detection only when needed

The fundamental GC tradeoff—pause time vs. throughput—remains. ZGC/Shenandoah achieve latency at cost of **5-15% throughput reduction** and additional memory overhead. For backend applications where tail latency matters more than average throughput, this trade is often worthwhile.

## Compiler infrastructure requirements are well-understood

Building a memory-safe language requires significant compiler infrastructure, but the components are well-documented. The essential pieces:

**Ownership and borrowing** require type system extensions: tracking affinity/linearity annotations, implementing context splitting during type checking (linear variables must be partitioned between subexpressions), and move/copy analysis. A borrow checker needs CFG construction, liveness analysis, and reaching definitions computation. Rust's Non-Lexical Lifetimes use dataflow analysis to track active borrows.

**Region/lifetime inference** involves constraint generation (expressing "region A must outlive region B") and constraint solving (finding minimal regions satisfying all constraints). Full inference (MLKit-style) proved impractical; hybrid approaches with inference inside functions and explicit annotations at boundaries are more tractable.

**Escape analysis** builds on points-to analysis (determining which pointers can reference which allocations) and requires interprocedural analysis for precision. The tradeoff is precision vs. compilation time—Steensgaard's unification-based algorithm runs in near-linear time but is less precise than Andersen's subset-based approach.

**LLVM integration** is straightforward: emit `noalias` attributes for mutable references, use Type-Based Alias Analysis (TBAA) metadata, and mark functions with `readnone`/`readonly` where applicable. These annotations enable LLVM's optimizer to perform better code generation. Rust marks all `&mut` arguments as `noalias`, enabling significant optimization.

**SSA form** (Static Single Assignment) simplifies all these analyses by ensuring each variable has exactly one definition. LLVM IR is SSA-based; Cranelift uses block parameters instead of φ-nodes but achieves similar benefits. Memory SSA extends this to track memory state changes, enabling alias disambiguation.

The RustBelt project demonstrated that **formal verification of memory safety** is achievable: using Iris separation logic in Coq, they verified Rust's standard library abstractions including `Arc`, `Rc`, `Cell`, `RefCell`, and `Mutex`. A new language need not achieve this level of formalization initially, but the existence of such proofs provides confidence that the underlying theory is sound.

## Language designer wisdom points toward specific choices

Go's team explicitly rejected generational GC because **write barrier overhead was too high** and escape analysis already handles short-lived objects by stack-allocating them. The "value-oriented" language design (like C, unlike Java) is "the most important thing that differentiates Go from other GCed languages"—values don't require heap allocation by default.

Go's attempted "Request-Oriented Collector" (request-scoped GC) was abandoned due to 30-50% slowdowns from always-on write barriers. This validates the approach of making request-scoped allocation explicit (arenas) rather than automatic.

Swift's team warns against relying on "observed object lifetimes"—when deallocation actually happens versus when it's guaranteed to happen. Xcode 13's "Optimize Object Lifetimes" exposed hidden bugs in code that depended on specific deallocation timing. The lesson: languages should provide clear, documented lifetime guarantees that don't depend on optimization behavior.

Mojo's design philosophy—**start simple, allow opt-in complexity**—appears most aligned with "stupid simplicity." Default to borrowed arguments, require explicit ownership transfer, provide zero-cost abstractions for those who need them. The `def` vs `fn` distinction (Python-like flexibility vs. systems-programming strictness) allows gradual adoption of advanced features.

For backend specifically, server-side patterns create natural region boundaries. Web requests have clear start and end points; connections have lifetimes; transactions have boundaries. LinkedIn's engineers discovered that **allocator choice matters even in Java**—switching from glibc malloc to jemalloc eliminated mysterious memory growth from fragmentation.

## A recommended approach for "stupid simplicity with maximum benefits"

Based on this research, a language targeting backend developers with minimal cognitive overhead should combine:

**Default memory model: Optimized reference counting**
- Compile-time RC elimination using Lobster/Perceus-style ownership analysis
- No cycle collection needed if language design prevents cycles (functional core, explicit graph types)
- Falls back to full RC only where ownership can't be statically determined
- Expected overhead: **~5% runtime cost** for remaining RC operations

**Explicit regions for request-scoped patterns**
- Simple `region { }` blocks for arena allocation
- Objects allocated in region freed together at block end
- Perfect fit for HTTP request handlers, transaction processing, batch operations
- No LIFO constraint across regions; LIFO within regions only

**Second-class references for simplicity**
- References exist only within function scopes; cannot be stored in data structures
- Eliminates lifetime annotations entirely
- Borrows are default for function arguments; ownership transfer requires explicit syntax
- For stored references, use RC (`Ref<T>`) or weak references (`Weak<T>`)

**Aggressive escape analysis**
- Stack-allocate values proven not to escape
- Automatically use region allocation when escape analysis identifies region-safe patterns
- Compiler hints showing allocation decisions (like Go's `-gcflags="-m"`)

**Opt-in GC for specific use cases**
- Allow marking types as GC-managed for cyclic data structures (graphs, trees with parent pointers)
- Concurrent, low-pause collection (ZGC-style) for GC'd objects only
- Clear documentation of which operations trigger GC

This combination achieves:
- **Simplicity**: No lifetime annotations, no fighting the borrow checker, obvious memory behavior
- **Performance**: Near-Rust speed for common patterns (regions, stack allocation, optimized RC)
- **Server fit**: Natural alignment with request-scoped, connection-scoped, transaction-scoped patterns
- **Escape hatches**: Full GC when needed, explicit RC for shared data, unsafe blocks for FFI

The mental model reduces to: **"Values are owned. Functions borrow by default. Use regions for batches. Use Ref for sharing."** This is substantially simpler than Rust's model while achieving most of its benefits for backend workloads.

## Conclusion: Composing proven techniques achieves the goal

The research reveals that "Python's readability, Go's simplicity, Rust's performance" is achievable—not through a single novel technique, but through **strategic composition** of proven approaches. Perceus shows RC can be nearly free with sufficient analysis. Cyclone and Rust proved regions and ownership compose well. Go demonstrated escape analysis substitutes for generational GC. Modern low-pause collectors prove concurrent memory management is tractable.

The key insight for a "stupid simple" language is that **backend workloads have favorable memory patterns**. Request-scoped allocation, short-lived objects, clear ownership boundaries—these align naturally with regions and optimized RC. The complexity of Rust's borrow checker largely addresses problems (cyclic graphs, self-referential structs, interior mutability) that backend code rarely encounters.

A language following the recommended approach would offer genuinely simple memory management—allocate freely, trust the compiler to optimize, use explicit regions for batch operations. No lifetime puzzles, no GC tuning, no manual malloc/free. For the 5% of code that needs more control, explicit RC and optional GC provide escape hatches without polluting the common case.

The research confidence is high for the core recommendation but moderate for specific overhead numbers—production validation of the combined approach would be needed. The theoretical foundations (separation logic, substructural types) are solid; the implementation techniques (SSA-based analysis, LLVM integration) are well-understood. What remains is engineering: building the compiler passes, tuning the heuristics, and refining the ergonomics through user feedback.