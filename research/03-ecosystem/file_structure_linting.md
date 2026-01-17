# Compile-time and linter checks for file and project structure

**File structure enforcement represents an underexplored opportunity for language design.** Most languages treat project organization as purely conventional, leaving structure validation to external linters or team discipline. Yet languages that do enforce structure—Go's `internal/` directories, Java's package-to-directory mapping, Rust's module file requirements—report consistently better developer experiences for onboarding and cross-team collaboration. For Indent, targeting DevOps engineers who inherit and maintain scripts across teams, predictable project structure isn't a nice-to-have; it's essential for the "one obvious way" philosophy.

The research reveals a spectrum of enforcement strategies: from compile-time hard errors (Java's package requirements) to purely advisory warnings (ESLint's file-naming rules). The optimal approach for Indent combines **compile-time enforcement for structure that affects semantics** (module resolution, visibility) with **configurable linter rules for conventions** (naming, organization). This hybrid ensures correctness while accommodating legitimate variation across project types.

## Languages with enforced file structure: what works

### Go: Convention-driven compilation

Go's approach is pragmatic: the compiler enforces minimal structure while conventions handle the rest. The enforced rules are:

**Package = directory**: All `.go` files in a directory must declare the same package. This isn't a convention—it's a compile error to mix packages in one directory. The payoff is immediate: import paths map directly to filesystem paths, enabling the legendary "go get imports work instantly" experience.

**Test file conventions**: Files ending in `_test.go` are compiled only during testing. The compiler enforces this—you cannot import `_test.go` symbols from non-test code. This separation enables parallel test compilation without polluting the production binary.

**`internal/` directory**: Since Go 1.4, any package in a directory named `internal` can only be imported by packages rooted at the parent of `internal`. This is a hard compiler error:

```
// In myproject/internal/secrets/keys.go
package secrets

func LoadKey() []byte { ... }

// In external/package/main.go - COMPILE ERROR
import "myproject/internal/secrets"  // cannot access internal package
```

The `internal/` rule solves a real problem: Go's capitalization-based visibility only offers public/private at the symbol level, not the package level. `internal/` provides package-level encapsulation without language changes.

**Build constraints via filename**: Files like `network_linux.go` compile only on Linux. The compiler pattern-matches filenames against known OS/architecture suffixes. This enables platform-specific code without preprocessor conditionals.

What Go doesn't enforce: file naming (beyond `_test.go` and build constraints), directory depth, organization within packages, or consistency across packages. Teams develop their own conventions, leading to significant structural variation in the ecosystem.

### Rust: Explicit module declaration

Rust's module system was famously confusing until the 2018 edition. The original requirement—`mod.rs` files for directory modules—created "walls of tabs named mod.rs" in editors. The 2018 changes improved ergonomics while maintaining enforcement:

**File ↔ module correspondence**: A file `foo.rs` or directory `foo/` creates module `foo`, but you must declare it in the parent with `mod foo;`. This explicit declaration is a compile-time requirement—orphaned files are ignored (with a warning), not silently included.

```rust
// In lib.rs
mod handlers;  // Requires handlers.rs or handlers/mod.rs

// handlers.rs without this declaration? Warning: "file is not included"
```

**Directory modules**: A directory `handlers/` requires either `handlers.rs` or `handlers/mod.rs` as the module root. The 2018 edition made `handlers.rs` alongside `handlers/` work, reducing the `mod.rs` proliferation.

**Visibility is path-based**: `pub(crate)`, `pub(super)`, `pub(in path)` enforce visibility relative to the module tree. The module tree is derived from file structure, making filesystem organization semantically meaningful.

What Rust doesn't enforce: file naming conventions (snake_case is convention, not requirement), directory depth, test file location (though `tests/` for integration tests is convention), or specific required files beyond `Cargo.toml`.

### Java: Package = directory, strictly

Java's enforcement is the strictest of mainstream languages:

**Package declaration must match directory path**: A file declaring `package com.example.util;` must live in `com/example/util/`. This isn't advisory—the compiler refuses to find the class otherwise.

```java
// File: src/com/example/util/StringHelper.java
package com.example.util;  // MUST match directory structure

public class StringHelper { ... }

// File in wrong directory: src/wrong/StringHelper.java
package com.example.util;  // COMPILE ERROR: package does not exist
```

**Source roots are configurable**: Build tools (Maven, Gradle) declare source roots (`src/main/java/`, `src/test/java/`), but within those roots, directory structure is non-negotiable.

**Convention-as-enforcement via tooling**: Maven's "convention over configuration" made specific directory structures (`src/main/java`, `src/main/resources`, `src/test/java`) nearly universal. While not compiler-enforced, build tool adoption made them effectively mandatory.

The Java approach demonstrates both benefits and costs. Benefits: any Java developer can navigate any Java project; IDE tooling is trivial to implement; package membership is unambiguous. Costs: verbose directory trees for simple projects; package renaming requires filesystem restructuring; namespace clashes require organizational coordination.

### Python: From chaos to optional structure

Python's evolution illustrates the progression from "no rules" to "configurable enforcement":

**`__init__.py` requirement (pre-3.3)**: Directories needed `__init__.py` to be importable as packages. This was compile-time enforced (ImportError without it).

**Implicit namespace packages (3.3+)**: PEP 420 made `__init__.py` optional, enabling namespace packages that span multiple directories. This increased flexibility but added confusion—developers now must understand two package types with different behaviors.

**Linter-based structure enforcement**: Tools like `flake8-import-order`, `isort`, and project-specific rules enforce conventions. Ruff's `I001` (import sorting) and `INP001` (implicit namespace check) demonstrate modern linter-based structure checking.

**`pyproject.toml` standardization**: PEP 517/518 established a required project root file, ending the `setup.py`/`setup.cfg`/`requirements.txt` fragmentation. This is tooling-enforced, not language-enforced.

Python's lesson: runtime import semantics make compile-time structure enforcement difficult. Linters fill the gap, but enforcement is opt-in and varies across projects.

### Other notable approaches

**Elm**: Enforces `module Name exposing (...)` must match filepath. File `User/Profile.elm` must declare `module User.Profile`. Hard compiler error otherwise. Combined with enforced semantic versioning for packages, Elm achieves remarkable ecosystem consistency.

**Haskell**: Module names traditionally match filepaths, but enforcement varies by build system. Cabal and Stack enforce it for project files; GHCi is more permissive for REPL use.

**Deno**: Enforces URL-style imports with optional import maps. No directory structure requirements—everything is URL-addressed. Structure emerges from convention, not enforcement.

**Zig**: Build system (`build.zig`) is code, enabling programmatic structure validation. No compiler-level enforcement, but the pattern of "structure as code" enables project-specific rules.

## What could be checked: a taxonomy

### Semantic structure (affects compilation)

These directly affect how code compiles and should be **compile-time enforced**:

| Check | Description | Go | Rust | Java |
|-------|-------------|----|----|------|
| Module ↔ directory mapping | Module path matches filesystem | Implicit | Declared | Enforced |
| Internal/private directories | Visibility boundaries | `internal/` | `pub(crate)` | Package-private |
| Test file separation | Test code isolated | `_test.go` | `#[cfg(test)]` | `src/test/` |
| Platform-specific files | Conditional compilation | `_linux.go` | `#[cfg]` | Preprocessor |
| Circular dependency detection | Module cycles prevented | No cycles | No cycles | Allowed |

For Indent, semantic structure should follow Go's implicit mapping with Rust's explicit visibility:
- Directory = module (like Go)
- `internal/` for project-private modules (like Go)
- `_test.idt` for test files (like Go's `_test.go`)
- No circular module dependencies (compile error)

### Naming conventions (affects readability)

These affect human readability and tooling but not compilation. Should be **linter-enforced with configurable severity**:

| Convention | Common choices | Tooling impact |
|------------|---------------|----------------|
| File naming | `snake_case.ext`, `PascalCase.ext`, `kebab-case.ext` | Editor search, imports |
| Directory naming | `snake_case`, `lowercase` | Module names in code |
| Type ↔ file correspondence | One public type per file | Navigation, refactoring |
| Acronym handling | `HTTPServer` vs `HttpServer` | Consistency |

Go's choice of lowercase package names is enforced by the compiler (import paths are lowercased). Rust's snake_case for files is convention only (compiler warns but compiles).

For Indent targeting Python familiarity: **snake_case for files and directories**. This should be a linter warning (not error) with project-level override capability.

### Required files (affects tooling)

Projects need certain files for tooling to work:

| File | Purpose | Enforcement level |
|------|---------|-------------------|
| `project.mod` / `Cargo.toml` | Manifest | Compile error if missing |
| `README.md` | Documentation | Linter warning |
| `LICENSE` | Legal | Linter warning (for public packages) |
| `.indent/config.toml` | Local config | Optional |

The manifest is essential—without it, the compiler doesn't know project boundaries. Documentation files are valuable but shouldn't block compilation.

### Organization patterns (affects maintainability)

Higher-level structural patterns that affect long-term maintainability:

| Pattern | Example | Enforcement approach |
|---------|---------|---------------------|
| Flat vs nested | `handlers.idt` vs `handlers/users.idt` | Advisory only |
| Separation of concerns | `models/`, `services/`, `handlers/` | Advisory only |
| Test co-location | Tests next to source vs separate `tests/` | Configurable preference |
| Binary vs library roots | `src/bin/`, `src/lib/` | Compiler-inferred |
| Examples location | `examples/` | Convention |

These are too project-specific for universal enforcement. The linter should **recognize common patterns** and offer consistency suggestions, not mandate specific structures.

## Compile-time vs linter vs both: the enforcement spectrum

### Hard errors: semantic correctness

Compile-time errors should be reserved for structure that affects program semantics:

1. **Module resolution**: If directory structure determines module paths (which it should for compilation speed), mismatches are errors.

2. **Visibility enforcement**: `internal/` directories can't be imported from outside—this is a capability boundary, not a suggestion.

3. **Circular dependencies**: Cross-module cycles prevent compilation (topological sort fails). Some languages allow cycles (Java) but pay performance costs.

4. **Manifest presence**: Can't compile without knowing project boundaries and dependencies.

5. **Platform file exclusion**: Files ending in `_windows.idt` shouldn't compile on Linux—incorrect structure means incorrect output.

The developer experience for hard errors must be excellent:

```
error[E0401]: cannot import internal module from external package
  --> external/lib.idt:3:1
   |
 3 | import myproject.internal.secrets
   |        ^^^^^^^^^^^^^^^^^^^^^^^^^^ this module is internal to 'myproject'
   |
   = help: internal modules can only be imported by 'myproject' and its submodules
   = note: see https://indent-lang.org/guide/visibility for visibility rules
```

### Warnings: style consistency

Linter warnings for structure that affects readability but not correctness:

1. **Naming conventions**: `my-file.idt` when project uses `snake_case`
2. **Missing documentation files**: No README in publishable package
3. **Unusual organization**: 50 files in one directory (suggest subdirectories)
4. **Unused modules**: File exists but no path leads to it
5. **Overly deep nesting**: `src/a/b/c/d/e/f/file.idt` suggests restructuring

Warnings should be:
- Suppressible per-file: `# indent:allow(deep_nesting)`
- Configurable per-project: `[lint] allow = ["naming.kebab-case"]`
- Groupable: `indent lint --warnings=structure` vs `indent lint --warnings=all`

### Info/suggestions: best practices

Non-blocking suggestions that help but don't indicate problems:

1. **"Consider extracting"**: Large file might benefit from splitting
2. **"Common pattern detected"**: This looks like a CLI app, consider `src/bin/` structure
3. **"Test file detected"**: `utils_test.idt` uses test naming; ensure it's in test configuration

These should be opt-in (`indent lint --pedantic`) or IDE-only, not in default lint output.

### The IDE experience

File structure rules significantly impact IDE tooling:

**Compile-time rules enable**:
- Instant "go to definition" (paths are deterministic)
- Accurate "find references" (visibility is known)
- Reliable refactoring (rename module = rename directory)
- Fast project scanning (structure implies content)

**Linter rules enable**:
- On-save structure validation
- Quick-fix suggestions ("rename to match convention")
- Project-wide consistency reports
- Configuration-aware highlighting

**What to avoid**:
- Rules that require full project analysis for file-level diagnostics
- Structure validation that blocks other IDE features
- Inconsistency between `indent lint` and IDE warnings

The LSP server should expose structure violations as diagnostics, with severity matching the CLI lint output.

## Benefits and drawbacks of structure enforcement

### Benefits

**Onboarding acceleration**: New team members navigate unfamiliar codebases faster when structure is predictable. Studies of Go adoption consistently cite "I can understand any Go project" as a top benefit. For DevOps teams with high rotation and many inherited scripts, this matters enormously.

**Tooling reliability**: IDEs, linters, and documentation generators work better with predictable structure. rust-analyzer's "find module definition" works because Rust enforces module declaration; Python's equivalent is probabilistic.

**Reduced bike-shedding**: Teams spend less time debating structure when the language decides. Go's gofmt is the famous example for formatting; structure enforcement extends this benefit to organization.

**Implicit documentation**: Directory structure communicates architecture. `internal/` says "don't depend on this"; `handlers/` says "HTTP endpoints here". This only works if structure is consistent.

**Compilation optimization**: Enforced structure enables build system optimizations. Go's "package = directory = compilation unit" enables trivially parallel builds. Languages with flexible structure must do more analysis to determine build order.

### Drawbacks

**Migration friction**: Existing codebases may not match enforced conventions. Converting a large project to new structure rules is nontrivial work with no functional benefit—pure chore.

**One size doesn't fit all**: A CLI tool, a library, a web service, and a script have different structural needs. Rigid enforcement may make some project types awkward.

**Ecosystem fragmentation**: If structure rules are configurable, different projects choose differently, eroding the "predictable structure" benefit.

**Learning curve**: Beginners must learn structure rules before writing working code. Python's "just create a file and run it" simplicity partly explains its educational popularity.

**False positives**: Overly aggressive structure linting creates noise. Developers ignore or disable linters that cry wolf.

### Mitigating drawbacks

**Gradual adoption**: Introduce structure rules as warnings first. `indent migrate --check` reports violations without failing; `indent migrate --fix` automates fixes where possible.

**Project type templates**: `indent new --template=cli` vs `indent new --template=library` generates appropriate structure. Templates codify best practices without universal mandates.

**Escape hatches**: Allow per-project configuration that disables specific rules with explicit justification. The default should be opinionated; overrides should be possible but documented.

**Excellent error messages**: When structure rules block progress, the error must explain why and how to fix it. "File not found" is useless; "Module 'handlers' not found. Did you mean to create 'handlers.idt' or 'handlers/' directory?" is actionable.

## Recommendations for Indent

Based on the research and Indent's design goals (DevOps focus, Python familiarity, "one obvious way" philosophy), here are concrete recommendations:

### Compile-time enforcement (hard errors)

1. **Directory = module**: A directory `foo/` containing `.idt` files creates module `foo`. No declaration needed (unlike Rust). This enables Go-speed compilation.

2. **`internal/` visibility**: Any `internal/` directory is only importable by the parent module and its children. Hard compiler error otherwise.

3. **No circular modules**: Module import cycles are compile errors. The error message should show the cycle path.

4. **Manifest required for projects**: `project.mod` required in project root. Single-file scripts can run without one (`indent run script.idt`), but multi-file projects need it.

5. **Test file convention**: `_test.idt` suffix marks test files. These are:
   - Compiled only during `indent test`
   - Can access `internal` symbols of their parent module
   - Cannot be imported from non-test code

6. **Platform suffixes**: `_linux.idt`, `_darwin.idt`, `_windows.idt` compile conditionally. Unknown suffixes are errors (catches typos like `_linuxx.idt`).

### Linter enforcement (warnings by default)

1. **File naming**: `snake_case.idt` is the default convention. Violations are warnings.
   ```toml
   # project.mod - override if needed
   [lint.naming]
   files = "kebab-case"  # or "any" to disable
   ```

2. **Directory naming**: `snake_case` directories for modules. Same override mechanism.

3. **Single public item per file** (optional): When one file exports multiple public items, suggest splitting. Disabled by default, enable for large projects.
   ```toml
   [lint.structure]
   single-public-item = "warn"
   ```

4. **Missing README for publishable packages**: If `publish = true` in manifest, warn without README.

5. **Deep nesting warning**: More than 4 directory levels triggers suggestion to restructure. Configurable threshold.

6. **Orphaned files**: Files not reachable from module tree get warnings (might indicate typos or forgotten code).

7. **Large module warning**: Module with >20 files suggests subdirectories. Configurable threshold.

### Recommended project structures

Rather than mandating one structure, document recommended patterns:

**CLI application**:
```
myapp/
├── project.mod
├── main.idt           # Entry point
├── commands/          # CLI commands
│   ├── init.idt
│   └── run.idt
├── internal/          # Not for external use
│   └── config.idt
└── tests/             # Integration tests
    └── cli_test.idt
```

**Library**:
```
mylib/
├── project.mod
├── lib.idt            # Public API re-exports
├── types.idt          # Core types
├── internal/          # Implementation details
│   └── helpers.idt
└── tests/
    └── integration_test.idt
```

**DevOps script collection**:
```
scripts/
├── project.mod
├── deploy.idt
├── backup.idt
├── shared/            # Common utilities
│   └── ssh.idt
└── tests/
    └── shared_test.idt
```

### Integration with `indent lint`

The lint command should have clear structure-related options:

```bash
# Full lint including structure
indent lint

# Structure checks only
indent lint --category=structure

# Ignore structure warnings
indent lint --ignore=structure

# Strict mode (warnings become errors)
indent lint --strict

# Show available structure rules
indent lint --list-rules | grep structure
```

Rule configuration in `project.mod`:

```toml
[lint]
# Disable specific rules
allow = [
    "structure.deep-nesting",  # We have legitimate deep hierarchies
]

# Upgrade warnings to errors for CI
deny = [
    "structure.orphaned-files",  # Catch forgotten code
]

# Rule-specific configuration
[lint.structure]
max-nesting-depth = 6
max-files-per-module = 30
```

### Migration tooling

For projects adopting Indent from other languages:

```bash
# Check what would need to change
indent structure check

# Auto-fix what can be fixed (renames, directory creation)
indent structure fix

# Generate migration report
indent structure report > migration.md
```

The migration tool should:
- Detect current structure patterns (Python-style, Go-style, ad-hoc)
- Suggest Indent-compatible reorganization
- Offer `--dry-run` mode
- Create git commits with sensible messages
- Handle test file identification

### What we explicitly don't enforce

Following the "one obvious way" philosophy means making decisions, but some decisions should be left to projects:

1. **Flat vs nested organization**: Small scripts are fine in one directory; large applications need hierarchy. No universal rule.

2. **Test location**: `_test.idt` next to source or in `tests/` directory. Both are valid; project chooses.

3. **Example location**: `examples/` is convention, not requirement.

4. **Documentation location**: `docs/` is convention. The linter might suggest it for larger projects.

5. **Binary entry point naming**: `main.idt`, `app.idt`, `cli.idt` are all fine. The manifest declares the entry point.

## Comparison table: Indent vs existing languages

| Aspect | Go | Rust | Java | Python | **Indent** |
|--------|----|----|------|--------|----------|
| **Dir = module** | Implicit | Declared | Enforced | Convention | Implicit |
| **Internal visibility** | `internal/` dir | `pub(crate)` | Package | `_` prefix | `internal/` dir |
| **Test files** | `_test.go` | `#[cfg(test)]` | `src/test/` | Convention | `_test.idt` |
| **Circular deps** | Error | Error | Allowed | Runtime | Error |
| **File naming** | Convention | Convention (warn) | Any | Convention | Lint warning |
| **Required manifest** | `go.mod` | `Cargo.toml` | `pom.xml` | `pyproject.toml` | `project.mod` |
| **Platform files** | `_os.go` | `#[cfg]` | N/A | `platform` | `_os.idt` |

## Implementation notes

### Compiler integration

Structure enforcement belongs in the module resolution phase:

1. **Scan project directory** at compilation start
2. **Build module tree** from directory structure
3. **Validate internal boundaries** during import resolution
4. **Reject cycles** during topological sort
5. **Store structure metadata** in compiled artifacts for dependent builds

This adds minimal compilation overhead (filesystem scanning is fast) while enabling downstream optimizations.

### Linter architecture

Structure linting can be separate from type checking:

```
Source files → Structure linter (fast, pre-parse)
            → Parser → Type checker → Semantic linter
```

The structure linter runs first and can fail fast without full parsing. This means `indent lint --category=structure` is nearly instantaneous even for large projects.

### Caching considerations

Structure rarely changes. The lint cache should:
- Invalidate only on file creation/deletion/rename
- Not invalidate on content changes (for structure rules)
- Share state with the compilation cache

This makes structure linting effectively free in watch mode.

## Summary of recommendations

| Category | Enforcement | Rationale |
|----------|-------------|-----------|
| Directory = module | Compile error | Enables fast compilation |
| `internal/` visibility | Compile error | Security boundary |
| Circular dependencies | Compile error | Correctness + build speed |
| Test file convention | Compile error | Correct test isolation |
| Platform file convention | Compile error | Correct conditional compilation |
| Manifest required | Compile error | Project identity |
| File naming convention | Lint warning | Readability, not correctness |
| Deep nesting | Lint warning | Maintainability suggestion |
| Orphaned files | Lint warning | Catches mistakes |
| Missing README | Lint info | Documentation encouragement |
| Module size | Lint info | Scaling suggestion |

The key insight: **enforce what affects compilation semantics; lint what affects human readability**. This gives Indent the "predictable structure" benefits of Go and Java while maintaining the flexibility DevOps scripts need. Excellent error messages and migration tooling ensure that enforcement helps rather than hinders adoption.
