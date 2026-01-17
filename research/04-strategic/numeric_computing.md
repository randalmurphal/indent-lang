# Numeric computing foundations for the Indent programming language

A backend-oriented language requires integer safety without runtime panic overhead, IEEE 754 compliance with opt-in fast-math, Zig-style first-class SIMD vectors, explicit numeric conversions with Rust-inspired traits, and zero-cost iterator abstractions. This report synthesizes design decisions from Rust, Go, Zig, Swift, and Julia to provide actionable specifications for Indent's numeric subsystem.

The core tension in numeric language design is **safety versus performance**—Rust's debug-mode overflow checking catches bugs but adds 5-10% overhead; Go's silent wrapping enables CVE-class vulnerabilities. For backend workloads processing high volumes of data, Indent needs designs that eliminate entire bug classes at compile time while generating code competitive with hand-optimized C. The recommendations below prioritize compile-time guarantees over runtime checks wherever possible, with explicit opt-in mechanisms for both stricter safety and relaxed performance modes.

## Integer overflow: checked by default with explicit wrapping operators

**Recommendation**: Panic on overflow in debug builds, configurable behavior in release (default: checked), with explicit wrapping/saturating operators for intentional modular arithmetic.

The evidence strongly favors Rust's approach over Go's. Integer overflow causes real security vulnerabilities—CWE-190 ranks #13 in Common Weakness Enumeration, with exploits including WhatsApp RCE (CVE-2022-36934) and Chrome Skia (CVE-2023-2136). Trail of Bits forked the Go compiler specifically to detect "silent arithmetic bugs" because Go's wrapping semantics are "a potential source of serious vulnerabilities." Meanwhile, Rust's debug-mode checking catches real bugs during development without penalizing production performance when disabled.

Performance overhead of checked arithmetic is **5-10%** on integer-heavy workloads according to benchmarks from John Regehr and Dan Luu. LLVM's constraint elimination and range analysis can eliminate many redundant checks. The key insight from Sanjoy Das's LLVM work: systematic optimization reduced SPEC overhead to **8.7%**, and delayed panic optimization (coalescing checks across operations) further reduces overhead. For tight loops where this matters, explicit `wrapping_add` operations signal intent while bypassing checks.

| Operation Type | Syntax | Behavior |
|---------------|--------|----------|
| Default arithmetic | `a + b` | Checked (panic on overflow) |
| Wrapping | `a +% b` or `a.wrapping_add(b)` | Two's complement modular |
| Saturating | `a +| b` or `a.saturating_add(b)` | Clamps to MIN/MAX |
| Checked returning Option | `a.checked_add(b)` | Returns `None` on overflow |

For BigInt, the recommendation is **explicit promotion only**—no automatic widening. Fixed-width integers remain the default for performance; `BigInt` is a separate type in the standard library backed by GMP or similar. Julia's approach validates this: "type promotion between primitive types and BigInt is not automatic and must be explicitly stated." Consider 128-bit integers (`i128`/`u128`) as an intermediate step before arbitrary precision.

## Floating point: IEEE 754 strict by default with scoped fast-math

**Recommendation**: Default to IEEE 754 compliance with SSE2/AVX (not x87), provide decimal types for money, implement both partial and total ordering, and offer per-function fast-math annotations.

The cross-platform reproducibility problem is severe. x87 performs operations at 80-bit precision internally, rounding only when storing to memory—meaning **debug and release builds produce different results** depending on register allocation. The solution: mandate SSE2 as the minimum floating-point target (`-mfpmath=sse`), which provides consistent 32/64-bit precision matching IEEE 754.

For performance-critical code, `-ffast-math` enables dangerous optimizations: `-ffinite-math-only` removes `isnan()` checks entirely, `-fassociative-math` breaks Kahan summation. Critically, flush-to-zero behavior is **set at link time and affects all code in the thread**, including third-party libraries. The recommendation is scoped fast-math via function attributes:

```
@[fast_math]
fn simd_dot_product(a: []f32, b: []f32) -> f32 { ... }
```

**Decimal types** are essential for financial applications. Binary floating point cannot exactly represent `0.1`—the classic `0.1 + 0.2 ≠ 0.3` problem. Indent should provide a `decimal64` or `decimal128` type (IEEE 754-2008) for money calculations, roughly 10-20x slower than binary floats but exact for decimal fractions. C#'s `decimal` (128-bit, 28-29 significant digits) provides a good reference implementation.

**NaN comparison semantics** require explicit design. IEEE 754's `NaN != NaN` breaks sorting, hash maps, and equality testing. Rust's solution—`PartialOrd` for standard comparison, `total_cmp()` for total ordering—is elegant. Indent should:
- Implement partial ordering by default (NaN comparisons return `None`)
- Provide `total_cmp()` that places NaNs after infinity
- Disallow floats as hash map keys without explicit wrapper type

## SIMD: first-class vector types with automatic LLVM lowering

**Recommendation**: Adopt Zig-style `Vector(N, T)` as a first-class type with standard operators, letting LLVM handle target-specific lowering and runtime dispatch.

Modern compilers can auto-vectorize many patterns, but explicit SIMD provides **4-25x speedups** for critical operations. simdjson achieves 7+ GB/s JSON parsing—25x faster than nlohmann/json—through explicit vectorization. UTF-8 validation reaches **23x faster** than standard library on non-ASCII text with SIMD.

Zig's approach is the cleanest design for a compiled language:

```
const a: @Vector(4, f32) = .{ 1, 2, 3, 4 };
const b: @Vector(4, f32) = .{ 5, 6, 7, 8 };
const c = a + b;  // Standard operators work element-wise
```

This compiles directly to LLVM vector IR (`<4 x float>`), which handles target-specific lowering. Operations on vectors shorter than native SIMD width become single instructions; longer vectors become multiple instructions; no SIMD support falls back to scalar loops. Built-in functions should include:

- `splat(value)` — broadcast scalar to all lanes
- `reduce(op, vector)` — horizontal sum/min/max/and/or
- `shuffle(a, b, mask)` — permute elements
- `select(mask, a, b)` — conditional per-lane selection

For runtime dispatch across CPU generations (SSE4.2 → AVX2 → AVX-512), Indent should support function multiversioning:

```
@[target("avx2")]
fn process_batch_avx2(data: []f32) { ... }

@[target("sse4")]  
fn process_batch_sse4(data: []f32) { ... }
```

The runtime detects CPU features once at startup and dispatches to the optimal implementation.

## Numeric type hierarchy: explicit conversions with From/TryFrom traits

**Recommendation**: No implicit numeric conversions, polymorphic literals, and trait-based conversion tracking (lossless vs. potentially lossy).

C's integer promotion rules cause subtle bugs: mixing signed/unsigned of the same rank converts signed to unsigned, making `-1 < 1u` return false. Modern languages (Rust, Swift, Zig) converge on requiring explicit conversions between all numeric types. This catches bugs at compile time rather than producing surprising runtime behavior.

**Core type hierarchy**:

```
Numeric
├── Integer
│   ├── Signed: i8, i16, i32, i64, i128, isize
│   └── Unsigned: u8, u16, u32, u64, u128, usize
├── Float: f16, f32, f64
└── (Library: Decimal, Complex, Rational, BigInt)
```

**Conversion traits** (following Rust's design):

| Trait | Semantics | Example |
|-------|-----------|---------|
| `From<T>` | Infallible, lossless | `i32: From<i16>` |
| `TryFrom<T>` | Fallible, may truncate | `i32::try_from(i64_val)?` |

The `From` trait should only exist for conversions that are infallible, lossless, value-preserving, and obvious. All other conversions require `try_from()` or explicit `as` cast. The `as` keyword performs truncating casts without checking—useful for low-level code but should trigger linter warnings in safe code.

**Literal inference**: Treat integer literals as compile-time arbitrary precision (like Zig's `comptime_int`), with type determined by context. Default to `i32` for integers and `f64` for floats when context doesn't constrain. This enables `const BILLION = 1_000_000_000` without explicit type annotation while catching overflow at compile time.

**Complex and rational numbers** should be library-only. Most backend code doesn't use them, and library implementations with operator overloading provide adequate ergonomics without complicating the core language.

## Array operations: strided views with explicit broadcasting

**Recommendation**: Separate storage from view (Rust ndarray pattern), row-major default, views by default for slicing, explicit broadcasting syntax.

NumPy's ergonomics come from separating the *view* (shape, strides, dtype) from the underlying *buffer*. Transpose is a stride swap—no data copy. Slicing creates a new descriptor pointing to the same data. This pattern enables zero-copy operations for most transformations.

**Memory layout**: Default to row-major (C order) for interoperability with most C libraries, but support column-major (Fortran order) for BLAS-heavy workloads. Performance impact is dramatic: column-wise access on row-major data is **6-25x slower** for large matrices due to cache misses.

**Slicing semantics**: Views by default, explicit `.copy()` when needed. Borrowing rules (à la Rust) prevent aliased mutable access:

```
let slice = data[1..5];        // View, borrows data
let copy = data[1..5].copy();  // Independent copy
```

For disjoint mutable access, provide `split_at_mut()`-style primitives that return two non-overlapping mutable slices.

**Broadcasting**: Support NumPy-compatible rules for familiarity, but consider explicit syntax (like Julia's `.+`) to make broadcasting visible:

```
// Explicit broadcasting operator
result = matrix .+ vector;  // Broadcasts vector across rows

// Or require explicit broadcast call
result = broadcast(matrix, vector, |a, b| a + b);
```

Implicit broadcasting is a common source of shape-mismatch bugs. Compile-time shape checking where possible catches errors earlier.

**BLAS integration**: Provide pluggable BLAS backend via build configuration (OpenBLAS, Intel MKL, Apple Accelerate) with pure-language fallback for portability. Use CBLAS interface to avoid Fortran ABI issues. For matrices smaller than ~32×32, direct implementation may outperform BLAS due to call overhead.

## Data processing primitives: zero-cost iterators with opt-in parallelism

**Recommendation**: Rust-style lazy iterators with guaranteed fusion, explicit `.par_iter()` for parallelism, arena allocators for batch processing.

Rust iterators compile to code **as fast or faster than hand-written loops**—benchmarks show iterator chains at 3.6µs versus 6.6µs for direct loops due to LLVM's ability to unroll and vectorize the fused loop. Java streams show the opposite: ~6x slower than loops due to allocation and megamorphic dispatch overhead.

**Iterator design principles**:

1. **Lazy by default**: No work until terminal operation (`.collect()`, `.sum()`, `.for_each()`)
2. **Monomorphization**: Generic adapters specialize at compile time
3. **Inlining**: Each adapter is a struct with inlined `next()` method
4. **Size hints**: Enable pre-allocation for `.collect()`

**Parallel iteration** should be explicit opt-in via `.par_iter()` (Rayon model). The N×Q rule applies: only parallelize when `elements × cost_per_element > 10,000`. Work-stealing scheduler with binary tree splitting handles load balancing. Critical guarantee: data-race freedom at compile time through ownership system.

| Pattern | When to Use | Overhead |
|---------|-------------|----------|
| Sequential iterator | Default | ~0% vs manual loop |
| Parallel iterator | CPU-bound, >10K elements | Thread coordination |
| Chunked processing | Data larger than memory | I/O latency |

**Memory efficiency patterns**:

- **Arena allocators**: Pre-allocate memory block, bump-pointer allocation, free everything at once. Essential for per-request allocation in servers—10-50x faster than malloc.
- **Small-buffer optimization**: Store small strings/vectors inline (16-22 bytes) without heap allocation. 5x faster creation for short strings.
- **Object pooling**: Reuse expensive-to-construct objects. Benchmarks show 6-8x speedup for pooled allocation.

## Performance patterns: bounds elimination and cache optimization

**Recommendation**: Use iterators to eliminate bounds checks by construction, prefer SoA layouts for batch operations, expose LLVM pragmas for explicit control.

Bounds check overhead is **2-5%** in typical code, but can reach 10-15% in tight numeric loops. The best strategy is *elimination by construction*—use iterators instead of indexed access, or create slices before loops to communicate known bounds to the optimizer:

```
// Better: slice communicates bounds to optimizer
let slice = &data[..n];
for i in 0..slice.len() {
    slice[i] = 0;  // Check eliminated
}

// Best: iterator eliminates checks entirely
for item in data.iter_mut() {
    *item = 0;  // No bounds check possible
}
```

**Data layout** has dramatic impact. Struct-of-Arrays (SoA) enables **40-60% faster** batch operations on single fields compared to Array-of-Structs (AoS), because cache lines hold 16 floats in SoA versus 2-3 structs in AoS. Consider language support for automatic SoA generation from struct definitions.

**Cache optimization checklist**:

- Align to 64-byte cache lines for false-sharing prevention
- Align to SIMD register width (32 bytes for AVX, 64 for AVX-512)
- Split hot/cold data to keep working set in cache
- Prefer sequential access patterns that hardware prefetcher can detect

**Profile-guided optimization** provides 10-20% improvement with minimal effort. Indent should integrate PGO into its build system:

```bash
indent build --profile-generate
./run_benchmarks
indent build --profile-use=profile.data
```

LLVM's PGO enables branch weight annotation, hot/cold splitting, and optimized inlining decisions based on actual call frequencies.

## Summary of design recommendations

| Area | Recommendation | Rationale |
|------|----------------|-----------|
| **Integer overflow** | Debug=panic, release=configurable (default checked), explicit wrapping ops | Catches bugs early, CVE prevention |
| **Float semantics** | IEEE 754 strict, SSE2 minimum, per-function fast-math | Reproducibility + opt-in performance |
| **Decimal type** | `decimal64`/`decimal128` in stdlib | Financial calculations require exactness |
| **NaN handling** | Partial ordering default, `total_cmp()` for sorting | Prevents hash/sort bugs |
| **SIMD** | First-class `Vector(N, T)` with operators | Clean syntax, LLVM handles lowering |
| **Type conversions** | Explicit only, `From`/`TryFrom` traits | Prevents silent precision loss |
| **Literals** | Compile-time arbitrary precision, contextual typing | Ergonomic without surprise overflow |
| **Array views** | Strided descriptors, views by default | Zero-copy slicing |
| **Broadcasting** | Explicit syntax (`.+` or `broadcast()`) | Prevents shape-mismatch bugs |
| **Iterators** | Lazy, fused, parallel opt-in | Zero-cost abstraction |
| **Memory** | Arena allocators, SBO, pooling in stdlib | 10-50x allocation speedup |
| **Bounds checks** | Eliminate via iterators, slicing patterns | 2-5% overhead when unavoidable |

These design choices position Indent as a pragmatic successor to Go and Rust for backend workloads—catching numeric bugs at compile time while generating competitive machine code. The explicit-by-default philosophy may require more keystrokes than C or Go, but prevents the silent corruption and security vulnerabilities that plague production systems built in those languages.