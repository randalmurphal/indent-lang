# What makes programming languages read like pseudocode

**Empirical research consistently shows that code readability stems from visual structure, reduced cognitive load, and leveraging developer familiarity—not from clever or terse syntax.** For a backend language targeting Python/Go developers, the evidence strongly favors Python-like indentation, explicit keywords over symbols, expression-oriented control flow, and the "one obvious way" philosophy over flexible alternatives. Academic studies using fMRI, eye-tracking, and controlled experiments reveal that factors like meaningful indentation, minimal nesting, and reduced vocabulary size have measurable impacts on comprehension, while popular beliefs about features like McCabe's cyclomatic complexity show no correlation with actual cognitive load.

The research points to a fundamental insight: **code is read far more than it is written**, and optimizing for the reader—especially one unfamiliar with the language—requires deliberate constraints on flexibility. This report synthesizes cognitive science research, language design histories, and documented anti-patterns to provide evidence-based guidance for designing a readable backend language.

---

## Cognitive science reveals what actually matters for comprehension

Neuroimaging research from Janet Siegmund's group at TU Chemnitz has transformed our understanding of code comprehension. Their fMRI studies show that **reading code activates the brain's multiple demand network (general reasoning) rather than language centers**—programmers process code as structured problem-solving, not as language parsing. This finding has direct implications for syntax design: code should minimize abstract reasoning demands, not just be "readable as English."

The strongest predictors of cognitive load are **vocabulary size** (number of unique identifiers) and **data-flow complexity** (how data moves through the code), with correlations of τ ≈ 0.32–0.41 with brain activation in semantic processing and working memory regions. Surprisingly, **McCabe's cyclomatic complexity showed no correlation** with cognitive load in fMRI studies, challenging decades of software engineering orthodoxy.

Working memory constraints are severe—Nelson Cowan's research suggests only **4 chunks** can be held simultaneously, not the traditional 7±2. This means every additional variable, nesting level, or implicit state the programmer must track degrades comprehension. The practical implication: design syntax that minimizes required mental bookkeeping.

Eye-tracking studies reveal that **code reading is fundamentally non-linear**. Expert programmers skip irrelevant sections entirely, while novices read sequentially and struggle to identify what matters. This pattern explains why visual structure is so critical—indentation and formatting serve as preattentive cues that enable experts to rapidly locate relevant code.

## Indentation provides massive, measurable comprehension benefits

Controlled experiments by Morzeck et al. (2023) quantified what programmers have long intuited: **non-indented code required 179% more time** to comprehend than properly indented code for nested conditionals. For JSON-like structures, the penalty reached **544% more time**. The researchers explained this through skip-ability—indentation allows readers to visually identify and skip irrelevant code blocks without conscious processing.

Python's significant whitespace enforces what developers should do anyway, eliminating the gap between visual appearance and semantic meaning. The classic "dangling else" bug in C-family languages—where indentation suggests one structure but braces define another—becomes impossible. As the Python documentation notes: "For a human to be able to read the code, indentation is a much better way of providing the visual cues about block structure. As indentation also contains all the information for the compiler, to use both would be redundant."

Guido van Rossum's rationale for significant whitespace reflects his visual thinking style: "Python for me is incredibly visual. When I read Python, I definitely see it as a two-dimensional structure rather than one-dimensional." This design originated in the ABC language, which was explicitly developed for teaching non-programmers, and the visual structure was found to significantly ease learning.

For a new language, the research strongly supports **mandatory indentation with spaces only** (following Nim's approach of a compile-time error for tabs). This eliminates the tab/space debate entirely while preserving all the comprehension benefits. The recommendation of **4-space indentation** (Python's standard) provides sufficient visual distinction without excessive horizontal sprawl.

## Python's design philosophy aligns with cognitive research

PEP 20's "Zen of Python" principles, written by Tim Peters to capture Guido van Rossum's design philosophy, anticipate many findings from cognitive science. The most critical principles for readability are:

**"Explicit is better than implicit"** reduces cognitive load by eliminating hidden behaviors. Every operation should be visible in the code—no implicit type conversions, no silent failures, no context-dependent meanings. Guido emphasized: "In Python, every symbol you type is essential."

**"There should be one—and preferably only one—obvious way to do it"** (TOOWTDI) was a deliberate counter to Perl's TIMTOWTDI philosophy. The cognitive overhead of multiple equivalent syntaxes is substantial: developers reading unfamiliar code must recognize all variants, and code reviews become debates about style rather than substance. Guido noted that Python's clarity compared to Perl "was definitely one of the reasons why Python took over Perl in the early aughts."

**"Flat is better than nested"** aligns directly with research showing that minimizing nesting decreases reading time and increases comprehension confidence. Deep nesting forces programmers to maintain more context in working memory and makes it harder to identify which block ends where.

**"Readability counts"** reflects Guido's foundational insight: "Code is read much more often than it is written." This principle drove Python's trade-offs—accepting slower execution for faster development and maintenance, prioritizing human time over machine time in an era when developer salaries dwarf computing costs.

Python's syntax choices implement these principles through **keywords over symbols** (`and`, `or`, `not` instead of `&&`, `||`, `!`), **English-like constructs** (`if x in list`, `for item in collection`), and **minimal special characters**. The result is code that Guido describes as accessible to "very small snippets that require very little understanding of terminology and concepts from programming before they make sense."

## Lua demonstrates that minimal mechanisms enable maximum power

Lua's design philosophy—**"mechanisms instead of policies"**—offers a complementary perspective on achieving readability through simplicity. With only **22 keywords** (compared to Python's 38, Ruby's 41, or Swift's 97), Lua achieves remarkable expressiveness through a small set of composable primitives.

The language provides exactly one general mechanism for each major programming aspect: **tables for data, functions for abstraction, and coroutines for control**. Tables serve as arrays, dictionaries, records, objects, modules, and sets—not through special syntax but through conventions. This economy of concepts means the entire reference manual spans only ~100 pages covering the language, libraries, and C API.

Lua's creators explicitly prioritized this minimalism for learnability. Roberto Ierusalimschy noted: "Because many potential users of the language were not professional programmers, the language should avoid cryptic syntax and semantics." The language deliberately omits features other languages consider essential—no built-in class system (use metatables), no module keyword (return a table), no try-catch syntax (use `pcall`)—forcing programmers to understand the underlying mechanisms rather than memorizing special cases.

The trade-off is acknowledged honestly: Lua "is not as good as other scripting languages for writing quick-and-dirty programs" and requires "an initial phase for programmers to set up the language." But this explicitness creates code that is easier to understand because the mechanisms are visible rather than hidden behind syntactic sugar. For a backend language where long-term maintainability matters more than quick prototyping, this philosophy has merit.

## Multiple ways to do things impose real cognitive costs

Ruby's inheritance of Perl's TIMTOWTDI philosophy demonstrates the readability costs of syntactic flexibility. The Ruby community itself recommends constrained style guides because the language's flexibility creates "mental overhead due to inconsistency" when reading unfamiliar code.

The `and`/`or` versus `&&`/`||` precedence trap exemplifies the problem. These operators have different precedence in Ruby, leading to surprising behavior:

```ruby
foo = true and false   # foo = true (parsed as: (foo = true) and false)
bar = true && false    # bar = false (parsed as: bar = (true && false))
```

The Ruby Style Guide explicitly warns against `and`/`or` because they "add cognitive overhead, as they don't behave like similarly named logical operators in other languages." This is precisely the trap Python avoided by having only one set of boolean operators.

Perl's "symbol soup"—punctuation variables like `$!`, `$/`, `@_`, `$#`—created what programmers call "write-only code." Paul Graham quipped that "Perl is the only language that looks the same before and after RSA encryption." The Perl community acknowledged these problems; Perl 6 (Raku) explicitly redesigned to remove "the majority of the punctuation variables" that caused "line noise" criticism.

**Variant sigils** compound the confusion. In Perl, `@array`, `$array[0]`, and `@array[0,1]` all refer to the same underlying array with different sigil prefixes depending on context—array, scalar element, or slice. This context-dependence forces readers to track additional state and is documented as "a major point of confusion for newcomers."

The pattern that emerges: **readability requires constraint**. Features that feel powerful when writing become burdens when reading, especially when reading code written by others or by one's past self.

## Context-dependent parsing creates lasting problems

C++'s "most vexing parse"—where `TimeKeeper time_keeper(Timer());` declares a function rather than an object—demonstrates how grammar ambiguity creates silent bugs. The C++ committee added uniform initialization syntax (`{}`) in C++11 specifically to work around this problem, but legacy code and habits persist.

The deeper issue is **context-dependent parsing**, where the same token sequence has different meanings depending on earlier declarations. In C, `T * x;` could be a pointer declaration or multiplication depending on whether `T` was previously typedef'd. This requires the lexer to track type definitions—the language cannot be parsed with context-free grammar.

The costs compound across tooling. Every IDE, linter, formatter, and editor must implement the same complex parsing logic. Different compilers may interpret edge cases differently. Formal verification becomes significantly harder. Error messages confuse users because the parser's interpretation differs from the programmer's intent.

For language design, the recommendation is clear: **use context-free grammar** that can be parsed purely from syntax, without semantic information. Avoid overloading operators for unrelated purposes (C++'s use of `<>` for both templates and comparisons). Use distinct tokens for distinct operations.

## Expression-oriented design offers benefits with caveats

Rust's design philosophy that "most forms of value-producing evaluation are directed by the uniform syntax category of expressions" enables more concise code with less mutable state. Where Go requires:

```go
var status string
if cpu.temperature <= MAX_TEMP {
    status = "ok"
} else {
    status = "error"
}
```

Rust allows:

```rust
let status = if cpu.temperature <= MAX_TEMP { "ok" } else { "error" };
```

The expression-oriented approach eliminates the intermediate mutable variable and makes data flow explicit. The pattern matching integrates naturally with assignment, and blocks can return values directly. Developers report that "code feels more natural to compose once you embrace expressions as a core mechanic."

However, expression-orientation can harm readability when overused. Complex chained expressions become hard to parse visually, and the significance of semicolons (which change an expression into a statement in Rust) adds a learning curve. Critics note that expression-orientation enables "an entire class of programming mistakes wherein a programmer accidentally codes an assignment expression, which replaces a variable with an expression rather than testing it for equality."

For Python/Go developers, neither language uses expression-oriented control flow—Python's `if` is a statement (only the ternary `x if cond else y` is an expression), and Go is entirely statement-based. A new language should **introduce expression orientation gradually**: make `if`/`match` expressions optional features with clear documentation, but keep explicit `return` available and idiomatic for complex functions.

## Keyword design matters less than consistency

The debate over `fn` versus `func` versus `def` versus `fun` versus `function` generates strong opinions but limited evidence. Research on keyword recognition speed is sparse, and the documented rationale from language designers focuses more on consistency than optimization.

Rust chose `fn` for brevity and alignment with its other two-letter keywords (`pub`, `mut`, `ref`). The justification: "`fn` simply doesn't need any more letters to make its meaning obvious," and the abbreviation is already familiar from keyboard function keys. Go chose `func` as a balance between brevity and clarity. Python's `def` (short for "define") aligns with its English-like philosophy. Kotlin's `fun` was chosen because "we like it" and provides visual consistency with `val` and `var`.

The emerging consensus from design discussions: **for high-frequency keywords, brevity matters; for low-frequency keywords, clarity matters**. Function definitions are high-frequency, justifying short forms. Error handling constructs may be lower frequency, allowing more explicit keywords.

For Python/Go developers, **`func` offers the best familiarity-brevity balance**—familiar to Go developers, a clear abbreviation, and not excessively verbose. `fn` is acceptable if targeting a more systems-oriented feel, while `def` would ease Python adoption.

## Whitespace handling requires clear, strict rules

Comparing Python, Haskell, and Nim's approaches to significant whitespace reveals important implementation details. Python uses INDENT/DEDENT tokens during lexical analysis, requires colons before blocks (helping editors auto-indent), and since Python 3 **disallows mixing tabs and spaces entirely** (throwing `TabError`).

Haskell's "layout rule" is more complex—it applies only after specific keywords (`where`, `let`, `do`, `of`) and supports both braces and indentation. This flexibility complicates tooling: as the Haskell-mode documentation notes, "For indentation to know when blocks close, it must fully implement Haskell parser that accepts every Haskell language prefix."

Nim takes the **strictest approach: spaces only, with tabs causing a compile-time syntax error**. This eliminates the tab/space debate entirely and has proven effective in practice.

The copy-paste problem—where indentation breaks when pasting code between different indentation levels—affects all significant-whitespace languages. Mitigations include IDE smart paste features, rectangle selection for block operations, and format-on-paste normalization. For a new language, building a **Black-style opinionated formatter from day one** significantly reduces this friction.

For tooling, LSP implementations face challenges because indentation decisions require full parse tree analysis, which is expensive during real-time editing. Solutions include tree-sitter for incremental parsing, heuristic-based indentation with grammar fallback, and format-on-save rather than real-time formatting.

## Familiarity consistently outweighs theoretical readability advantages

Perhaps the most important finding from cognitive research: **training and familiarity consistently dominate over any inherent readability differences**. Studies on camelCase versus snake_case found that performance follows the style developers learned first, not any objective property. Expert programmers show different reading patterns because of experience, not innate ability.

This has profound implications for language design. Borrowing heavily from languages your target users already know—Python and Go in this case—provides immediate comprehension benefits that novel "improvements" may not overcome. Novel syntax should be introduced sparingly, with clear rationale, and ideally with a transition path that leverages existing mental models.

The "principle of least surprise," first articulated in PL/I documentation in 1967, operationalizes this insight: "Every construct in the system should behave exactly as its syntax suggests." For Python/Go developers, this means `if`, `for`, and `func` should work as expected; deviations, even if theoretically superior, impose learning costs that must be justified by substantial benefits.

## Practical recommendations for a new backend language

Based on the research synthesis, a backend language targeting Python/Go developers should prioritize:

**Adopt from Python**: Significant indentation with INDENT/DEDENT tokens; colons before blocks; implicit continuation inside brackets and parentheses; 4-space standard indentation; keywords over symbols for boolean operators (`and`/`or`/`not`); the "one obvious way" philosophy enforced through limited syntactic choices.

**Adopt from Nim**: Spaces-only enforcement at compile time (eliminating tab/space debates); flexible continuation after operators and commas.

**Adopt from Rust/Kotlin**: Expression-oriented `if`/`match` constructs (optional, with explicit `return` available); Result-based error handling with explicit propagation.

**Adopt from Go**: `func` keyword for familiarity; explicit type annotations; straightforward concurrency primitives.

**Avoid explicitly**: Multiple ways to do the same thing (Ruby's trap); punctuation-heavy special variables (Perl's trap); context-dependent parsing (C's trap); operator overloading that changes meaning (C++'s trap); implicit behaviors that hide control flow.

**Build tooling from day one**: An opinionated formatter (Black-style); LSP with format-on-save; paste-and-indent editor plugins; a Concrete Syntax Tree library for code transformation tools.

## Conclusion

The research converges on a counterintuitive insight: **constraints create readability**. Python's one obvious way, Lua's minimal mechanisms, and research showing indentation's massive comprehension benefits all point toward deliberate limitation rather than flexible power. The language that reads like pseudocode does so not because it offers many ways to express ideas, but because it offers one clear way that matches what developers already expect.

For Python/Go developers, the path is clear: leverage their existing mental models through familiar keywords and control structures, enforce visual structure through mandatory indentation, provide expression-oriented constructs as ergonomic improvements rather than required paradigm shifts, and build opinionated tooling that eliminates style debates. The goal—code that senior developers can read without learning the language—is achievable precisely because familiarity and visual structure matter more than theoretical elegance.