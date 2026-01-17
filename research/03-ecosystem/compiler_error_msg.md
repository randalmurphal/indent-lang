# Designing world-class error messages for the Indent compiler

Error messages are your compiler's most important user interface. Research shows developers spend up to **25% of debugging time** on "blind alleys" from poor diagnostics, while compilers like Rust and Elm have proven that investing in error quality dramatically improves developer experience. This report synthesizes best practices from Rust, Elm, Zig, and Clang to design a diagnostic system for Indent that teaches while it corrects.

## The case for treating errors as educational tools

Modern compiler design has shifted from viewing errors as failure reports to treating them as teaching moments. Elm's creator Evan Czaplicki established the guiding principle: **"Compilers should be assistants, not adversaries."** Rust adopted this philosophy after RFC 1644, which explicitly cited Elm as inspiration for their error redesign. The key insight is that errors answer two questions: "What went wrong?" and "How do I fix it?"

Academic research supports this approach. A 2017 IEEE/ACM ICSE study using eye-tracking found that **interpreting error messages is as cognitively demanding as reading source code itself**. Developers must constantly context-switch between error output and source files, making proximity and clarity critical. Enhanced error messages resulted in significantly shorter debugging times and higher self-reported usefulness scores.

Rust's error format demonstrates these principles in practice:

```
error[E0382]: borrow of moved value: `data`
  --> src/main.rs:8:20
   |
6  |     let data = vec![1, 2, 3];
   |         ---- move occurs because `data` has type `Vec<i32>`
7  |     process(data);
   |             ---- value moved here
8  |     println!("{:?}", data);
   |                      ^^^^ value borrowed here after move
   |
help: consider cloning the value if the performance cost is acceptable
   |
7  |     process(data.clone());
   |                 ++++++++
```

This format shows the "what" (borrow of moved value), the "why" (moved on line 7), and the "how to fix" (consider cloning) in a single cohesive message.

## Four philosophies of error design

### Rust's structured precision

Rust's diagnostic system uses a **primary/secondary span architecture**. Primary spans (marked with `^^^`) explain what went wrong, while secondary spans (marked with `---`) explain why. This distinction reduces cognitive load by separating the error location from its causes. Rust's compiler guidelines emphasize using complete sentences, present tense for ongoing conditions, and never making users feel at fault. Error codes like `E0382` enable direct lookup via `rustc --explain E0382`, which displays a full tutorial on the concept.

### Elm's conversational approach

Elm's errors read like a helpful colleague explaining the problem. They use headers to categorize error types, conversational language without jargon, and embedded hints with links to documentation. Elm has no runtime exceptions—every error is caught at compile time with messages designed to educate. John Carmack called Elm's approach "an inspiration for every error message." For Indent targeting DevOps developers who may not have deep compiler theory background, this accessibility matters.

### Zig's compile-time transparency

Zig unifies compile-time execution and metaprogramming through `comptime`, allowing arbitrary code execution during compilation. When errors occur, Zig shows clear call stacks through the comptime evaluation path. The `@compileError()` function lets library authors provide custom error messages. However, users report that Zig's error traces can become very long for complex comptime failures, suggesting a need for trace summarization.

### Clang's surgical precision

Clang pioneered several techniques that became industry standards: accurate column numbers (not just line numbers), source range highlighting with `^^^` underlines, and **Fix-It hints** that suggest concrete repairs. Clang's rule for Fix-Its: only suggest changes when "very likely they match the user's intent." This avoids the C++ template problem where compilers suggest dozens of irrelevant alternatives due to overloading.

## Implementing "did you mean?" suggestions

Typo correction significantly improves developer experience. The foundational algorithm is **Damerau-Levenshtein distance**, which calculates the minimum edits (insertions, deletions, substitutions, transpositions) between strings. Research by Damerau found that over **80% of human misspellings** involve exactly one of these four operations.

However, raw string distance isn't enough. Modern compilers use context-aware suggestions combining:

- **Scope filtering**: Only suggest identifiers actually visible in the current scope
- **Type compatibility**: Suggest variables that would type-check in context  
- **Semantic relevance**: Weight suggestions by how likely they are to be the intended symbol
- **Word boundary bonuses**: Matching at `camelCase` or `snake_case` boundaries ranks higher

For efficiency over large symbol tables, Levenshtein automata (finite state machines recognizing all strings within edit distance N) avoid recalculating distance for every candidate. Tools like Nucleo use Smith-Waterman algorithms with affine gap penalties—originally developed for DNA sequence alignment—for sophisticated fuzzy matching.

## Recommended error message format for Indent

Based on this research, here is the proposed format for Indent errors:

```
error[IND0042]: cannot assign to immutable variable `config`
  --> deploy/main.ind:15:5
   |
12 |     let config = load_config()
   |         ------ variable declared here as immutable
   |
15 |     config.timeout = 30
   |     ^^^^^^ cannot assign to immutable variable
   |
help: consider declaring `config` as mutable
   |
12 |     let mut config = load_config()
   |         +++
   |
note: for more information about mutability, see https://indent-lang.org/learn/mutability
```

This format incorporates:

- **Error code prefix** (`IND`) distinguishing Indent errors from ecosystem tools
- **Four-digit codes** allowing ~10,000 unique errors with room for categorization
- **Primary span** showing the error location with explanation
- **Secondary span** showing the declaration that caused the constraint
- **Actionable help** with inline diff showing exactly what to type
- **Documentation link** for deeper learning

## Designing the error code system

| Aspect | Recommendation | Rationale |
|--------|----------------|-----------|
| **Format** | `IND` + 4 digits (IND0001-IND9999) | Prefix distinguishes from other tools; 4 digits balance memorability with capacity |
| **Categorization** | IND0xxx syntax, IND1xxx types, IND2xxx regions, IND3xxx modules | Numeric ranges for logical grouping without complex prefixes |
| **Stability** | Never reuse or change meaning | Enables stable Stack Overflow threads and tooling |
| **CLI explain** | `indent --explain IND0042` | Matches Rust's successful pattern for learning |
| **Documentation** | Per-code pages with examples | Each code gets erroneous example, explanation, and fix |

The error code system serves three audiences: developers searching for solutions, tooling that filters or acts on specific errors, and the Indent team tracking diagnostic quality. TypeScript's ~2,000 error codes and Rust's 700+ demonstrate that large code spaces work well in practice.

Each documentation page should include:

1. One-sentence description of the error
2. Minimal code example that triggers it
3. Explanation of why this is an error (concepts involved)
4. Fixed code example
5. Common variations and edge cases
6. Links to relevant Indent documentation chapters

## Technical architecture for the diagnostic system

### Core data structures

The diagnostic system should center on a `Diagnostic` structure containing hierarchical information:

```
Diagnostic:
  - level: Error | Warning | Note | Help
  - code: Option<ErrorCode>
  - message: String
  - primary_spans: Vec<LabeledSpan>
  - secondary_spans: Vec<LabeledSpan>
  - children: Vec<Diagnostic>
  - suggestions: Vec<Suggestion>

LabeledSpan:
  - file: SourceFile
  - byte_range: Range<usize>
  - line_col: (start_line, start_col, end_line, end_col)
  - label: Option<String>
  - is_primary: bool

Suggestion:
  - message: String
  - span: Span
  - replacement: String
  - applicability: MachineApplicable | MaybeIncorrect | HasPlaceholders
```

Rust's `suggestion_applicability` levels are particularly valuable: `MachineApplicable` fixes can be auto-applied by tools, `MaybeIncorrect` requires human review, and `HasPlaceholders` contains `(...)` markers needing user input.

### Multi-span rendering

When errors involve multiple locations (which region errors will frequently), the renderer must handle:

- **Vertical layout**: Stack related spans with clear visual connection
- **Ellipsis for gaps**: Skip irrelevant lines between spans with `...` markers
- **Span overlap**: When labels would overlap, use vertical stacking with connector lines
- **File boundaries**: Cross-file errors need clear file headers for each location

### Terminal output formatting

For terminal rendering, follow these conventions:

| Element | Color | ANSI Code | Fallback |
|---------|-------|-----------|----------|
| Error header | Bold red | `\e[1;31m` | `error:` prefix |
| Warning header | Bold yellow | `\e[1;33m` | `warning:` prefix |
| Note | Bold cyan | `\e[1;36m` | `note:` prefix |
| Line numbers | Bold blue | `\e[1;34m` | Plain text |
| Primary caret | Bold red | `\e[1;31m` | `^^^` markers |
| Secondary caret | Blue | `\e[34m` | `---` markers |

Indent must support the **NO_COLOR** standard: if the `NO_COLOR` environment variable is set and non-empty, disable all color output. Command-line flags (`--color=always|never|auto`) should override environment variables. Accessibility matters—never rely on color alone; always use symbols (`^^^`, `---`, `...`) as primary indicators.

### Machine-readable output

Indent should support two output formats for tooling integration:

**JSON format** (for Indent-specific tools):
```json
{
  "$message_type": "diagnostic",
  "level": "error",
  "code": {"code": "IND0042", "explanation_url": "https://..."},
  "message": "cannot assign to immutable variable `config`",
  "spans": [{
    "file_name": "deploy/main.ind",
    "byte_start": 245, "byte_end": 251,
    "line_start": 15, "line_end": 15,
    "column_start": 5, "column_end": 11,
    "is_primary": true,
    "label": "cannot assign to immutable variable",
    "suggested_replacement": null
  }],
  "children": [...],
  "rendered": "error[IND0042]: cannot assign..."
}
```

**SARIF format** (for ecosystem integration):

SARIF (Static Analysis Results Interchange Format) is an OASIS standard used by GitHub code scanning, VS Code, Visual Studio, and many CI/CD systems. Supporting SARIF enables:

- GitHub Security tab integration without custom tooling
- VS Code problem panel population via standard extensions
- Aggregation with other static analysis tools in unified dashboards

Key SARIF fields include `runs` (tool execution instances), `results` (individual findings with rule IDs and locations), and `fixes` (suggested changes). The format is more verbose than custom JSON but enables broad interoperability.

## Handling complex error scenarios

### Type inference errors require showing both sides

The fundamental challenge with type inference errors is that the compiler discovers an inconsistency but doesn't know which constraint the programmer intended. Research on PolySubML identified the key principle: **never show just one side of a type conflict**. 

Bad: "Expected `int`, found `string`" (why was `int` expected?)
Good: Show where the `int` expectation originated AND where the `string` value came from

For Indent, type errors should:
- Always show both the "expected" and "found" types with their source locations
- Trace expectations back to their origin (function signature, variable declaration, etc.)
- Suggest specific locations for type annotations to narrow down the error
- Use consistent terminology ("expected X" vs "found Y") across all type errors

### Region errors need visual lifetime annotations

Indent's "region system" (analogous to Rust's borrow checker) will produce some of the most complex errors. Rust's NLL (Non-Lexical Lifetimes) diagnostic improvements provide the model:

```
error: region may not live long enough
  --> src/lib.rs:5:20
   |
3  | fn process<'a, 'b>(x: &'a Data) -> &'b Result {
   |            --  -- region `'b` defined here
   |            |
   |            region `'a` defined here
5  |     compute(x)
   |     ^^^^^^^^^^ returning data with region `'a` but region `'b` required
   |
help: consider adding: `'a: 'b` (region `'a` outlives region `'b`)
```

Key techniques: inline region annotations showing where lifetimes are defined, clear explanation of the conflict, and concrete suggestions for fixes. Rust's transition from the old borrow checker (30+ line confusing errors) to NLL (12-line clear errors) demonstrates that investment in visualization pays off dramatically.

### Macro and comptime expansion errors need traces

For Indent's metaprogramming features, error attribution is critical. When errors occur in generated code:

1. **Show the call stack**: Display the chain from macro invocation to error location
2. **Attribute correctly**: Errors in macro-generated code should point to the macro call site first, with notes pointing to the macro definition
3. **Allow inspection**: A command like `indent expand main.ind` should show expanded code for debugging
4. **Custom errors**: Provide `@compileError("message")` for library authors to emit domain-specific errors

C++ template errors demonstrate what to avoid: massive type names, exponential alternative exploration due to SFINAE, and loss of programmer-written typedefs. Constrain metaprogramming errors early with clear messages.

## Parser recovery for IDE responsiveness

### The LSP imperative

From VS Code's LSP documentation: "Most of the time, the code in the editor is incomplete and syntactically incorrect, but developers would still expect autocomplete and other language features to work." An error-tolerant parser is mandatory for a language server.

### Recommended recovery strategy

For Indent's parser, combine these techniques:

**Panic mode with smart synchronization**: When encountering an error, skip tokens until reaching a synchronization point. For Python-like syntax without semicolons, synchronization tokens include:
- Line starts at lower or equal indentation
- Keywords that start statements (`def`, `if`, `for`, `class`, etc.)
- Closing brackets at appropriate nesting level

**Recovery expressions**: Create placeholder AST nodes (like Clang's `RecoveryExpr`) for invalid expressions. These nodes carry approximate type information and prevent cascading errors downstream.

**Error productions**: Add grammar rules for common mistakes:
```
# Allow parsing of common errors with helpful messages
statement → "=" expression  # Error: assignment requires target
         → expression "=" expression "=" expression  # Error: chained assignment
```

**Error limiting**: Stop after 20 errors by default (configurable via `--max-errors`). Beyond this threshold, messages become noise. Rust, GCC, and Clang all use similar limits.

### Concrete syntax trees for IDE support

The parser should produce a **lossless concrete syntax tree** (CST) preserving all tokens including whitespace, comments, and error nodes. This approach, pioneered by Roslyn and adopted by rust-analyzer's `rowan` library, enables:

- Syntax highlighting even with errors
- Formatting that preserves user intent
- Incremental reparsing (only re-parse changed regions)
- Error recovery that maintains partial information (`foo.` parsed as incomplete member access, still recording `foo` was referenced)

For incremental parsing, consider integrating Tree-sitter's approach or implementing sentential-form incremental LR parsing. The goal: re-parse on every keystroke with sub-10ms latency.

## Comparison of approaches across languages

| Feature | Rust | Elm | Zig | Clang | **Indent (Proposed)** |
|---------|------|-----|-----|-------|----------------------|
| Error codes | E0000 format, 700+ | None | None | None | IND0000 format |
| CLI explain | `--explain E0382` | N/A | N/A | N/A | `--explain IND0042` |
| Suggestions | Inline diffs | Prose hints | @compileError | Fix-Its | Inline diffs + applicability |
| Multi-span | Primary + secondary | Single location | Call stacks | Ranges | Primary + secondary + traces |
| Machine format | JSON (custom) | None | None | JSON, SARIF | JSON + SARIF |
| Documentation | Per-code pages | Handbook | Docs | Minimal | Per-code pages + handbook |
| Tone | Technical but friendly | Conversational | Minimal | Technical | Conversational for DevOps |
| Color support | Full + NO_COLOR | Terminal-aware | Basic | Full + NO_COLOR | Full + NO_COLOR |

## Implementation priorities

For the Indent compiler team, the recommended implementation order:

1. **Core diagnostic structure**: Build the `Diagnostic` type with spans, labels, and children
2. **Human-readable renderer**: Terminal output with colors, line numbers, and span annotations
3. **Error code framework**: Establish IND prefix, documentation templates, and `--explain` command
4. **JSON output**: Enable tooling integration early in development
5. **Suggestion system**: Add `Suggestion` types with applicability levels
6. **Parser recovery**: Implement panic mode with Python-aware synchronization tokens
7. **SARIF output**: Add for CI/CD and GitHub integration
8. **Advanced diagnostics**: Type inference traces, region visualization, macro expansion context

The research is clear: investment in error message quality compounds over time. Every hour spent on diagnostics saves thousands of hours of developer frustration. Rust's decade-long refinement of errors, guided by hundreds of contributors, demonstrates that world-class diagnostics require sustained attention—but the payoff in developer experience and language adoption is substantial.