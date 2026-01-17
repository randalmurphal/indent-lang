# AI-Friendly Programming Language Design for Indent

A new programming language optimized for AI code generation faces a fundamental design challenge: **94% of LLM-generated compilation errors are type-check failures**, according to GitHub's January 2026 research. This single statistic reshapes everything about how "Indent" should be designed. The language must prioritize explicit, constrained syntax with mandatory static typing—not despite AI assistants, but because of them. Research on domain-specific languages shows that explicit, keyword-heavy syntax delivers a **40 percentage point accuracy improvement** over Python's flexible patterns, while still maintaining human readability. The path forward is clear: design Indent for maximum predictability, leverage structural similarity to well-supported languages like Python and Go, and build verification into the core tooling from day one.

## LLM performance reveals which syntax patterns actually work

Modern code LLMs demonstrate dramatic performance variations across programming languages. The MultiPL-E benchmark (Cassano et al., IEEE TSE 2023), which translated HumanEval and MBPP problems to 18 languages, found that Codex performs best on JavaScript and equally well on C++, Scala, and TypeScript as on Python. Performance correlates with language popularity, but crucially, **some niche languages perform as well as popular ones** when their syntax is consistent and explicit.

Token efficiency analysis across 19 languages using identical RosettaCode tasks revealed a **2.6x difference** between the least efficient (C at ~280 tokens) and most efficient (Clojure at ~109 tokens) languages. The key drivers of efficiency include:

- **Significant indentation** eliminates brace and semicolon tokens (matching Indent's namesake)
- **Aggressive type inference** achieves near-dynamic efficiency with static typing benefits
- **First-class functions** enable composition over repetition
- **Pattern matching** reduces verbose conditional chains

The most common LLM code generation errors fall into predictable categories: code block errors account for **43-60%** of mistakes, followed by conditional errors, loop boundary problems, and incorrect return values. Critically, most LLM-generated errors are syntactically valid but semantically wrong—the models have learned syntax but struggle with logic. GPT-4's paradox is notable: when it makes mistakes, those mistakes deviate **more** from correct solutions than smaller models, requiring more edits to fix.

## Explicit syntax dramatically outperforms flexible alternatives

The Anka DSL study (arXiv 2512.23214) provides the most compelling evidence for explicit syntax design. This domain-specific language for data pipelines achieved **100% accuracy** on multi-step tasks compared to Python's 60%—a 40 percentage point improvement. Claude 3.5 Haiku achieved **99.9% parse success** and **95.8% overall task accuracy** on Anka despite zero prior training on the language.

The design principles that drove this success should guide Indent's development:

| Principle | Implementation | Evidence |
|-----------|----------------|----------|
| Named blocks | Use `STEP validate:` and `END STEP` markers | Provides scaffolding that guides LLMs through sequential operations |
| English keywords | Prefer `and`/`or` over `&&`/`||` | "LLMs excel at natural language"—keywords leverage this strength |
| Mandatory types | Require type annotations on function signatures | 94% of errors are type failures (GitHub 2026) |
| Explicit returns | Require `return` keyword | Removes ambiguity about function behavior |
| One canonical form | Single way to express each pattern | Reduces LLM confusion from multiple valid syntaxes |

Type-constrained decoding research (arXiv 2504.09246) demonstrates that using type systems to guide LLM generation **reduces compilation errors by more than half** and significantly increases functional correctness across code synthesis, translation, and repair tasks. This works across LLMs of various sizes and model families, suggesting type systems are the single most impactful feature for AI code quality.

The recommendation is clear: Indent should use mandatory static typing with inference for local variables, explicit type annotations on function signatures, and rich type features including generics, union types, and optional types. Gradual typing creates ambiguity that LLMs exploit—avoid it.

## Documentation structure that AI systems can actually parse

Structured documentation dramatically improves code generation quality. Research shows functions with detailed docstrings generate **2-3x more accurate implementations** than undocumented code. The documentation format should follow a machine-parseable structure:

```indent
@doc """
  Brief: Calculate compound interest over time.
  
  @param principal : Float - Initial investment amount
  @param rate : Float - Annual interest rate (0.05 = 5%)
  @param years : Int - Number of years to compound
  @returns : Float - Final amount after compounding
  
  @example
    result = compound_interest(1000.0, 0.05, 10)
    # Expected: 1628.89
  
  @complexity O(1) time, O(1) space
  @see simple_interest, continuous_compound
"""
func compound_interest(principal: Float, rate: Float, years: Int) -> Float:
    return principal * pow(1 + rate, years)
end func
```

Each element serves a specific purpose for LLM consumption. The **Brief** line provides a single-sentence summary that helps models understand purpose quickly. Typed parameters reduce hallucination in generated code. The **@example** section with expected outputs serves as an implicit test for generated implementations. Semantic tags like `@see` and `@complexity` enable better retrieval-augmented generation for code search.

## Error messages designed for AI correction, not just human reading

Rust's error message design (RFC 1644) provides the model for AI-friendly diagnostics. The structure uses primary labels to explain "what" went wrong and secondary labels to explain "why," creating a narrative that both humans and AI can follow:

```
error[I0042]: Type mismatch in function argument
  --> app/main.indent:15:12
   |
14 |  func greet(name: String):
   |             ---- parameter expects `String`
15 |      greet(42)
   |            ^^ found `Int`, expected `String`
   |
   = help: Convert the integer to a string using `to_string(42)`
   = note: See https://indent-lang.org/errors/I0042 for more examples
```

The critical innovation for AI consumption is **machine-readable output** with suggestion applicability levels. Following Rust's pattern, each suggestion should be classified:

- **MachineApplicable**: Can be applied automatically by AI/tools
- **HasPlaceholders**: Contains placeholders like `<type>` requiring user input
- **MaybeIncorrect**: May or may not fix the issue
- **Unspecified**: Unknown confidence level

Error messages should output as structured JSON when requested (`--error-format=json`), including a `rendered` field with human-readable text so AI tools can access both structured data and formatted output. The SARIF format provides additional compatibility with VS Code diagnostics and static analysis tools.

## Bootstrapping AI support without training data

The cold-start problem for a new language is solvable. Research on cross-lingual code transfer (Baltaji et al., arXiv 2310.16937) across 1,808 experiments and 41 languages reveals that **learning transfers well across programming languages** for both classification and generation tasks. The best source languages for transfer are Kotlin and JavaScript; the best target languages are Java, Go, Dart, and TypeScript.

This research directly informs Indent's syntax design strategy:

- **Python-like indentation semantics** maximize transfer from the largest code corpus
- **Go-like type annotations** align with high-transfer target languages
- **TypeScript-like typing patterns** transfer well across model families
- **Familiar keywords** (`func`, `if`, `for`, `return`) prevent confusion with novel terminology
- **Avoid C++ patterns**—C++ is consistently a poor source and target for transfer

For few-shot prompting in a new language, the optimal approach uses 3-5 carefully selected examples covering diverse patterns (loops, conditionals, functions, types), with consistent formatting and structure across all examples. The prompt should include a brief language specification in natural language that LLMs understand, explicit mappings to familiar constructs ("X in Indent is like Y in Python"), and input-output examples demonstrating language behavior.

Synthetic training data generation follows a practical pipeline: seed data creation (100-200 high-quality manual examples), transpilation-based expansion from Python/Go/TypeScript, LLM-assisted generation with execution verification, and Case2Code synthesis pairing inputs/outputs with implementations. Research on teaching GPT-4o-mini a new language (Hyperlambda case study) required approximately **3,500 training snippets** with ~20 lines each—roughly 80,000 lines of code total.

## Tooling architecture for the AI-assisted future

Modern AI coding tools reveal patterns that should be built into Indent's design from the start. Cursor IDE's architecture demonstrates key principles: multi-cloud, edge-optimized, model-agnostic backends delivering sub-100ms completions through caching, with codebase indexing for whole-project understanding. The `.cursorrules` files for project-specific AI instructions suggest that Indent should have first-class support for AI configuration at the language level.

Sourcegraph Cody's architecture shows that Tree-sitter integration for concrete syntax trees enables precise language understanding. Clear indentation rules help completion engines distinguish single-line versus multi-line completion needs. Predictable block boundaries help AI understand completion intent.

For agent-based programming—an increasingly important use case as tools like Devin, OpenHands, and Claude Code become more capable—the language should support:

- **Sandboxed execution blocks** where AI-generated code runs with restricted permissions
- **Deterministic builds** for reproducibility across agent sessions
- **Transaction-like code blocks** enabling rollback on failure
- **Structured error messages** with machine-applicable fix suggestions
- **Audit trails** for code modifications by agents

The SWE-bench evaluation framework, where OpenHands achieved a **72% resolve rate** with Claude Sonnet 4.5, tests issue resolution without breaking existing tests, multi-file modifications, and dependency management. Languages supporting isolated module compilation, test discovery protocols, and clear import/dependency graphs perform better in these scenarios.

## Verification features that make AI-generated code trustworthy

Formal verification of AI-generated code is becoming practical. Research shows GPT-4 achieved a **58% success rate** generating verified Dafny methods with Chain-of-Thought prompts. The Clover framework from Stanford demonstrates a closed-loop verification approach: generate candidate solutions, formally check correctness, accept only verified code.

Indent should build verification into its core syntax through design-by-contract features:

```indent
func binary_search(arr: Array<Int>, target: Int) -> Option<Int>
    requires arr.is_sorted()
    ensures result.is_some() implies arr[result.unwrap()] == target
    ensures result.is_none() implies target not_in arr
:
    // Implementation
end func
```

Property-based testing should be a first-class language feature rather than a library concern:

```indent
property test "list operations":
    generators:
        lst: List<Int>.random(max_size: 100)
    
    check "reverse is involutive":
        lst.reverse().reverse() == lst
    
    check "sort is idempotent":
        lst.sort().sort() == lst.sort()
end property
```

These verification features serve dual purposes: they catch AI hallucinations before runtime, and they provide specifications that AI can use to generate correct implementations.

## Language comparison reveals AI success patterns

Synthesizing benchmark data across multiple studies reveals clear patterns in AI code generation success:

| Language Feature | AI Success Impact | Recommendation for Indent |
|-----------------|-------------------|---------------------------|
| Static typing | +50% reduction in compilation errors | Mandatory, with inference for locals |
| Explicit keywords | +40% accuracy on complex tasks | Prefer `and`/`or`, `func`, `match` |
| Consistent patterns | Reduces "garbage code" from 38% to <10% | One canonical form per operation |
| Type inference | Matches dynamic typing efficiency | Use Hindley-Milner style inference |
| Named parameters | Prevents argument order errors | Default for 3+ parameter functions |
| Significant whitespace | ~20% token savings | Use indentation (namesake feature) |
| Pattern matching | Reduces conditional errors | Primary control flow mechanism |

The token efficiency comparison is particularly instructive. Dynamic languages save 30-40% of tokens through omitting type declarations, but **type inference wins overall**—Haskell and F# achieve near-dynamic efficiency with full static typing benefits. Functional composition through higher-order functions reduces token count without sacrificing expressiveness.

## Recommended syntax specification for Indent

Based on all research findings, the recommended core syntax prioritizes explicitness, consistency, and AI-friendliness:

```indent
# Module declaration with AI context
@ai_context("Payment processing with fraud detection")
module Payments:

# Type definitions
type PaymentResult = Success(amount: Float) | Failure(reason: String)

# Function with contracts
func process_payment(user: User, amount: Float) -> PaymentResult
    requires amount > 0
    requires user.is_verified()
    ensures result is Success implies user.balance >= 0
:
    if user.balance < amount:
        return Failure("Insufficient funds")
    end if
    
    user.balance = user.balance - amount
    return Success(amount)
end func

# Property-based test
property test "payment invariants":
    check "balance never negative":
        forall user: User, amount: Float where amount > 0:
            let result = process_payment(user, amount)
            user.balance >= 0
end property

end module
```

Key design decisions reflected in this syntax: explicit `end` keywords for block termination (reduces LLM confusion about scope), named block structures throughout, contracts as first-class syntax, property testing integrated at the language level, and AI context annotations for tooling.

## Implementation roadmap for AI-first tooling

The recommended development sequence prioritizes AI capabilities from day one:

**Months 1-2**: Create a language specification document readable by LLMs, write 200 canonical examples covering all constructs, build few-shot prompt templates, and create an evaluation benchmark by translating 50 HumanEval problems to Indent.

**Months 3-4**: Build a transpiler from Python to Indent, generate 3,000 synthetic examples via transpilation, create Case2Code training pairs, and validate all examples via execution.

**Months 5-6**: Fine-tune StarCoder-3B with LoRA on generated data, measure on the Indent-HumanEval benchmark, iterate on data quality based on failure modes, and release an initial "Indent Copilot."

**Months 7-12**: Expand to 10,000+ examples, fine-tune larger models (7B-15B), implement full LSP with AI extensions, and build community contributions of code and examples.

The tooling architecture should include an LSP server with standard capabilities plus AI-specific extensions (`indent/aiSuggestFix`, `indent/verifyContract`), structured JSON diagnostic output with machine-applicable suggestions, integration with the emerging Agent Client Protocol from Zed Editor, and property-based test generation from specifications.

## Conclusion: design for the AI-augmented future

The research converges on a clear design philosophy: **Indent should be more constrained than existing languages, not less**. The flexibility that makes Python attractive to human programmers is precisely what causes LLMs to generate incorrect code. Explicit syntax, mandatory static typing, keyword-heavy constructs, and built-in verification create guardrails that transform AI assistants from unreliable suggestionmakers into trustworthy code generators.

The most counterintuitive finding is that verbose, explicit syntax actually improves both AI generation accuracy and human readability—these goals are aligned, not in conflict. Chris Lattner's insight that "the readability of a programming language is more important than how easy it is to write" becomes even more relevant when AI agents are doing much of the writing.

Three design principles should guide every syntax decision: **predictability** (one canonical way to express each pattern), **verifiability** (contracts and property tests as first-class features), and **familiarity** (Python-like structure with Go-like types for maximum transfer learning). A language built on these principles can achieve competitive AI support with only 3,000-5,000 quality training examples—far less than the millions of lines training data for established languages. The path to an AI-first programming language is achievable today with the research and tooling that now exists.