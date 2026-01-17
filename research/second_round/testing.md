# Designing a built-in testing framework: Go's simplicity meets modern ergonomics

A new language targeting backend developers with Go's compilation speed, Rust's memory model, and Python's syntax has an opportunity to create testing that combines **Go's "just works" philosophy with significantly better assertion ergonomics**. After analyzing testing approaches across Go, Rust, Python, Haskell, and Java ecosystems, this report provides concrete design recommendations with proposed syntax.

## Go's testing succeeds through radical simplicity

Go's testing package is praised because **tests are just code**—no special syntax, no framework magic, no inheritance hierarchies. The `testing.T` type provides exactly what's needed: `Error()` for failures, `Fatal()` for early termination, `Run()` for subtests. Table-driven tests became idiomatic because Go's anonymous structs and composite literals make them natural:

```go
tests := []struct{ input string; want int }{
    {"hello", 5},
    {"world", 5},
}
for _, tc := range tests {
    t.Run(tc.input, func(t *testing.T) {
        if got := Len(tc.input); got != tc.want {
            t.Errorf("got %d, want %d", got, tc.want)
        }
    })
}
```

However, **Go's assertion ergonomics are widely criticized**. The idiomatic `if got != want { t.Errorf(...) }` pattern creates boilerplate and makes it easy to swap `got`/`want` in error messages. The testify library exists because developers want `assert.Equal(t, got, want)` with automatic diff output. Go 1.24's `b.Loop()` for benchmarks shows the core team continues addressing pain points pragmatically.

**Key Go insights for the new language:** Name-based discovery (`TestXxx`) works well and requires no special syntax. Subtests with `t.Run()` enable both organization and selective execution. Table-driven patterns need language-level support to reduce boilerplate. Benchmarks need automatic iteration control.

## Rust's compile-time test discovery enables dead code elimination

Rust's `#[cfg(test)]` attribute provides **conditional compilation**—test code is entirely removed from release binaries, saving compile time and binary size. The `#[test]` attribute triggers test harness generation: rustc creates a main function that discovers all test functions, builds an array of function pointers, and links with libtest.

```rust
#[cfg(test)]
mod tests {
    use super::*;  // Can test private functions
    
    #[test]
    fn test_internal() -> Result<(), String> {
        let result = internal_function()?;  // ? works in tests
        assert_eq!(result, expected);
        Ok(())
    }
}
```

**Result-returning tests** are particularly relevant—they enable using `?` for error propagation, which aligns perfectly with the new language's `Result<T, E>` pattern. Rust's `#[should_panic(expected = "message")]` tests panic behavior with substring matching.

Rust deliberately excludes benchmarks from stable, leaving `#[bench]` on nightly only. The community adopted **Criterion** because it provides statistical rigor (confidence intervals, regression detection, HTML reports) that libtest lacks. This suggests benchmarking benefits from ecosystem evolution rather than premature standardization.

**Key Rust insights:** Conditional compilation for tests should be built-in. Result-returning tests are essential for `?`-based languages. Inline tests in the same file enable private function testing. Leave sophisticated benchmarking to ecosystem initially, but provide stable benchmark primitives.

## pytest's assertion rewriting demonstrates the power of AST transformation

pytest's most impressive feature is making plain `assert` statements produce detailed error messages. When you write `assert [1, 2, 3] == [1, 2, 4]`, pytest shows `At index 2 diff: 3 != 4`. This happens through **AST rewriting at import time**:

1. pytest installs a PEP 302 import hook
2. Test modules are parsed to AST before execution
3. `assert` statements are transformed to capture subexpressions in temporary variables
4. On failure, these captured values are formatted into detailed error messages

The transformation turns `assert a + b == 3` into code that stores `a + b`, stores `3`, stores the comparison result, and on failure builds a message showing all intermediate values. **This eliminates the need for an assertion library while providing better errors than most assertion libraries.**

pytest's fixture system uses **dependency injection via function parameters**. Fixtures are discovered automatically by name matching—no imports needed. The `@pytest.fixture` decorator with scope parameters (`function`, `module`, `session`) controls lifecycle. The `yield` pattern handles teardown elegantly.

**Key pytest insights:** Compiler-level assertion rewriting eliminates the assertion library problem entirely. Fixture discovery by parameter name is intuitive. Fixture scopes handle resource lifecycle cleanly. Parametrization with `@pytest.mark.parametrize` reduces table-driven test boilerplate.

## Property-based testing works better as an ecosystem library

QuickCheck (Haskell) and Hypothesis (Python) generate random inputs to test properties rather than specific examples. Properties like "reversing twice equals identity" or "serializing then deserializing returns the original" find edge cases humans wouldn't write.

**Shrinking** is critical: when a property fails on complex input, the framework reduces it to a minimal failing case. Hypothesis's **integrated shrinking** is superior to QuickCheck's type-based approach—it works by shrinking the underlying random byte sequence, automatically respecting all generation constraints. This prevents shrunk values from violating invariants.

Go's `testing/quick` is in stdlib but became outdated and lacks shrinking. The Rust community rejected stdlib inclusion; proptest (Hypothesis-inspired) and quickcheck compete, driving innovation. **The pattern is clear: languages that included early, basic implementations found them insufficient.**

**Recommendation:** Exclude property-based testing from stdlib. Provide hooks for test framework integration (custom generators, shrinking interface). Let the ecosystem mature—integrated shrinking techniques are still evolving.

## Mocking benefits from implicit interface satisfaction

Go's mocking story is good because **implicit interface satisfaction** enables simple hand-written mocks. Any struct implementing the right methods automatically satisfies an interface—no explicit declaration needed:

```go
type MockStore struct{ data map[string]string }
func (m *MockStore) Get(key string) string { return m.data[key] }
// MockStore automatically satisfies any interface with Get(string) string
```

The "accept interfaces, return structs" pattern maximizes flexibility: consumers define minimal interfaces they need, producers return concrete types. This is **superior to Java's Mockito**, which requires runtime proxy generation via reflection because Java interfaces require explicit `implements` declarations.

Rust's mockall crate uses procedural macros for compile-time mock generation. It works but struggles with ownership—mocks must handle `Send` bounds for parallel tests, non-`Clone` return values need `return_once()`, and generic methods require `'static` bounds.

**Test doubles taxonomy** (Fowler/Meszaros) distinguishes: **Dummies** (fill parameters, never used), **Stubs** (canned answers), **Spies** (record calls), **Mocks** (verify expectations), **Fakes** (working implementations with shortcuts). Most "mocks" in practice are stubs or spies.

**Recommendation:** Use implicit interface satisfaction (structural typing). Provide no built-in mocking framework—the interface system handles simple cases; ecosystem libraries can provide sophisticated features. Document test double patterns clearly.

## JUnit XML is the only portable CI format

Every major CI system—GitHub Actions, GitLab CI, Jenkins, CircleCI—supports **JUnit XML** natively. It became the de facto standard despite having no official schema because Jenkins popularized it early. TAP (Test Anything Protocol) is simpler and streamable but requires conversion to JUnit XML for most CI systems.

**Go's `-json` flag** outputs newline-delimited JSON (NDJSON) where each line is a complete event. This streaming format enables real-time progress display while remaining machine-parseable. Events include `run`, `pass`, `fail`, `output` with test name and timing.

For coverage, **Cobertura XML** has the best CI integration (GitLab shows coverage overlays in diffs). **LCOV** is simpler and widely supported. Both work with Codecov and similar services.

**Recommendation:** Default to human-readable streaming output. Support `--json` for NDJSON streaming, `--junit=file.xml` for CI integration. Support Cobertura and LCOV for coverage. Always include timing data.

## Source-based coverage works with any backend

LLVM's coverage instrumentation works at the IR level: the compiler inserts counter increments before optimization, embeds source-to-counter mappings, and the runtime writes profiles on exit. This produces accurate, low-overhead (~3%) coverage.

**Go took a different approach**: the `cover` tool rewrites source code before compilation, inserting `GoCover.Count[N] = 1` statements. This is backend-independent and produces a single "move" instruction per counter.

**Cranelift doesn't support coverage instrumentation natively**. For a language using Cranelift for dev and LLVM for release, the Go-style approach is ideal: implement coverage rewriting at the language IR level, before backend selection. This gives consistent behavior across both backends.

**Coverage metrics research** (Inozemtseva & Holmes, ICSE 2014) found coverage is **weakly correlated** with test effectiveness when controlling for suite size. Coverage is useful as a **negative indicator** (low coverage = likely problems) but not as a quality target. Branch coverage catches decision path gaps that line coverage misses. Mutation testing is a better quality indicator but much slower.

**Recommendation:** Implement coverage at IR level (Go-style) for backend independence. Support both line and branch coverage. Don't encourage coverage targets; surface coverage as diagnostic information.

---

# Concrete design recommendations

## Test declaration syntax

Use **attribute-based discovery** combined with **naming conventions**, taking the best of both Rust and Go:

```
// Basic test - detected by #[test] attribute
#[test]
fn test_addition() {
    assert add(2, 2) == 4
}

// Result-returning test - ? operator works naturally
#[test]
fn test_file_parsing() -> Result<(), ParseError> {
    let config = parse_config("test.toml")?
    assert config.version == "1.0"
    Ok(())
}

// Expected failure test
#[test]
#[should_fail(contains: "division by zero")]
fn test_divide_by_zero() {
    divide(1, 0)
}

// Test with setup/teardown via fixture
#[test]
fn test_database_query(db: Database) {
    let result = db.query("SELECT 1")
    assert result == [1]
}
```

The `#[test]` attribute enables compile-time discovery and conditional compilation. Tests can return `()` or `Result<(), E>`. **Fixtures are injected via parameter names**, discovered from test module scope.

## Assertion design using compiler support

**Implement assertion rewriting at compilation**, similar to pytest but integrated into the language:

```
// Simple assertion - compiler captures subexpressions
assert user.age >= 18

// On failure, output shows:
// Assertion failed: user.age >= 18
//   user.age = 16
//   18 = 18

// Comparison assertions get automatic diff
assert users == expected_users

// On failure for collections:
// Assertion failed: users == expected_users
//   Difference at index 2:
//     got:    User { name: "Alice", age: 30 }
//     expect: User { name: "Alice", age: 31 }
```

**No assertion library needed**—the compiler rewrites `assert` to capture intermediate values and format meaningful errors. This works because:
1. The new language controls compilation end-to-end
2. Python-like syntax makes AST manipulation straightforward
3. Value types by default means capturing values is cheap

For **soft assertions** (continue after failure), provide `check`:

```
#[test]
fn test_user_properties(t: Test) {
    let user = create_user()
    t.check user.name != ""       // Records failure, continues
    t.check user.age > 0          // Records failure, continues
    t.check user.email.contains("@")
    // Test fails if any check failed, showing all failures
}
```

## Fixture mechanism design

Fixtures use **parameter injection with explicit scopes**:

```
// Define a fixture in test module
#[fixture]
fn database() -> Database {
    let db = Database.connect("test://memory")
    defer db.close()  // Cleanup runs after fixture users complete
    db
}

// Module-scoped fixture (created once per module)
#[fixture(scope: module)]
fn server() -> TestServer {
    let server = TestServer.start()
    defer server.stop()
    server
}

// Fixtures can depend on other fixtures
#[fixture]
fn authenticated_client(server: TestServer) -> Client {
    let client = Client.new(server.url)
    client.login("test", "password")
    client
}

// Tests receive fixtures by parameter name
#[test]
fn test_api_call(authenticated_client: Client, db: Database) {
    let response = authenticated_client.get("/users")
    assert response.status == 200
    assert db.query("SELECT COUNT(*) FROM users")[0] > 0
}
```

**Fixture scopes:** `fn` (per-test, default), `module` (per-file), `session` (per-test-run). The `defer` statement handles cleanup naturally, leveraging the language's existing cleanup mechanism.

**Fixture discovery:** Search current module, then `test_fixtures.lang` in same directory, then parent directories. This mirrors pytest's conftest.py but with explicit file naming.

## Table-driven test support

**Parametrization as first-class syntax** to reduce boilerplate:

```
// Parametrized test - generates one test per case
#[test]
#[cases(
    ("empty", "", 0),
    ("single", "a", 1),
    ("unicode", "日本", 2),
)]
fn test_string_length(name: String, input: String, expected: Int) {
    assert input.chars().count() == expected
}

// Output:
// test_string_length/empty ... ok
// test_string_length/single ... ok  
// test_string_length/unicode ... ok
```

For complex cases, use **inline table definition**:

```
#[test]
fn test_http_status_codes() {
    let cases = [
        (200, "OK", true),
        (404, "Not Found", false),
        (500, "Internal Server Error", false),
    ]
    
    for (code, message, is_success) in cases {
        test_case("{code} {message}") {
            let status = HttpStatus(code)
            assert status.message == message
            assert status.is_success() == is_success
        }
    }
}
```

The `test_case` block creates a named subtest, enabling independent failure tracking and selective execution.

## Benchmark syntax

**Explicit benchmark attribute with automatic iteration control**:

```
#[bench]
fn bench_string_concat(b: Bencher) {
    let strings = ["hello", "world", "foo", "bar"]
    
    // b.iter() handles iteration count automatically
    b.iter(|| {
        let result = strings.join(", ")
        // Use result to prevent optimization
        black_box(result)
    })
}

// Benchmark with setup (not measured)
#[bench]
fn bench_sort(b: Bencher) {
    let data = random_vec(10000)  // Setup not timed
    
    b.iter(|| {
        let mut copy = data.clone()  // Clone inside iter if needed
        copy.sort()
        black_box(copy)
    })
}

// Memory allocation tracking
#[bench]
fn bench_allocations(b: Bencher) {
    b.report_allocs()  // Include allocation stats
    b.iter(|| {
        let v: Vec<Int> = (0..1000).collect()
        black_box(v)
    })
}
```

**Keep benchmarks in stdlib but simple.** Provide `iter()` for timing, `report_allocs()` for memory, `black_box()` for preventing optimization. Leave statistical analysis, HTML reports, and regression detection to ecosystem libraries.

## Testing concurrent code

**Integration with structured concurrency**:

```
#[test]
fn test_concurrent_operations() {
    // concurrent {} blocks work naturally in tests
    concurrent {
        spawn check_service_a()
        spawn check_service_b()
        spawn check_service_c()
    }
    // Assertions after concurrent block
    assert all_services_healthy()
}

// Test with timeout
#[test]
#[timeout(5.seconds)]
fn test_slow_operation() {
    let result = slow_computation()
    assert result.is_ok()
}

// Test that operations complete within time budget
#[test]
fn test_response_time(b: Bencher) {
    let elapsed = measure(|| {
        make_request()
    })
    assert elapsed < 100.milliseconds
}
```

**For testing race conditions**, provide deterministic scheduling in test mode:

```
#[test]
#[scheduler(deterministic, seed: 12345)]
fn test_concurrent_map_access() {
    let map = ConcurrentMap.new()
    concurrent {
        spawn map.insert("a", 1)
        spawn map.insert("b", 2)
        spawn map.get("a")
    }
    assert map.len() == 2
}
```

## Mock and stub patterns

**Rely on implicit interface satisfaction** (structural typing):

```
// Production code accepts interface
interface Storage {
    fn get(key: String) -> Result<String, Error>
    fn set(key: String, value: String) -> Result<(), Error>
}

fn cache_lookup(storage: impl Storage, key: String) -> String {
    storage.get(key).unwrap_or("default")
}

// Test provides simple stub - no framework needed
struct StubStorage {
    data: Map<String, String>
}

impl StubStorage {
    fn get(self, key: String) -> Result<String, Error> {
        self.data.get(key).ok_or(Error.not_found())
    }
    fn set(mut self, key: String, value: String) -> Result<(), Error> {
        self.data.insert(key, value)
        Ok(())
    }
}

#[test]
fn test_cache_lookup() {
    let storage = StubStorage { data: [("key", "value")].into() }
    assert cache_lookup(storage, "key") == "value"
    assert cache_lookup(storage, "missing") == "default"
}
```

No built-in mocking framework. The interface system handles simple cases; ecosystem libraries can provide expectation-based mocking if needed.

## CLI interface design

```bash
# Run all tests
lang test

# Run specific test file
lang test tests/auth_test.lang

# Run tests matching pattern
lang test -run "login"

# Run specific test by full path
lang test -run "tests/auth_test.lang::test_login_success"

# Run with verbose output
lang test -v

# Run with very verbose (show passing test output)
lang test -vv

# Output formats
lang test --json              # NDJSON streaming to stdout
lang test --junit=report.xml  # JUnit XML for CI
lang test --tap               # TAP format

# Benchmarks
lang test -bench              # Run all benchmarks
lang test -bench "string"     # Run matching benchmarks
lang test -bench -count 5     # Run each benchmark 5 times

# Coverage
lang test -cover                        # Enable coverage
lang test -cover -coverprofile=cov.out  # Write coverage data
lang test -cover -coverformat=lcov      # LCOV output
lang test -cover -coverformat=cobertura # Cobertura XML

# Parallelization
lang test -parallel 4         # Max 4 tests concurrently
lang test -parallel 1         # Sequential execution

# Filtering
lang test -tags integration   # Run tests with tag
lang test -skip slow          # Skip tests matching pattern
lang test -timeout 30s        # Per-test timeout

# Fail fast
lang test -failfast           # Stop on first failure
```

## Output format defaults

**Default human output (streaming)**:
```
Running tests in auth/

  test_login_success ✓ (12ms)
  test_login_wrong_password ✓ (8ms)
  test_login_rate_limit ✗ (45ms)
    
    Assertion failed: response.status == 429
      response.status = 200
      429 = 429
    
    at auth/login_test.lang:45

  test_logout ✓ (5ms)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FAILED: 1 failed, 3 passed (70ms)
```

**JSON streaming output (`--json`)**:
```json
{"action":"run","test":"auth::test_login_success","time":"2026-01-17T10:30:00Z"}
{"action":"pass","test":"auth::test_login_success","elapsed":0.012}
{"action":"run","test":"auth::test_login_rate_limit"}
{"action":"fail","test":"auth::test_login_rate_limit","elapsed":0.045,"message":"Assertion failed..."}
```

## Integration with Result<T, E> error handling

Tests naturally support `Result` returns and the `?` operator:

```
#[test]
fn test_file_operations() -> Result<(), IoError> {
    let temp = TempDir.create()?
    let file = temp.write_file("test.txt", "content")?
    
    let content = file.read_to_string()?
    assert content == "content"
    
    Ok(())  // Implicit cleanup via TempDir's drop
}

// For testing error conditions
#[test]
fn test_invalid_config() {
    let result = parse_config("invalid.toml")
    
    assert result.is_err()
    assert result.unwrap_err().kind == ErrorKind.InvalidSyntax
}

// Pattern matching on Result in assertions
#[test]  
fn test_specific_error() {
    let result = connect("invalid://url")
    
    assert match result {
        Err(ConnectionError.InvalidProtocol(proto)) => proto == "invalid",
        _ => false,
    }
}
```

## Tests in same file vs separate files

**Support both patterns** with clear conventions:

```
// src/parser.lang - production code with inline tests

pub fn parse(input: String) -> Result<Ast, ParseError> {
    // implementation
}

// Inline tests - compiled only in test mode
#[cfg(test)]
mod tests {
    use super::*
    
    #[test]
    fn test_parse_empty() {
        assert parse("").is_err()
    }
    
    #[test]  
    fn test_parse_simple() -> Result<(), ParseError> {
        let ast = parse("1 + 2")?
        assert ast.root.is_binary_op()
        Ok(())
    }
}
```

**Convention:**
- Inline `#[cfg(test)] mod tests` for unit tests (can test private functions)
- `tests/` directory for integration tests (can only access public API)
- `tests/` files are automatically discovered, no `#[cfg(test)]` needed

```
myproject/
├── src/
│   ├── lib.lang         # May contain inline tests
│   └── parser.lang      # May contain inline tests
└── tests/
    ├── fixtures.lang    # Shared test fixtures
    └── integration.lang # Integration tests
```

## Summary of key design decisions

| Decision | Recommendation | Rationale |
|----------|---------------|-----------|
| **Test discovery** | `#[test]` attribute | Enables conditional compilation, explicit over implicit |
| **Assertions** | Compiler-rewritten `assert` | Eliminates assertion library debate, superior ergonomics |
| **Fixtures** | Parameter injection with scopes | Clean dependency declaration, automatic lifecycle |
| **Table tests** | `#[cases]` attribute + `test_case` blocks | Reduces Go's table-driven boilerplate |
| **Benchmarks** | Built-in primitives, ecosystem sophistication | Avoid Go's testing/quick stagnation problem |
| **Property testing** | Ecosystem only | Techniques still evolving rapidly |
| **Mocking** | Rely on structural typing | Simple cases need no framework |
| **Coverage** | IR-level instrumentation | Backend-independent (Cranelift + LLVM) |
| **Output formats** | Human default, JSON/JUnit/TAP flags | Streaming for dev, buffered for CI |
| **Test location** | Both inline and separate files | Unit tests inline, integration tests in tests/ |

This design achieves **Go's "tests are just code" simplicity** while providing **pytest-level ergonomics** through compiler support. The `assert` rewriting eliminates the assertion library debate. Fixtures provide clean setup/teardown without inheritance. The `#[cases]` attribute makes table-driven tests first-class. Result-returning tests integrate naturally with the language's error handling. Coverage works across both backends. The testing framework feels native to the language rather than bolted on.