# Comprehensive testing patterns for a Rust-like compiled language

A new compiled language called Indent—featuring `#[test]` attributes, compiler-rewritten assert macros, and trait-based polymorphism—can implement a testing ecosystem that combines the best patterns from Rust, Go, Python, Java, and C++. The recommended approach centers on **compile-time mock generation via procedural macros**, **attribute-based test isolation with scoped fixtures**, **LLVM-integrated coverage instrumentation**, **YAML-based snapshot testing with redaction selectors**, **phased mutation testing starting with source-level mutations**, and a **hybrid test organization** combining inline unit tests with a separate integration test directory.

This design prioritizes zero runtime overhead in production, type-safe test configurations verified at compile time, and developer ergonomics comparable to dynamic language frameworks while respecting the constraints of a statically-typed compiled language without runtime reflection.

## Mocking strategy: procedural macro generation on traits

Languages without runtime reflection—including Rust, C++, and Go—cannot use the bytecode manipulation that powers Java's Mockito or the runtime object patching that enables Python's unittest.mock. Instead, they must rely on compile-time code generation combined with trait/interface abstraction. After analyzing mockall (Rust), gomock (Go), Google Mock (C++), and Mockito (Java), the recommended approach for Indent is a **procedural macro system that generates mock implementations from trait definitions**.

The `#[mock]` attribute parses a trait definition at compile time and generates a corresponding `MockTraitName` struct containing expectation storage, call counters, and builder methods for each trait method. This approach requires no runtime reflection metadata in production binaries and catches configuration errors at compile time rather than test runtime.

```indent
#[mock]
trait PaymentGateway {
    fn process(&self, amount: f64) -> Result<TransactionId, Error>;
    async fn refund(&self, tx_id: TransactionId) -> Result<(), Error>;
}

#[test]
fn test_order_processing() {
    let mut mock = MockPaymentGateway::new();
    mock.expect_process()
        .with(arg::eq(99.99))
        .times(1)
        .returning(|_| Ok(TransactionId::new("tx-123")));
    
    let service = OrderService::new(mock);
    assert!(service.checkout(99.99).is_ok());
}
```

The mock struct stores expectations in a `HashMap<MethodId, Vec<Expectation>>` and validates them either on explicit `verify()` calls or automatically when the mock is dropped. Argument matching uses a type-safe predicate system with combinators like `eq()`, `any()`, and `custom(|x| predicate)`. For external traits not defined with `#[mock]`, a `mock! {}` declarative macro allows users to define the trait signature manually and generate the corresponding mock.

The critical design constraint is that **mocking requires trait abstraction**. Code that instantiates concrete types directly cannot be mocked without refactoring. This is an acceptable trade-off that encourages dependency injection and clean architecture. Constructor injection should be the primary pattern: services accept trait-bounded type parameters rather than constructing dependencies internally.

For HTTP mocking specifically, Indent should provide a built-in mock server fixture similar to Rust's wiremock, where each test gets an isolated server on a random port that automatically shuts down after the test completes.

## Test isolation: attributes, fixtures, and resource locks

Parallel test execution creates isolation challenges for database connections, file systems, environment variables, and global state. After studying Rust's serial_test crate, Go's `t.TempDir()` and `t.Parallel()`, pytest's scoped fixtures, and JUnit 5's resource locking, Indent should implement a **multi-layered isolation system** combining execution control attributes, scoped fixtures, and explicit resource locks.

The default behavior should run tests in parallel across multiple threads within a single process. Tests that mutate shared state opt into serialization via attributes:

```indent
#[test]
#[serial]                      // Never runs with other #[serial] tests
fn test_global_config() { }

#[test]  
#[serial(database)]            // Serialized only against same-keyed tests
fn test_db_migration() { }

#[test]
#[isolated]                    // Runs in separate process with own memory space
fn test_environment_manipulation() { }

#[test]
#[resource_lock("shared_cache", mode = read)]   // Read lock allows concurrency
fn test_cache_read() { }

#[test]
#[resource_lock("shared_cache", mode = read_write)]  // Write lock is exclusive
fn test_cache_write() { }
```

The fixture system should support **scoped lifecycle management** inspired by pytest. Fixtures are functions annotated with `#[fixture]` that return a resource, optionally with cleanup via `defer` blocks. Scope determines sharing: function scope creates fresh instances per test, module scope shares across tests in a file, and session scope persists for the entire test run.

```indent
#[fixture(scope = function)]
fn temp_directory() -> TempDir {
    TempDir::new()
}  // Automatically deleted when test completes

#[fixture(scope = module)]
fn database_pool() -> Pool {
    let pool = Pool::connect(&test_url());
    defer { pool.close(); }
    pool
}

#[test]
fn test_with_fixtures(dir: TempDir, db: &Pool) {
    // dir is fresh (function scope), db is shared (module scope, borrowed)
}
```

For database isolation specifically, Indent should support three patterns: **transaction rollback** (wrap test in transaction that rolls back), **template databases** (clone from template for each test), and **containerized databases** via Testcontainers integration. The `#[db_transaction]` attribute provides the rollback pattern:

```indent
#[test]
#[db_transaction]
fn test_user_creation(tx: &Transaction) {
    tx.execute("INSERT INTO users (name) VALUES ('test')");
    assert_eq!(tx.query_one("SELECT COUNT(*) FROM users"), 1);
}  // Automatically rolled back
```

Environment variable isolation requires special handling because env vars are process-global state. The `#[env(...)]` attribute should set variables for the test duration and automatically restore original values, using an internal mutex to prevent concurrent modification by parallel tests:

```indent
#[test]
#[env(API_KEY = "test_key", DEBUG = "true")]
fn test_with_env_vars() {
    // Variables are set; restored after test
}
```

## Coverage tooling: LLVM source-based instrumentation

Code coverage for compiled languages can be implemented through source rewriting (Go's approach), bytecode instrumentation (JaCoCo's approach), or compiler-integrated instrumentation (LLVM's approach). For a language targeting LLVM, the **compiler-integrated approach** provides the best combination of precision, optimization resistance, and advanced coverage types including MC/DC.

LLVM source-based coverage works by inserting counter increment instructions at the IR level during compilation, along with coverage mappings that connect counters to source code regions. The binary embeds these mappings in a `__llvm_covmap` section. When tests run, counters increment and write to `.profraw` files. Post-processing tools merge profiles and generate reports.

The recommended architecture for Indent:

```
Compiler Pipeline:
  Parser → AST → Coverage Transform at MIR → LLVM IR with Counters + Mappings

Runtime:
  Counter arrays + .profraw writer with LLVM_PROFILE_FILE support

Tooling:
  indent-cov merge|show|report|export|html
```

Indent should support four coverage levels. **Line coverage** measures executed source lines and serves as the basic metric. **Branch coverage** tracks whether both true and false paths of conditionals execute. **Region coverage** provides finer granularity by tracking contiguous code regions between control flow points. **MC/DC coverage** (Modified Condition/Decision Coverage) verifies that each condition independently affects decision outcomes—required by safety-critical standards like DO-178C and ISO 26262.

The primary output format should be **LCOV** for broad CI/CD compatibility with Codecov, Coveralls, and SonarQube. Secondary support for **Cobertura XML** enables integration with GitLab and Jenkins. A JSON export format supports custom tooling. HTML reports with syntax-highlighted source provide developer-friendly visualization.

For CI/CD integration, Indent should enforce **differential coverage** on changed lines rather than just overall coverage thresholds. This approach—requiring 80-90% coverage of lines modified in a PR while accepting lower overall coverage—is more achievable for teams and directly attributes testing responsibility:

```bash
indent test --coverage
indent coverage --fail-under=70              # Overall threshold
indent coverage --diff-fail-under=85 main    # Changed lines threshold
indent coverage --lcov -o coverage.info      # Export for CI services
```

Configuration in `indent.toml` should specify thresholds, excluded patterns, and output formats:

```toml
[coverage]
types = ["line", "branch"]
thresholds = { overall = 70, new_code = 85 }
exclude = ["tests/*", "*_generated.indent"]
formats = ["html", "lcov"]
```

## Snapshot testing: external files with selector-based redaction

Snapshot testing—comparing test outputs against stored reference values—excels for large structured outputs, compiler error messages, serialized data structures, and regression testing during refactoring. After analyzing Rust's insta, JavaScript's Jest, Go's go-snaps, and Python's syrupy, Indent should implement a **macro-based snapshot system with external file storage and path-based redaction**.

The core assertion macros capture output in various formats:

```indent
#[test]
fn test_compiler_output() {
    let ast = parse("let x = 1 + 2");
    
    assert_snapshot!(ast);                    // Uses Debug formatting
    assert_json_snapshot!(api_response);      // JSON serialization
    assert_yaml_snapshot!(config);            // YAML serialization
    assert_snapshot!(name = "custom", value); // Named snapshot
    assert_snapshot!(value, @"inline");       // Inline in source
}
```

Snapshots are stored in `snapshots/` directories alongside test files, using a YAML metadata header followed by the snapshot content:

```yaml
---
source: src/parser/tests.indent
expression: parse("let x = 1 + 2")
---
BinaryExpr {
    left: Literal(1),
    op: Add,
    right: Literal(2),
}
```

Non-deterministic data like timestamps, UUIDs, and random IDs must be handled through **redaction selectors** that replace values with placeholders:

```indent
assert_json_snapshot!(user, {
    ".id" => "[uuid]",
    ".created_at" => "[timestamp]",
    ".sessions[]" => "[session]",
    ".metadata.**" => "[redacted]",
});
```

Selectors support path navigation (`.key`, `[index]`), wildcards (`[]` for all items, `.*` for all keys), and recursive matching (`.**` for deep paths). Dynamic redactions can validate format before replacing, and `sorted_redaction()` handles non-deterministic ordering in sets and maps.

The update workflow requires explicit acceptance. Running tests creates `.snap.new` files for mismatches. The `indent snap review` command provides an interactive TUI for accepting or rejecting changes. Environment variables control behavior:

```bash
INDENT_SNAPSHOT_UPDATE=no       # CI mode: fail on mismatch (default in CI)
INDENT_SNAPSHOT_UPDATE=auto     # Development: create .snap.new files
INDENT_SNAPSHOT_UPDATE=always   # Force accept all (use sparingly)
```

For code review, snapshot changes should receive the same scrutiny as source changes. Large snapshot diffs warrant investigation rather than blind acceptance. The `--unreferenced=reject` flag catches orphaned snapshots from deleted tests.

## Mutation testing: phased implementation with coverage-based optimization

Mutation testing—introducing small code changes and checking if tests detect them—provides stronger quality assurance than coverage alone. A test suite with 100% line coverage might have zero meaningful assertions; mutation testing reveals whether tests actually verify behavior. After analyzing cargo-mutants (Rust), PIT (Java), mutmut (Python), and go-mutesting (Go), mutation testing is **feasible for Indent but requires aggressive optimization** to manage recompilation costs.

The primary challenge for compiled languages is that each mutation requires rebuilding at least part of the binary. Java's PIT achieves superior performance through bytecode-level mutation that avoids recompilation entirely—mutations are applied at runtime via the Java instrumentation API. For Indent, the recommended approach is a **phased implementation** starting with source-level mutation and evolving toward IR-level mutation.

**Phase 1** implements source-level mutation with incremental compilation. The mutation engine parses source files, generates mutations via AST manipulation, writes modified files, and uses incremental builds (reusing unchanged compilation artifacts). This approach works immediately but remains slower than bytecode mutation.

**Phase 2** moves mutation to the compiler IR level. After parsing and type-checking succeed, the engine injects mutations just before code generation. This avoids re-parsing and re-type-checking for each mutation, providing **10-100x performance improvement** over source-level mutation.

The most critical optimization is **coverage-based test selection**. Rather than running the entire test suite for each mutant, the engine maps tests to code they execute and runs only relevant tests. This typically reduces test execution to 5-10% of the full suite per mutant.

Mutation operators should focus on those with highest fault-detection value:

| Operator | Mutation | Catches |
|----------|----------|---------|
| Relational boundary | `>` → `>=`, `<` → `<=` | Off-by-one errors |
| Negate conditional | `==` → `!=`, `&&` → `\|\|` | Logic inversions |
| Arithmetic | `+` → `-`, `*` → `/` | Calculation errors |
| Return value | Replace with default | Missing assertions |
| Statement deletion | Remove statement | Untested side effects |

For CI integration, mutation testing should focus on changed code via `--in-diff`:

```bash
indent mutants --in-diff origin/main    # Only mutate changed lines
indent mutants --shard 0/4              # Distributed across CI workers
indent mutants --timeout 300            # Kill stuck mutants
```

The bottom line: mutation testing is practical and valuable for Indent, but **coverage-based test selection is mandatory** for acceptable performance. Without it, mutation testing on moderate codebases becomes intractable. The long-term goal should be IR-level mutation to match PIT's performance characteristics.

## Test organization: inline unit tests with separate integration directory

Test file organization affects discoverability, access to private APIs, and separation of test types. After comparing Rust's inline modules, Go's adjacent `*_test.go` files, Python's `tests/` directory, and Java's mirrored `src/test/` structure, Indent should adopt a **hybrid approach** combining Rust's inline unit tests with a dedicated integration test directory.

Unit tests live inline within source files inside `#[cfg(test)]` modules. This placement provides access to private functions, keeps tests close to implementation, and excludes test code from production builds:

```indent
// src/parser.indent
pub fn parse(input: &str) -> Ast { /* ... */ }

fn tokenize(input: &str) -> Vec<Token> { /* private */ }

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn parses_simple_expression() {
        let ast = parse("1 + 2");
        assert!(matches!(ast, Ast::BinaryExpr { .. }));
    }
    
    #[test]
    fn tokenizes_operators() {
        let tokens = tokenize("+ - *");  // Can test private function
        assert_eq!(tokens.len(), 3);
    }
}
```

Integration tests live in the `tests/` directory and can only access public APIs. Each file compiles as a separate crate, mirroring external consumer usage:

```
project/
├── src/
│   ├── lib.indent          # Library root
│   └── parser.indent       # Contains inline #[cfg(test)] module
├── tests/
│   ├── common/
│   │   └── mod.indent      # Shared helpers (not a test file)
│   ├── api_tests.indent    # Integration tests
│   └── e2e_tests.indent
└── testdata/               # Test fixtures
    └── fixtures/
```

The `tests/common/` directory is special: it's excluded from test discovery, allowing shared helper functions without cluttering test runs.

Test discovery uses the `#[test]` attribute. Functions marked `#[test]` must take no arguments (except fixture parameters) and return `()` or `Result<(), E>`. This explicit marking is preferable to naming conventions because it's unambiguous and enables IDE tooling.

Tags provide flexible test categorization and filtering:

```indent
#[test]
#[tag("slow")]
#[tag("integration")]
fn test_full_compilation() { }
```

Tags are registered in `indent.toml` with descriptions and filtered via CLI:

```bash
indent test --tags "not slow"
indent test --tags "integration and not flaky"
```

Conditional execution supports both compile-time and runtime skipping:

```indent
#[test]
#[cfg(target_os = "linux")]              // Compile-time exclusion
fn linux_specific_test() { }

#[test]
#[skip_if(cfg(windows), reason = "Requires POSIX signals")]
fn signals_test() { }                     // Runtime skip with message

#[test]
#[xfail(reason = "Known bug #456")]       // Expected failure
fn currently_broken() { }
```

Test naming should use descriptive snake_case that describes behavior rather than implementation: `delivery_with_past_date_is_invalid` rather than `test_validateDeliveryDate_returnsFalse`. The `#[test]` attribute handles discovery, so a `test_` prefix is unnecessary.

## Implementation summary and integration points

The testing patterns described form a cohesive system where each component integrates with the others. Mocking depends on trait-based design enforced through dependency injection. Test isolation uses fixtures that provide scoped resources to tests. Coverage instrumentation feeds into mutation testing for test selection. Snapshot testing uses the same serialization traits as the mocking system.

The compiler must support conditional compilation (`#[cfg(test)]`), procedural macro expansion for `#[mock]` and `#[fixture]`, and instrumentation hooks for coverage. The test runner handles parallel execution, fixture lifecycle, isolation attributes, and result reporting. Post-processing tools manage coverage aggregation, snapshot review, and mutation analysis.

Configuration consolidates in `indent.toml`:

```toml
[test]
parallel = true
threads = "auto"
default_timeout = "30s"

[test.coverage]
types = ["line", "branch"]
thresholds = { overall = 70, new_code = 85 }
formats = ["html", "lcov"]

[test.snapshots]
format = "yaml"
update_behavior = "auto"

[test.tags]
slow = "Tests taking >1s"
integration = "Tests requiring external services"
```

The CLI provides unified access:

```bash
indent test                        # Run all tests
indent test --coverage             # With coverage collection
indent test --tags "not slow"      # Filtered by tags
indent snap review                 # Interactive snapshot review
indent mutants --in-diff main      # Mutation testing on changes
indent coverage --html             # Generate coverage report
```

This design synthesizes proven patterns from mature testing ecosystems while respecting the constraints of a compiled language without runtime reflection. The resulting system provides comprehensive testing capabilities—mocking, isolation, coverage, snapshots, mutation testing, and flexible organization—through a coherent, type-safe API that catches configuration errors at compile time and imposes zero overhead on production builds.