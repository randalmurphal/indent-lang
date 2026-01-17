# Documentation system design for the Indent language

For a whitespace-significant language like Indent, **Rust-style `///` line comments placed before declarations** offer the best documentation syntax—they completely avoid indentation complexities that plague Python's trailing docstrings while providing clear visual separation and simple parsing. The optimal documentation system combines Rust's proven doc-testing model, CommonMark Markdown with intra-doc links, and a docs.rs-style automatic hosting architecture. This report provides concrete specifications, implementation plans, and comparative analysis to guide Indent's documentation design.

## Why line comments outperform block strings for significant-whitespace languages

The fundamental tension in documenting whitespace-significant languages centers on where documentation lives relative to the item it describes. Python places docstrings *inside* function bodies as trailing strings, which creates three significant problems that Indent should avoid.

First, **indentation trimming becomes mandatory**. When a docstring appears inside an indented block, its raw content inherits that indentation—Python's `__doc__` attribute contains leading whitespace that must be stripped using PEP 257's algorithm, which finds the minimum indentation of non-blank lines after line 1 and removes that amount from all lines. Every tool consuming docstrings must implement this algorithm correctly.

Second, trailing docstrings create **ambiguity with empty blocks**. In Python, a function containing only a docstring requires an additional `pass` statement; whether docstrings count as valid statements varies by context. This complexity propagates through the parser.

Third, the docstring's position inside the block means **parsing must handle string literals specially** as potential documentation rather than executable code.

Rust-style line comments (`///`) placed before declarations elegantly sidestep all three issues. Each line is independent—no relative indentation to track. The parser encounters the documentation first, then the declaration—no ambiguity about block structure. The syntax is `/// ` followed by content, which Markdown parses directly without preprocessing.

```indent
/// Calculates the factorial of a non-negative integer.
///
/// This function uses iterative multiplication rather than recursion
/// to avoid stack overflow for large inputs.
///
/// # Arguments
///
/// * `n` - The non-negative integer
///
/// # Returns
///
/// The factorial of n, or an error if n is negative
///
/// # Examples
///
/// ```
/// let result = factorial(5)
/// assert_eq(result, 120)
/// ```
fn factorial(n: Int) -> Result<Int, Error>:
    if n < 0:
        return Error("Negative input")
    var result = 1
    for i in 1..=n:
        result *= i
    result
```

For module-level documentation, Indent should adopt `//!` inner doc comments that document the enclosing item rather than the following item. This pattern works well for file headers and module descriptions.

## Recommended doc comment syntax specification

The syntax specification for Indent documentation should formalize these patterns:

**Outer documentation (`///`)** documents the immediately following item. Three slashes exactly—four or more becomes a regular comment. Content after the space is CommonMark Markdown. Multiple consecutive `///` lines form a single documentation block.

**Inner documentation (`//!`)** documents the enclosing module or crate. Typically appears at file top. Useful for describing module purpose, providing usage examples, and listing re-exports.

**Attribute equivalence** means the compiler transforms `/// text` into `#[doc = "text"]`, enabling programmatic documentation manipulation, external file inclusion via `#[doc = include_str!("README.md")]`, and conditional documentation with `#[cfg_attr(feature = "doc", doc = "...")]`.

**Markdown support** should implement CommonMark specification with extensions for tables (GitHub-style pipe syntax), fenced code blocks with language hints and attributes, and admonitions via HTML `<div class="warning">` blocks.

## Cross-reference syntax enables rich linking

Documentation becomes significantly more useful when types, functions, and modules can reference each other. Indent should adopt Rust's bracket-based intra-doc link syntax with **scope-based resolution**.

The basic pattern uses backtick-bracketed names: `` [`OtherType`] `` creates a link to `OtherType`, resolved relative to the current scope. Full paths work with `` [`module::submodule::Type`] ``. Custom link text uses Markdown syntax: `[click here](`Type`)`.

**Disambiguators** resolve conflicts when names exist in multiple namespaces. When a function and type share the same name, `[fn@calculate]` links to the function while `[type@calculate]` links to the type. Shorthand suffixes like `[calculate()]` for functions and `[calculate!]` for macros provide readable alternatives.

Resolution order proceeds outward: current scope first, then parent modules, then crate root, then standard library. Broken links should generate compiler warnings to catch documentation drift.

## Parameter documentation works best as prose for typed languages

Statically-typed languages like Indent already express parameter types in signatures, making JSDoc-style `@param {type} name` annotations redundant. Rust's approach—describing parameters in prose under conventional headers—proves both readable and sufficient.

The recommended pattern uses Markdown headers for structure:

```indent
/// Writes data to the specified file path.
///
/// # Arguments
///
/// * `path` - The destination file path, created if nonexistent
/// * `data` - Bytes to write, completely replacing existing content
/// * `options` - Optional configuration for permissions and atomicity
///
/// # Returns
///
/// The number of bytes written, or an error if the write fails.
///
/// # Errors
///
/// Returns `IoError::PermissionDenied` if the path is not writable.
/// Returns `IoError::DiskFull` if insufficient space exists.
///
/// # Panics
///
/// Panics if `data` length exceeds `isize::MAX`.
fn write_file(path: Path, data: Bytes, options: Option<WriteOptions>) -> Result<usize, IoError>
```

This structure enables doc generators to parse sections reliably while keeping documentation human-readable. The convention should support **Arguments**, **Returns**, **Errors**, **Panics**, **Examples**, **See Also**, and **Safety** (for unsafe code) as standard headers.

## Deprecation requires both attributes and documentation

Effective deprecation combines compiler-level attributes with documentation-level notices. The attribute provides machine-readable metadata that enables IDE warnings and lints, while the documentation explains migration paths.

```indent
#[deprecated(since = "2.1.0", note = "Use `new_function` instead—this will be removed in 3.0")]
/// Processes input using the legacy algorithm.
///
/// **Deprecated since 2.1.0**: This function uses O(n²) processing.
/// Use [`new_function`] which achieves O(n log n) complexity.
fn old_function(input: Data) -> Output
```

The doc generator should render deprecated items with a distinct visual treatment—a yellow badge, strikethrough styling, or warning box—and optionally hide them from default navigation while keeping them accessible via direct links.

## Doc tests implementation plan

Executable documentation—code examples that run as tests—represents one of Rust's most valuable documentation innovations. When examples compile and execute correctly, they cannot become stale. Indent should implement a full doctest system.

### Extraction phase

The documentation parser extracts fenced code blocks from doc comments. Each block's info string specifies the language and attributes: ` ```indent,no_run ` indicates Indent code that should compile but not execute. The parser records source location (file, line, column) for error reporting that maps back to documentation.

### Preprocessing and compilation

Each doctest undergoes preprocessing before compilation. Lines beginning with `# ` (hash space) are **compiled but not displayed** in rendered documentation—essential for hiding imports, error handling setup, and cleanup code that distracts from the example's core purpose.

```indent
/// ```
/// # use mylib::Config
/// # fn main() -> Result<(), Error>:
/// let config = Config::load("settings.toml")?
/// assert(config.validate())
/// # Ok(())
/// ```
```

The rendered documentation shows only the configuration loading, while the hidden lines provide the necessary context for compilation.

If no `fn main` exists, the preprocessor wraps the code in one. For examples using the `?` operator, a hidden `fn main() -> Result<T, E>` wrapper with trailing `Ok(())` enables clean error handling examples.

### Execution and verification

The test runner compiles each doctest as an independent program. Compilation isolation ensures examples don't accidentally depend on each other's setup. Execution occurs in sandboxed processes with time limits, working directory control, and resource constraints.

Verification operates in **assertion mode** by default (Rust model): tests pass if they compile and execute without panic. No stdout comparison occurs. This avoids brittleness to output formatting changes while ensuring examples actually work.

### Code block attributes

| Attribute | Behavior |
|-----------|----------|
| `ignore` | Skip entirely—neither compile nor run |
| `no_run` | Compile but don't execute (for destructive or network operations) |
| `should_panic` | Test passes only if execution panics |
| `compile_fail` | Test passes only if compilation fails |
| `edition2024` | Compile with specific language edition |

## Documentation generation architecture

Indent's documentation generator should ship with the compiler toolchain (like rustdoc) rather than require external installation (like Sphinx). Built-in tools achieve **deep compiler integration**—accurate type information, cross-reference resolution, and consistent behavior across the ecosystem.

### Output formats

**HTML** serves as the primary output with interactive features: hierarchical navigation, client-side search, theme switching (light/dark/system), and mobile responsiveness. Each documented item gets a stable URL anchor for linking.

**JSON** output enables tooling integration. A well-defined schema lets IDEs extract hover documentation, linters analyze documentation coverage, and custom frontends render documentation differently. The schema should be versioned and considered stable API.

**Man pages** serve command-line tools specifically. Generated from the same source documentation, they integrate with Unix `man` infrastructure.

### Search implementation

Client-side JavaScript search with pre-generated indexes offers the best balance of speed and simplicity. Rustdoc's search implementation demonstrates effective techniques: highly compressed indexes using parallel arrays and VLQ encoding, type-signature search (`-> Result` finds functions returning Result), and fuzzy name matching with Levenshtein distance.

### Hosting architecture

A docs.rs-style automatic hosting service dramatically improves ecosystem documentation quality. The architecture comprises:

**Registry watcher** monitors the package registry for new publishes. Each publish triggers a documentation build job added to a queue.

**Sandboxed builders** run documentation generation in containers (Docker/Podman) with network isolation, time limits, and resource constraints. Builds use a fixed compiler version for reproducibility.

**Storage** uses S3-compatible object storage for generated documentation. PostgreSQL tracks metadata: crate names, versions, build status, feature flags.

**Serving** delivers static files from storage through a CDN. URLs follow the pattern `/crate-name/version/module/struct.Type.html` with semver-aware redirects: `/crate/1.0/` resolves to the latest 1.0.x version.

**Version management** keeps all published versions accessible while handling yanked packages specially—yanked versions exclude from "latest" redirects but remain accessible via direct version URLs.

## Comparison of Rust, Python, and Go documentation systems

### Python: runtime-accessible but whitespace-troubled

Python pioneered the docstring concept—documentation as first-class runtime data accessible via `__doc__`. This enables introspection, dynamic help systems, and REPL exploration. The `help()` function and `inspect.getdoc()` leverage this accessibility.

However, Python's approach carries significant costs. **Indentation trimming** requires tooling to implement PEP 257's algorithm correctly. **No standard format** means the ecosystem fragments across Google style (`Args:`), NumPy style (underlined headers), Sphinx reST (`:param:`), and others. **pydoc is minimal**—serious documentation requires Sphinx, a powerful but configuration-heavy external tool. **Doctest brittleness** stems from exact stdout matching; minor formatting changes break tests.

The fragmentation shows in tooling: Google style emphasizes readable source, NumPy style emphasizes comprehensive documentation, Sphinx style emphasizes cross-reference power—but none achieved universal adoption.

### Rust: the documentation gold standard

Rust's documentation system represents the state of the art through several reinforcing design decisions.

**Line comments before declarations** sidestep indentation entirely. The `///` syntax is visually distinctive and trivially parseable. Inner comments `//!` handle module-level documentation elegantly.

**Markdown is universal**—no competing format standards. CommonMark provides a well-specified foundation with tables and fenced code blocks added.

**Intra-doc links** make cross-references first-class with scope-based resolution and disambiguators. Broken links generate compiler warnings.

**Doctests run as tests** ensuring examples stay current. Hidden lines, error handling support, and attribute control provide flexibility. The 2024 edition's doctest merging dramatically improves performance.

**rustdoc ships with the compiler**, requiring zero setup. JSON output enables tooling integration.

**docs.rs provides automatic hosting** for every published crate, establishing ecosystem-wide documentation availability as the default.

The primary limitation is **verbosity**—each line requires `/// ` prefix, making very long documentation slightly tedious to write.

### Go: minimalist by design

Go's documentation philosophy prioritizes simplicity above all. Plain `//` comments before declarations become documentation—no special syntax. godoc renders Markdown-like formatting from plain text conventions.

**Strengths** include zero learning curve (comments are just comments), testable examples through `ExampleFunctionName()` convention, and `doc.go` files for package-level documentation.

**Limitations** become apparent at scale. No cross-reference syntax means linking to other packages requires full URLs. No hidden code in examples means all setup appears in rendered documentation. Limited formatting (no tables, no admonitions) constrains complex API documentation. The deprecated godoc gave way to pkg.go.dev, but tooling remains simpler than rustdoc.

Go's approach works well for Go's domain (systems programming with simple APIs) but provides insufficient structure for complex libraries.

### Summary comparison

| Aspect | Python | Rust | Go |
|--------|--------|------|-----|
| Syntax | `"""` after declaration | `///` before declaration | `//` before declaration |
| Indentation interaction | Complex trimming required | None | None |
| Format standardization | Fragmented | CommonMark universal | Plain text conventions |
| Cross-references | Sphinx roles (external) | `[`Type`]` built-in | None |
| Doc tests | Output-matching (brittle) | Compile-and-run (robust) | ExampleFunc (good) |
| Built-in generator | pydoc (minimal) | rustdoc (comprehensive) | godoc (moderate) |
| Hosting | Read the Docs (external) | docs.rs (automatic) | pkg.go.dev (automatic) |

## IDE integration via Language Server Protocol

The Language Server Protocol provides the interface between documentation and editors. Three LSP features particularly matter for documentation.

**Hover documentation** (`textDocument/hover`) displays documentation when users position their cursor over identifiers. The server should return Markdown content including the signature as a fenced code block, the first paragraph as summary, and optionally full documentation with examples. rust-analyzer's configurable `hoverKind` (full documentation, synopsis only, or none) provides a good model.

```json
{
  "contents": {
    "kind": "markdown",
    "value": "```indent\nfn factorial(n: Int) -> Result<Int, Error>\n```\n\nCalculates the factorial of a non-negative integer.\n\nUses iterative multiplication to avoid stack overflow."
  }
}
```

**Signature help** (`textDocument/signatureHelp`) displays parameter documentation as users type function arguments. The `activeParameter` index highlights which parameter the cursor is on, and per-parameter documentation in `ParameterInformation.documentation` provides inline help.

**Documentation completion** assists users writing doc comments. A code action can generate documentation stubs pre-populated with parameter names from the signature, placeholder text for description, and standard sections (Arguments, Returns, Errors).

## Separating internal documentation from public API docs

Internal documentation serves maintainers; public documentation serves users. These audiences need different information.

**Visibility-based separation** uses `#[doc(hidden)]` for items that are public (for macro hygiene or cross-crate implementation reasons) but shouldn't appear in documentation. The `--document-private-items` flag generates internal documentation including non-public items when needed.

**Comment style separation** reserves `///` for user-facing documentation while using regular `//` comments for implementation notes, TODOs, and maintainer context:

```indent
/// Validates and normalizes user input.
///
/// # Arguments
///
/// * `input` - Raw user input to validate
fn validate(input: String) -> Result<NormalizedInput, ValidationError>:
    // PERF: This regex is compiled once at module load
    // TODO: Consider caching validation results for repeated inputs
    // NOTE: See ADR-047 for why we chose this validation approach
    pattern.validate(input)
```

**Architecture Decision Records** capture the "why" behind significant decisions. Store ADRs in `docs/adr/` using sequential numbering (`0001-record-decisions.md`). Link from code documentation to relevant ADRs. Use adr-tools or similar for consistent management.

A recommended project structure:

```
project/
├── docs/
│   ├── adr/
│   │   ├── 0001-use-adrs.md
│   │   ├── 0002-choose-database.md
│   │   └── README.md
│   └── ARCHITECTURE.md
├── src/
│   └── lib.indent
├── CONTRIBUTING.md
└── README.md
```

## Rendering examples for Indent documentation

The doc generator should render documentation with clear visual hierarchy. A function's documentation might render as:

---

### `fn write_file`

```indent
fn write_file(path: Path, data: Bytes, options: Option<WriteOptions>) -> Result<usize, IoError>
```

Writes data to the specified file path.

Creates the file if it doesn't exist, or completely replaces existing content if it does. For atomic writes that prevent partial writes on crash, use `WriteOptions::atomic()`.

**Arguments**

- `path` – The destination file path, created if nonexistent
- `data` – Bytes to write, completely replacing existing content  
- `options` – Optional configuration for permissions and atomicity

**Returns**

The number of bytes written on success.

**Errors**

Returns `IoError::PermissionDenied` if the path is not writable.  
Returns `IoError::DiskFull` if insufficient space exists.

**Example**

```indent
let bytes_written = write_file(Path::new("output.txt"), b"Hello, world!", None)?
assert_eq(bytes_written, 13)
```

---

This rendering demonstrates the recommended patterns: signature in code block, prose description, structured sections for different information types, and a runnable example.

## Summary recommendations for Indent

The documentation system for Indent should adopt `///` outer doc comments and `//!` inner doc comments placed before declarations, completely avoiding whitespace interaction issues. CommonMark Markdown with table extensions provides the content format. Bracket-based cross-references (`[`Type`]`) with scope-based resolution enable rich linking. Prose-based parameter documentation under conventional headers (Arguments, Returns, Errors, Panics, Examples) suits typed languages well.

For doc tests, implement Rust's model: fenced code blocks compile and execute as tests, lines prefixed with `#` are hidden but compiled, and attributes like `no_run` and `should_panic` control test behavior. Start with individual compilation, then optimize to merged compilation for performance.

The doc generator should ship with the compiler, outputting HTML with client-side search and JSON for tooling. Automatic hosting for all published packages, versioned documentation with semver-aware URLs, and yanked package handling complete the ecosystem infrastructure.

This design synthesizes the best proven patterns: Rust's documentation excellence, Go's simplicity where appropriate, and lessons from Python's fragmentation to avoid. The result should be a documentation system that encourages comprehensive, tested, well-linked documentation across the Indent ecosystem.