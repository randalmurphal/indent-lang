# Designing type systems that feel dynamic but compile fast

**For a backend language targeting server and DevOps engineers, the optimal design combines bidirectional type checking with local inference boundaries, a hybrid monomorphization strategy, and Go-style structural traits with explicit opt-in.** This achieves Python's ergonomics while maintaining sub-second incremental compiles. The key insight from modern language research: full Hindley-Milner inference is unnecessary—TypeScript, Kotlin, and Swift prove that targeted bidirectional inference with rich control flow analysis delivers superior developer experience with better compile times and error messages.

The research reveals a clear design path: require explicit function signatures (enabling parallelization), use bidirectional checking within function bodies (enabling contextual typing for lambdas), monomorphize primitives while using witness tables for complex types (balancing performance and compile time), and invest heavily in diagnostic infrastructure from day one.

---

## Type inference: why local beats global for your use case

The theoretical complexity of Hindley-Milner inference is **DEXPTIME-complete** in the worst case, but this rarely manifests in practice—OCaml compiles ~170K lines in roughly 60 seconds from scratch. The real issue isn't algorithmic complexity but *design predictability* and *error localization*.

Go's designers explicitly rejected HM-style inference after experiencing 45-minute C++ builds at Google. Their rationale: "Experience with other languages suggests that unexpected type inference can lead to considerable confusion when reading and debugging a program." Go restricts inference to short variable declarations (`x := expr`) within function bodies only, with function signatures always explicit. This enables their hallmark compilation speed—a factor of **50x less data processed** compared to equivalent C++ through their dependency model.

Rust extends HM with subtyping and lifetimes, allowing inference across statements within functions:

```rust
let mut vec = Vec::new();  // Type unknown here
vec.push("thing");          // Now inferred as Vec<&str>
```

But crucially, Rust requires explicit function signatures. From compiler profiling data, type inference itself isn't Rust's bottleneck—**LLVM codegen and monomorphization dominate** compile time, not the inference algorithm.

**For your language**: Adopt bidirectional type checking rather than HM. TypeScript's design team states explicitly: "None of these techniques are Hindley-Milner type inference. Instead, TypeScript adds a few ad-hoc inference techniques to its normal type-checking." Bidirectional checking combines synthesis (bottom-up type derivation) with checking (top-down type verification), naturally supporting subtyping and union types that HM struggles with. This is why TypeScript, Scala 3, Swift, and Kotlin all use bidirectional approaches.

| Approach | Worst-Case Complexity | Practical Speed | Subtyping Support | Annotation Burden |
|----------|----------------------|-----------------|-------------------|-------------------|
| Hindley-Milner | DEXPTIME | ~O(n) typical | Poor | Minimal |
| Bidirectional | ~Linear | O(n) | Excellent | Moderate |
| Go Local-Only | O(n) | Very Fast | N/A | Function signatures |
| Rust Extended-HM | Polynomial | Moderate | Via traits | Function signatures |

The compile-time benchmarks are instructive: OCaml's HM-based compiler processes **~40K LOC in 4 seconds** for well-structured projects. With bidirectional checking scoped to single expressions (Swift's approach), you gain better error locality while avoiding HM's edge-case complexity. Swift's constraint solver handles each expression independently—exponential worst-case but irrelevant for typical single-expression scope.

---

## Generics implementation: the hybrid approach wins

The monomorphization vs type erasure decision profoundly impacts both compile time and runtime performance. Research reveals that **full monomorphization creates severe scaling problems**, while pure type erasure sacrifices too much performance for systems programming.

Real-world monomorphization costs are sobering. The Burn.dev matrix multiplication benchmark dropped from **108 seconds to 1 second** after aggressive optimization targeting monomorphization bloat. A type-erased BTree experiment in Rust reduced compile time from **6 minutes to 30 seconds**. Binary sizes tell a similar story: one CLI tool grew from 2MB to **37MB** (18.5x increase) due to generic data structures; xi-editor's 5.9MB release binary lists serde serialization as the biggest bloat source.

Go 1.18's "GCShape stenciling with dictionaries" attempts a middle ground but shows concerning overhead. PlanetScale's benchmarks reveal that generic method calls with pointer types run **5% slower than interface dispatch**, and generic functions with complex interface constraints can be **248% slower** than manually monomorphized code. The culprit: two pointer dereferences (dictionary → itab → function pointer) plus `runtime.assertI2I` calls for every method invocation when passing interfaces to generic functions.

**Swift's witness tables offer the most promising model** for your requirements. The approach:
- Protocol Witness Table (PWT): function pointers to protocol method implementations
- Value Witness Table (VWT): lifecycle operations and type size/alignment
- Existential Container: 5-word runtime structure (3-word inline buffer for small values)

Swift defaults to dictionary passing for ABI stability but opportunistically monomorphizes when the compiler sees both use and definition in the same module. The `@inlinable` attribute enables cross-module specialization. From WWDC benchmarks: specialized generics with value types achieve **zero allocation, zero reference counting, and static dispatch**, while unspecialized generics with small values still fit in the inline buffer with no heap allocation.

**.NET's partial specialization** deserves serious consideration: monomorphize value types (varying sizes require specialized code), share code for all reference types using a canonical `System.__Canon` representation. This elegantly balances performance and code sharing since reference types have uniform pointer size.

**Recommended strategy for your language**:
1. Monomorphize primitives and small value types (integers, floats, small structs)
2. Use witness tables for interface-constrained generics with reference types
3. Provide an explicit `@specialize` annotation for performance-critical paths
4. Consider .NET's approach: automatically share code for pointer-sized types

---

## Structural traits: Go's model with escape hatches

Your goal of "structural traits (Go-like)" aligns with the research, but the findings reveal important caveats about scale and safety.

Go's structural interfaces enable powerful patterns—types created in separate packages satisfy interfaces without modification, enabling unanticipated reuse. Kubernetes' client-go library extensively leverages this for heterogeneous resource handling. However, IDE tooling suffers: "Find All Implementations" via LSP works but is "not very intuitive"—users must invoke it from struct types rather than interfaces, and there are no visual indicators showing when a method implements an interface.

The **accidental compatibility problem** is real. The CMU paper "Integrating Nominal and Structural Subtyping" documents the classic `cowboy.draw()` vs `circle.draw()` collision—structurally identical methods with completely different semantics. In Go, two unrelated types with matching method signatures accidentally satisfy the same interface. When external dispatch or overriding exists, structural matching causes ambiguity problems.

TypeScript's structural typing at scale reveals performance and correctness challenges. Large codebases (100k+ LOC) with complex type inference experience serious degradation—one team documented "complex type inference failing silently in 23% of cases." Heavily generic libraries (Prisma, Kysely, ts-pattern) cause type-checking bottlenecks requiring workarounds like inlining queries to reduce type complexity.

The popularity of **branded types** in TypeScript is telling:

```typescript
type Brand<K, T> = K & { __brand: T };
type UserId = Brand<number, "UserId">;
type OrderId = Brand<number, "OrderId">;
// Now UserId and OrderId cannot be accidentally mixed
```

The TypeScript compiler team uses this pattern internally. Its existence demonstrates that structural typing alone is insufficient for large-scale type safety.

**Recommended hybrid design**:
1. Default to structural matching (Go-style implicit satisfaction) for simplicity
2. Provide an explicit `implements` declaration that opts into nominal checking
3. Support branded/newtype patterns for domain types requiring identity
4. Consider Rust's approach: allow implementing traits for non-local types (Go forbids this, limiting extensibility)

The CMU research concludes that both paradigms have value: nominal subtyping for explicit design intent, structural subtyping for flexible reuse. A language supporting both—defaulting to structural with opt-in nominal—offers maximum flexibility.

---

## How TypeScript, Kotlin, and Swift feel dynamic while staying static

These three languages achieve excellent inference through carefully limited scope plus compensating mechanisms—not through implementing full HM.

**TypeScript's three inference techniques**:
1. **Initializer inference**: Types derived from right-hand side (`let x = 123` → `number`)
2. **Contextual typing**: Looks *upward* in AST for type annotations, walks back *down* through the type to apply to parameters
3. **Type parameter inference**: Two-pass system—first pass skips contextually typed arguments, second pass uses inference priorities (lone type variables highest, return positions lower)

TypeScript's control flow analysis compensates for limited inference. The checker constructs control flow graphs greedily in the binder, then lazily evaluates them backward from reference points. This enables powerful narrowing: `typeof`, `instanceof`, discriminated unions, truthiness checks, and user-defined type guards (`x is Type`).

**Kotlin's smart casts** implement data-flow analysis over a lattice structure tracking what types values *are* vs *are not*. The critical "stability" requirement means smart casts work only on stable references—immutable `val` properties are always stable, mutable `var` locals are stable only if effectively immutable (no nested redefinitions). This prevents soundness holes from concurrent modification.

**Swift's constraint-based checker** most resembles HM but scopes solving to single expressions. The architecture:
1. **Constraint generation**: Convert source to constraint system with type variables (`$T0`, `$T1`)
2. **Constraint solving**: Union-find for equivalence classes, disjunction constraints for overloads
3. **Solution application**: Make implicit conversions explicit

Swift's bidirectional flow enables both bottom-up inference (`let age = 26` → `Int`) and top-down checking (`let value: Double = 42` makes literal `Double`, not `Int`). The key insight: scoping to single expressions avoids exponential blowup while enabling rich inference within expressions.

**Common patterns across all three**:
- Require explicit function parameter types (always)
- Allow return type inference from expression bodies
- Use contextual typing for lambdas/closures
- Implement rich control flow analysis to compensate for limited inference
- Apply heuristics and priorities rather than complete algorithms

---

## Building Rust-quality error messages from the start

Rust's diagnostic infrastructure represents the state of the art. Understanding its architecture enables replicating the quality in a new language.

**The Span system** tracks source locations pervasively. Every AST node carries a `Span` (byte offsets into source). Spans can be looked up in a `SourceMap` to retrieve source code snippets for display. `MultiSpan` supports diagnostics involving multiple locations (essential for "expected X here, but found Y there" messages). Crucially, spans track macro expansion context—enabling errors in macro-generated code to point back to the macro invocation.

**Diagnostic structure**:
```
error[E0308]: main error message
  --> file.rs:LL:CC
   |
LL | <code>
   | -^^^^- secondary label
   |  |
   |  primary label
   |
   = note: additional context
help: suggestion message
   |
LL | suggested code
   |     ^^^
```

Rust's diagnostic system uses **derive macros** for type-safe diagnostic definitions:

```rust
#[derive(Diagnostic)]
#[diag(hir_analysis_field_already_declared, code = E0124)]
pub struct FieldAlreadyDeclared {
    #[primary_span] #[label] pub span: Span,
    #[label(previous_decl_label)] pub prev_span: Span,
}
```

**The suggestion system** is particularly sophisticated. Each suggestion carries an `Applicability` level: `MachineApplicable` (safe to auto-apply), `MaybeIncorrect`, `HasPlaceholders`, or `Unspecified`. This enables `cargo fix` to automatically apply safe suggestions while prompting for uncertain ones. Span manipulation methods like `find_ancestor_inside()` prevent suggesting changes in external/macro code.

**Elm's complementary approach** focuses on human-centered design. The "compiler as assistant" philosophy prioritizes teaching over reporting: "Help users learn—error messages should teach, not just report." Elm's type error messages show type diffs, use conversational language ("Looks like a record is missing the field..."), and link directly to documentation.

Academic research on type error diagnosis is relevant. The "Learning to Blame" paper (NATE, OOPSLA 2017) achieved **72% accuracy** in predicting the exact sub-expression causing type errors using machine learning on a corpus of ill-typed programs—16 points higher than constraint-based approaches. Key features: syntactic (is-literal, is-function-app) and semantic (is-list, is-integer).

**Implementation recommendations**:
1. Attach spans to all AST/IR nodes from parsing onward
2. Separate diagnostic data structures from rendering (enable JSON, terminal, LSP output)
3. Build the suggestion system alongside the type checker, not as an afterthought
4. Use applicability levels for suggestions from day one
5. Support Fluent-style internationalization for messages
6. Maintain an error catalog (like `elm/error-message-catalog`) driven by real user reports
7. Apply bidirectional "blame" heuristics: trust type signatures, weight by distance from annotations

---

## The achievable frontier: sub-second incremental with rich inference

Research identifies a clear path to maximum inference with Go-level compile times. The Salsa framework (powering rust-analyzer) demonstrates the architecture, while Zig shows the type system design.

**Salsa's red-green algorithm** enables sub-second incremental checking:
1. After compilation, save all query results plus the query DAG (which queries depend on which)
2. On input change, use "try-mark-green" to identify recomputation needs
3. **Critical optimization**: If a query's inputs change but it produces the same result, mark it "green" and skip re-executing dependents
4. This means changing a function body often doesn't trigger recompilation of callers—only if the signature changes

rust-analyzer achieves IDE-level responsiveness (**tens of milliseconds** for incremental updates) using Salsa despite Rust's complex type system.

**Zig's design proves fast compilation with useful inference is achievable**:
- No unification—"type inference is a simple single-pass recursion" (per matklad)
- Explicit type parameters at generic call sites (`fn max(comptime T: type, a: T, b: T)`)
- Mandatory return type annotations (enables skipping function bodies during parallel compilation)
- Methods "close over" generic parameters, covering 80% of use cases without annotation burden

**What's achievable with <1s incremental compiles**:

| Feature | Achievable? | Notes |
|---------|-------------|-------|
| Local type inference | ✅ | Within expressions/statements |
| Bidirectional checking | ✅ | Standard for modern languages |
| HM within modules | ✅ | With incremental recomputation |
| Cross-module inference | ⚠️ | Requires careful caching |
| Higher-rank polymorphism | ✅ | With annotations at rank-2+ |
| GADTs | ⚠️ | Local type argument inference |
| Dependent types | ❌ | Normalization too expensive |
| Full System F inference | ❌ | Undecidable |

**Experimental languages pushing boundaries**:
- **Ante**: Lifetime inference reducing Rust's annotation burden, algebraic effects, refinement types
- **Vale**: Generational references for memory safety without GC, region borrowing for zero-cost pre-existing memory access
- **Koka** (Microsoft Research): Row-polymorphic effect types with efficient compilation via selective CPS

---

## Concrete design recommendations

Based on this research, here's a synthesis optimized for your requirements (Python readability, Go simplicity, near-Rust performance, backend/DevOps engineers):

**Type inference**:
- Bidirectional type checking with local scope (single statement/expression)
- Require explicit function signatures for parameters
- Infer return types from expression bodies
- Implement rich control flow analysis for type narrowing (TypeScript-style)
- Use contextual typing for lambdas and collection literals

**Generics**:
- Hybrid approach: monomorphize primitives and small value types, witness tables for interface-constrained reference types
- Explicit type parameters at call sites when not inferable from arguments
- Consider .NET-style sharing for pointer-sized types
- Provide `@specialize` annotation for performance-critical paths

**Structural traits**:
- Default to structural matching (Go-style) for simplicity
- Provide explicit `implements` for opt-in nominal checking
- Support newtype patterns for branded domain types
- Allow trait implementations for external types (unlike Go)

**Compile speed architecture**:
- Mandatory function signatures enabling parallel compilation
- Salsa-style incremental framework with red-green revalidation
- Aggressive type interning (32-bit indices into canonical pool)
- No cyclic module dependencies

**Error messages**:
- Attach spans to all nodes from parsing
- Structured diagnostic types separate from rendering
- Machine-applicable suggestions with applicability levels
- JSON output for tooling integration
- Learn from Elm: conversational tone, link to documentation

The research demonstrates that this combination is achievable. TypeScript, Kotlin, and Swift prove excellent inference doesn't require HM. Salsa proves sub-second incremental with complex types is possible. Swift's witness tables prove hybrid generics can balance performance and compile time. The path forward is clear—it's an engineering challenge, not a research gap.