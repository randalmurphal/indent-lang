# Indent Language Design Decisions

**Date**: 2026-01-17
**Status**: Complete - 37 research investigations synthesized, ready for prototype
**Language Name**: Indent
**File Extension**: `.idt`

This document synthesizes findings from 37 deep research investigations across four rounds into a coherent design for the Indent programming language.

---

## Executive Summary

The research validates our core vision while sharpening specific technical choices:

| Area | Original Direction | Research Verdict | Confidence |
|------|-------------------|------------------|------------|
| Memory | Regions + ARC | **Validated** - Perceus/Lobster prove 95% RC elimination possible | High |
| Compilation | Cranelift + LLVM | **Validated** - 10x faster compile, 14% slower runtime acceptable | High |
| Concurrency | Structured + channels | **Validated** - colorless functions + explicit FFI blocking | High |
| Types | Bidirectional inference | **Validated** - HM unnecessary, TypeScript/Kotlin approach wins | High |
| Errors | Result + `?` operator | **Validated** - add anonymous sum types (`E1 \| E2`) | High |
| Syntax | Python-like | **Validated** - spaces only, 4-space indent, `and`/`or`/`not` | High |
| Modules | Go-style MVS | **Validated** - no lock files, three-level visibility | High |
| Metaprogramming | Minimal | **Validated** - built-in derives + comptime, NO AST macros | High |
| Interop | C FFI + Python embed | **Validated** - zero-overhead C, PyO3-style Python | High |
| Testing | Built-in framework | **Validated** - compiler-rewritten assert, fixtures | High |
| Security | Capability-based | **Validated** - three-layer model (compile/runtime/OS) | High |
| Scripting | Fast AOT + cache | **Validated** - <100ms cold, <10ms cached achievable | High |
| Stdlib | Targeted batteries | **Refined** - specific module priorities identified | High |
| Tooling | Single binary | **Validated** - Go model + Salsa-based LSP | High |
| Cloud-native | Built-in patterns | **Expanded** - more should be language primitives | High |
| Strings | UTF-8 + graphemes | **Validated** - grapheme-first len(), no integer indexing | High |
| Collections | Iterator protocol | **Validated** - Option-based, Swiss Tables, comprehensions | High |
| Async Runtime | Stackful coroutines | **Validated** - work-stealing, io_uring, 2KB stacks | High |
| Serialization | serde-style | **Validated** - format-agnostic, derive-based | High |
| Time/Date | java.time model | **Validated** - separate Instant/LocalDateTime, explicit TZ | High |
| Numerics | Checked by default | **Validated** - explicit wrapping ops, first-class SIMD | High |
| Observability | Built-in tracing | **Validated** - <2% overhead, compile-time elimination | High |
| Hot Reload | Late binding default | **Validated** - BEAM-style two-version coexistence | Medium |
| Formal Methods | Contracts + PBT | **Validated** - require/ensure blocks, property testing | High |
| Embedding | Three-tier plugins | **Validated** - Native/WASM/gRPC for different trust levels | High |
| Error Messages | Rust-style | **Validated** - source location, suggestions, color | High |
| Resource Mgmt | Explicit cleanup | **Validated** - Resource trait, depends_on ordering | High |

**Key insight**: The research reveals we can achieve more than originally hoped. Every major design decision has been validated by real-world implementations. The "one obvious way" philosophy is achievable across all language features.

---

## Memory Model: Final Design

### Research Findings

1. **Perceus (Koka)**: Eliminates ~95% of RC operations through compile-time analysis
2. **Lobster**: Similar results with ownership specialization
3. **Vale's generational references**: 10.84% overhead vs 25% for traditional RC
4. **Go's escape analysis**: Proves 80%+ of allocations can be stack-promoted
5. **Region inference (MLKit)**: Failed in practice - explicit regions work, inference doesn't

### Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Mental Model                         │
│  "Allocate freely, use regions for batches, Ref for sharing" │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Compiler Optimizations                       │
│  • Escape analysis → stack allocation                        │
│  • Lobster-style ownership analysis → 95% RC elimination     │
│  • ASAP destruction (Mojo) → deterministic, no drop flags    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Runtime Fallbacks                         │
│  • Optimized RC for remaining cases (~5%)                    │
│  • Explicit regions for request-scoped allocation            │
│  • Optional cycle collector for GC-managed types (rare)      │
└─────────────────────────────────────────────────────────────┘
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Default semantics | Value types, copy/move by size | Mojo validated this |
| Function arguments | Borrow by default | Reduces annotation burden 87% |
| Stored references | Require `Ref<T>` (explicit RC) | Clear ownership boundaries |
| Scopes | Explicit `scope { }` blocks | MLKit inference failed |
| Lifetimes | Second-class (cannot be stored) | Eliminates annotations entirely |
| Cycles | Optional `@gc` annotation for types | Backend code rarely has cycles |

### Expected Performance

- **~5% runtime overhead** from remaining RC operations
- **Zero overhead** for stack-allocated values (majority)
- **Region allocation**: Near-instant bulk free

---

## Compilation Stack: Final Design

### Research Findings

1. **LLVM consumes 99%+ of compile time** - measured at 115,500μs vs 800μs for frontend
2. **Cranelift is 10x faster** with only 14% slower output
3. **Rust's cg_clif**: 20-80% faster debug builds, production-ready in 2025
4. **Zig's self-hosted backend**: 275ms vs 918ms for hello world
5. **Go achieves speed through language design**, not compiler heroics

### Recommended Architecture

```
┌──────────────────┐     ┌──────────────────┐
│   Script Mode    │     │   Release Mode   │
│   indent run    │     │  indent build   │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         ▼                        ▼
┌──────────────────┐     ┌──────────────────┐
│    Cranelift     │     │      LLVM        │
│   Fast compile   │     │  Full optimize   │
│   Good output    │     │  Best output     │
└────────┬─────────┘     └────────┬─────────┘
         │                        │
         ▼                        ▼
    60-100ms cold            2-10s for
    <10ms cached            medium project
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Dev backend | Cranelift | 10x faster, good enough output |
| Release backend | LLVM -O2 | -O3 is 2x slower for 10-20% gain |
| IR design | Backend-agnostic MIR-level | Critical for dual-backend |
| Incremental | File-level + dependency caching | Go-style simplicity first |
| Linking | mold or custom minimal | Linker is 20-40% of time |

### Language Design for Fast Compilation

From Go's lessons, **enforce these in the language**:

1. **No cyclic imports** - enables parallel compilation
2. **Explicit public signatures** - don't infer across modules
3. **Export data in object files** - read dependencies once
4. **Unused imports are errors** - prevent dependency bloat

---

## Concurrency Model: Final Design

### Research Findings

1. **Go goroutines**: 2KB stack, 1-5μs spawn, simple but leak-prone
2. **Rust tokio**: 64 bytes/task, 10ns spawn, colored functions
3. **Kotlin structured concurrency**: Best of both with Job hierarchy
4. **FFI is the hard problem**: All green threads struggle with blocking C
5. **Discord's Go→Rust migration**: Eliminated 40ms GC latency spikes

### Recommended Architecture

**Colorless functions with structured scopes + explicit FFI blocking**

```
# Colorless: any function can be used concurrently
func fetch(url: str) -> Response:
    return http.get(url)  # May suspend, caller doesn't know

# Structured: all spawned work completes before scope exits
func fetch_all(urls: list[str]) -> list[Response]:
    concurrent:
        results = [spawn fetch(url) for url in urls]
    return results  # Guaranteed all complete

# FFI blocking: explicit annotation required
@blocking  # Runs on dedicated thread pool
func call_legacy_c_lib(data: bytes) -> bytes:
    return c_lib.process(data)
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Task model | M:N with work-stealing | Go's model but with smaller tasks |
| Task size | Target 64-256 bytes | Rust-level efficiency |
| Structured scope | `concurrent { }` block | Cannot leak by construction |
| Detached tasks | Explicit `detach { }` | Heavier syntax = discouraged |
| FFI blocking | `@blocking` annotation | Prevents scheduler starvation |
| Channels | Built-in, typed | Go's channels are excellent |
| Select | Built-in `select { }` | Essential for multiplexing |

### Key Insight: No Colored Functions

Research confirms colorless functions are achievable:
- Runtime handles suspension transparently (like Go's netpoller)
- Mark potential yield points in source for readability: `await` optional keyword
- Compiles to same code either way

---

## Type System: Final Design

### Research Findings

1. **HM is unnecessary** - TypeScript/Kotlin/Swift prove bidirectional is better
2. **Monomorphization kills compile time** - Burn.dev: 108s → 1s after fixing
3. **Swift witness tables**: Best balance of performance and compile time
4. **Structural typing at scale**: TypeScript shows 23% silent inference failures
5. **Control flow analysis** compensates for limited inference

### Recommended Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Type Checking                             │
│  • Bidirectional (not HM)                                    │
│  • Per-expression scope (Swift-style)                        │
│  • Explicit function signatures                              │
│  • Rich control flow narrowing                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Generics Strategy                         │
│  • Monomorphize: primitives, small value types               │
│  • Witness tables: reference types, complex generics         │
│  • @specialize annotation for hot paths                      │
└─────────────────────────────────────────────────────────────┘
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Inference scope | Bidirectional, per-expression | Fast + good errors |
| Function params | Always explicit | Enables parallelization |
| Return types | Inferred from body | Reduces annotation |
| Generics | Hybrid mono/witness | Balance perf/compile |
| Traits | Structural default, opt-in nominal | Go flexibility + safety |
| Null | `Option<T>`, no null | Billion dollar mistake |
| Pattern matching | Exhaustive required | Catch missing cases |

### Branded Types (From TypeScript Research)

Support newtype pattern for domain safety:
```
newtype UserId(str)
newtype OrderId(str)
# UserId and OrderId cannot be confused despite both being strings
```

---

## Error Handling: Final Design

### Research Findings

1. **`?` operator is zero-cost** - desugars to simple match, LLVM inlines completely
2. **Anonymous sum types solve Rust's biggest pain** - `E1 | E2` without naming
3. **Algebraic effects are premature** - unclear benefit for backend code
4. **Swift 6 typed throws validates direction** - industry converging on Result
5. **Context chaining beats backtraces** - 256ms for backtrace vs ~0 for context

### Recommended Syntax

```
# Function signature shows error types
func load_config(path: str) -> Config | IOError | ParseError:
    content = read_file(path).context("reading config")?
    return parse_json(content).context("parsing config")?

# Anonymous sum types compose automatically
func initialize() -> App | IOError | ParseError | DBError:
    config = load_config("app.json")?  # IOError | ParseError
    db = connect_db(config.db_url)?     # DBError
    return App(config, db)

# Pattern matching is exhaustive
match load_config(path):
    case Ok(c): use(c)
    case Err(IOError.NotFound): create_default()
    case Err(IOError.Permission(p)): fail(f"denied: {p}")
    case Err(ParseError.Syntax(s)): fail(f"invalid: {s}")
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Core type | `Result[T, E]` | Industry standard |
| Syntax sugar | `T \| E` for `Result[T, E]` | Concise signatures |
| Propagation | `?` operator | Rust proved it works |
| Error composition | Anonymous sum types | No naming every combo |
| Context | `.context(msg)` method | Cheap semantic info |
| Location | Compile-time tracking | Zero-cost, always on |
| Backtraces | Debug builds only | Too expensive otherwise |

---

## Syntax: Final Design

### Research Findings

1. **Cognitive load research validates Python**: Syntax choices directly impact comprehension speed
2. **Spaces-only indentation eliminates ambiguity**: Tab/space mixing causes 15% of Python-related SO questions
3. **"One obvious way" is achievable**: Go and Python prove that limiting choices improves ecosystem consistency
4. **Keyword operators aid readability**: `and`/`or`/`not` parse as words; `&&`/`||`/`!` require symbol-switching
5. **Expression-oriented control flow**: Returning values from `if`/`match` reduces temporary variables

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Indentation | 4 spaces, no tabs | Nim validated this - eliminates debate |
| Block syntax | Significant whitespace | Python readability, enforced consistency |
| Boolean operators | `and`, `or`, `not` | Cognitive research favors words |
| Comparison chaining | `a < b < c` | Python's most loved feature |
| Control flow | Expression-oriented | `x = if cond: a else: b` |
| String interpolation | `f"value: {x}"` | Python familiarity |
| Comments | `#` line, no block | One way to comment |
| Function syntax | `func name(args) -> Type:` | Explicit, searchable |

### Syntax Sample

```
func fetch_user(db: Database, id: UserId) -> User | NotFound | DBError:
    user = db.query_one("SELECT * FROM users WHERE id = ?", id)?

    if user.is_active:
        log.info(f"fetched active user {id}")
        return user
    else:
        return NotFound(f"user {id} inactive")

func process_batch(items: list[Item]) -> list[Result]:
    concurrent:
        results = [spawn process(item) for item in items]
    return results
```

### What We Explicitly Reject

- Multiple ways to define functions (no `fn`, `def`, `function`)
- Block comments (leads to commented-out code)
- Semicolons (noise)
- Braces for blocks (indentation is cleaner)
- Tabs (mixing causes bugs)
- `async`/`await` coloring (handled by runtime)

---

## Module System: Final Design

### Research Findings

1. **Go's MVS eliminates lock files**: Minimum Version Selection provides inherent reproducibility
2. **Transitive exports enable fast compilation**: Go's object files contain dependency type info
3. **Three visibility levels is the sweet spot**: private/internal/pub covers all real needs
4. **Semantic import versioning solves diamond dependency**: `/v2` in path = separate module
5. **URL-based imports are self-documenting**: No central registry required

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Version selection | MVS (Minimum Version) | Polynomial complexity, inherent reproducibility |
| Lock files | None required | `project.mod` alone determines versions |
| Visibility | private (default), `internal`, `pub` | Rust-inspired but simpler |
| Import paths | URL-based | Decentralized, self-documenting |
| Major versions | In path (`/v2`) | Different majors = different modules |
| Cyclic imports | Compile error | Enables parallel compilation |
| Unused imports | Compile error | Prevents dependency bloat |

### Module Structure

```
# project.mod
module github.com/user/myapp
version 1.0.0

require (
    github.com/lib/http v1.2.0
    github.com/lib/json v2.1.0
)

# File structure
myapp/
├── project.mod
├── main.idt
├── handlers/
│   ├── users.idt      # internal by default
│   └── api.idt        # pub items exported
└── internal/           # Never exported
    └── helpers.idt
```

### Visibility Model

```
# Default: private to current module
func helper():
    pass

# Visible within project, not to dependents
internal func shared_util():
    pass

# Public API
pub func handle_request(req: Request) -> Response:
    pass
```

---

## Metaprogramming: Final Design

### Research Findings

1. **Rust proc macros cost 11-40% compile time**: IPC overhead + no caching
2. **Zig comptime is elegant**: Same language, no AST manipulation, type reflection
3. **90% of derive needs are covered by 5 traits**: Serialize, Debug, Clone, Eq, Hash
4. **Go's `go generate` proves explicit generation works**: Committed, visible, debuggable
5. **Full AST macros enable DSL proliferation**: Counter to "one obvious way"

### Recommended Architecture: Three Layers

**Layer 1: Built-in Compiler Derivations (70% of use cases)**
```
@derive(Serialize, Debug, Clone, Eq, Hash)
struct User:
    id: UserId
    name: str
    email: str
```

Compiler generates optimal code with perfect error messages. No user-facing metaprogramming.

**Layer 2: Comptime Type Reflection (20% of use cases)**
```
func serialize[T](value: T, writer: Writer):
    comptime for field in type_fields(T):
        writer.write_field(field.name, @field(value, field.name))
```

Zig-style: types are values at compile time, field iteration, no AST manipulation.

**Layer 3: Explicit Code Generation (10% of use cases)**
```
# In project.mod or build file
generate:
    protoc --lang_out=. api.proto
    mockgen interfaces.idt
```

For schemas, mocks, FFI bindings. Generated files committed, verified in CI.

### What We Explicitly Reject

- Full AST macros (too complex, enables DSL proliferation)
- TokenStream manipulation (Rust's pain point)
- Custom syntax creation (counter to "one obvious way")
- Compile-time I/O (breaks hermetic builds)
- Method injection at compile time (breaks type locality)

---

## Interop: Final Design

### Research Findings

1. **Zero-overhead C FFI is achievable**: Zig proves ABI-compatible types work
2. **Compile-time header translation beats runtime**: Zig's Aro approach
3. **Ownership annotations solve safety**: `@owned`, `@borrowed` at boundaries
4. **PyO3 is the model for Python embedding**: Compile-time GIL tracking
5. **Free-threaded Python is coming**: Design for GIL-less future

### C Interop

```
# Zero-overhead: ABI-compatible types
@ffi("sqlite3.h")
extern func sqlite3_open(filename: *const c_char, db: **sqlite3) -> c_int

# Safe wrapper with ownership annotations
@ffi("mylib.h")
extern:
    @returns_owned
    func create_handle() -> *Handle

    @borrows(self)
    func get_data(h: *Handle) -> *const c_char

    @consumes(self)
    func destroy_handle(h: *Handle)
```

### Python Interop

```
# Embed Python with GIL tracking
func call_python_ml_model(data: Tensor) -> Tensor:
    with python.gil():
        # GIL held, can call Python
        model = python.import("my_model")
        return model.predict(data.to_numpy())

# Release GIL for parallel work
func parallel_native_work(items: list[Item]):
    python.release_gil:
        concurrent:
            [spawn process(item) for item in items]
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| C ABI | Built-in compatible types | Zero overhead |
| Header translation | Compile-time (Zig-style) | No runtime parsing |
| Ownership | Explicit annotations | Safety at boundaries |
| Python | PyO3-style GIL tracking | Compile-time safety |
| Error bridging | Thread-local storage | Standard pattern |
| Callbacks | Trampoline + user_data | Works with all C APIs |

---

## Testing: Final Design

### Research Findings

1. **Compiler-rewritten assert is powerful**: Show actual values, not just "assertion failed"
2. **pytest fixtures > xUnit setUp**: Dependency injection scales better
3. **Table-driven tests reduce boilerplate**: Go and Rust patterns
4. **Result-returning tests enable async**: `func test_x() -> Result[(), Error]`
5. **Same-package test access works**: Go's `_test.go` pattern

### Test Syntax

```
# Basic test with compiler-rewritten assert
#[test]
func test_user_creation():
    user = User.new("alice", "alice@example.com")
    assert user.name == "alice"  # On failure: "alice" == "bob" (shows values)
    assert user.is_valid()       # On failure: user.is_valid() was false

# Table-driven tests
#[cases(
    ("empty", "", false),
    ("valid", "test@example.com", true),
    ("no_at", "invalid", false),
)]
func test_email_validation(name: str, email: str, expected: bool):
    assert validate_email(email) == expected

# Fixtures via parameter injection
#[test]
func test_with_database(db: TestDatabase):
    # db automatically created, transaction rolled back after
    user = db.insert(User.new("test", "test@test.com"))
    assert db.find_user(user.id).is_some()

# Async/Result-returning tests
#[test]
func test_api_call() -> Result[(), HTTPError]:
    response = http.get("https://api.example.com/health")?
    assert response.status == 200
    return Ok(())
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Discovery | `#[test]` attribute | Explicit, searchable |
| Assertions | Compiler-rewritten | Rich failure messages |
| Table tests | `#[cases(...)]` | Reduces boilerplate |
| Fixtures | Parameter injection | pytest-style, composable |
| Test isolation | Same package access | Test private items |
| Async tests | Result return type | Natural error handling |

---

## Security: Final Design

### Research Findings

1. **Deno's capability model works**: Explicit permissions, fail-closed
2. **Compile-time verification catches most issues**: Static analysis is cheap
3. **OS-level sandboxing is essential**: seccomp, Landlock, namespaces
4. **`Secret<T>` prevents accidental leaks**: Type system enforcement
5. **Three-layer defense**: Compile-time + runtime + OS

### Three-Layer Security Model

**Layer 1: Compile-Time Capability Verification**
```
# Capabilities declared in module manifest
capabilities:
    net: [api.example.com, *.internal.corp]
    fs: [read: /config, write: /tmp]
    env: [DATABASE_URL, API_KEY]

# Compiler verifies all capability uses
func fetch_data() -> Data:
    # Compile error if api.example.com not in allowed hosts
    return http.get("https://api.example.com/data")
```

**Layer 2: Runtime Capability Enforcement**
```
# Runtime verifies dynamic values
func read_config(path: str) -> Config:
    # Runtime check: is path under /config?
    return fs.read(path)
```

**Layer 3: OS-Level Sandboxing**
```
# Process-level isolation
func run_untrusted(code: str) -> Result[Output, SandboxError]:
    sandbox:
        capabilities: none
        timeout: 5s
        memory: 100mb
    return eval(code)
```

### Secret Type

```
# Secret<T> cannot be logged, displayed, or serialized
func load_credentials() -> Secret[str]:
    return Secret(env.get("API_KEY"))

func make_request(key: Secret[str]):
    log.info(f"using key: {key}")  # Compile error: cannot display Secret
    http.header("Authorization", key.expose())  # Explicit unwrap required
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Default | Deny all | Fail-closed security |
| Verification | Compile-time first | Cheap, catches most issues |
| Runtime | Enforcement layer | For dynamic values |
| OS sandbox | seccomp + Landlock | Defense in depth |
| Secrets | `Secret<T>` type | Cannot accidentally leak |
| Permissions | Declared in manifest | Auditable, explicit |

---

## Tooling: Final Design

### Research Findings

1. **LSP speed from laziness, not incrementality** - skip code, don't recompute
2. **gofmt is fast because it doesn't reflow** - single pass, no backtracking
3. **Single binary wins for small teams** - Go model over Rust ecosystem
4. **Tree-sitter takes 4-8 weeks** for moderate complexity
5. **DWARF + lldb-dap = free debugging** - no custom DAP needed initially

### Recommended Architecture

```
indent                    # Single binary
├── build                  # Compile to binary
├── run                    # Compile and run (cached)
├── test                   # Run tests
├── fmt                    # Opinionated formatter
├── lint                   # Static analysis
├── check                  # Type check only
├── doc                    # Generate docs
├── repl                   # Interactive mode
└── lsp                    # Language server (may be separate binary)
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Distribution | Single binary | Go's biggest UX win |
| Formatter | gofmt-style, no config | One output, instant |
| LSP architecture | Salsa-based queries | rust-analyzer proved it |
| Package registry | Start decentralized (URL+hash) | Avoid npm security issues |
| Debugger | DWARF + lldb-dap | Free debugging |
| Tree-sitter | Budget 4-8 weeks | For editor highlighting |

### Timeline

| Phase | Months | Deliverable |
|-------|--------|-------------|
| MVP | 1-3 | build, run, fmt, basic test |
| Usable | 4-6 | LSP (basic), package mgmt, lint |
| Complete | 7-12 | Full LSP, debugger, coverage |

---

## Scripting Mode: Final Design

### Research Findings

1. **rust-script achieves <10ms cached** - just runs pre-compiled binary
2. **Go's 160ms floor is from no binary caching** - fixable
3. **Cranelift math works**: 160ms Go × 0.1 (Cranelift speedup) = 16ms codegen
4. **JIT is wrong for scripts** - warmup requires hundreds of iterations
5. **V8 snapshots show 20x speedup** for runtime initialization

### Target Performance

| Scenario | Target | How |
|----------|--------|-----|
| Cache hit | <10ms | Run pre-compiled binary |
| Cold (tiny) | <100ms | Cranelift + minimal linking |
| Cold (small) | <200ms | Parallel compilation |
| Cold (medium) | <500ms | Incremental caching kicks in |

### Implementation Checklist

```
Cache lookup:     <5ms   # mmap'd index, BLAKE3 hash
Runtime init:     <5ms   # Pre-serialized snapshot
Parsing:          <10ms  # Simple grammar, hand-written
Type checking:    <20ms  # Incremental, cached signatures
Cranelift codegen: <30ms # Parallel, minimal optimization
Linking:          <30ms  # mold or custom minimal
─────────────────────────
Total cold:       <100ms
```

---

## Standard Library: Final Design

### Research Findings

1. **Same packages everywhere**: HTTP, JSON, DB, logging, config
2. **Go's `net/http` flaw**: No default timeouts - critical bug
3. **Serde is 3-10x faster than Go's encoding/json** - compile-time wins
4. **Go's `sql.Null*` is painful** - design for `Option<T>` from day one
5. **slog is state of the art** for logging API design

### Day One Modules

| Module | Priority | Key Design Choice |
|--------|----------|-------------------|
| `http` | Critical | 30s default timeout, method+path routing |
| `json` | Critical | Compile-time derive, streaming |
| `sql` | Critical | Connection pooling, `Option<T>` nulls |
| `log` | Critical | slog-style Handler interface |
| `fs` | Critical | Standard file operations |
| `time` | Critical | Duration type, formatting |
| `env` | Critical | Typed config from environment |
| `cli` | High | Argument parsing |
| `test` | High | Built-in test framework |

### Explicit Non-Goals

- GUI, graphics, image processing
- Email, SMTP
- Machine learning
- Specific database drivers (ecosystem)

---

## Cloud-Native: Final Design

### Research Findings

1. **80% of k8s code is identical boilerplate** - language can eliminate
2. **Liveness probes checking external deps = cascading failures** - enforce at compile time
3. **Context propagation is the #1 pain point** - needs language support
4. **`Secret<T>` type that can't leak** - compiler must enforce
5. **Service mesh header propagation should be invisible**

### Language Primitives (Not Libraries)

| Primitive | Rationale |
|-----------|-----------|
| `Secret<T>` | Compiler prevents logging/display |
| `TraceContext` | Automatic propagation across async |
| Graceful shutdown | Default server behavior |
| Health endpoints | Auto-generated from resource state |

### Runtime Behaviors (Automatic)

| Behavior | Trigger |
|----------|---------|
| SIGTERM handling | Always, 30s default drain |
| `/health/live` endpoint | Always (static 200) |
| `/health/ready` endpoint | Based on declared resources |
| Trace header propagation | All HTTP clients |

### Example Server

```
service API:
    shutdown_timeout: 30s

    resource db = Postgres.connect(env.DATABASE_URL)
    resource cache = Redis.connect(env.REDIS_URL)

    # Health automatically reflects resource state
    # Graceful shutdown closes in reverse order
    # Trace context propagates automatically

    route GET "/users/{id}" -> User | NotFound:
        return db.query_one("SELECT * FROM users WHERE id = ?", id)?
```

---

## Strings & Unicode: Final Design

### Research Findings

1. **UTF-8 wins universally** - All modern languages (Rust, Go, Swift) use UTF-8 internally
2. **Grapheme clusters are what users expect** - `len("é")` should be 1, not 2
3. **Integer indexing is harmful** - O(n) for UTF-8, encourages bugs
4. **Owned vs borrowed is essential** - `String` (heap) vs `str` (view)
5. **Linear-time regex prevents ReDoS** - RE2 semantics for safety

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Internal encoding | UTF-8 always | Memory efficiency, web compatibility |
| `len()` semantics | Grapheme cluster count | Matches user intuition |
| Indexing | Iterator-based only | Forbid `s[i]`, use `s.chars()`, `s.graphemes()` |
| String types | `String` (owned) + `str` (borrowed) | Clear ownership |
| Interpolation | `f"Hello {name}"` | Python familiarity |
| Regex engine | RE2 semantics (linear time) | ReDoS prevention |
| Normalization | Explicit `.normalize(NFC)` | No silent changes |
| Case operations | Locale-aware `.to_upper(locale)` | Turkish i problem |

### String API

```
val s: String = "Hello, 世界! 👨‍👩‍👧‍👦"

s.len()           # 4 graphemes (Hello + comma + space + world + ! + space + family)
s.byte_len()      # Byte count (for allocation)
s.chars()         # Iterator over Unicode scalars
s.graphemes()     # Iterator over grapheme clusters
s.lines()         # Iterator over lines
s.split(sep)      # Split by separator
f"Value: {x:03d}" # Formatted interpolation
```

---

## Collections & Iterators: Final Design

### Research Findings

1. **Option-based iteration beats exceptions** - `next() -> Option[T]` vs StopIteration
2. **Lazy adapters compile to tight loops** - Rust iterators match hand-written code
3. **Swiss Tables are fastest** - Google's hashmap: 2-3x faster than std::unordered_map
4. **1.5x growth factor is optimal** - Better cache behavior than 2x
5. **Comprehensions + methods both valuable** - Don't force choice

### Core Collection Types

| Type | Implementation | Notes |
|------|----------------|-------|
| `List[T]` | Dynamic array, 1.5x growth | Small-buffer optimization |
| `Dict[K, V]` | Swiss Table (flat, SIMD probing) | Insertion-ordered by default |
| `Set[T]` | Swiss Table variant | Hash-based |
| `Deque[T]` | Ring buffer | O(1) both ends |
| `Sorted[T]` | B-tree | Ordered iteration |

### Iterator Protocol

```
interface Iterator[T]:
    func next(var self) -> Option[T]
    func size_hint() -> (int, Option[int])

# Three iteration modes
for item in collection:           # Borrows
for var item in collection.iter_mut():  # Mutable borrow
for item in collection.into_iter():     # Consumes
```

### Comprehension Syntax

```
# List comprehension (eager, allocates)
squares = [x * x for x in range(10)]

# Generator expression (lazy)
squares = (x * x for x in range(10))

# Dict comprehension
lookup = {item.id: item for item in items}

# With filter
evens = [x for x in nums if x % 2 == 0]
```

---

## Extended Syntax: Final Design

### Lambda Syntax

```
# Short form
items.map(x -> x * 2)

# Multi-parameter
items.reduce((a, b) -> a + b)

# With type annotation
items.map((x: int) -> x.to_string())

# Block form for complex lambdas
items.map(x ->
    val processed = transform(x)
    processed.finalize()
)
```

### Pattern Matching

```
match value:
    case 0:
        "zero"
    case n if n < 0:
        "negative"
    case Point(x, 0):
        f"on x-axis at {x}"
    case Point(x, y) as p if x == y:
        f"diagonal point {p}"
    case [first, ...rest]:
        f"list starting with {first}"
    case {"key": value}:
        f"dict with key={value}"
    case _:
        "anything else"
```

### Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `\|>` | Pipeline | `data \|> transform \|> format` |
| `??` | Null coalesce | `x ?? default` |
| `?.` | Optional chain | `user?.address?.city` |
| `//` | Integer division | `7 // 3 == 2` |
| `**` | Exponentiation | `2 ** 10 == 1024` |
| `in` | Containment | `x in collection` |
| `not in` | Negated containment | `x not in collection` |
| `is` | Identity | `a is b` |
| `is not` | Negated identity | `a is not b` |
| `++` / `--` | Increment/Decrement | `i++` (statement only, Go-style) |

### Collection Operators

Python-style intuitive collection operations:

```
# Concatenation
[1, 2] + [3, 4]         # [1, 2, 3, 4]
"hello" + " world"      # "hello world"

# Repetition
[0] * 5                 # [0, 0, 0, 0, 0]
"-" * 20                # "--------------------"

# Set operations
a | b                   # Union
a & b                   # Intersection
a - b                   # Difference
a ^ b                   # Symmetric difference
```

### Default Arguments & Variadics

```
func greet(name: str, greeting: str = "Hello") -> str:
    return f"{greeting}, {name}!"

func sum(...numbers: int) -> int:
    return numbers.reduce((a, b) -> a + b, 0)

func configure(**options: Value):
    for key, value in options:
        set_option(key, value)
```

---

## Async Runtime Architecture: Final Design

### Research Findings

1. **Stackful coroutines enable colorless async** - No function coloring
2. **Work-stealing is optimal for CPU-bound** - Tokio, Go, Rayon all use it
3. **io_uring is 40% faster than epoll** - But needs fallback for older kernels
4. **2KB stacks are sufficient** - Growable to 1GB when needed
5. **SIGURG enables safe preemption** - Go 1.14+ approach

### Runtime Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Code                                 │
│  • Colorless: all functions can suspend                     │
│  • @blocking marks C FFI that may block                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Scheduler                                 │
│  • Work-stealing with per-processor queues                  │
│  • LIFO slot for hot task cache                             │
│  • Hierarchical timing wheels for timers                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    I/O Backend                               │
│  • io_uring primary (Linux 5.1+)                            │
│  • epoll fallback (older Linux)                             │
│  • kqueue (macOS/BSD)                                       │
└─────────────────────────────────────────────────────────────┘
```

### Specific Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Coroutine model | Stackful, 2KB initial | Colorless async |
| Stack growth | Segmented, up to 1GB | Handle deep recursion |
| Scheduler | Work-stealing, N:M | Best for mixed workloads |
| I/O primary | io_uring | 40% faster than epoll |
| Timer implementation | Hierarchical timing wheels | O(1) insertion |
| Preemption | SIGURG-based (Go style) | Safe points at function entry |
| Blocking FFI | `@blocking` annotation | Dedicated thread pool |

---

## Serialization: Final Design

### Research Findings

1. **Serde's format-agnostic design is superior** - Separate type from format
2. **JSON + MessagePack cover 90% of needs** - Built-in, no schema required
3. **Protocol Buffers for gRPC** - Optional support for strict schemas
4. **Zero-copy is specialized** - FlatBuffers/rkyv for memory-mapped files
5. **Security limits are non-negotiable** - Max depth, size, string length

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Types                                │
│  @derive(Serialize, Deserialize)                            │
│  struct User { id: int, name: str, email: str }            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Model                                │
│  ~25 types: primitives, strings, sequences, maps, etc.     │
│  Format-independent representation                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Format Implementations                    │
│  JSON, MessagePack (built-in)                               │
│  Protobuf, CBOR, TOML (optional)                            │
└─────────────────────────────────────────────────────────────┘
```

### Security Defaults

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| Max depth | 64 | Prevent stack overflow |
| Max string | 20 MB | Prevent allocation bombs |
| Max payload | 10 MB | Match gRPC default |
| Max array elements | 100,000 | Application-dependent |

---

## Time & Date: Final Design

### Research Findings

1. **Machine time ≠ human time** - Instant vs LocalDateTime must be separate
2. **Duration ≠ Period** - 24 hours ≠ "1 day" during DST
3. **java.time is the gold standard** - Adopt its type hierarchy
4. **Explicit timezone required** - Converting local→instant needs zone
5. **DST gaps/overlaps need explicit resolution** - No silent defaults

### Type Hierarchy

| Type | Purpose | Example |
|------|---------|---------|
| `Instant` | UTC nanoseconds since epoch | Timestamps, logging |
| `LocalDate` | Calendar date, no timezone | Birthdays |
| `LocalTime` | Wall-clock time, no date | "9:00 AM" |
| `LocalDateTime` | Date + time, no timezone | Recurring events |
| `ZonedDateTime` | Full datetime with timezone | Meeting scheduling |
| `Duration` | Seconds + nanoseconds | Elapsed time |
| `Period` | Years + months + days | Calendar intervals |
| `TimeZone` | IANA identifier | "America/New_York" |

### Clock Abstraction (for testing)

```
interface Clock:
    func now() -> Instant
    func zone() -> TimeZone

# Implementations
SystemClock       # Production
FixedClock        # Testing: frozen time
SimulatedClock    # Testing: controllable advancement

# Testable code
func is_overdue(deadline: Instant, clock: Clock = SystemClock) -> bool:
    Instant.now(clock).is_after(deadline)
```

---

## Numeric Computing: Final Design

### Research Findings

1. **Checked arithmetic by default** - Catches bugs, CVE prevention
2. **IEEE 754 strict by default** - SSE2 minimum, per-function fast-math opt-in
3. **First-class SIMD vectors** - Zig-style `Vector(N, T)` with operators
4. **Explicit conversions only** - No silent truncation
5. **Decimal types for money** - Binary floats can't represent 0.1 exactly

### Overflow Handling

| Operation | Syntax | Behavior |
|-----------|--------|----------|
| Default | `a + b` | Checked (panic on overflow) |
| Wrapping | `a +% b` | Two's complement modular |
| Saturating | `a +\| b` | Clamps to MIN/MAX |
| Checked | `a.checked_add(b)` | Returns `Option[T]` |

### SIMD Support

```
val a: Vector(4, f32) = [1.0, 2.0, 3.0, 4.0]
val b: Vector(4, f32) = [5.0, 6.0, 7.0, 8.0]
val c = a + b  # Element-wise addition

# Built-in operations
splat(value)      # Broadcast to all lanes
reduce(op, vec)   # Horizontal operation
shuffle(a, b, mask)  # Permute elements
```

---

## Resource Management: Final Design

### Research Findings

1. **Close should return Result** - Unlike Rust's Drop, cleanup can fail
2. **Async cleanup needs explicit support** - Separate AsyncResource trait
3. **Cleanup order matters** - Reverse allocation order, explicit dependencies
4. **Pool patterns are common** - Language support for connection pooling
5. **Leak detection enables debugging** - Runtime tracking with stack traces

### Resource Traits

```
interface Resource:
    func close(var self) -> Result[(), CloseError]

interface AsyncResource: Resource:
    async func async_close(var self) -> Result[(), CloseError]
```

### Region-Scoped Cleanup

```
scope request:
    val conn = db_pool.get()      # Cleaned up third
    val file = File.open("log")   # Cleaned up second
    val lock = mutex.lock()       # Cleaned up first
# Scope exit triggers cleanup cascade in reverse order
```

### Explicit Dependency Ordering

```
val pool = ConnectionPool.new()
val conn = pool.get() depends_on pool  # Compiler ensures conn closes before pool
```

### Connection Pooling

```
# Built-in pool semantics
struct Pool[T: Resource]:
    func get() -> PoolGuard[T]
    async func async_get() -> PoolGuard[T]
    func shutdown(timeout: Duration)

# Pool returns to pool, not closed
with conn = pool.get():
    conn.query(...)
# conn returns to pool here
```

---

## Observability: Final Design

### Research Findings

1. **Compiler injection eliminates boilerplate** - Automatic span creation
2. **Structured logging is 60x faster** - Parameterized vs string interpolation
3. **W3C Trace Context is standard** - Use for cross-service propagation
4. **<2% overhead is achievable** - Compile-time elimination when disabled
5. **Type-safe metrics prevent cardinality explosion** - Enum labels

### Automatic Tracing

```
@trace(level: Info, fields: {user_id: ctx.user_id})
func process_order(ctx: Context, order: Order) -> Result[Receipt, OrderError]:
    # Compiler injects span lifecycle, argument recording, error capture
    validate_order(order)?
    charge_payment(ctx, order.total)?
    Ok(generate_receipt(order))
```

### Type-Safe Metrics

```
@derive(LabelSet)
struct HttpLabels:
    method: HttpMethod     # Enum: GET, POST, PUT, DELETE
    status: StatusClass    # Enum: 2xx, 3xx, 4xx, 5xx
    path: str

val requests = Counter[HttpLabels].new(
    name: "http_requests_total",
    help: "Total HTTP requests"
)

requests.with(HttpLabels { method: GET, status: Success2xx, path: "/api" }).inc()
```

### Zero-Cost When Disabled

```
# Build with: indent build --max-log-level=info
# Result: all trace!() and debug!() calls removed from binary entirely
```

---

## Error Messages: Final Design

### Research Findings

1. **Source location + code snippet essential** - Point to exact problem
2. **Suggestions dramatically help** - "Did you mean X?"
3. **Error codes enable documentation** - `E0001`, `indent explain E0001`
4. **Colors improve scanning** - But support `--no-color`
5. **Progressive disclosure** - Main message first, details available

### Error Format

```
error[E0308]: type mismatch
  --> src/main.idt:15:12
   |
15 |     val x: int = "hello"
   |            ---   ^^^^^^^ expected int, found str
   |            |
   |            expected due to this type annotation
   |
help: consider using parse
   |
15 |     val x: int = "hello".parse()?
   |                         ++++++++
```

### Error Code System

| Range | Category |
|-------|----------|
| E0000-E0099 | Syntax errors |
| E0100-E0199 | Type errors |
| E0200-E0299 | Lifetime/borrow errors |
| E0300-E0399 | Pattern matching |
| E0400-E0499 | Module system |
| W0000-W0099 | Warnings |

---

## Hot Reload: Final Design

### Research Findings

1. **Late binding is non-negotiable** - Function calls via runtime registry
2. **State and behavior must be separable** - Explicit reload-stable state
3. **Two-version limit is a feature** - Forces migration completion
4. **Closures are the hardest problem** - Document stale reference behavior
5. **Actor model provides best foundation** - Process isolation + message passing

### Phase 1 (v1.0) Foundation

- Late-bound function calls through runtime registry
- Two-version module coexistence (current/old)
- `@stable` annotation for reload-surviving state
- Module lifecycle hooks: `before_reload`, `after_reload`
- State migration callback: `migrate_state(old_state, old_version) -> new_state`

### Development Experience

```
# File watcher with dependency analysis
indent dev --watch

# Validation before load - reject code with errors, keep old version
# Error overlay without breaking application state
```

---

## Formal Methods: Final Design

### Research Findings

1. **Contracts have 30 years of proven value** - Eiffel's Design by Contract
2. **Property-based testing succeeds where others fail** - Highest adoption
3. **Refinement types are premature** - SMT solver complexity, poor errors
4. **Typestate provides zero-cost safety** - Compile-time state tracking
5. **Built-in fuzzing catches 13,000+ bugs** - OSS-Fuzz demonstrates value

### Contracts

```
func deposit(account: Account, amount: Money) -> None:
    require:
        amount >= 0, "amount must be non-negative"
        account.is_open, "account must be open"
    ensure:
        account.balance == old(account.balance) + amount

    account._balance += amount
```

### Property-Based Testing

```
@property_test(iterations: 1000)
func test_sort_preserves_length(xs: list[int]):
    assert len(sort(xs)) == len(xs)

@property_test
func test_json_roundtrip(data: Json from json_generator()):
    assert Json.parse(data.to_string()) == data
```

### Typestate for Resources

```
state Connection:
    Disconnected
    Connected { socket: Socket }
    Closed

implement Connection[Disconnected]:
    func connect(self, addr: Address) -> Connection[Connected]:
        ...

implement Connection[Connected]:
    func send(self, data: bytes) -> None:
        ...
    func close(self) -> Connection[Closed]:
        ...

# Compile error: send() not available on Disconnected
val conn = Connection.Disconnected()
conn.send(b"hello")  # Error!
```

### Priority

| Priority | Feature |
|----------|---------|
| P0 | Runtime contracts (require/ensure) |
| P0 | Property-based testing |
| P0 | Null safety (Option types) |
| P1 | Built-in fuzzing |
| P1 | Class/struct invariants |
| P1 | Typestate for resources |
| P2 | Loop invariants |
| Future | Refinement types |

---

## Embedding & Plugins: Final Design

### Research Findings

1. **Region model is embedding advantage** - Predictable, no GC pauses
2. **Three tiers serve different needs** - Native/WASM/gRPC
3. **Capability types enable sandboxing** - Built into type system
4. **<500KB binary, <10ms startup achievable** - Match Lua's embeddability
5. **Config mode competes with CUE/Dhall** - Termination checking, hash imports

### Three-Tier Plugin Architecture

| Tier | Mechanism | Trust Level | Use Case |
|------|-----------|-------------|----------|
| 1 | Native C ABI | Full trust | Database drivers, performance-critical |
| 2 | WASM sandbox | Untrusted | User scripts, marketplace plugins |
| 3 | gRPC subprocess | Isolated | Cloud integrations, crash-tolerant |

### Capability Types

```
func read_config(path: Path) -> Config requires fs:read(path.parent):
    file = open(path)?
    return parse(file.read_all()?)

# Compile-time verification of capability requirements
```

### Embedding API

```c
// Handle-based with arena support
indent_arena_t* arena = indent_arena_new(ctx, 64 * 1024);
indent_value_t result = indent_call(ctx, func, args);
indent_arena_free(arena);  // Single deallocation
```

---

## Production Failure Prevention: Final Design

### Research Findings

1. **70% of severe bugs are memory safety** - Google, Microsoft data
2. **Unbounded queues cause cascading failures** - Every queue needs max size
3. **Missing timeouts cause resource exhaustion** - Infinite is never safe default
4. **Container awareness is mandatory** - Read cgroup limits, not host
5. **Compile-time prevention >> runtime detection** - Go race detector misses most

### Compile-Time Prevention (Highest Priority)

- Ownership system for memory safety and deterministic cleanup
- Non-nullable types by default with explicit `Option[T]`
- `Result[T, E]` with must_use - compiler-enforced error handling
- Send/Sync traits for compile-time thread safety
- Bounded channels and collections by default

### Runtime Defaults

| Behavior | Default | Rationale |
|----------|---------|-----------|
| Container awareness | Auto-detect cgroup limits | Configure GC/threads correctly |
| SIGTERM handling | 15-second drain | Graceful shutdown |
| HTTP timeout | 30 seconds | Never infinite |
| Health endpoints | Auto `/healthz`, `/readyz` | Kubernetes compatibility |
| OOM handling | Heap dump on OOM | Post-mortem debugging |

### Patterns to Make Hard

```
# These require explicit annotation:
@unbounded  # Unbounded queue/collection
@no_timeout # Missing network timeout
@fire_and_forget  # Untracked background work
@ignore_error  # Discarding error result
```

---

## Remaining Open Questions

### High Priority (Decide Before Prototype)

All high-priority questions have been resolved. See "Resolved" section below.

### Medium Priority (Decide During Prototype)

1. **Package registry** - When community needs one (URL-based imports work initially)
2. **IDE plugins beyond VS Code** - Community can help
3. **Windows support** - Linux/macOS first

### Low Priority (Decide Later)

1. **Custom allocators** - API design for custom memory allocation
2. **Async generators** - Syntax for async iteration
3. **Effect system** - Beyond Result, for cancellation/context

### Resolved

| Question | Resolution |
|----------|------------|
| Language name | **Indent** with `.idt` extension |
| Syntax style | Python-like: significant whitespace, 4 spaces, `and`/`or`/`not` |
| Scope syntax | `scope { }` blocks for memory scopes |
| Concurrency syntax | `concurrent { }` for structured, `spawn` for tasks |
| Standard library async | Colorless - runtime handles suspension transparently |
| Testing framework | Built-in with `#[test]`, compiler-rewritten assert |
| Security model | Three-layer: compile-time caps, runtime enforcement, OS sandbox |
| Metaprogramming | Built-in derives + comptime reflection, NO AST macros |
| Lambda syntax | `x -> expr` with optional parens for multiple params |
| String semantics | UTF-8 internal, grapheme-first `len()`, no integer indexing |
| Iterator protocol | Option-based `next()`, three modes (borrow, mut borrow, consume) |
| Comprehensions | Python-style `[x for x in xs if cond]` alongside method chains |
| Integer overflow | Checked by default, explicit wrapping/saturating operators |
| Time types | java.time model with Instant/LocalDateTime separation |
| Serialization | serde-style format-agnostic with Serialize/Deserialize derives |
| Observability | Built-in tracing with `@trace` attribute, <2% overhead |
| Resource cleanup | Resource trait with Result-returning `close()` |
| Formal methods | Contracts (require/ensure) + property-based testing |
| Hot reload | Late binding by default, two-version module coexistence |
| Error messages | Rust-style with source snippets, suggestions, error codes |
| Mutable vs immutable | Mutable by default (like Python/Go), but function params borrow immutably |
| Associated types | Supported with simple syntax: `type Item` in trait definitions |
| Method call syntax | `obj.method()` only, no UFCS ("one obvious way") |
| Operator overloading | Limited: standard operators via traits (Add, Sub, etc.), no custom operators |
| Increment/Decrement | `++`/`--` as statements only (Go-style), not expressions |
| Collection operators | Python-style: `+` (concat), `*` (repeat), `in`, set ops (`\|`, `&`, `-`, `^`) |

---

## Prototype Roadmap

### Phase 1: Validate Core (Months 1-3)

**Goal**: Prove the memory model and compilation stack work together

- [ ] Parser for minimal syntax (functions, types, expressions)
- [ ] Type checker with bidirectional inference
- [ ] Cranelift code generation
- [ ] Basic memory model (stack, simple RC)
- [ ] Hello world compiles in <100ms

### Phase 2: Language Usable (Months 4-6)

**Goal**: Write real programs

- [ ] Full type system (generics, traits)
- [ ] Error handling (`Result`, `?`)
- [ ] Structured concurrency
- [ ] Basic stdlib (io, strings)
- [ ] Formatter

### Phase 3: Production Path (Months 7-12)

**Goal**: Deployable applications

- [ ] LLVM backend for release builds
- [ ] HTTP module with cloud-native defaults
- [ ] Database connectivity
- [ ] LSP server
- [ ] Package management

---

## Confidence Assessment

| Decision | Confidence | Risk if Wrong |
|----------|------------|---------------|
| Cranelift for dev builds | Very High | Low - can adjust thresholds |
| Bidirectional type inference | Very High | Low - well-proven approach |
| Python-like syntax | Very High | Low - cognitive research validates |
| Go's MVS for packages | Very High | Low - proven at scale |
| UTF-8 strings + grapheme len() | Very High | Low - modern consensus |
| Option-based iterators | Very High | Low - Rust proves superiority |
| Checked integer arithmetic | Very High | Low - catches real CVEs |
| java.time model for dates | Very High | Low - industry gold standard |
| Serde-style serialization | Very High | Low - proven at scale |
| Rust-style error messages | Very High | Low - universally praised |
| Structured concurrency | High | Medium - may need escape hatches |
| RC with 95% elimination | High | Medium - needs prototype validation |
| No colored functions | High | Low - Go proves it works |
| Anonymous sum types for errors | High | Low - just syntax sugar |
| Built-in derives + comptime | High | Low - Zig proves this works |
| Three-layer security | High | Low - Deno validates approach |
| Compiler-rewritten assert | High | Low - pytest/Zig prove value |
| Zero-overhead C FFI | High | Low - Zig proves achievable |
| Built-in observability | High | Low - <2% overhead proven |
| Resource trait with Result | High | Low - addresses Drop limitations |
| Contracts (require/ensure) | High | Low - 30 years of Eiffel experience |
| Property-based testing | High | Low - highest adoption success |
| Three-tier plugin system | High | Low - addresses different trust levels |
| Region-based allocation | Medium | Medium - interface needs iteration |
| Cloud-native primitives | Medium | Low - can start as runtime, promote later |
| Hot reload foundation | Medium | Medium - requires careful runtime design |
| Typestate for resources | Medium | Medium - ergonomics need iteration |

---

## Language Identity

**Name**: Indent
**Extension**: `.idt`
**Command**: `indent`

**Philosophy**: "One obvious way" - Go's simplicity, Python's readability, Rust's performance
**Target**: Backend developers, DevOps engineers, server-side applications
**Key differentiators**:
- Reads like pseudocode (Python syntax) but compiles to native (Rust speed)
- No GC, no manual memory management (automatic RC with 95% elimination)
- Colorless concurrency (no async/await virus)
- Cloud-native by default (graceful shutdown, health checks, tracing built-in)
- Security as a first-class concern (capabilities, Secret<T>, sandboxing)
- Sub-100ms script execution (bash replacement with type safety)

**Vibe**: Professional, pragmatic, boring-in-a-good-way, productive, safe

**Why "Indent"**: The name directly references significant whitespace - the most visible syntactic feature. It's a common English word, easy to say, slightly self-aware without being clever. Like Python and Rust, it's a real word that becomes associated with the language over time.

---

## Research Foundation

This design is informed by 37 deep research investigations across four rounds:

**Round 1 - Core Mechanics (9 papers)**: Memory model, compilation stack, concurrency, types, errors, stdlib, tooling, scripting, cloud-native

**Round 2 - Language Features (6 papers)**: Syntax, modules, metaprogramming, interop, testing, security

**Round 3 - Ecosystem Gaps (10 papers)**: Error messages, strings/unicode, collections/iterators, extended syntax, async runtime, database access, configuration/build, documentation, profiling, testing patterns

**Round 4 - Strategic (12 papers)**: Adoption strategy, production failures, AI code generation, hot reload, numerics, time/date, serialization, resource management, observability, developer experience, embedding/plugins, formal methods

All research papers are available in `/research/` organized by category.

---

*This synthesis represents the culmination of 37 deep research investigations across all aspects of language design. Every major design decision has been validated by real-world implementations. The "one obvious way" philosophy is achievable across all features. The path forward is clear - time to build Indent.*
