# Compilation Stack Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Affects compile times, interop, and what's feasible
**Key Question**: What gives us Go-like compile speeds with Rust-like output?

## The Problem Space

We want:
- ✅ Fast compilation (Go compiles huge projects in seconds)
- ✅ Fast runtime (Rust-level performance)
- ✅ Good debugging experience
- ✅ Easy C interop (for calling libraries)
- ✅ Cross-platform (Linux, macOS, Windows at minimum)
- ✅ Small binaries (for containers)
- ✅ Reasonable implementation complexity (we're a small team)

The fundamental tension: **optimize for compile time OR runtime, hard to get both**.

## Why Compilation Speed Matters for Us

Target users: backend devs, devops, scripting replacement

| Use Case | Compile Time Requirement |
|----------|-------------------------|
| Development iteration | < 1 second for incremental |
| Full rebuild | < 10 seconds for medium project |
| CI/CD | Fast enough to not be bottleneck |
| Scripting use | Near-instant for small files |

Rust's compile times (minutes for large projects) are unacceptable for our use case.

## Existing Compilation Stacks

### 1. LLVM

**What it is**: Industry-standard compiler infrastructure. Frontend emits LLVM IR, LLVM optimizes and generates machine code.

**Used by**: Rust, Swift, Clang (C/C++), Julia, Mojo, Zig (optionally)

**Architecture**:
```
Source → [Frontend] → LLVM IR → [LLVM Optimizer] → [LLVM Backend] → Machine Code
                                     ↑
                            (This is slow)
```

**Pros**:
- Excellent optimization (decades of work)
- Outstanding code generation
- All platforms supported
- Mature debugging support (DWARF, CodeView)
- C interop is trivial (same ABI)
- Well-documented, large community

**Cons**:
- **Slow compilation** (optimization passes are expensive)
- Large dependency (~100MB+ for LLVM libraries)
- Complex to work with (IR is verbose)
- Compile times scale poorly with code size
- LLVM itself is complex C++ codebase

**Compile Time Reality**:
```
Rust (LLVM, optimized):    ~10-60 seconds for medium project
Rust (LLVM, debug):        ~5-30 seconds for medium project
Swift (LLVM):              Similar to Rust
Clang (C++, optimized):    ~30 seconds for medium project
```

**How Rust uses it**:
- MIR (Mid-level IR) → LLVM IR → Machine code
- Borrow checker is BEFORE LLVM (not the slow part)
- LLVM optimization passes ARE the slow part
- Incremental compilation helps but not enough

---

### 2. Cranelift

**What it is**: Fast code generator originally built for WebAssembly, now used by Rust for debug builds.

**Used by**: Rust (rustc_codegen_cranelift), Wasmtime

**Architecture**:
```
Source → [Frontend] → Cranelift IR → [Cranelift] → Machine Code
                                          ↑
                          (Designed for speed over optimization)
```

**Pros**:
- **Very fast compilation** (designed for JIT, fast for AOT too)
- Produces decent code (not as optimized as LLVM, but good)
- Written in Rust (we can modify it easily)
- Simpler than LLVM
- Actively developed

**Cons**:
- Less optimized output than LLVM (10-30% slower code typical)
- Fewer platforms (x86_64, aarch64 well-supported, others limited)
- Smaller community than LLVM
- Less mature debugging support
- Still evolving

**Compile Time Reality**:
```
Rust (Cranelift, debug):   ~50% faster than LLVM debug builds
Cranelift itself:          Designed for < 1ms per function
```

**Potential for us**: Use Cranelift for development, LLVM for release builds.

---

### 3. GCC

**What it is**: GNU Compiler Collection. Oldest major open-source compiler.

**Used by**: C, C++, Go (gccgo), Fortran, Ada

**Pros**:
- Excellent optimization (competes with LLVM)
- All platforms supported (more than LLVM)
- Very mature
- Go frontend exists (gccgo)

**Cons**:
- Slower than LLVM typically
- GPL license (may matter for some uses)
- Less modular than LLVM
- C/C++ codebase
- Less active development than LLVM lately

**Verdict**: Not a good fit. LLVM is better for our needs.

---

### 4. Go's Compiler (gc)

**What it is**: Go's custom compiler toolchain. Not LLVM or GCC.

**Architecture**:
```
Source → [Parse] → [Type Check] → [SSA] → [Machine Code]
                                    ↑
                    (Custom, optimized for Go's needs)
```

**Pros**:
- **Incredibly fast compilation** (compiles itself in < 1 minute)
- Produces good code (not LLVM-level, but close enough)
- Purpose-built for the language
- Simple cross-compilation

**Cons**:
- Go-specific, can't reuse
- Less optimized than LLVM output
- Custom == more work to build

**Compile Time Reality**:
```
Go (gc):                   ~1-5 seconds for medium project
Go (full rebuild):         ~10-30 seconds for large project
```

**What makes it fast**:
1. Simple language (no generics until recently, limited features)
2. No heavy optimization passes
3. Designed from day 1 for fast compilation
4. Concurrent compilation
5. Dependency management eliminates redundant work

**Key insight**: Go traded some runtime performance for compile speed. Their users accepted this trade-off.

---

### 5. Zig's Compiler

**What it is**: Self-hosted compiler with optional LLVM backend.

**Architecture**:
```
                          ┌→ [Zig Backend] → Machine Code (fast, less optimized)
Source → [Frontend] → IR ─┤
                          └→ [LLVM Backend] → Machine Code (slow, optimized)
```

**Pros**:
- Fast self-hosted backend for debug/development
- LLVM for release builds when needed
- Written in Zig (bootstrapped)
- Good C interop
- Interesting incremental compilation approach

**Cons**:
- Zig-specific
- Self-hosted backend still less optimized
- Smaller community

**Key insight**: The two-backend approach is promising for our needs.

---

### 6. QBE

**What it is**: Extremely simple compiler backend. < 15k lines of C.

**Philosophy**: "80% of LLVM's performance with 1% of the complexity"

**Pros**:
- Tiny, understandable
- Fast compilation
- Reasonable output quality
- Easy to modify

**Cons**:
- x86_64 and aarch64 only
- No Windows support
- Less optimized than LLVM/Cranelift
- Very small community

**Verdict**: Interesting for learning/prototyping, not for production.

---

### 7. Custom Backend (Like Go)

**What it is**: Build our own code generator.

**Pros**:
- Complete control
- Optimize for our specific needs
- No external dependencies

**Cons**:
- Massive undertaking
- Need to implement all optimizations ourselves
- Platform support is our problem
- Debugging support is our problem
- Years of work

**Verdict**: Not realistic for a small team starting out. Maybe later.

---

## Hybrid Approaches

### Rust's Approach
```
Development:  rustc → MIR → Cranelift → Fast binary (10-30% slower)
Release:      rustc → MIR → LLVM → Optimized binary
```

This gives fast iteration AND optimized releases.

### Zig's Approach
```
Development:  zig → IR → Self-hosted backend → Fast binary
Release:      zig → IR → LLVM → Optimized binary
Debug/Safe:   zig → IR → Debug backend (with checks)
```

### Our Potential Approach
```
Scripting:    our-lang → IR → JIT (Cranelift) → Execute
Development:  our-lang → IR → Cranelift → Binary (fast compile)
Release:      our-lang → IR → LLVM → Binary (optimized)
```

## Compile Time Analysis

What makes compilation slow?

| Factor | Impact | Mitigation |
|--------|--------|------------|
| Parsing | Low (~5%) | Efficient parser |
| Type checking | Medium (~10-20%) | Incremental, parallel |
| Borrow checking | Medium (~10-15%) | (Only if we have it) |
| IR generation | Low (~5%) | Efficient codegen |
| **Optimization** | **High (~40-60%)** | Skip for dev, use Cranelift |
| Code generation | Medium (~15-20%) | Parallel per function |
| Linking | Medium (~10-20%) | Incremental linking |

**Key insight**: LLVM optimization passes are the bottleneck. Skip them for development.

## Incremental Compilation

Critical for development speed.

| Approach | Granularity | Complexity |
|----------|-------------|------------|
| File-level | Recompile changed files | Low |
| Function-level | Recompile changed functions | Medium |
| Query-based (Rust/Salsa) | Recompile changed queries | High |

Go achieves fast compiles with file-level + good dependency tracking.
Rust has sophisticated incremental compilation but still slow.

**For us**: Start with file-level, good dependency tracking. Add function-level later if needed.

## Language Design Affects Compile Time

| Feature | Compile Time Impact | Notes |
|---------|-------------------|-------|
| Generics (monomorphization) | HIGH | Each instantiation = more code |
| Generics (type erasure) | LOW | Box overhead at runtime |
| Complex type inference | MEDIUM | More work for type checker |
| Macros | MEDIUM-HIGH | Code generation at compile time |
| Whole-program optimization | HIGH | Can't do separate compilation |
| Modules with clear boundaries | POSITIVE | Enables parallel compilation |
| Forward declarations | POSITIVE | Two-pass not required |

**Generics decision matters**:
- Monomorphization (Rust) → Fast runtime, slow compile
- Type erasure (Go interface boxing) → Fast compile, some runtime cost
- **Hybrid**: Monomorphize hot paths, erase cold paths?

## C Interop Requirements

Backend work needs to call C libraries (SSL, compression, databases, etc.)

| Approach | Pros | Cons |
|----------|------|------|
| Same ABI as C (like Zig) | Direct calls, no overhead | Must match C ABI exactly |
| FFI layer (like Rust) | Safe boundary | Some overhead, verbosity |
| C compiled together | Seamless | Need C compiler in toolchain |

**LLVM advantage**: Same codegen as clang → perfect C ABI compatibility.
**Cranelift**: Also has good C ABI support.

## My Recommendation

### Phase 1: Prototype (Months 1-6)
**Use Cranelift exclusively**

- Fast compile times from day 1
- Good enough output for validation
- Rust ecosystem (we likely write compiler in Rust)
- Focus on language design, not optimization

### Phase 2: Production Ready (Months 6-12)
**Add LLVM backend for release builds**

```
our-lang dev   → Cranelift → Fast binary (debug)
our-lang build → LLVM → Optimized binary (release)
```

- Best of both worlds
- Cranelift for development iteration
- LLVM for production deployments
- Can tune the threshold (debug/release)

### Phase 3: Polish (Months 12+)
**Add JIT for scripting mode**

```
our-lang run script.ol  → Cranelift JIT → Execute
```

- Near-instant "compilation" for scripts
- REPL support
- Same code paths as AOT, just JIT'd

## Implementation Language

What do we write the compiler in?

| Option | Pros | Cons |
|--------|------|------|
| **Rust** | Cranelift is Rust, type safety, good perf | Learning curve, compile times |
| Zig | Fast compilation, simple | Smaller ecosystem |
| Go | Fast compilation, simple | GC (ironic), worse for compilers |
| C++ | Fast, LLVM is C++ | Memory safety issues |
| Our language | Dogfooding, cool | Need bootstrap path |

**Recommendation**: **Rust**
- Cranelift integration is natural
- Can use LLVM bindings when ready
- Type safety for compiler correctness
- Good tooling

**Bootstrap path**: Write compiler in Rust first. Eventually self-host (write compiler in our language).

## Comparison Summary

| Stack | Compile Speed | Output Speed | Complexity | C Interop | Rec |
|-------|--------------|--------------|------------|-----------|-----|
| LLVM only | Slow | Excellent | Medium | Excellent | ❌ |
| Cranelift only | Fast | Good | Low | Good | Dev only |
| Cranelift + LLVM | Fast dev, slow release | Best of both | Medium | Excellent | ✅ |
| Custom | Fast (eventually) | Variable | Very High | Our problem | ❌ |
| Go-style custom | Very fast | Good | High | Medium | Future? |

## Open Questions

1. **Generics strategy?** Monomorphization kills compile times. Do we:
   - Type erase by default, monomorphize on request?
   - Monomorphize only for primitive types?
   - Something else?

2. **How much optimization for debug builds?** Some optimization is actually cheap and helps.

3. **Incremental compilation granularity?** File-level enough? Function-level needed?

4. **Debug info format?** DWARF is standard but complex. Do we need full fidelity?

5. **Cross-compilation story?** How important? Affects toolchain complexity.

## Next Steps

1. [ ] Prototype parser and basic IR in Rust
2. [ ] Integrate Cranelift for simple code generation
3. [ ] Benchmark compile times vs Go for equivalent programs
4. [ ] Evaluate Cranelift output quality for backend workloads
5. [ ] Plan LLVM integration path

---

## Research Links

- [Cranelift Documentation](https://cranelift.dev/)
- [LLVM Language Reference](https://llvm.org/docs/LangRef.html)
- [How Go Compiles So Fast](https://www.reddit.com/r/golang/comments/2xmnvs/how_does_go_compile_so_quickly/)
- [Zig Compiler Internals](https://mitchellh.com/zig)
- [QBE Compiler Backend](https://c9x.me/compile/)
- [Rust Incremental Compilation](https://blog.rust-lang.org/2016/09/08/incremental.html)

---

*Last updated: 2024-01-17*
*Decision: PENDING - Prototype with Cranelift first*
