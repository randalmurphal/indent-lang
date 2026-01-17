# Error handling that's explicit but not annoying: state of the art

**Your proposed approach—`Result[T, E]` with `T | E` sugar, `?` operator, and `.context()` method—aligns remarkably well with emerging best practices.** After deep analysis of Rust's compiler implementation, algebraic effects research, Swift's error model, production error handling patterns, and debugging strategies, several key insights emerge that could refine your design further. The core tension in error handling—between explicitness and ergonomics—has been solved by modern languages through strategic syntax sugar over sound semantic foundations.

## The `?` operator works because it's beautifully simple inside

Rust's `?` operator achieves genuine zero-cost abstraction through a surprisingly minimal desugaring. The operator transforms `expr?` into a match on `ControlFlow::Continue/Break` variants via the `Try` trait:

```rust
match Try::branch(expr) {
    ControlFlow::Continue(val) => val,
    ControlFlow::Break(residual) => return FromResidual::from_residual(residual),
}
```

The critical insight is **separating the "what" from the "how"**—the desugaring itself is trivial, while type-specific behavior lives in trait implementations that get monomorphized away. LLVM then inlines these implementations completely, producing assembly identical to hand-written match statements. Benchmarks consistently show zero measurable overhead in release builds.

Three design decisions enable this zero-cost property: **monomorphization** (generating specialized code per concrete type), **aggressive inlining** (all trait methods marked `#[inline]`), and **niche optimization** (exploiting type layouts so `Result<&T, E>` has the same size as a raw pointer). For your language, this suggests the `?` operator should desugar to the simplest possible form, pushing complexity into overridable trait methods that your compiler can fully inline.

The `From::from` conversion for error types happens *inside* the `FromResidual` trait implementation, not in the desugaring itself. This subtle placement enables automatic error type conversion while keeping the core operator semantics clean—a pattern worth preserving in your design.

## Anonymous sum types would solve Rust's biggest pain point

The most frequent complaint from Rust developers involves **error type proliferation**. When a function can fail with `IoError` or `ParseError`, you must either define a named enum, use `Box<dyn Error>`, or reach for crates like `anyhow`. Your proposed `T | E` syntax for anonymous sum types addresses this directly.

Production Rust codebases reveal a clear pattern: libraries use `thiserror` to define explicit error enums with `#[from]` attributes for automatic conversion, while applications use `anyhow` for type-erased errors with context chaining. The `thiserror`/`anyhow` split exists precisely because Rust lacks anonymous sum types—your language can unify these patterns:

```
// Hypothetical syntax combining the best of both worlds
fn load_config(path: Path) -> Result[Config, IoError | ParseError] {
    let contents = read_file(path)?         // IoError
    let config = parse_json(contents)?      // ParseError
    Ok(config)
}
```

The compiler can automatically handle `?` propagation when the current function's error type is a superset of the expression's error type. This eliminates the need for explicit `From` implementations while preserving exhaustive pattern matching. OCaml's polymorphic variants demonstrate this works well in practice—errors compose naturally without naming every combination.

## Algebraic effects are promising but premature for mainstream adoption

Algebraic effects offer elegant error handling through "resumable exceptions"—handlers can intercept effects and either terminate or resume computation with a value. Koka demonstrates this works well for a research language, achieving **performance within 10% of C++** through evidence-passing compilation. OCaml 5's effect handlers show the concept can retrofit onto an existing language.

However, several factors argue against full algebraic effects for a mainstream backend language in 2026:

- **Developer unfamiliarity**: The "resumable exception" mental model is foreign to most developers
- **Implementation complexity**: Evidence passing or fiber-based runtimes require significant engineering
- **Limited ecosystem**: No major production systems use effect-based error handling
- **Uncertain tooling**: IDE support for effect navigation remains nascent

A **simplified subset** could work well: one-shot continuations only (like OCaml), built-in effects for common patterns (Error, Async), and familiar `try/handle` syntax. But this adds complexity for uncertain gain over your current proposal.

**Recommendation**: Stick with `Result[T, E]` and `?` for error handling. Algebraic effects might be worth revisiting for async/concurrency features where resumable computations have clearer value.

## Swift 6's typed throws validates the errors-as-values direction

Swift's evolution from untyped to typed throws confirms the industry is converging on Rust's approach. Swift 6 introduced `throws(E)` syntax after years of developer requests:

```swift
func processFile() throws(FileError) -> Data {
    throw .fileNotFound  // Type-safe, exhaustive matching in catch blocks
}
```

Notably, Swift's error handling uses a **dedicated register** for error returns (`x21` on ARM64), requiring just one instruction per call to check for errors. This is slightly slower on the happy path than Rust's zero-cost Result (which needs no check when inlined), but much faster on the error path than C++ exceptions.

The key lessons from Swift's journey:

1. **Untyped errors hurt libraries**: Callers can't know what errors to handle without documentation
2. **Migration matters**: Swift supports both typed and untyped throws, enabling gradual adoption  
3. **Inference reduces annotation burden**: Swift infers error types through closures and generics
4. **Explicit callsites improve readability**: Both `try` and `?` mark where errors can occur

Your language should be **typed from the start** (Swift's recommendation is that untyped throws were a mistake for libraries), but inference should minimize annotation requirements for application code.

## Context chaining beats backtraces for debugging ergonomics

Stack traces are expensive—Rust's backtrace capture costs **~256ms with symbol resolution**, making them impractical to enable universally. The modern approach separates concerns:

| Strategy | Cost | Information | Use Case |
|----------|------|-------------|----------|
| Compile-time location (`#[track_caller]`) | **Zero** | File, line, column | Always-on, panic messages |
| Context chaining (`.context()`) | **Cheap** | Semantic description | Error propagation |
| Error return tracing (Zig-style) | **Linear** | Propagation path | Debug builds |
| Full backtrace | **Expensive** | Complete stack | Investigation only |

Zig's error return tracing is particularly clever—instead of capturing the entire stack at error creation, it records each point where an error is *returned*. This costs just a few ALU ops plus one memory write per `?` operation, scaling linearly with propagation depth rather than stack depth.

**For your language, consider a hybrid approach:**

```
// Every error gets zero-cost compile-time location
struct Error[E] {
    value: E,
    location: Location,  // File, line, column, function - compiler-inserted
    context: Option[String],
    return_trace: Option[ReturnTrace],  // Debug builds only
}

// Context is cheap and semantic
read_file(path).context("loading config")?

// Return traces enabled via environment or compiler flag
DEBUG_ERRORS=1 myprogram
```

The `.context()` method in your proposal is exactly right—it provides semantic debugging information at minimal cost while keeping full backtraces optional for investigation.

## Concrete recommendations for your language

Based on this research, here's how to refine your proposed `Result[T, E]` with `T | E` syntax:

**Core type system:**
- `Result[T, E]` as the foundational error type with `Ok(T)` and `Err(E)` variants
- Anonymous sum types: `IoError | ParseError` composes without naming
- Automatic subtyping: `Result[T, E1]` converts to `Result[T, E1 | E2]` via `?`
- Exhaustive pattern matching on error variants

**The `?` operator:**
- Desugar to simple match on Continue/Break (Rust's try_trait_v2 design)
- Automatic error conversion when return type is superset
- Require explicit `.into()` when narrowing error types
- Full inlining to achieve zero-cost abstraction

**Context and debugging:**
- Built-in `.context(msg)` and `.with_context(|| msg)` methods
- Compile-time location tracking (file, line, column, function) on all errors
- Optional error return tracing for debug builds
- Environment-controlled full backtraces for investigation

**Error definition ergonomics:**
- Derive macro or compiler support for `Error`, `Display`, `From` equivalents
- `#[from]` attribute for automatic variant conversion
- Anonymous sum types eliminate need to name every combination

**Explicit but not annoying syntax:**
```
// Function signature shows error types (explicit)
fn load_config(path: Path) -> Result[Config, IoError | ParseError]

// Error propagation is concise (not annoying)
fn initialize() -> Result[App, InitError] {
    let config = load_config("app.json")
        .context("loading configuration")?
    let db = connect_database(config.db_url)?
    Ok(App { config, db })
}

// Pattern matching is exhaustive
match load_config(path) {
    Ok(config) => use(config),
    Err(IoError::NotFound) => create_default(),
    Err(IoError::Permission(p)) => fail("permission denied: {p}"),
    Err(ParseError::Syntax(s)) => fail("invalid config: {s}"),
}
```

## Novel ideas worth exploring

Beyond established patterns, several innovations could differentiate your language:

**Gradual error typing**: Allow `Result[T, _]` in application code where the compiler infers the error type union, while requiring explicit types in public library APIs. This mirrors the thiserror/anyhow split without requiring two different approaches.

**Semantic error categories**: Built-in traits like `Retryable`, `UserFacing`, `Transient` that errors can implement, enabling generic recovery logic without pattern matching on specific types.

**Error budgets**: Compile-time tracking of which functions can fail, with optional "error budget" annotations that limit error type proliferation in module boundaries.

**Async-aware context**: Integrate error context with structured logging (like Rust's `tracing` crate) so errors automatically capture which async operation was in progress—solving a major pain point in Result-based async code.

Your proposed design is already well-positioned. The key refinements are: anonymous sum types for ergonomic composition, compile-time location tracking for zero-cost debugging, and considering error return tracing as a debug-build feature. These additions would give your language error handling that genuinely achieves "Rust's safety with less boilerplate than Go."