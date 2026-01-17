# Formatter and Linter Design for the Indent Language

The tooling landscape has converged on a clear lesson: **opinionated, fast, zero-configuration tools win adoption**. gofmt's "one true style" philosophy, Black's "uncompromising" approach, and Ruff's 100x speed advantage over Pylint all demonstrate that developers value consistency and speed over configurability. For Indent, this research informs a cohesive formatter and linter strategy that aligns with the language's "one obvious way" philosophy.

---

## 1. Formatter Philosophy: Lessons from the Field

### gofmt: The Gold Standard for Opinionated Formatting

gofmt established the paradigm that Indent should follow. As [Rob Pike famously stated](https://go.dev/blog/gofmt): "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."

The key design decisions that made gofmt successful:

| Design Choice | Rationale | Outcome |
|---------------|-----------|---------|
| Zero configuration options | Eliminates style debates entirely | Universal adoption |
| Ships with the language | Formatting is not optional | Consistent ecosystem |
| Single-pass algorithm | No line-width fitting or backtracking | Sub-millisecond performance |
| Canonical output | Same input always produces same output | Reliable CI integration |

[Russ Cox emphasizes](https://research.swtch.com/gofmt) that gofmt "doesn't enforce style, in the sense of The Elements of Programming Style... The only style that gofmt imposes is consistency." This distinction matters: the formatter handles layout, not quality.

**Benefits documented by the Go team:**
- Code is easier to read when formatting is uniform
- Mechanical changes produce minimal diffs
- Code reviews focus on logic, not style
- Enables powerful source-to-source transformation tools

The [Go Style Guide](https://google.github.io/styleguide/go/) notes: "When you open an unfamiliar Go program, your brain doesn't get distracted, even subconsciously, about why that brace is in the wrong place; you can focus on the code, not the formatting."

### rustfmt: The Configurability Cautionary Tale

rustfmt provides a counterexample. While [technically capable](https://github.com/rust-lang/rustfmt), it has accumulated [dozens of configuration options](https://rust-lang.github.io/rustfmt/) that:

- Fragment the ecosystem (different projects use different settings)
- Create decision fatigue for new projects
- Complicate CI/CD setup
- Generate inconsistency in open-source contributions

The [rustfmt team documented](https://users.rust-lang.org/t/does-anyone-use-a-customized-configuration-for-rustfmt/66420) controversies around defaults. For example, struct literal formatting: "people who think of struct literals like function calls (single-line by default) finding it odd, whereas people who think of struct literals like struct declarations with values find it more natural."

**Lesson for Indent:** Configuration options create maintenance burden without proportional benefit. Every option is a choice developers must make and maintain.

### Black: "Uncompromising" as a Feature

[Black's philosophy](https://github.com/psf/black) explicitly trades flexibility for productivity:

> "By using Black, you agree to cede control over minutiae of hand-formatting. In return, Black gives you speed, determinism, and freedom from pycodestyle nagging about formatting."

Black's adoption by major projects (pytest, Django, SQLAlchemy, pandas, Dropbox, Mozilla) validates the approach. [The name itself](https://black.readthedocs.io/) references Henry Ford: "Any customer can have a car painted any colour that he wants so long as it is black."

Key Black design decisions:

| Feature | Implementation | Rationale |
|---------|----------------|-----------|
| Minimal options | Only line length and string quote style | Reduces bikeshedding |
| AST validation | Verifies reformatted code produces identical AST | Safety guarantee |
| Stable output | No major style changes after v1.0 | Predictable diffs |
| PEP 8 compliant | Compatible with existing Python style guides | Easy adoption |

### Prettier: Multi-Language Complexity

[Prettier](https://prettier.io/) demonstrates both the benefits and costs of multi-language support. Its [plugin architecture](https://deepwiki.com/prettier/prettier) enables formatting JavaScript, TypeScript, CSS, HTML, JSON, Markdown, and more through a unified interface.

The architecture follows a pipeline pattern:
1. Configuration resolution
2. Parsing to AST
3. AST transformation
4. Printing to intermediate representation (Doc)
5. Final string generation

However, Prettier's [Wadler-Lindig algorithm](https://research.swtch.com/gofmt) for line-width fitting is computationally expensive. It explores multiple layouts to find optimal line breaks, creating:
- 2-10x slower performance than single-pass formatters
- Complexity in implementation (10,000-30,000 LOC)
- Occasionally surprising output for edge cases

**Indent's advantage:** With significant whitespace, many of Prettier's line-breaking decisions become unnecessary. The programmer's indentation decisions are preserved.

---

## 2. What the Formatter Should Handle

Based on research and Indent's language characteristics, the formatter should handle:

### Must Handle (Core Formatting)

| Category | Specifics | Rationale |
|----------|-----------|-----------|
| **Indentation** | Enforce 4-space indentation | Core language identity |
| **Whitespace around operators** | `a + b` not `a+b` | Readability |
| **Trailing whitespace** | Remove | Clean diffs |
| **Final newline** | Ensure single trailing newline | POSIX compatibility |
| **Blank line normalization** | Max 2 consecutive blank lines | Prevent excessive spacing |
| **Operator spacing** | Consistent spacing around `=`, `->`, `|>` | Visual consistency |

### Should Handle (Style Normalization)

| Category | Specifics | Rationale |
|----------|-----------|-----------|
| **Import ordering** | Stdlib, external packages, local modules (alphabetically within groups) | Deterministic ordering reduces merge conflicts |
| **Trailing commas** | Always in multi-line constructs, never in single-line | Cleaner diffs for additions |
| **String quote normalization** | Double quotes preferred (unless containing double quotes) | Consistency |
| **Collection literal formatting** | Consistent line breaks for long literals | Readability |

### Should NOT Handle (Leave to Programmer)

| Category | Rationale |
|----------|-----------|
| **Line length enforcement** | Indent's significant whitespace means programmers make explicit break decisions |
| **Expression grouping** | When to break long expressions is semantic |
| **Comment placement** | Comments are inherently semantic |
| **Blank line placement** | Logical grouping is semantic |

### Import Ordering Specification

```
# Order: stdlib, external, local (blank line between groups)
# Alphabetical within each group

import json
import os

import http
import sqlalchemy

import myproject.config
import myproject.handlers
```

This matches Python conventions (PEP 8, isort) and Go's goimports behavior.

---

## 3. Linter Philosophy: Lessons from Established Tools

### Clippy: The Gold Standard for Categorized Lints

[Clippy](https://github.com/rust-lang/rust-clippy) provides the model for comprehensive linting with 750+ lints organized into well-defined categories. The [Clippy documentation](https://doc.rust-lang.org/stable/clippy/lints.html) defines:

| Category | Default Level | Purpose |
|----------|---------------|---------|
| **correctness** | deny | Outright wrong or useless code; free of false positives |
| **suspicious** | warn | Code that is "sus" and should likely be fixed |
| **complexity** | warn | Code that can be simplified |
| **perf** | warn | Performance improvements the compiler can't optimize automatically |
| **style** | warn | Idiomatic code patterns |
| **pedantic** | allow | Aggressive lints prone to false positives |
| **nursery** | allow | Experimental or buggy lints |
| **restriction** | allow | Case-by-case lints that may contradict others |
| **cargo** | warn | Cargo.toml improvements |

The key insight is **graduated defaults**: correctness lints abort compilation (no false positives guarantee), while pedantic lints are opt-in for projects wanting stricter standards.

### Ruff: Speed as a Feature

[Ruff](https://github.com/astral-sh/ruff) demonstrates that linter speed is not a nice-to-have but a fundamental requirement:

> Ruff is 10-100x faster than existing linters (like Flake8) and formatters (like Black).

[Performance comparisons](https://trunk.io/learn/comparing-ruff-flake8-and-pylint-linting-speed) show:
- CPython codebase: Traditional linters take 30-60 seconds; Ruff takes 300 milliseconds
- dagster (250k LOC): Pylint takes 2.5 minutes parallelized; Ruff takes 0.4 seconds

This speed difference changes how developers use the tool:
- Format-on-save becomes viable
- Pre-commit hooks don't slow development
- CI pipelines run faster
- Large monorepos remain manageable

[Ruff's architecture](https://pythonspeed.com/articles/pylint-flake8-ruff/) achieves this through:
- Rust implementation (vs Python for Pylint/Flake8)
- Doing only necessary work for each check
- Single-pass analysis where possible
- Efficient memory representation

**Trade-off:** Ruff is not extensible by Python programmers (unlike Pylint/Flake8). For Indent, this is acceptable since the linter ships with the language.

### ESLint: Configurability Complexity

[ESLint](https://eslint.org/) demonstrates the costs of extreme configurability:

- The eslintrc configuration system was ["a significant source of complexity"](https://eslint.org/blog/2022/08/new-config-system-part-2/)
- Plugin ecosystem fragmentation: "wild west in the ecosystem where plugins were all doing different things"
- The [v9.0.0 transition](https://eslint.org/blog/2025/05/eslint-v9.0.0-retrospective/) revealed that bundling breaking changes created ecosystem disruption
- [Flat config](https://eslint.org/blog/2025/03/flat-config-extends-define-config-global-ignores/) was introduced to simplify configuration

[2025 developments](https://eslint.org/blog/2026/01/eslint-2025-year-review/) show continued evolution: multi-language support (CSS, HTML) and [AI integration via MCP](https://eslint.org/blog/2026/01/eslint-2025-year-review/).

**Lesson for Indent:** Start simple, add complexity only when needed. ESLint's decade of evolution shows that configurability creates long-term maintenance burden.

### golint/staticcheck: Focused Approach

Go's linting ecosystem takes a more focused approach:
- **golint**: Deprecated, focused on style suggestions
- **staticcheck**: Currently recommended, focuses on correctness and performance
- **go vet**: Ships with Go, catches obvious bugs

This layered approach works because each tool has a clear purpose. staticcheck doesn't try to be everything.

---

## 4. What the Linter Should Check

### Category 1: Correctness (deny-by-default)

Zero false positives guarantee. These abort compilation.

| Check | Example | Rationale |
|-------|---------|-----------|
| Unreachable code after return | `return x; print("never runs")` | Dead code is always a bug |
| Unused variables | `val x = 5` (never used) | Indicates incomplete code |
| Unused imports | `import json` (never used) | Dependency bloat |
| Shadowing of keywords | `val match = 5` | Future keyword conflicts |
| Invalid pattern matching | Non-exhaustive match on closed enum | Runtime panic prevention |
| Division by zero (constant) | `x / 0` | Guaranteed runtime error |
| Invalid format strings | `f"Hello {nonexistent}"` | Guaranteed runtime error |

### Category 2: Suspicious (warn-by-default)

Likely bugs but could be intentional.

| Check | Example | Rationale |
|-------|---------|-----------|
| Comparison to same variable | `x == x` | Usually a typo |
| Unused function parameters | `func foo(x, y): return x` | Incomplete implementation |
| Empty blocks | `if cond:` with no body | Incomplete code |
| Redundant `else` after `return` | `if x: return 1 else: return 2` | Style, but suspicious |
| Floating-point equality | `x == 0.0` | Usually should use epsilon |
| Self-assignment | `x = x` | Usually a typo |
| Condition always true/false | `if True:` | Dead code or debugging leftover |

### Category 3: Performance (warn-by-default)

Patterns the compiler can't optimize.

| Check | Example | Rationale |
|-------|---------|-----------|
| Repeated string concatenation in loop | `s = s + item` | Use StringBuilder pattern |
| Map lookup followed by insert | `if key not in d: d[key] = val` | Use `d.setdefault()` or equivalent |
| Unnecessary allocations | `list[0:len(list)]` | Just use `list` |
| Inefficient collection iteration | `for i in range(len(list))` | Use `for item in list` |
| Collecting iterator to re-iterate | `.to_list().iter()` | Just iterate directly |

### Category 4: Style (warn-by-default)

Idiomatic Indent code.

| Check | Example | Rationale |
|-------|---------|-----------|
| Naming conventions | `camelCase` variable | Should be `snake_case` |
| Boolean literal comparison | `if x == True:` | Use `if x:` |
| Negated conditions | `if not x == y:` | Use `if x != y:` |
| Single-use comprehension | `[x for x in items][0]` | Use `items[0]` |
| Redundant type annotations | `val x: int = 5` | Type is obvious |
| Non-descriptive names | `val a = fetch_users()` | Use `val users = fetch_users()` |

### Category 5: Complexity (warn-by-default)

Simplification opportunities.

| Check | Example | Rationale |
|-------|---------|-----------|
| Deeply nested conditionals | 5+ levels of `if` | Extract functions |
| Long functions | 100+ lines | Extract functions |
| High cyclomatic complexity | 20+ paths | Simplify logic |
| Redundant pattern arms | Covered by previous arm | Remove dead code |
| Manual iteration patterns | `filter` + `map` manually | Use iterator methods |

### Category 6: Security (warn-by-default)

Common security issues.

| Check | Example | Rationale |
|-------|---------|-----------|
| Hardcoded credentials | `password = "secret"` | Use environment/secrets |
| SQL injection patterns | String concatenation in queries | Use parameterized queries |
| Path traversal | User input in file paths | Validate paths |
| Unbounded input | No size limits on user data | Set explicit limits |

### Category 7: Pedantic (allow-by-default)

Opt-in for strict projects.

| Check | Example | Rationale |
|-------|---------|-----------|
| Missing documentation | Public function without doc | Enforce documentation |
| Magic numbers | `if count > 42:` | Use named constants |
| Implicit type conversions | Even safe ones | Prefer explicit |
| Single-character names | `for x in items:` | Some prefer `for item in items:` |

---

## 5. Integration Considerations

### Speed Requirements

Based on [LSP integration research](https://docs.rubocop.org/rubocop/usage/lsp.html) and developer experience studies:

| Operation | Target Latency | Rationale |
|-----------|----------------|-----------|
| Format single file | < 10ms | Format-on-save must be imperceptible |
| Format project | < 500ms | Pre-commit hooks |
| Lint single file | < 50ms | Real-time diagnostics in editor |
| Lint project | < 5s | CI pipeline speed |
| Incremental re-lint | < 100ms | After file save |

The existing tooling research notes that rust-analyzer achieves sub-100ms response times through demand-driven computation. Similar principles apply:
- **Lazy evaluation**: Only analyze what's needed
- **Caching**: Don't recompute unchanged files
- **Parallelization**: Use all available cores

### LSP Integration

The formatter and linter must integrate with the language server:

```
┌─────────────────────────────────────────────────────────────┐
│                    Editor (VS Code, Neovim, etc.)           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    indent lsp                                │
│  ├── textDocument/formatting → calls formatter              │
│  ├── textDocument/publishDiagnostics → calls linter         │
│  └── codeAction → applies quick fixes                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────┐     ┌─────────────────┐
│  indent fmt     │     │  indent lint    │
│  (formatter)    │     │  (linter)       │
└─────────────────┘     └─────────────────┘
```

Key requirements:
- **Format-on-save**: LSP's `textDocument/formatting` triggers formatter
- **Real-time diagnostics**: Lint as user types (debounced)
- **Quick fixes**: `codeAction` provides automated fixes
- **Range formatting**: Format only selection

### CI/CD Integration

```yaml
# Example CI workflow
lint:
  - indent fmt --check      # Fail if not formatted
  - indent lint --deny-warnings  # Treat warnings as errors
```

Key behaviors:
- `indent fmt --check`: Exit non-zero if formatting would change files
- `indent lint --deny-warnings`: Promote all warnings to errors for CI
- Machine-readable output: JSON for tooling integration
- GitHub Actions / GitLab CI integration out of the box

### Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: indent-fmt
        name: Format Indent files
        entry: indent fmt
        language: system
        types: [indent]
      - id: indent-lint
        name: Lint Indent files
        entry: indent lint
        language: system
        types: [indent]
```

### Editor Plugins

Minimum viable editor support:
- **VS Code**: Extension using LSP
- **Neovim**: Native LSP configuration
- **JetBrains**: Plugin using LSP

Format-on-save should be the default experience.

---

## 6. Recommendations for Indent

### Naming: "indent fmt" and "indent lint"

**Recommendation:** Use `indent fmt` and `indent lint` as subcommands of the main binary.

Arguments against standalone naming like "indenter":
- Complicates the single-binary model
- Creates confusion about whether it's separate software
- "indent fmt" is self-documenting
- Matches Go's `go fmt`, Rust's `cargo fmt`, Zig's `zig fmt`

The tooling research already recommends this architecture:
```
indent                    # Single binary
├── build                  # Compile to binary
├── run                    # Compile and run
├── test                   # Run tests
├── fmt                    # Format code
├── lint                   # Static analysis
└── ...
```

### Configuration: Zero-Config with Escape Hatches

**Formatter configuration:** None.

The formatter has no options. This is a feature, not a limitation:
- No debates about configuration
- All Indent code looks the same
- Simpler implementation
- Easier onboarding

**Linter configuration:** Minimal, project-level only.

```toml
# project.mod

[lint]
# Promote warnings to errors
deny = ["suspicious", "security"]

# Disable specific lints
allow = ["missing-docs", "single-char-names"]

# Enable opt-in lints
warn = ["pedantic"]
```

**Lint-level comments for exceptions:**

```
# Allow specific lint on next line
#[allow(unused_variable)]
val _intentionally_unused = compute_side_effect()

# Allow for entire function
#[allow(high_complexity)]
func necessarily_complex_algorithm():
    ...
```

### Lint Categories and Severity

| Category | Default | Rationale |
|----------|---------|-----------|
| **correctness** | error | No false positives, always wrong |
| **suspicious** | warning | Likely wrong, needs review |
| **performance** | warning | Optimization opportunity |
| **style** | warning | Idiomatic code |
| **complexity** | warning | Maintainability |
| **security** | warning | Common vulnerabilities |
| **pedantic** | off | Opt-in for strict projects |
| **deprecated** | warning | Upcoming breaking changes |

### Default Lint Set for New Projects

New projects should have a sensible default that:
- Catches real bugs (correctness, suspicious)
- Enforces idiomatic code (style)
- Flags obvious performance issues (perf)
- Doesn't overwhelm with pedantic warnings

```toml
# Default (implicit for new projects)
[lint]
deny = ["correctness"]
warn = ["suspicious", "style", "performance", "complexity", "security"]
allow = ["pedantic"]
```

For strict projects (libraries, infrastructure):

```toml
[lint]
deny = ["correctness", "suspicious", "security"]
warn = ["style", "performance", "complexity", "pedantic"]
```

### Implementation Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Parser (shared with compiler)            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    AST + Source Mapping                     │
└─────────────────────────────────────────────────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────────┐
│        Formatter        │     │           Linter            │
│  ├── Indent normalization│     │  ├── Lint registry          │
│  ├── Whitespace cleanup │     │  ├── Pattern-based checks   │
│  ├── Import sorting     │     │  ├── Type-aware checks      │
│  └── Pretty printer     │     │  └── Quick fix generator    │
└─────────────────────────┘     └─────────────────────────────┘
```

**Shared infrastructure with compiler:**
- Parser: Same parser, same AST
- Type information: Linter uses type checker results
- Source mapping: Precise error locations
- Incremental computation: Salsa-based caching

### Implementation Priority

**Phase 1 (MVP, Months 1-3):**
- Formatter: indentation, whitespace, basic pretty-printing
- Linter: correctness category only
- CLI: `indent fmt`, `indent lint`
- Format-on-save via LSP

**Phase 2 (Usable, Months 4-6):**
- Formatter: import sorting, trailing commas
- Linter: suspicious, style, performance categories
- Quick fixes for common lints
- CI integration (`--check`, `--deny-warnings`)

**Phase 3 (Complete, Months 7-12):**
- Linter: complexity, security, pedantic categories
- Project-level configuration
- Comprehensive quick fixes
- Editor-specific optimizations

---

## 7. Formatter Technical Design

### Algorithm: Single-Pass Pretty Printer

Unlike Prettier's Wadler-Lindig algorithm, Indent's formatter uses a single-pass approach similar to gofmt:

1. Parse source to AST (preserve source positions)
2. Walk AST, emitting formatted output
3. Preserve programmer's line break decisions (significant whitespace)
4. Normalize whitespace within lines

This is possible because Indent's significant whitespace already encodes structure:
- Line breaks are semantically meaningful
- Indentation defines scope
- No "should this be one line or multiple lines?" decisions

### Handling Edge Cases

**Comments:** Preserve position relative to code:
```
# Comment before statement
val x = 5  # Inline comment

# Block of comments
# explaining something
# important
func process():
    ...
```

**Long lines:** The formatter does NOT break long lines. This is intentional:
- Line breaks in Indent are semantic
- The programmer's decision is preserved
- The linter can warn about excessively long lines

**Trailing commas:**
```
# Single-line: no trailing comma
val point = (x, y)

# Multi-line: trailing comma
val config = {
    "host": "localhost",
    "port": 8080,
}
```

### Import Sorting Algorithm

```
# 1. Group: stdlib
import collections
import json
import os

# 2. Group: external packages (alphabetically)
import http
import sqlalchemy

# 3. Group: local modules (alphabetically)
import myproject.config
import myproject.handlers

# Within groups: alphabetically by module path
# Within braced imports: alphabetically by name
import http.{Client, Server}
```

---

## 8. Linter Technical Design

### Lint Registration

Each lint is registered with metadata:

```rust
// Internal lint definition (Rust pseudocode for implementation)
Lint {
    id: "unused_variable",
    category: Correctness,
    default_level: Deny,
    description: "Variable is never used",
    explanation: "Unused variables indicate incomplete code...",
    auto_fix: Some(|span| Suggestion::Remove(span)),
}
```

### Check Types

**AST-only checks:** Fast, no type information needed
- Unused imports (syntactic)
- Naming conventions
- Style patterns
- Dead code after return

**Type-aware checks:** Require type checker results
- Unused variables (may be used in closures)
- Incorrect type comparisons
- API misuse
- Method availability

**Inter-procedural checks:** Cross-function analysis
- Unused function parameters
- Dead code paths
- Security patterns (taint tracking)

### Quick Fix Generation

Every lint that can be automatically fixed should provide a fix:

```
error[E0101]: unused variable `x`
  --> src/main.idt:15:5
   |
15 |     val x = compute()
   |         ^ variable `x` is never used
   |
   = fix: prefix with underscore to indicate intentional disuse
   |
15 |     val _x = compute()
   |         ++
```

Fix types:
- **Replace:** Suggest better pattern
- **Remove:** Delete dead code
- **Insert:** Add missing elements
- **Rename:** Fix naming conventions

### Performance Optimization

**Parallel lint execution:**
```
Files ──→ Parse ──→ Type Check ──→ Lint Passes ──→ Collect Results
  │         │           │              │
  ├─ f1.idt │           │         ┌────┴────┐
  ├─ f2.idt │           │         │ worker 1 │
  ├─ f3.idt │           │         │ worker 2 │
  └─ ...    │           │         │ worker N │
            │           │         └────┬────┘
            │           │              │
            └───────────┴──────────────┘
                   (shared cache)
```

**Incremental re-linting:**
- Track file modification times
- Only re-lint changed files
- Invalidate dependents when signatures change
- Cache lint results per file hash

---

## 9. Summary

### Key Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| **Formatter name** | `indent fmt` | Single-binary model, self-documenting |
| **Formatter config** | Zero options | "One obvious way", eliminates debates |
| **Formatter algorithm** | Single-pass | Speed, simplicity, significant whitespace |
| **Linter name** | `indent lint` | Consistent with formatter |
| **Linter config** | Minimal, project-level | Simple but flexible |
| **Lint categories** | 7 categories (Clippy-inspired) | Clear organization, graduated defaults |
| **Default for new projects** | Deny correctness, warn others | Catches bugs without overwhelming |
| **Speed targets** | <10ms format, <50ms lint (single file) | Format-on-save, real-time diagnostics |
| **LSP integration** | Built-in | First-class editor experience |

### Implementation Checklist

**Formatter:**
- [ ] Single-pass pretty printer
- [ ] Indentation normalization
- [ ] Whitespace cleanup
- [ ] Import sorting
- [ ] Trailing comma normalization
- [ ] Comment preservation
- [ ] `--check` mode for CI

**Linter:**
- [ ] Lint registry with metadata
- [ ] Correctness category (Phase 1)
- [ ] Suspicious, style, perf categories (Phase 2)
- [ ] Complexity, security, pedantic (Phase 3)
- [ ] Quick fix generation
- [ ] Project-level configuration
- [ ] `--deny-warnings` for CI
- [ ] JSON output for tooling

**Integration:**
- [ ] LSP textDocument/formatting
- [ ] LSP textDocument/publishDiagnostics
- [ ] LSP codeAction for quick fixes
- [ ] Pre-commit hook support
- [ ] GitHub/GitLab CI examples

---

## Sources

- [gofmt design philosophy](https://go.dev/blog/gofmt)
- [Russ Cox on gofmt](https://research.swtch.com/gofmt)
- [Go Style Guide](https://google.github.io/styleguide/go/)
- [rustfmt configuration](https://github.com/rust-lang/rustfmt)
- [Black formatter](https://github.com/psf/black)
- [Prettier architecture](https://deepwiki.com/prettier/prettier)
- [Clippy lint categories](https://doc.rust-lang.org/stable/clippy/lints.html)
- [Clippy repository](https://github.com/rust-lang/rust-clippy)
- [Ruff performance](https://github.com/astral-sh/ruff)
- [Ruff vs Pylint comparison](https://trunk.io/learn/comparing-ruff-flake8-and-pylint-linting-speed)
- [ESLint 2025 retrospective](https://eslint.org/blog/2026/01/eslint-2025-year-review/)
- [ESLint flat config](https://eslint.org/blog/2022/08/new-config-system-part-2/)
