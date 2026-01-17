# Indent

A new programming language designed for backend developers, server engineers, and DevOps.

**File Extension**: `.idt`
**Command**: `indent`

## Vision Statement

**"Python's readability, Go's simplicity, Rust's performance - without the papercuts of any."**

Create a language that:
- Reads like pseudocode
- Compiles fast (Go-level)
- Runs fast (near Rust-level)
- Handles memory automatically (no manual management in 95% of code)
- Makes the right thing easy and the wrong thing hard
- Replaces bash scripts with something maintainable
- Is cloud-native from the ground up

## Example

```
fn fetch_user(db: Database, id: UserId) -> User | NotFound | DBError:
    user = db.query_one("SELECT * FROM users WHERE id = ?", id)?

    if user.is_active:
        log.info(f"fetched active user {id}")
        return user
    else:
        return NotFound(f"user {id} inactive")

fn process_batch(items: list[Item]) -> list[Result]:
    concurrent:
        results = [spawn process(item) for item in items]
    return results
```

## Target Users

| User Type | Pain Points We Solve |
|-----------|---------------------|
| Backend developers | Go verbosity, Python performance, Rust complexity |
| DevOps engineers | Bash unmaintainability, Python deployment headaches |
| Server engineers | Memory bugs, concurrency footguns, deployment complexity |
| Python devs wanting perf | Keep readability, gain 10-100x performance |

## Design Principles

### 1. Obvious Over Clever
One way to do things. If you can read Python, you can read this.

### 2. Explicit Over Magic
No hidden control flow. Errors are values. Side effects are visible.

### 3. Simple Over Easy
Simple = fewer concepts to learn, composable.
Easy = familiar, possibly complex underneath.
We choose simple.

### 4. Fast By Default
Zero-cost abstractions where possible. Predictable performance.
No "oh that innocent-looking line is O(n^2)" surprises.

### 5. Progressive Complexity
95% of code needs no special knowledge.
5% can drop down to explicit memory control when needed.
The boundary is clear and auditable.

### 6. Cloud-Native DNA
Not a library. Built into the language:
- Graceful shutdown
- Health checks
- Configuration from environment
- Structured logging
- Observability hooks

## Key Design Decisions

| Area | Decision |
|------|----------|
| Memory | Regions + ARC with 95% RC elimination via compile-time analysis |
| Compilation | Cranelift for dev builds, LLVM for release |
| Concurrency | Structured concurrency, colorless functions, M:N threading |
| Types | Bidirectional inference, explicit function signatures |
| Errors | `Result[T, E]` with `?` operator, anonymous sum types |
| Syntax | Python-like indentation, 4 spaces, `and`/`or`/`not` |
| Modules | Go's MVS, three-level visibility (private/internal/pub) |
| Metaprogramming | Built-in derives + comptime, NO AST macros |
| Interop | Zero-overhead C FFI, PyO3-style Python embedding |
| Testing | `#[test]` attributes, compiler-rewritten assert |
| Security | Compile-time capabilities, runtime enforcement, OS sandboxing |

## CLI

```
indent                    # Single binary
├── build                 # Compile to binary
├── run                   # Compile and run (cached)
├── test                  # Run tests
├── fmt                   # Opinionated formatter
├── lint                  # Static analysis
├── check                 # Type check only
├── doc                   # Generate docs
├── repl                  # Interactive mode
└── lsp                   # Language server
```

## Documentation

| Document | Purpose |
|----------|---------|
| [SYNTHESIS.md](./SYNTHESIS.md) | Complete design synthesis from 37 research papers |
| [SPECIFICATION.md](./SPECIFICATION.md) | Formal language grammar, types, and semantics |

## Success Criteria

A minimal viable language would:
- [ ] Compile a "hello world" to native binary
- [ ] Have basic types (int, float, string, list, dict)
- [ ] Have functions and type definitions
- [ ] Handle errors explicitly
- [ ] Run concurrent code safely
- [ ] Call C libraries
- [ ] Have a working REPL or fast compile cycle
- [ ] Produce binaries deployable in containers

---

*Last updated: 2026-01-17*
*Status: Design complete (37 research papers), ready for prototype*
