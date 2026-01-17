# Building world-class language tooling with a small team

A small team can absolutely achieve Go's unified tooling experience with Rust's quality—but the path requires strategic prioritization and architectural decisions made early. The research reveals that **modern language tooling success depends more on architectural choices than team size**: rust-analyzer's incremental computation patterns, gofmt's opinionated design philosophy, and Go's checksum database for packages all demonstrate that complexity can be tamed through smart design.

The core insight is this: **start with a single-binary modular monolith**, emit proper DWARF debug info from day one, and defer infrastructure-heavy features like a package registry until the community demands them. This approach lets a 2-5 person team deliver professional-grade tooling within 12-18 months.

## LSP performance comes from laziness, not just incrementality

The rust-analyzer and gopls teams discovered a counterintuitive truth: **"It's not incrementality that makes an IDE fast—it's laziness."** The ability to skip huge swaths of code entirely matters more than efficiently recomputing changed portions. Both servers achieve sub-100ms response times through demand-driven computation that only analyzes what's immediately needed.

rust-analyzer uses the **Salsa framework**, which implements a "red-green algorithm" for incremental computation. The core abstraction treats compiler functions as tracked queries that automatically record dependencies. When inputs change, Salsa validates whether cached results are still valid, propagating changes only where necessary. A critical optimization is **durability levels**—marking standard library and external dependencies as "high durability" so they're never re-validated when only user code changes. This single optimization reduced response times from ~300ms to imperceptible.

gopls took a different approach with **persistent file-based caching**. Version 0.12 achieved **75% memory reduction** by storing per-package summaries on disk like compiler object files. The typed syntax trees are approximately **30x larger than source text**, making selective memory usage essential for large codebases. gopls maintains a lightweight in-memory graph to determine invalidation scope without loading everything.

For implementation complexity, the architecture choice depends on your language's semantics:

| Language characteristic | Recommended architecture |
|------------------------|-------------------------|
| Clean module system with fully-qualified names | Map-reduce (like IntelliJ/Sorbet) |
| Headers or interface files | Header-snapshotting (like clangd) |
| Complex macros or mutual recursion | Query-based with Salsa |

A minimal LSP server with parsing takes **1-2 months**; adding name resolution and go-to-definition adds **2-4 months**; type-aware completion requires **6-12 months** more. The critical architectural invariant to maintain: **typing inside a function body should never invalidate facts about other functions**. This requires separating signature queries from body queries in your compiler design.

## Opinionated formatting enables instant performance

gofmt achieves sub-millisecond formatting through a radically different philosophy than Prettier: **it doesn't reflow code to fit line widths**. While Prettier uses Wadler's algorithm to explore multiple layouts and choose the best fit, gofmt makes a single pass through the AST emitting formatted output immediately. No backtracking, no line-width measurements, no "try flat, then break" logic.

The technical reasons for gofmt's speed compound multiplicatively:

| Factor | Performance impact |
|--------|-------------------|
| Compiled language (Go vs JavaScript) | 10-35x |
| Single-pass, no backtracking | 2-5x |
| No line-width fitting | 2-10x |
| Zero configuration options | eliminates branching |
| Language syntax designed for formatting | enables simplicity |

Biome (the Rust rewrite of Prettier's algorithm) demonstrates that **35x speedup** is achievable through implementation language alone—but gofmt remains faster still because it solves a simpler problem. The Wadler-Lindig algorithm's complexity comes from the `group` operator that enables "flat or broken" layout choices; gofmt sidesteps this entirely by respecting the programmer's existing line break decisions.

For a new language, the recommendation is clear: **start with gofmt-style formatting**. Be extremely opinionated—one output for any input. Design your syntax to make formatting trivial: no semicolon insertion ambiguity, fixed brace placement, simple expression grammar. If users later demand reflowing for line widths, you can add it, but most backend developers prefer explicit control over their line breaks.

Implementation complexity for gofmt-style: **2,000-5,000 LOC** and **2-4 weeks** of development. For Prettier-style with full line fitting: **10,000-30,000 LOC** and **2-6 months**. The additional complexity buys you automatic line breaking but costs significant performance and implementation effort.

## The single-binary model wins for small teams

Go's `go` command bundles everything—build, test, fmt, vet, doc, mod—into one binary with unified UX, consistent versioning, and a single release process. Rust's ecosystem distributes tooling across cargo, rustfmt, clippy, and rust-analyzer with different maintainers and release cadences. For a small team, the Go model is clearly superior.

The coordination overhead of separate tools is substantial. The Clippy team notes that managing "over 1,000 open issues with regularly between 25-35 open PRs" strains their resources. rust-analyzer development was largely external until recently. Version compatibility between tools creates user confusion. Meanwhile, Go's single release cycle (every 6 months major, plus patches) keeps everything aligned.

The recommended approach combines both philosophies:

**Single binary with subcommands** for everything users touch daily: `yourlang build`, `yourlang test`, `yourlang fmt`, `yourlang lint`, `yourlang run`. Structure the codebase as a **modular monolith** with clear internal APIs between components—loose coupling enables future extraction if needed.

**Consider the language server separately**. rust-analyzer explicitly designs its syntax crate to be "completely independent from the rest"—IDE requirements (responsiveness, cancellation, constant interaction) differ significantly from batch compilation. A separate LSP binary is reasonable, but should share core libraries (parser, type system) with the main tool.

**Enable extensibility through convention**: Any `yourlang-foo` binary in PATH becomes available as `yourlang foo` (cargo's extension model). This lets community tools integrate without burdening core development.

For distribution, the single binary approach yields **50-100MB downloads** versus **200-400MB** for Rust's full toolchain. More importantly, users get "one download, everything works" rather than managing components through a tool multiplexer.

## Tree-sitter grammars take weeks to months, not days

The Tree-sitter documentation warns that development has a "steep initial learning curve" but becomes "zen-like" with practice. Real-world developer experiences vary dramatically based on language complexity—from "a few hours for simple languages" to "several months for complex languages with context-sensitive features."

Timeline estimates by language complexity tier:

| Complexity | Example languages | Initial grammar | Production-ready |
|------------|------------------|-----------------|------------------|
| Simple (DSLs, Lisp-like) | Config files | 1-3 days | 1-2 weeks |
| Moderate (clean syntax) | Go-like | 1-2 weeks | 3-4 weeks |
| Complex (context-sensitive) | Python, TypeScript, Rust | 4-8 weeks | 2-4 months |
| Highly complex (proofs, shells) | TLA⁺, Bash | 3-6 months | 6-12 months |

The most dangerous pitfall is **state explosion**. Putting too much logic in a single grammar rule causes exponential growth in parser states, massively slowing generation. The solution is extracting complex rules into hidden subrules (`_rule` prefix). Tree-sitter provides `--report-states-for-rule` to identify problems.

**External scanners** are required for context-sensitive features like Python's indentation-based scoping, here-documents, or string interpolation. These are written in C and add significant complexity—typically **200-500+ lines** with tricky state management. Debug by adding macros that log every character consumed.

Error recovery is Tree-sitter's biggest limitation for language tooling. The recovery algorithm is automatic but provides **limited control**—you cannot customize error messages or recovery strategies. One experienced developer notes: "If autocompletion or custom error messages are important, I'd recommend rolling your own parser... tree-sitter places a ceiling on possibilities for future improvement." For a language with its own compiler, Tree-sitter is best viewed as a fast editor integration layer, not the primary parser.

Testing strategies are essential: **corpus tests** for every grammar rule (stored in `test/corpus/*.txt`), **fuzzing** to catch crashes and infinite loops (the `tree-sitter/fuzz-action` GitHub Action is invaluable), and **real-world validation** by parsing actual codebases (`tree-sitter parse /path/to/code --quiet --stat`).

## Debug support comes nearly free with proper DWARF

The fastest path to a working debugger requires no custom code: **emit proper DWARF debug information, and lldb-dap provides debugging support immediately**. LLVM's `DIBuilder` API makes this straightforward if you're using LLVM for code generation.

Essential DWARF elements to emit:

```cpp
// Compile unit (one per source file)
DBuilder->createCompileUnit(dwarf::DW_LANG_C, File, "MyLang", ...);

// Function metadata
DISubprogram *SP = DBuilder->createFunction(Scope, Name, LinkageName, ...);

// Source location on every instruction
Builder->SetCurrentDebugLocation(DILocation::get(Ctx, Line, Col, Scope));

// Variable declarations
DBuilder->insertDeclare(Alloca, VarInfo, Expression, Location, Block);
```

Use `dwarf::DW_LANG_C` for C-like ABI compatibility—debuggers understand C calling conventions and can evaluate expressions, call functions, and display values correctly.

Once DWARF is emitting correctly, test with raw lldb or gdb before building any DAP adapter. If basic debugging works there, it will work everywhere. Then simply use `lldb-dap --port 12345` (ships with LLVM 18+) and configure VS Code to connect—zero adapter code needed.

When language-specific features justify custom work, build a **thin wrapper** around LLDB's stable SB API. Implementation estimates:

| Component | Minimal LOC | Full-featured LOC |
|-----------|------------|-------------------|
| Protocol handling (JSON) | 300-500 | 500-800 |
| Breakpoint handling | 300-500 | 600-1,000 |
| Stack/scope/variable inspection | 500-800 | 1,000-2,000 |
| LLDB integration | 500-1,000 | 1,500-3,000 |
| **Total** | **2,000-3,500** | **4,500-8,000** |

For initial release, implement only: `initialize`, `launch`, `setBreakpoints`, `stackTrace`, `scopes`, `variables`, stepping commands, and `disconnect`. Defer expression evaluation, conditional breakpoints, watchpoints, and disassembly to later releases.

## Start with zero-infrastructure package management

The XZ Utils backdoor in March 2024—where an attacker spent **two years** building trust as a legitimate maintainer before inserting a sophisticated SSH backdoor—changed everything about supply chain security assumptions. Combined with npm's ongoing malware campaigns (40+ packages compromised in September 2025 alone) and PyPI's typosquatting waves, the lesson is clear: **central registries create central attack surfaces**.

For a new language, start with **Zig's model: fully decentralized, content-addressed dependencies**. No central registry, no infrastructure to maintain, no moderation burden.

```
# Dependencies specified by URL + cryptographic hash
dependencies:
  httplib: 
    url: "https://github.com/author/httplib/archive/v1.2.3.tar.gz"
    hash: "sha256:a1b2c3d4..."
```

Content hashes guarantee **immutability without central authority**. Even the package author cannot change bits after first access (if users share their lock files). This is strictly more secure than version numbers, which can be reused after package deletion.

Phase your development:

| Phase | Timeline | What to build |
|-------|----------|--------------|
| MVP | 0-6 months | URL+hash dependencies, lock file, `pkg add/fetch` |
| Discovery | 6-12 months | Static site package index (GitHub Pages, $0) |
| Caching | 12-24 months | Optional proxy/mirror for reliability |
| Full registry | Only if warranted | When 1000+ packages and corporate sponsorship |

If you eventually build a registry, learn from npm's mistakes: **use scoped namespaces from day one** (`@author/package`). This prevents dependency confusion attacks and name squatting. Also integrate **Sigstore** for keyless package signing—it's becoming the industry standard, with npm, PyPI, and Maven all adopting it.

Go's checksum database (`sum.golang.org`) is worth studying even if you don't replicate it. The Merkle tree design ensures that the proxy cannot serve different content to different users without detection—a powerful guarantee that doesn't require trusting the proxy operator.

## What's realistic for a small team

A 2-5 person team can achieve Go's unified tooling experience with Rust's quality in **12-18 months** by making smart architectural choices:

**Months 1-3**: Single-binary CLI skeleton with subcommands, basic build system, gofmt-style opinionated formatter, DWARF emission from compiler.

**Months 4-6**: LSP server with parsing and go-to-definition (no type-aware completion yet), Tree-sitter grammar for editor highlighting, content-addressed package management.

**Months 7-12**: Type-aware completion in LSP, incremental compilation for large projects (consider Salsa), basic linting integrated into main binary.

**Months 12-18**: Polish, performance optimization, community feedback incorporation, package index if community needs one.

The key insight is that **quality comes from design constraints, not team size**. gofmt is fast because it solves a constrained problem. rust-analyzer is responsive because Salsa enables laziness. Go's tooling is coherent because it's one binary. Make the constraining decisions early—opinionated formatting, single binary, content-addressed packages—and excellence becomes achievable for a small team.

Defer the hard problems: full expression evaluation in the debugger, complex refactoring tools, automatic line reflowing. These can come in year two or three. What matters for day one is that developers can write code, format it instantly, get useful diagnostics in their editor, debug when things go wrong, and manage dependencies reproducibly. That's achievable.

## Key architectural recommendations summary

- **LSP**: Use query-based architecture if your language has complex macros; otherwise map-reduce is simpler. Body-level isolation is essential for performance.
- **Formatter**: Be extremely opinionated. Single pass, no line-width fitting, one output for any input. gofmt-style, not Prettier-style.
- **Distribution**: Single binary with subcommands. Share core libraries between CLI and LSP server. Enable `yourlang-*` extensions.
- **Tree-sitter**: Budget 4-8 weeks for a moderate-complexity grammar. External scanners are complex—avoid if possible through language design.
- **Debugging**: Emit proper DWARF, use lldb-dap immediately, build custom DAP adapter only when language-specific features justify it.
- **Packages**: Start decentralized with URL+hash. Content hashes beat version numbers. Add infrastructure only when community demands it.