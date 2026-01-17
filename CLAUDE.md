# Indent Language Design

**Purpose**: Design specification for Indent, a compiled language targeting backend developers with Python readability, Go simplicity, and Rust performance.

**Status**: Pre-implementation - 37 research papers complete, specification ready for prototype.

---

## Quick Navigation

| Need to... | Go to... |
|------------|----------|
| Understand design decisions | `docs/SYNTHESIS.md` |
| Read formal language spec | `docs/SPECIFICATION.md` |
| Find research on a topic | `research/<category>/` |
| See project overview | `docs/README.md` |

---

## Project Structure

```
lang-design/
├── CLAUDE.md              # This file
├── docs/
│   ├── SYNTHESIS.md       # All design decisions (1600 lines)
│   ├── SPECIFICATION.md   # Formal language spec (1000 lines)
│   └── README.md          # Project overview
└── research/              # 37 research papers
    ├── 01-core/           # Memory, compiler, concurrency, types, errors
    ├── 02-language/       # Syntax, modules, metaprogramming, interop, testing, security
    ├── 03-ecosystem/      # Stdlib, tooling, async, collections, strings, etc.
    └── 04-strategic/      # Adoption, failures, AI, hot reload, numerics, etc.
```

---

## Core Language Identity

| Aspect | Decision |
|--------|----------|
| **Name** | Indent |
| **Extension** | `.idt` |
| **Target** | Backend developers, DevOps, server applications |
| **Philosophy** | "One obvious way" - Go simplicity, Python readability, Rust performance |
| **Memory** | Regions + ARC with 95% compile-time RC elimination |
| **Concurrency** | Colorless async, structured concurrency |
| **Typing** | Static, bidirectional inference, explicit function signatures |

---

## Key Design Decisions

| Area | Decision | Confidence |
|------|----------|------------|
| Syntax | Python-like: 4 spaces, `and`/`or`/`not`, significant whitespace | Very High |
| Memory | Regions + ARC, Perceus-style optimization | High |
| Compilation | Cranelift (dev) + LLVM (release) | Very High |
| Concurrency | Stackful coroutines, work-stealing, io_uring | High |
| Errors | `Result[T, E]` with `?` operator, anonymous sum types | Very High |
| Strings | UTF-8, grapheme-first `len()`, no integer indexing | Very High |
| Generics | Hybrid mono/witness tables, `@specialize` for hot paths | High |
| Metaprogramming | Built-in derives + comptime, NO AST macros | High |
| Security | Three-layer: compile-time caps, runtime enforcement, OS sandbox | High |

---

## Research Categories

| Category | Papers | Topics |
|----------|--------|--------|
| `01-core/` | 5 | memory, compiler, concurrency, error_handling, type |
| `02-language/` | 6 | syntax, modules, metaprogramming, interop, testing, security |
| `03-ecosystem/` | 14 | stdlib, tooling, async_runtime, collections, strings, db, config, etc. |
| `04-strategic/` | 12 | adoption, failures, AI, hot_reload, numerics, time, serialization, etc. |

---

## Syntax Quick Reference

```python
# Function definition
fn add(a: int, b: int) -> int:
    return a + b

# Error handling
fn fetch(url: str) -> Data | IOError | ParseError:
    content = http.get(url)?
    return parse(content)?

# Structured concurrency
concurrent:
    results = [spawn fetch(url) for url in urls]

# Pattern matching
match value:
    case Some(x) if x > 0: handle_positive(x)
    case Some(x): handle_negative(x)
    case None: handle_none()

# Lambda
items.map(x -> x * 2)

# Pipeline
data |> transform |> filter |> format
```

---

## When Working on This Project

### Do
- Reference `docs/SYNTHESIS.md` for authoritative design decisions
- Check research papers in `research/` for detailed rationale
- Follow the "one obvious way" philosophy
- Keep syntax Python-like and readable

### Ask First
- Changes to core design decisions in SYNTHESIS.md
- New language features not covered by existing research
- Deviations from established patterns

### Don't
- Add multiple ways to do the same thing
- Introduce features without research backing
- Change decisions marked "Very High" confidence without discussion

---

## Prototype Roadmap

| Phase | Timeline | Focus |
|-------|----------|-------|
| 1 | Months 1-3 | Core: parser, type checker, Cranelift codegen, basic memory |
| 2 | Months 4-6 | Usable: full types, errors, concurrency, basic stdlib, formatter |
| 3 | Months 7-12 | Production: LLVM backend, HTTP, database, LSP, packages |

---

## Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `docs/SYNTHESIS.md` | ~1600 | All design decisions from 37 research papers |
| `docs/SPECIFICATION.md` | ~1000 | Formal language grammar and semantics |
| `research/README.md` | ~80 | Index of all research papers |

---

## Common Gotchas

1. **No colored functions** - All functions can suspend; runtime handles it transparently
2. **Grapheme-first strings** - `len("é")` returns 1, not byte count
3. **Checked arithmetic** - Integer overflow panics by default; use `+%` for wrapping
4. **Borrowing default** - Function params borrow immutably unless marked `mut`
5. **No null** - Use `Option[T]` instead
6. **Regions not GC** - Memory freed at region exit, not by garbage collector

---

## Related Documentation

- Research index: `research/README.md`
- Full synthesis: `docs/SYNTHESIS.md`
- Language spec: `docs/SPECIFICATION.md`
