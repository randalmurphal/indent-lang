# Designing a module system for fast, parallel compilation

**A new compiled language targeting backend developers needs a module system that combines Go's compilation speed, Rust's fine-grained visibility, and the simplicity developers expect.** The research reveals clear patterns: Go's Minimum Version Selection eliminates lock file complexity while enabling reproducible builds; explicit module boundaries—not inferred types across files—unlock parallel compilation; and convention-based visibility (Go's capitalization) trades flexibility for simplicity in ways most developers accept. The critical insight is that every module system design decision directly impacts compilation speed, and the choices that feel "limiting" often enable the <100ms compile times your language targets.

## Go's import system enables fast builds through radical simplicity

Go's fundamental insight is that **import paths should equal code locations**, creating a predictable filesystem-to-namespace mapping. When the compiler sees `import "github.com/user/repo/pkg"`, it knows exactly where to find the code—no complex resolution algorithms, no searching multiple paths, no ambiguity about which version loads.

The URL-based import design has significant tradeoffs. The benefits are compelling: decentralization (anyone can host packages without a central registry), self-documenting imports (the source is visible in the path), and no name-squatting problems. However, URL stability becomes critical—if a repository moves, import paths break globally. Go solves this partially through the module proxy system (proxy.golang.org), which caches modules and provides the Go Checksum Database for integrity verification.

**Minimum Version Selection (MVS)** is Go's most underappreciated innovation. Traditional package managers (npm, Cargo) select the *newest* version satisfying constraints, requiring lock files for reproducibility. MVS inverts this: it selects the *minimum* version satisfying all requirements. The algorithm works by recursively collecting requirements from all dependencies, then keeping only the highest required version of each module—which is still the *minimum* version that works.

The practical implications are significant:

- **No lock file needed for reproducibility**—`go.mod` alone deterministically specifies versions
- **Polynomial-time complexity**—MVS lies in the intersection of 2-SAT, Horn-SAT, and Dual-Horn-SAT, meaning it has a unique minimal solution with no NP-complete SAT solving
- **High-fidelity builds**—users build with versions close to what authors actually tested
- **Predictable upgrades**—new releases don't change builds until explicit `go get -u`

The `go.mod` file itself is minimal: module path, Go version, requirements, and optional `replace`/`exclude` directives. The `go.sum` file is *not* a lock file—it's purely cryptographic checksums for security verification.

For handling breaking changes, Go requires major versions v2+ to include the version in the import path: `import "github.com/example/mod/v2"`. This "semantic import versioning" treats different major versions as entirely separate modules that can coexist, solving the diamond dependency problem cleanly.

## Rust's module system power comes with an initial learning curve

Rust's module system is powerful but has historically confused newcomers because it violates expectations from other languages. **Creating a file doesn't automatically create a module**—you must explicitly declare modules with `mod`, then optionally alias them with `use`. This two-step process (declaration + aliasing) differs from most languages where import does both.

The 2018 edition changes addressed the infamous `mod.rs` problem. Previously, adding submodules to a module required restructuring files:
```
# Rust 2015: foo/ required foo/mod.rs
# Rust 2018: foo.rs + foo/ directory works together
```

This eliminated the "multiple tabs named mod.rs" problem in editors and reduced refactoring overhead when modules grow.

Rust's visibility system provides **five levels of granularity**: private (default), `pub(self)`, `pub(super)`, `pub(crate)`, `pub(in path)`, and `pub`. The most practically useful are:

- **Private (default)**: Current module and descendants only
- **`pub(crate)`**: Visible anywhere within the current crate—perfect for internal helpers that multiple modules need
- **`pub`**: Fully public API (if parent modules are also public)

Most Rust developers use only `pub` and private, with `pub(crate)` for crate-wide internal utilities. The finer-grained options (`pub(super)`, `pub(in path)`) are rarely needed but valuable for large codebases with deep module hierarchies.

Cargo workspaces enable monorepo patterns effectively. A workspace shares:
- Single `Cargo.lock` ensuring consistent dependency versions
- Unified `target/` directory for faster builds and shared compilation
- `[workspace.dependencies]` for declaring shared dependency versions once

The `pub use` pattern (re-exports) is essential for creating clean public APIs that hide internal structure. Libraries often have deep internal hierarchies but re-export key types at the crate root for ergonomic access.

## Python's import system reveals what not to do

Python's import system is considered one of the language's most confusing aspects, with StackOverflow questions like "Relative imports for the billionth time" accumulating thousands of votes. The fundamental problem is that **imports execute at runtime**, not compile time—when Python imports a module, it executes the file line-by-line, creating module attributes as it goes.

This execution model creates several cascading problems:

**Circular imports fail in ways that seem arbitrary.** If module A imports B and B imports A, whether it works depends on *what names* each module accesses and *when*. If B tries to access a name from A that hasn't been defined yet (because A is still loading), you get `AttributeError: partially initialized module`. The same code structure that works in compiled languages fails in Python.

**Relative imports work only in specific execution modes.** Running `python script.py` executes the file as `__main__` with `__package__` set to `None`, breaking all relative imports. You must run `python -m package.script` for relative imports to resolve correctly. This invocation-dependent behavior confuses developers constantly.

The evolution of `__init__.py` added further confusion. Pre-Python 3.3, it was required to mark directories as packages. PEP 420 introduced implicit namespace packages (directories without `__init__.py`), meaning it's now easier to create a namespace package than a regular package—but namespace packages behave differently, leading to subtle bugs.

**Key lessons for a new language:**
- Don't require runtime execution during import
- Make the execution mode irrelevant to import behavior
- Eliminate the possibility of circular import failures at compile time
- Have one clear rule for what constitutes a module/package

## Module design choices directly determine compilation speed

Go achieves legendary compilation speed through a key design decision: **transitive export information in object files**. When compiling a package P, Go writes not just P's exports but also type information for everything P imports. When compiling a client of P, the compiler reads *only* P's object file—never P's dependencies. This transforms potentially exponential dependency traversal into linear work.

Contrast this with C++'s header model, where `#include` literally copies header text into each translation unit. A header like `<iostream>` may expand to hundreds of thousands of lines after transitive includes. In a project with 1000 source files, the same headers are parsed, tokenized, and analyzed thousands of times. Measured inclusion times show `<regex>` taking ~500ms and `windows.h` ~200-500ms per translation unit.

C++20 modules address this by compiling to Binary Module Interfaces (BMI) that are read rather than textually included. However, benchmarks from WG21 reveal an interesting tradeoff: at low parallelism (4-8 threads), modular builds are consistently faster. At high parallelism (128 threads), non-modular builds can be faster because every translation unit is independent, enabling unlimited parallelism. Modules create DAG constraints where compilation must wait for dependencies.

Rust's approach differs: **query-based incremental compilation with the Salsa framework**. Instead of compiling phases (parse → type-check → codegen), Rust uses on-demand computation with memoization. Each query (e.g., `type_of(definition)`) has tracked dependencies. On recompilation:

1. Check if all inputs to each query are unchanged ("green")
2. If any input changed ("red"), recompute the query
3. After recomputation, check if the *result* changed—if not, mark green anyway ("early cutoff")

The early cutoff is powerful: changing whitespace might change the source text query but produce an identical AST, preventing downstream queries from recomputing. Salsa's "durability" system further optimizes by assigning stability levels—standard library queries rarely change, so they're checked less frequently.

**For your target of <100ms compile times:**
- Store transitive type information in compiled artifacts (Go's approach)
- Require explicit public signatures (no cross-module type inference)
- Enable package-level parallelism with known dependency DAG
- Disallow cycles (enables topological sort scheduling)
- Make unused imports errors (prevents dependency bloat)
- Require imports at file start (dependency graph known without parsing entire files)

## Visibility approaches reveal a fundamental design tension

The spectrum runs from convention-based (Python, Go) to keyword-based (Rust, Swift, Kotlin) with meaningful tradeoffs.

**Go's capitalization convention** (`ExportedName` vs `unexportedName`) is brilliantly simple: no keywords, visibility is immediately apparent, and tooling trivially determines access levels. The cost is severe: **renaming a public symbol breaks API**. You cannot change `ProcessData` to `processData` without breaking all consumers. This couples naming to visibility in ways other languages avoid.

**Rust's keyword approach** (`pub`, `pub(crate)`, etc.) decouples naming from visibility entirely. Refactoring private code is always safe; making something public is an explicit decision. The tradeoff is verbosity and cognitive load—five visibility levels can overwhelm newcomers, though most developers stick to `pub`/private with occasional `pub(crate)`.

**Python's underscore conventions** (`_internal`, `__mangled`) are pure convention with no enforcement. The `__double_underscore` prefix triggers name mangling (`_ClassName__name`), providing accidental-collision prevention rather than privacy. This matches Python's "consenting adults" philosophy but offers no compile-time guarantees.

**Swift's five levels** (`open`, `public`, `internal`, `fileprivate`, `private`) distinguish between "can call from outside module" (`public`) and "can also subclass/override from outside module" (`open`). This granularity helps library authors control extension points but adds complexity.

**Kotlin's `internal` modifier** provides module-level visibility that's particularly useful for multi-module applications—internal members are visible within the entire compilation unit (IntelliJ module, Maven project) but not outside. This hits a sweet spot between package-private and fully public.

**Testing implications matter.** Go tests in the same package access unexported members; Rust's `#[cfg(test)]` child modules inherit parent access; Swift's `@testable import` exposes `internal` members to tests. Your language should have a clear story for testing private code.

**Recommendation for your language:** Use explicit keywords (like Rust) but limit to three levels: private (default), `internal` (crate/module-wide), and `pub` (fully public). Avoid coupling names to visibility (unlike Go). Support test access to internal items via explicit mechanism.

## Package versioning requires choosing between flexibility and simplicity

The fundamental choice is between **lock files + "newest version" selection** (Cargo, npm) and **Minimum Version Selection** (Go).

Lock files provide exact reproducibility when committed but create two sources of truth (manifest + lock) that can drift. They enable rich version constraints (`^1.2.3`, `~1.2`, `>=1.0 <2.0`) and SAT-solver-based resolution. The downsides: merge conflicts in collaborative development, NP-complete resolution complexity, and the temptation to skip committing lock files for libraries.

MVS provides inherent reproducibility—the manifest alone determines versions. It uses polynomial-time resolution (no SAT solver), produces high-fidelity builds (users get versions close to what authors tested), and eliminates the lock file question entirely. The tradeoffs: less flexibility (no upper bounds), relies heavily on ecosystem semver discipline, and doesn't automatically pick up security patches.

**Diamond dependencies** illustrate the differences clearly. When A depends on B and C, and both depend on different versions of D:

- **npm**: Allows multiple D versions (nested in node_modules)—always finds a solution but risks type mismatches and bloat
- **Go**: MVS picks max(minimum requirements) within major version; different major versions are separate modules (`pkg/v2`)
- **Cargo**: Different major versions can coexist; within major version, picks one (may require backtracking)

**Vendoring** complements any versioning strategy for offline/reproducible builds. Go's `go mod vendor` and Cargo's `cargo vendor` both copy dependencies locally. For enterprise environments and supply chain security, vendoring support is essential.

**Recommendation for your language:** Implement MVS. The simplicity, polynomial complexity, and inherent reproducibility align with your compilation speed goals. Require major versions in module paths (like Go's `/v2`) to handle breaking changes. Include integrity hashes in manifests (like `go.sum`) for security. Support vendoring for offline builds.

## Modern approaches offer valuable innovations

**Deno's URL imports** pioneered decentralized dependency management but revealed limitations at scale: URLs clutter codebases, random server reliability is problematic, and no automatic semver deduplication occurs. Deno's solution—import maps centralizing URL-to-alias mappings—essentially reinvents a local manifest. The lesson: direct URL imports work for small scripts but require abstraction for larger projects.

**Zig's programmatic build system** (`build.zig`) treats build configuration as code. Dependencies are specified in `build.zig.zon` with content-addressed hashes for integrity. The key innovation is **automatic module discovery from import statements**—the build system analyzes your imports rather than requiring manual declaration. This reduces manifest maintenance overhead.

**Odin's collections system** organizes packages into clear categories: `core:` (standard library), `vendor:` (third-party), `shared:` (system-wide). This namespace prefix makes dependency provenance immediately clear without examining manifests.

**Gleam's qualified-by-default imports** prevent namespace pollution by requiring `module.function()` rather than allowing unqualified imports. Combined with explicit selective imports, this enforces clear code without verbosity.

**Key patterns from modern languages:**
- Content-addressed dependencies (hash verification)
- Automatic dependency discovery from imports
- Clear namespace prefixes indicating dependency origin
- Qualified access as default, unqualified as opt-in

## Concrete recommendations for your new language

Based on the requirements (fast compilation, parallel builds, clear dependencies, reproducible builds), here are specific design recommendations:

**Module structure:** Follow Go's "directory = module" with explicit `pub` markers. A directory with source files becomes a module. No separate manifest file per module—just a project-level one. Require package declaration at file top.

**Import syntax:** Single form only—`import "path/to/module"` or `import alias = "path/to/module"`. No glob imports. No unqualified imports by default. Consider allowing explicit selective imports: `import { Type, function } from "module"`.

**Visibility:** Three levels with explicit keywords:
- Private (default, no keyword)
- `internal` (visible within project/crate but not to dependents)  
- `pub` (fully public API)

Don't tie visibility to naming. Consider `internal` directories (like Go) for package-level encapsulation.

**Compilation model:**
- Store transitive type information in compiled artifacts
- Require explicit type signatures on all public items (no cross-module inference)
- Compile at module level (not file level)
- Unused imports are errors (as you specified)
- No cyclic imports (enables parallel compilation)
- Compute dependency graph from imports before full parsing

**Versioning:** Implement MVS with semantic import versioning:
- Manifest file (`project.mod` or similar) declares dependencies with minimum versions
- Major versions in import paths: `"github.com/user/repo/v2"`
- Content-addressed verification (SHA-256 hashes)
- Support vendoring via `vendor/` directory
- Include proxy/mirror system for reliability

**Testing story:** Test files in same directory access private items. Integration tests in separate directory access only public API. This mirrors Go's `_test.go` pattern.

**Error on common mistakes:**
- Compile error on import cycles (not runtime)
- Error on unused imports
- Error on importing private items
- Warning on overly-broad visibility (`pub` where `internal` would suffice)

This design prioritizes the <100ms compilation target by enabling maximum parallelism (no cycles, module-level units), minimal parsing for dependency resolution (imports at top, explicit signatures), and cached compilation artifacts (transitive exports). The visibility system is simpler than Rust's while avoiding Go's naming-coupled-to-visibility problem. MVS provides reproducibility without lock file complexity, aligning with the "obvious to understand" requirement.