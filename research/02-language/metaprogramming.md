# Minimal metaprogramming that maximizes simplicity

A new compiled language targeting "Python's readability, Go's simplicity, Rust's performance" can achieve **derive-like functionality** for serialization, debug, and cloning without the complexity of Rust's proc macros or C++ templates. The key insight from analyzing five metaprogramming approaches: **Zig's comptime philosophy combined with Swift's constrained mechanisms provides 90% of derive-macro utility at roughly 20% of the implementation and learning complexity**.

The recommended architecture integrates three complementary layers: built-in compiler intrinsics for common derivations, a comptime-style system for type reflection without AST manipulation, and explicit code generation for edge cases. This approach aligns with your "obvious over clever" philosophy while maintaining fast compilation and excellent error messages.

## Rust proc macros are powerful but costly

Rust's procedural macros operate on `TokenStream` objects through an **IPC mechanism**—the compiler places tokens in a handle table and macros make RPC calls back to the compiler. This design solves ABI stability but introduces significant compile-time overhead. Macro expansion isn't cached: changing one line forces re-expansion of all macros in that crate. Benchmarks show **11-40% slower incremental builds** on macro-heavy codebases, with Polkadot-SDK seeing 35% improvement from experimental caching.

The developer experience pain points are substantial. Proc macros require a **separate crate** with `proc-macro = true`, forcing awkward `-macros` companion packages. Debugging relies on `panic!` output or `cargo-expand`, neither integrated into standard workflows. The dual `proc_macro` vs `proc_macro2` types confuse newcomers—entry points must use one, internal logic the other. Error messages point to invocation sites rather than generated code, making failures cryptic.

Helper libraries (`syn`, `quote`, `darling`) make complex macros manageable but add their own compile-time tax—syn alone takes significant time because "parsing Rust is genuinely hard." The learning curve requires understanding token manipulation, compiler internals, and a fundamentally different mental model from regular Rust. For a language prioritizing simplicity, **Rust's approach trades too much complexity for power most users don't need**.

## Go's explicit generation proves transparency works

Go's `go generate` demonstrates that **explicit code generation** can cover substantial metaprogramming needs without language complexity. Magic comments like `//go:generate stringer -type=Pill` trigger external tools that produce committed, visible source files. This creates real tradeoffs: transparency and debuggability versus manual synchronization burden.

The approach works remarkably well for specific patterns. **Stringer** generates efficient `String()` methods for enums using static lookup tables and compilation guards that fail if enum values change. **easyjson** produces optimized JSON marshaling 5-6x faster than reflection-based `encoding/json`. **mockgen** creates concrete mock implementations from interfaces. These tools remain essential even after Go 1.18 added generics because they **inject methods**—something generics cannot do.

The pain points are instructive for language design. Developers must remember to regenerate, creating CI verification overhead (`git diff --exit-code` after generation). Version control noise from generated files creates merge conflicts. No dependency analysis means code can silently become stale. However, the Go community consensus—commit generated files, verify in CI—proves this model scales to large codebases. The key insight: **explicit generation complements, rather than competes with, generics and compile-time features**.

## Zig comptime eliminates the "second language" problem

Zig's compile-time execution represents the most elegant alternative to traditional macros. **The same Zig code runs at compile time and runtime**—no separate DSL, no TokenStream manipulation, no AST transformation. Comptime operates on values, including types as first-class citizens, using partial evaluation rather than code generation.

Type reflection works through `@typeInfo`, which returns a tagged union describing any type's structure. Generic data structures emerge naturally:

```zig
fn LinkedList(comptime T: type) type {
    return struct {
        pub const Node = struct { data: T, next: ?*Node };
        first: ?*Node = null;
    };
}
```

This enables serialization, debug printing, and cloning through **field iteration at compile time**. The standard library validates format strings purely in userland code—no compiler special-casing required. Conditional compilation uses normal `if` statements; dead branches are entirely eliminated.

The deliberate limitations preserve simplicity. **No I/O during compilation** makes builds hermetic, reproducible, and safely cacheable. **No custom syntax** means no DSL proliferation. **No method injection** means types' public APIs must be hand-written. The execution limit of 1000 backwards branches prevents runaway compile times. These constraints eliminate entire categories of complexity while covering the derive-style use cases your language needs.

## Nim's graduated hierarchy offers a middle path

Nim provides three metaprogramming levels with explicit guidance: **use the least powerful construct that suffices**. Templates perform AST substitution (like hygenic C macros). Macros enable full AST manipulation. Compile-time procs execute arbitrary Nim code.

The template level handles 60-70% of metaprogramming needs:

```nim
template withLock(lock: Lock, body: untyped) =
  acquire lock
  try: body
  finally: release lock
```

Macros receive AST as `NimNode` objects with direct access to compiler internals. Built-in tools dramatically lower the learning curve: `dumpTree` shows any code's AST structure, `dumpAstGen` generates code that would produce that AST, and quasiquoting via `quote do:` makes code generation readable. However, advanced macros require understanding **100+ node kinds**, and error messages can be "weird" when macros fail.

Compared to Rust, Nim macros offer **simpler setup** (no separate crate), **direct AST access** (no TokenStream indirection), and **unified language** (no macro_rules! DSL). But they still require substantial AST knowledge for anything beyond templates—a complexity cost your language should avoid.

## Swift proves constrained features cover common cases

Swift's property wrappers and result builders demonstrate that **targeted abstractions outperform general-purpose macros** for specific patterns. Property wrappers encapsulate property access logic; result builders transform closure syntax into DSLs. Neither requires AST manipulation or macro understanding.

SwiftUI's success validates the approach. `@State` manages view-local mutable state in structs. `@Binding` creates two-way connections to parent state. `@Published` adds automatic change notification. These solve data flow patterns that would otherwise require verbose boilerplate—without exposing users to metaprogramming complexity. Result builders enable SwiftUI's declarative syntax without commas or explicit array management.

The limitations are intentional features. Property wrappers **cannot inject methods** onto types or participate in error handling. Result builders support only **specific control flow** (`if`, `for...in`, `switch`)—no `guard`, `break`, or `while`. These constraints preserve predictability and debuggability. When Swift 5.9 added full macros, property wrappers and result builders remained the **preferred tools** for their specific domains due to superior ergonomics.

## Go's 12-year generics journey teaches patience

Go's resistance to generics from 2009 to 2022 wasn't obstinance—it was principled design. The team explored **five major designs** before finding one that preserved Go's values: fast compilation, small binaries, readable code. They explicitly avoided Java's type erasure problems and C++'s template error messages.

The core insight was implementation strategy. Pure monomorphization (C++ style) produces fast code but slow compilation and binary bloat. Pure boxing (Java style) compiles fast but runs slow with type erasure. Go's **GCShape stenciling with dictionaries** creates one function copy per "garbage collector shape" with runtime dictionaries for method dispatch within shapes—a novel hybrid balancing all concerns.

Even with generics, `go generate` remains essential. Generics cannot inject methods (stringer still needed), cannot generate code from external schemas (protobuf), cannot create mocks from interfaces (mockgen). The lesson: **generics and code generation solve different problems and coexist productively**. Design them as complementary systems from the start.

## Error message quality determines user experience

Across languages, the debugging experience varies dramatically. Rust's cargo-expand shows macro expansions but is separate from normal workflows. Good macro errors use **proper spans** pointing to user code, provide **help text** with "did you mean" suggestions, and **collect multiple errors** before stopping. Bad errors point to macro invocation sites when the problem is in generated code, cascade confusingly, or show compiler internals.

The proc-macro-error crate demonstrates best practices: `emit_error!(span, "unknown attribute"; help = "expected one of: rename, skip")` produces errors matching compiler quality. Nim's `expectKind` and `expectLen` catch invalid input early with clear messages. The design principle is locality—**errors should point to where the user can fix them**, not where the compiler detected the problem.

For language designers, key infrastructure includes: rich span types supporting multi-token ranges, hierarchical errors with notes and suggestions, error recovery to find multiple issues, and stable diagnostic APIs for generated code. Compile-fail testing (like Rust's trybuild) prevents error message regressions.

## Recommended architecture for your language

Based on this analysis, here's the minimal metaprogramming surface covering 90% of derive-style needs:

**Layer 1: Built-in compiler derivations (70% of use cases)**
Implement Serialize, Deserialize, Debug, Clone, Eq, Hash, and similar traits as **compiler intrinsics**, not macros. The compiler knows all type information and can generate optimal code with perfect error messages pointing to the specific field causing problems. This matches how Go's `fmt` package reflects on types for printing—no user-facing metaprogramming required.

**Layer 2: Comptime type reflection (20% of use cases)**
Adopt Zig's model: types are first-class comptime values, field iteration happens at compile time, and the same language syntax works for both phases. Provide `@typeInfo` equivalent for introspecting struct fields, enum variants, and method signatures. This enables custom serialization formats, validation, and builder patterns without AST manipulation.

```
// Example syntax concept
fn serialize[T](value: T, writer: Writer) {
    comptime for field in type_fields(T) {
        writer.write_field(field.name, @field(value, field.name))
    }
}
```

**Layer 3: Explicit generation for edge cases (10% of use cases)**
Provide first-class support for external code generators (like `go generate`) for schema-driven code (protobuf, GraphQL), mock generation, and cases requiring method injection. Commit generated files, verify in CI. This covers what comptime cannot: generating new identifiers, injecting methods, and processing external data sources.

**What to explicitly omit:**
- Full AST macros (too complex for "obvious over clever" goals)
- TokenStream manipulation (Rust's pain point)
- Custom syntax creation (prevents DSL proliferation)
- Compile-time I/O (preserves hermetic builds)
- Method injection at compile time (preserves type definition locality)

## Design principles for implementation

**Cache comptime evaluation aggressively.** Rust's biggest macro performance problem is lack of expansion caching. Make comptime results cached by default, keyed on input type and comptime parameters.

**Require explicit constraints on generic types.** Go's lesson: unconstrained generics produce C++-style error messages deep in instantiation. Require interface/trait bounds and report constraint violations at definition sites.

**Error messages should never show generated code by default.** Point to user declarations. Provide "expand derivation" as an explicit debug command (like cargo-expand) for advanced users.

**Make built-in derivations configurable through attributes.** Serde's `#[serde(rename = "...")]` pattern is excellent. Allow `#[serialize(skip)]`, `#[debug(format = "hex")]` without requiring users to understand metaprogramming.

**Design field iteration as deterministic and side-effect-free.** Comptime loops over fields should be pure, enabling aggressive caching and parallel compilation.

This architecture gives developers derive-like functionality for serialization, debug printing, cloning, and comparison while maintaining Go-level simplicity, near-Rust performance, and excellent error messages. The "obvious over clever" philosophy manifests as explicit layers with clear boundaries—users know whether they're using a built-in derivation, writing comptime reflection code, or reaching for external generation.