# Profiling and performance tooling strategy for Indent

A new LLVM-based systems language with Rust-level performance and ARC memory management requires a carefully designed profiling ecosystem. **The optimal strategy combines minimal built-in runtime hooks with comprehensive compiler instrumentation support, enabling zero-cost profiling when disabled while maintaining compatibility with the rich ecosystem of external tools.** This approach mirrors Rust's philosophy while incorporating lessons from Go's production-ready pprof and Swift's ARC debugging capabilities.

The key architectural insight is that most profiling tools share common requirements—debug symbols, frame pointers for stack unwinding, and clear function boundaries. Implementing these fundamentals correctly enables broad ecosystem compatibility without reinventing wheels. For an ARC-based language specifically, compiler-level reference counting optimization is crucial, as Swift's experience demonstrates that well-optimized builds show dramatically fewer retain/release calls than naive implementations.

## CPU profiling requires frame pointers and pprof compatibility

**Sampling profiler integration** depends critically on stack unwinding capability. The modern consensus, reflected in Ubuntu 24.04 and Fedora 40+ defaulting frame pointers to ON, recognizes that the **0-2% performance overhead** is vastly outweighed by profiling benefits. Indent should default to frame pointers enabled on 64-bit platforms with an explicit flag to disable (`-no-frame-pointers`).

For stack unwinding, two approaches exist: frame pointer-based (fast, ~1 second to process large traces) and DWARF-based (slow, ~8+ minutes for equivalent data but works without frame pointers). Recommend supporting both via compiler flags:

```
indent build --release                        # Default: frame pointers ON
indent build --release --no-frame-pointers    # Explicit disable
indent build --release --debug=line-tables    # DWARF for profiling
```

**The pprof format should be the native profiling output** given its ecosystem dominance. Pprof uses a Protocol Buffer format with string table deduplication, multiple value types per sample, and rich metadata. Tools consuming pprof include Google's pprof CLI, Perfetto, speedscope, Pyroscope, and Parca. Go's built-in HTTP endpoint pattern (`/debug/pprof/*`) is worth emulating for production services.

Implementation requires a runtime sampling profiler using SIGPROF signals (Unix) or similar mechanisms. The core API:

```
const Profiler = @import("std").profiler;

pub fn main() void {
    var guard = Profiler.start(.{ .frequency = 100 });
    defer guard.stop();
    guard.writeProfile("profile.pb.gz");
}
```

**Flame graph generation** follows naturally from pprof support since tools like speedscope and `go tool pprof -http` render flame graphs directly. For standalone generation, support Brendan Gregg's folded stack format (semicolon-separated stack trace followed by sample count). Differential flame graphs for regression detection require storing baseline profiles and computing per-function time deltas—red indicates increased time, blue indicates decreased.

**Profile-Guided Optimization** leverages LLVM's infrastructure directly. The workflow involves instrumented builds (`-profile-generate`), running representative workloads, merging profiles with `llvm-profdata`, and optimized builds (`-profile-use`). Typical improvements range from **2-14% for Go applications to 10-30% for large C++ codebases**. Uber reports ~4% fleet-wide CPU reduction from Go PGO. Indent should provide ergonomic flags:

```
indent build --profile-generate=/tmp/profiles
indent build --profile-use=merged.profdata
```

## Memory profiling centers on allocator abstraction and ARC instrumentation

**Allocation tracking** requires a global allocator trait similar to Rust's `GlobalAlloc`, enabling custom allocators for profiling without code changes:

```
trait GlobalAllocator {
    fn alloc(layout: Layout) -> *mut u8
    fn dealloc(ptr: *mut u8, layout: Layout)
    // Optional profiling hooks
    fn on_alloc(ptr: *mut u8, layout: Layout) {}
    fn on_dealloc(ptr: *mut u8, layout: Layout) {}
}
```

Feature-flag profiling support ensures zero overhead in production. For memory tracking tools, jemalloc's statistical sampling (configurable via `lg_prof_sample`, ~6-15% overhead) works well for production, while heaptrack's full tracking suits development.

**ARC operation profiling is critical** for an ARC-based language. Swift's approach provides the model:

- Compiler inserts `retain`/`release` calls during intermediate representation generation
- Aggressive optimization passes eliminate redundant operations
- Three reference counts per object: strong, weak, unowned
- Side tables (Swift 4+) enable object deallocation while weak references exist

For debugging, expose ARC operation counters behind a compile-time flag with per-thread counters to avoid contention:

```
struct MemoryStats {
    total_allocated: u64,
    current_allocated: u64,
    peak_allocated: u64,
    retain_count: u64,
    release_count: u64,
}
```

**Retain cycle detection** requires graph-based analysis. Implement reference graph maintenance in debug mode with periodic cycle detection using Tarjan's algorithm. Export memory graphs for visualization, similar to Xcode's Memory Graph Debugger which shows purple markers for detected cycles.

**Memory leak detection** should integrate LLVM's LeakSanitizer (`-fsanitize=leak`), which has almost no overhead until process exit. AddressSanitizer (`-fsanitize=address`) provides broader memory error detection with ~2x overhead. Both work at LLVM IR level and require linking compiler-rt runtime library.

| Tool | Overhead | Use Case |
|------|----------|----------|
| LeakSanitizer | Minimal until exit | Production leak checking |
| AddressSanitizer | 2x CPU, 2-4x RAM | Development, CI |
| heaptrack | 2-10x | Deep debugging |
| Valgrind | 20-30x | Comprehensive analysis |

## Async profiling requires runtime instrumentation hooks

**Task timing analysis** follows Tokio Console's architecture: a two-component system with instrumentation in the runtime and a separate visualization tool communicating via gRPC. Key metrics per task:

| Metric | Description |
|--------|-------------|
| **Total** | Duration task has been alive |
| **Busy** | Time actively executing (being polled) |
| **Scheduled** | Time waiting to be polled |
| **Idle** | Time waiting to be woken |
| **Polls** | Number of times polled |

The wire protocol should use Protocol Buffers for cross-language compatibility. Core event types:

```
enum TaskEvent {
    Created { task_id, parent_id, name, location },
    Scheduled { task_id, timestamp },
    PollStart { task_id, timestamp },
    PollEnd { task_id, timestamp, yielded: bool },
    Woken { task_id, waker_task_id },
    Completed { task_id, timestamp }
}
```

**Channel utilization profiling** tracks queue depth, throughput, and blocked sender/receiver counts. Go's block profiler (`runtime.SetBlockProfileRate`) provides a model for configurable sampling of blocking operations.

**Lock contention analysis** benefits from multiple tools. For development, mutrace uses LD_PRELOAD to intercept pthread mutex operations with minimal performance impact, reporting lock ownership changes and actual contention events. For production, Intel VTune's Threading Analysis combines concurrency and lock analysis with ITT API support for custom instrumentation. Linux perf's lock contention analysis (`perf lock con`) works with BPF for live analysis.

**ThreadSanitizer** (`-fsanitize=thread`) detects data races using hybrid happens-before + lockset analysis. Overhead is **5-15x CPU and 5-10x memory**, making it suitable for testing rather than production. Mozilla Firefox successfully used TSan to eliminate data races.

**Scheduler overhead measurement** requires exposing runtime metrics similar to Go's `GODEBUG=schedtrace`:

- Global and per-worker queue lengths
- Steal counts for work-stealing schedulers
- Context switches
- Idle/spinning thread counts

Implement runtime warnings like Tokio Console's "clippy for async": detecting tasks running too long without yielding, self-waking tasks, and abnormal poll durations.

## Benchmarking needs built-in primitives with external statistical analysis

**Built-in framework requirements** are minimal but essential:

- **Black box primitive**: `std.hint.black_box()` equivalent preventing dead code elimination
- **High-resolution timer**: Monotonic, nanosecond precision
- **DoNotOptimize**: Compiler intrinsic consuming values
- **Automatic iteration scaling**: Adjust iterations to meet measurement time threshold
- **JSON output format**: Standardized schema for ecosystem tools

Go 1.24's `testing.B.Loop` innovation provides an elegant model—the compiler detects benchmark loops and disables inlining within them, preventing DCE without manual intervention.

**Statistical rigor should be external** (Criterion.rs model). The statistical methods required—bootstrap resampling with 10,000+ samples, linear regression fitting, T-tests for regression detection—are complex and evolving. Criterion.rs represents the gold standard:

- Four-phase cycle: warmup, measurement, analysis, comparison
- Modified Tukey's method for outlier detection (but doesn't drop outliers)
- Configurable significance level (default 0.05) and noise threshold (default 1%)
- HTML reports with distribution histograms and comparison charts

**CI integration** requires noise-resistant measurement. CodSpeed uses CPU simulation to achieve <1% variance in measurements, isolating from CI environment noise. Bencher provides cloud-hosted tracking with statistical thresholds and PR comments. For deterministic results, instruction counting (Iai for Rust) eliminates timing variance entirely.

Recommended API design:

```
benchmark "my_bench" {
    setup { /* runs once before measurement */ }
    measure { code_to_benchmark() }
    teardown { /* runs once after */ }
}
```

## Binary analysis tooling enables size optimization

**Bloaty McBloatface** is the gold standard for binary size analysis, providing dual-view architecture (FILE SIZE vs VM SIZE), multi-format support (ELF, Mach-O, PE, WebAssembly), and hierarchical profiles combining sections, compile units, symbols, and inlines.

**Dependency contribution analysis** requires crate/module-level attribution like cargo-bloat's `--crates` flag. Note that symbol-to-module attribution uses heuristics and isn't 100% accurate—document this limitation.

**Monomorphization bloat detection** is crucial for generics-heavy code. cargo-llvm-lines measures generic instantiations by LLVM IR lines generated. Mitigation strategies include:

- Use dynamic dispatch (`dyn Trait`) for large interfaces
- Factor out non-generic inner functions  
- Consider automatic degenericization passes

**Dead code elimination** operates at multiple levels:

1. **Compiler warnings**: Warn about unused items
2. **LTO**: `-lto=true` enables whole-program optimization
3. **Linker GC**: `--gc-sections` removes unreferenced sections
4. **Codegen units**: `codegen-units=1` allows better cross-module optimization

**Binary size optimization flags** should be comprehensive:

```toml
[profile.release-small]
strip = true          # Strip symbols
opt-level = "z"       # Optimize for size
lto = true            # Link-time optimization
codegen-units = 1     # Better optimization
panic = "abort"       # Remove unwinding code
```

Typical hello world sizes show the tradeoffs: C/Zig achieve ~2-8KB minimal, Rust achieves ~8KB with `#![no_std]` and all optimizations, while Go's runtime baseline is ~800KB optimized.

## Tool integration requires DWARF generation and probe points

**Linux perf integration** requires ELF symbols, DWARF debug info, and frame pointers. For JIT code, implement `PerfJITEventListener` writing to `/tmp/perf-PID.map` and creating JitDump files.

**USDT probes** (User Statically Defined Tracing) enable powerful eBPF/DTrace debugging. Add probes for runtime events:

- GC: `gc:start`, `gc:stop`, `gc:phase`
- Allocations: `alloc:malloc`, `alloc:free`
- ARC: `arc:retain`, `arc:release`

Probes compile to NOP sleds + ELF `.note.stapsdt` section metadata, enabling zero overhead when not traced.

**Tracy profiler integration** provides real-time profiling popular in game development. Core API wraps zone macros, memory tracking, and lock profiling:

```cpp
ZoneScoped;  // Automatic function profiling
TracyAlloc(ptr, size);  // Memory tracking
TracyLockable(std::mutex, myMutex);  // Lock profiling
```

**Perfetto trace format** (protobuf-based) enables CI/CD performance regression testing with the Perfetto UI for visualization. Support Chrome Tracing JSON format for broad compatibility.

**OpenTelemetry profiling** is emerging but unstable (added to OTLP v1.3.0). Monitor for stabilization—the eBPF-based agent donated by Elastic supports native profiling of C/C++, Rust, Zig, and Go without language-specific instrumentation.

**DWARF generation priorities**:

| Section | Profiler Need |
|---------|---------------|
| `.debug_line` | Source attribution (critical) |
| `.debug_frame` | Stack unwinding (critical) |
| `.debug_info` | Symbol resolution |
| `.debug_aranges` | Fast lookup |

Use LLVM's `DIBuilder` API for compile units, functions, and types. Support split debug info (`-gsplit-dwarf`) to reduce binary size while maintaining profiling capability.

## Language design should follow the hybrid model

**The recommended architecture** combines minimal runtime hooks with comprehensive compiler support and external tool compatibility:

**Built-in (zero-cost when disabled)**:
- Frame pointer control (default ON for 64-bit)
- Debug info levels: none, line-tables-only, full
- Allocator trait with profiling hooks
- Monotonic high-resolution timer
- Thread/task identification
- Basic memory statistics

**Compiler flags to support**:
```
-profile-generate=<path>         # PGO instrumentation
-profile-use=<path>              # PGO optimization
-coverage                        # Source-based coverage
-sanitize=address,thread,memory,undefined
-debug-info=none|line-tables|full
-force-frame-pointers=yes|no
-instrument-functions            # Function entry/exit hooks
-xray-instrument                 # LLVM XRay support
```

**ARC-specific requirements**:
- Expose retain/release counts behind debug flag
- Provide weak/unowned reference primitives
- Implement retain cycle detection (graph analysis in debug builds)
- Support compile-time ARC optimization analysis
- Consider probabilistic checking (GWP-ASan style) for production

**External tool compatibility**:
- pprof-compatible output (primary format)
- DWARF debug info for stack unwinding
- USDT probes for eBPF/DTrace
- Tracy protocol support
- Perfetto trace export

This hybrid approach achieves **zero-cost when disabled** while enabling comprehensive profiling when needed, positioning Indent to leverage the rich ecosystem of existing tools while providing first-class profiling support where it matters most—ARC debugging, async task analysis, and production-safe CPU/memory profiling.