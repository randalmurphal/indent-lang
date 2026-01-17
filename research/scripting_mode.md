# Achieving scripting-like speed with compiled languages

A **sub-100ms cold start is achievable** with Cranelift-based AOT compilation, but requires combining multiple optimization strategies: V8-style snapshotting, content-addressed binary caching, and Go-style dependency management. The key insight: Go achieves ~160ms with LLVM-equivalent compilation, and Cranelift compiles **10x faster**—making sub-100ms realistic for scripts under 100 lines.

---

## The startup time landscape reveals clear patterns

Current runtimes demonstrate what's possible across different architectures. The fastest cold starts come from interpreters (bash at **3-5ms**, Python at **30-50ms**), while AOT-compiled languages cluster around 130-180ms. The gap between these represents compilation overhead that caching can eliminate.

| Runtime | Cold Start | Warm/Cached | Architecture |
|---------|-----------|-------------|--------------|
| Bash | 3-5ms | 3-5ms | Interpreted |
| Python | 30-50ms | 30-50ms | Bytecode interpreted |
| LuaJIT | <1ms | <1ms | Tracing JIT + optimized interpreter |
| GraalVM Native Image | 7-24ms | 7-24ms | AOT compiled |
| Deno | 130-135ms | 30-40ms | V8 snapshots + TS caching |
| Go (`go run`) | 160-180ms | 160-180ms | AOT, no binary caching |
| rust-script | 30-60 seconds | **<10ms** | Cached binary execution |

The striking finding: **rust-script achieves <10ms cached execution** by simply running a pre-compiled binary. This proves the target is achievable—the challenge is minimizing cold compilation to under 100ms.

---

## Deno's three-layer optimization strategy

Deno achieves fast TypeScript execution through V8 snapshots, aggressive caching, and SWC-based transpilation. These techniques directly apply to a Cranelift-based language.

### V8 snapshots eliminate runtime initialization

V8 snapshots serialize heap state at build time, allowing context creation to drop from **40ms to under 2ms**—a 20x improvement. The snapshot contains:
- Initialized built-in objects (`String`, `Array`, `Object`)
- Compiled bytecode for runtime functions
- Pre-allocated language constants

For a Cranelift-based language, the equivalent is **pre-initializing the runtime and standard library into a serialized blob** loaded via memory-mapping. This avoids parsing and compiling standard library code on each invocation.

### Multi-level caching delivers cumulative wins

Deno implements three cache layers, each providing measurable improvements:

1. **TypeScript transpilation cache** (DENO_DIR/gen/): Stores JavaScript output keyed by source CRC hash. SWC transpilation is 8x faster than Microsoft's TSC, but caching eliminates even this cost on subsequent runs.

2. **V8 bytecode cache** (introduced in Deno 1.43): Caches compiled V8 bytecode, reducing parse/compile time by **20-40%**. Real-world impact ranges from 5% to 240% startup improvement depending on application size.

3. **Dependency cache** (DENO_DIR/deps/): Remote modules cached by URL structure with integrity hashes in lock files.

The V8 code cache follows a three-phase warmup: cold run (compile and execute), warm run (use in-memory cache, create disk cache), hot run (deserialize from disk, skip compilation entirely).

### SWC proves Rust-based tooling is fast enough

Deno's adoption of SWC (Speedy Web Compiler) for TypeScript transpilation demonstrates that Rust-based compilation can be extremely fast. SWC is **20x faster than Babel on a single thread** and 70x faster on four cores. Type-stripping (the core transpilation operation) is simple enough that even cold transpilation adds minimal latency.

---

## Go's compilation model offers transferable lessons

Go achieves ~160ms for `go run` on hello world programs, making it the fastest traditional AOT compiler for scripting scenarios. The architectural decisions are language-agnostic and directly applicable.

### Dependency management is the dominant factor

Google measured that a large C++ binary expanded source code by **2000x** after `#include` preprocessing (4.2MB → 8GB). An equivalent Go program showed only ~40x expansion—**50 times better**. This single difference explains most of Go's speed advantage.

Go achieves this through:
- **Export data in object files**: When compiling package A that imports B, the compiler reads only B's compiled export data—not B's source. This data appears first in object files, allowing early termination.
- **No circular imports**: The language prohibits cyclic dependencies, creating a clean DAG that enables parallel compilation.
- **Explicit, used-only imports**: Unused imports are compile errors, not warnings, preventing accidental dependency bloat.

### The grammar simplicity advantage is overrated

Rob Pike clarified in 2014 that Go's simple syntax (25 keywords, no symbol table needed for parsing) is **"not a significant factor for compilation speed"** compared to dependency management. This means a new language needn't sacrifice expressiveness for parsing speed—the module system matters more.

### Go's compiler intentionally trades optimization for speed

Go's compiler performs fewer optimization passes than LLVM, producing code that runs **~14% slower** but compiles **~5x faster**. This explicit tradeoff aligns with Cranelift's philosophy.

### The `go run` caching gap is instructive

Critically, `go run` does NOT cache the final linked binary (GitHub issue #33468, still open). Every invocation re-links, even with unchanged source. This explains why cached rust-script (<10ms) vastly outperforms cached `go run` (still 160ms). **Binary caching is essential** for sub-100ms targets.

---

## Cranelift vs LLVM makes the 100ms target realistic

Cranelift was explicitly designed for JIT compilation speed, making it ideal for scripting scenarios where compilation latency matters more than peak runtime performance.

### Compilation speed comparison

| Backend | Relative Speed | Code Quality | Use Case |
|---------|---------------|--------------|----------|
| LLVM | 1x (baseline) | Highest | Release builds |
| Cranelift | **~10x faster** | ~14% slower | Debug builds, JIT, scripting |
| Single-pass (Umbra-style) | ~160x faster | Lower | Extreme latency sensitivity |
| Copy-and-patch | ~100x faster | Lower | Pre-compiled stencils |

Research from TU Munich measured Cranelift at **20-35% faster than LLVM** in controlled tests, with some workloads showing 10x improvements. The Rust compiler's Cranelift backend (`rustc_codegen_cranelift`) achieves **20-80% faster** debug builds than LLVM.

### The math works for 100ms cold start

If Go achieves 160ms with an LLVM-equivalent backend, and Cranelift is 10x faster for code generation, the compilation phase could drop to 16ms. Adding parsing (5-15ms), linking (20-50ms), and startup overhead leaves headroom within 100ms for small scripts.

### Cranelift supports REPL-friendly features

Cranelift's `cranelift-jit` crate provides:
- **Hot-swapping**: `jit_builder.hotswap(true)` enables function redefinition
- **Incremental compilation**: IR caching keyed on stencil hash (added 2022)
- **Single IR design**: No lowering stages until machine code, reducing compilation overhead

---

## Cache lookup can achieve sub-10ms with proper design

The sub-10ms cached execution target is achievable based on ccache benchmarks and theoretical minimums.

### Real-world cache hit performance

ccache in direct mode achieves **4.8ms cache hits** for C files—this includes hashing, lookup, and copying the cached artifact. The breakdown:

| ccache Mode | Cache Hit Time | Overhead |
|-------------|---------------|----------|
| Direct mode | **4.8ms** | 145x faster than compile |
| Depend mode | 5.1ms | 137x faster |
| Preprocessor mode | 24.7ms | 28x faster |

### Theoretical minimum is under 1ms

With optimal design, cache lookup overhead can be minimized:
- File stat: ~1μs
- Hash 10KB source (BLAKE3): ~1μs  
- Memory-mapped index lookup: ~3.3ns per operation
- Result file stat: ~1μs
- Copy cached binary to temp: ~10-100μs

**Total theoretical minimum: <1ms** for small cached artifacts.

### Key optimizations for sub-10ms

1. **Memory-map the cache index**: mmap provides 100x+ speedup when pages are in OS page cache (3.3ns vs 333ns for traditional I/O)
2. **Use BLAKE3 or XXH3 for hashing**: 1-5 GB/s throughput vs SHA-256's 0.3-0.5 GB/s
3. **Keep cache metadata in memory**: sccache's server architecture maintains state in RAM, eliminating disk I/O for index lookups
4. **Pre-warm OS page cache**: Frequently accessed cache files stay in RAM; first access triggers minor page fault (~1μs)
5. **Store on NVMe/SSD**: Random read latency 0.1ms (SSD) vs 5-10ms (HDD)

---

## JIT warmup makes AOT superior for scripting

JIT compilation provides excellent peak performance but fundamentally conflicts with scripting use cases where code runs once or few times.

### JIT warmup requirements exclude short scripts

| Runtime | Iterations to Optimize | Notes |
|---------|----------------------|-------|
| JVM C1 compiler | ~1,500 | Light optimizations |
| JVM C2 compiler | ~10,000 | Full optimization |
| V8 Sparkplug | 8 | Minimal optimization |
| V8 TurboFan | Variable (hundreds+) | Peak performance tier |
| PyPy | ~3,000 | 1619 to trace + 1039 to compile |
| LuaJIT functions | ~112 | 2x loop threshold |

Scripts running fewer than these iteration counts see **no JIT benefit**—only interpreter performance or baseline compilation.

### Julia's "time to first plot" problem is cautionary

Julia's JIT-heavy design causes notorious startup latency: first-time plotting historically took **13-20 seconds** due to cascading method compilation. While Julia 1.9+ Package Images reduced this to ~5 seconds, the fundamental issue—deferred compilation creating unpredictable latency—makes JIT unsuitable for CLI tools.

### LuaJIT demonstrates the exception

LuaJIT achieves <1ms startup with excellent performance even without JIT compilation. Its secret: the interpreter itself is hand-optimized DynASM assembler, running **50% faster than standard Lua** without any JIT. For a new language, this suggests investing in interpreter optimization as a fallback provides scripting-appropriate latency.

---

## REPL implementation via LLVM ORC or Cranelift JIT

For interactive development, the language needs incremental compilation with state persistence across expressions.

### LLVM ORC provides production-ready REPL infrastructure

ORC (On-Request-Compilation) is LLVM's third-generation JIT framework, powering Swift REPL, Julia, and Clang-Repl. Key capabilities:

- **Lazy compilation via lazy-reexports**: Functions compile on first call, not definition
- **JITDylibs for symbol management**: Emulates dynamic library behavior for proper linking
- **Concurrent compilation**: Multiple threads can execute JIT'd code while compilers run
- **Removable code**: ResourceTracker enables "undo" functionality

### Cranelift JIT provides similar capabilities faster

Cranelift's `cranelift-jit` crate offers:
- Function redefinition via `prepare_for_function_redefine()`
- Module-level symbol management
- Hot-swapping support

The tradeoff: Cranelift compiles ~10x faster but generates ~14% slower code. For REPL scenarios where iteration speed matters more than runtime performance, Cranelift is the better choice.

### State persistence requires careful design

Swift's approach—embedding the compiler in LLDB with `PersistentParserState`—demonstrates that REPL state management requires:
- Parser state tracking across expressions
- Symbol tables accumulating definitions
- Type context maintained incrementally
- Dependency tracking for invalidation when definitions change

---

## Shebang execution patterns and their tradeoffs

Several compiled languages support script-like execution with varying performance characteristics.

### rust-script achieves the target for warm cache

rust-script demonstrates the ideal cached behavior:
- **Cold start with dependencies**: 30-60+ seconds (compiling dependencies)
- **Warm start (cached binary)**: **<10ms** (runs pre-compiled executable)

The implementation: generate a temporary Cargo.toml, compile via `cargo build`, cache the binary keyed by source hash + dependency versions. Subsequent runs with unchanged source simply execute the cached binary.

### D's rdmd achieves 20ms with dependency caching

rdmd caches compiled executables and dependency information:
- Before .deps caching: 300-1000ms startup
- After .deps caching improvement: **~20ms** rerun time

The key optimization: tracking `.deps` files to avoid re-invoking the compiler for import resolution.

### Zig's incremental compilation targets single-digit milliseconds

Zig's experimental incremental compilation (`-fincremental`) aims for **sub-millisecond rebuilds** via in-place binary patching. Current measurements on the Ghostty project:
- Zig 0.15.1: ~300ms build.zig (vs 1.1s on 0.14)
- Incremental rebuild (one-line change): ~6.2s (vs 25s on 0.14)
- Future goal with full incremental: single-digit milliseconds

---

## Recommended architecture for Cranelift-based scripting

Based on this research, here's the architecture that achieves <100ms cold start and <10ms cached execution:

### Core compilation pipeline

```
Script Source → Hash Check → [CACHE HIT?] → Run Cached Binary (<10ms)
                    ↓ MISS
              Parse (5-10ms)
                    ↓
              Type Check (10-20ms)
                    ↓
              Cranelift IR (10-20ms)
                    ↓
              Native Code (15-30ms)
                    ↓
              Link + Cache (20-40ms)
                    ↓
              Execute
         Total cold: 60-120ms
```

### Essential optimizations

1. **Pre-compiled runtime snapshot**: Serialize initialized runtime state (standard library, built-ins) at build time. Load via mmap on startup. Saves 20-40ms per Go's initialization data.

2. **Content-addressed binary cache**: Key by BLAKE3 hash of (source content, compiler version, target platform). Store in `~/.cache/language-name/`. Lookup overhead <5ms with mmap'd index.

3. **Go-style module system**: Export data in compiled artifacts. Read each dependency exactly once. Prohibit circular imports. This is the **single most impactful optimization** for compilation speed.

4. **Parallel compilation**: Cranelift supports concurrent code generation. With acyclic dependencies, all independent modules compile simultaneously.

5. **Lazy standard library**: Don't compile unused stdlib modules. Import analysis determines minimal set.

6. **Fast linker**: Use mold (50%+ faster than default linkers) or implement custom minimal linker for script binaries.

### Implementation checklist for <100ms

| Component | Target Latency | Approach |
|-----------|---------------|----------|
| Cache lookup | <5ms | mmap'd index, BLAKE3 hash |
| Runtime init | <5ms | Pre-serialized snapshot |
| Parsing | <10ms | Simple grammar, hand-written parser |
| Type checking | <20ms | Incremental, cached module signatures |
| Cranelift codegen | <30ms | Parallel, minimal optimization |
| Linking | <30ms | Custom linker or mold |
| **Total cold** | **<100ms** | |
| **Cached execution** | **<10ms** | Run pre-compiled binary |

### Tiered approach for production

Consider implementing two modes:
- **Script mode**: Cranelift with minimal optimization, <100ms cold start
- **Release mode**: LLVM with full optimization, accepts longer compile time

This mirrors V8's Ignition→TurboFan tiering and Wasmtime's Winch→Cranelift strategy.

---

## Conclusion

The <100ms cold start / <10ms cached targets are **achievable** with a Cranelift-based AOT approach. The key insights:

1. **Binary caching is non-negotiable**: rust-script's <10ms cached execution proves the warm-cache target. Go's failure to cache binaries explains its 160ms floor.

2. **Cranelift's 10x speed advantage changes the math**: If Go achieves 160ms with LLVM-equivalent compilation, Cranelift can achieve 60-80ms for equivalent programs.

3. **Module system design dominates compilation speed**: Go's 50x improvement over C++ comes from dependency management, not grammar simplicity. Adopt Go's export-data-in-object-files pattern.

4. **JIT is wrong for scripting**: Warmup requirements (hundreds to thousands of iterations) make JIT unsuitable for one-shot scripts. AOT with aggressive caching is the correct architecture.

5. **Pre-serialized runtime beats cold initialization**: V8 snapshots demonstrate 20x speedup (40ms→2ms). Apply the same technique to runtime/stdlib initialization.

The DevOps/backend developer replacing bash scripts needs predictable, instant-feeling execution. With this architecture, a Cranelift-based language can deliver that experience while providing the safety and performance benefits of a compiled language.