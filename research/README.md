# Indent Language Research

This directory contains all deep research investigations that informed the Indent language design.

## Structure

### 01-core/ - Foundation Research
Core language mechanics that everything else depends on.

| File | Topic | Key Decisions |
|------|-------|---------------|
| memory.md | Memory Model | Regions + ARC, Perceus-style 95% RC elimination |
| compiler.md | Compilation Stack | Cranelift (dev) + LLVM (release) |
| concurrency.md | Concurrency Model | Structured concurrency, colorless functions |
| error_handling.md | Error Handling | Result + `?` operator, anonymous sum types |
| type.md | Type System | Bidirectional inference, hybrid mono/witness |

### 02-language/ - Language Features
Specific language features and their design.

| File | Topic | Key Decisions |
|------|-------|---------------|
| syntax.md | Syntax Design | Python-like, 4 spaces, `and`/`or`/`not` |
| modules.md | Module System | Go's MVS, three-level visibility |
| metaprogramming.md | Metaprogramming | Built-in derives + comptime, NO macros |
| interop.md | FFI & Interop | Zero-overhead C, PyO3-style Python |
| testing.md | Testing Framework | `#[test]`, compiler-rewritten assert |
| security.md | Security Model | Three-layer: compile/runtime/OS |

### 03-ecosystem/ - Tooling & Ecosystem
Standard library, tooling, and ecosystem concerns.

| File | Topic | Key Decisions |
|------|-------|---------------|
| stdlib.md | Standard Library | HTTP, JSON, SQL, logging priorities |
| tooling.md | Developer Tooling | Single binary, Salsa-based LSP |
| scripting_mode.md | Scripting Mode | <100ms cold, <10ms cached |
| cloud-native.md | Cloud Native | Built-in health, tracing, graceful shutdown |
| async_runtime.md | Async Runtime | Work-stealing, io_uring, task design |
| collections_iterator.md | Collections & Iterators | Iterator protocol, comprehensions |
| compiler_error_msg.md | Error Messages | Rust-style diagnostics |
| configuration_and_build.md | Config & Build | Env handling, build profiles |
| db_access.md | Database Access | Connection pooling, query patterns |
| documentation.md | Documentation System | Doc comments, doc tests |
| profiling_performance.md | Profiling | pprof compatibility, benchmarks |
| str_unicode.md | Strings & Unicode | UTF-8, grapheme handling |
| syntax_design.md | Extended Syntax | Lambda, kwargs, comprehensions |
| testing_patterns.md | Extended Testing | Mocking, coverage, isolation |

### 04-strategic/ - Strategic & Cross-Cutting
Non-obvious research areas that affect language success.

| File | Topic | Key Insights |
|------|-------|--------------|
| adoption_strategy.md | Language Adoption | Why languages succeed/fail, ecosystem bootstrap |
| production_failures.md | Failure Modes | What crashes production, how to prevent |
| ai_code_generation.md | AI/LLM Optimization | Syntax patterns for AI tools |
| hot_reload.md | Hot Code Reload | Erlang-style live development feasibility |
| numeric_computing.md | Numeric Computing | Integer overflow, SIMD, floating point |
| time_date.md | Time & Date | Temporal types, timezone handling |
| serialization.md | Serialization | Wire formats, schema evolution |
| resource_management.md | Resource Management | Beyond memory: handles, connections |
| observability.md | Observability | Tracing, logging, metrics as primitives |
| developer_experience.md | Developer Psychology | Cognitive load, error messages, workflow |
| embedding_plugins.md | Embedding & Plugins | WASM plugins, embedding API |
| formal_methods.md | Formal Methods Lite | Contracts, property testing, refinements |

## Research Rounds

1. **Round 1** (9 papers): Core mechanics - memory, compiler, concurrency, types, errors, stdlib, tooling, scripting, cloud-native
2. **Round 2** (6 papers): Language features - syntax, modules, metaprogramming, interop, testing, security
3. **Round 3** (10 papers): Gaps - error messages, strings, collections, syntax details, async runtime, database, config, docs, profiling, testing patterns
4. **Round 4** (12 papers): Strategic - adoption, failures, AI, hot reload, numerics, time, serialization, resources, observability, DX, embedding, formal methods

## Total: 37 Research Papers

All findings are synthesized in [docs/SYNTHESIS.md](../docs/SYNTHESIS.md).
