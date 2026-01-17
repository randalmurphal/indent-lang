# Lightweight Formal Methods for Indent: A Design Guide

**Property-based testing and contracts offer the highest value-to-complexity ratio for backend developers.** Refinement types, while powerful, present significant adoption barriers that make them unsuitable for initial implementation. This guide synthesizes research from 30+ years of Design by Contract evolution, modern fuzzing techniques, and industry adoption patterns to provide concrete recommendations for Indent's formal methods strategy.

Indent's target audience—backend developers and DevOps engineers—values pragmatism over theoretical purity. AWS's experience with TLA+ is instructive: they avoid the words "formal," "verification," and "proof" entirely because of negative perceptions, instead calling their specifications "exhaustively testable pseudo-code." The formal methods features that succeed in practice share common traits: **familiar syntax, gradual adoption paths, immediate debugging value, and excellent error messages**.

## Contracts bring proven benefits with minimal cognitive overhead

Design by Contract has accumulated 30+ years of practical experience since Eiffel's introduction. The syntax patterns that work best share key characteristics: explicit keywords over annotations, named error messages for debugging, and clear handling of "old" values in postconditions.

For Indent's Python-like syntax, contracts should use **keyword-based blocks** that mirror the language's significant whitespace philosophy. The recommended syntax draws from Eiffel's proven semantics while adapting to Python's visual style:

```python
def deposit(account: Account, amount: Money) -> None:
    """Add amount to account balance."""
    require:
        amount >= 0, "amount must be non-negative"
        account.is_open, "account must be open"
    ensure:
        account.balance == old(account.balance) + amount
        account.transaction_count == old(account.transaction_count) + 1
    
    account._balance += amount
    account._transactions.append(Transaction(amount))
```

The `old()` function captures pre-call state—a cleaner approach than Ada's `'Old` attribute or D's lack of native support. Named error conditions (the string after each assertion) enable precise blame assignment when contracts fail.

**Inheritance semantics** must follow the Liskov Substitution Principle: preconditions can only be weakened (using `require or:`), postconditions can only be strengthened (using `ensure and:`), and invariants are always AND-combined with parent invariants. Eiffel's 30 years of experience proves these semantics are essential for contracts to work correctly in object-oriented code.

Performance overhead for contracts is typically **2-5% for precondition-only checking** and **10-20% for full assertion monitoring**. The recommended tiered approach: preconditions always enabled in production (cheap, catches caller bugs), postconditions and invariants disabled by default but configurable via compiler flags or module-level settings.

## Property-based testing succeeds where other formal methods fail

Property-based testing represents the most successful lightweight formal method adoption outside academia. Jane Street uses it extensively, Motorola reported **40-50% productivity gains** with model-based testing approaches, and the technique has been ported to 20+ languages.

The key success factors explain why PBT should be a first-class Indent feature:

- Properties look like regular code—no mathematical notation required
- Shrinking produces minimal counterexamples automatically
- Works alongside existing unit tests with gradual adoption
- Integrates with standard test runners without special tooling

Indent should adopt **Hypothesis's internal shrinking approach** rather than QuickCheck's external shrinking. Internal shrinking operates on the "choice sequence" (the random decisions made during generation) rather than values directly. This works uniformly across all data types, handles nested structures automatically, and eliminates the need for users to write custom shrink functions.

The recommended syntax follows Go's successful integration model while adding typed input domains:

```python
@property_test(iterations=1000)
def test_sort_preserves_length(xs: list[int]):
    assert len(sort(xs)) == len(xs)

@property_test
def test_json_roundtrip(data: Json from json_generator()):
    encoded = data.to_string()
    decoded = Json.parse(encoded)
    assert data == decoded
```

For stateful testing, which catches bugs in APIs with internal state, Indent should support state machine syntax:

```python
state_machine DatabaseTest:
    state:
        model: dict[str, bytes] = {}
        db: Database = Database()
    
    rule save(k: binary(), v: binary()):
        model[k] = v
        db.save(k, v)
    
    invariant consistency:
        for k, v in model.items():
            assert db.get(k) == v
```

The compiler should automatically derive generators from type definitions using the `@Fuzzable` attribute, eliminating the generator-writing friction that Jane Street identified as a key adoption barrier.

## Refinement types are not recommended for initial implementation

Refinement types—predicates attached to types like `type Port = int where 0 < _ < 65536`—offer powerful static guarantees but face **significant practical adoption barriers**. Research on LiquidHaskell users identified nine key barriers spanning developer experience, scalability, and verification understanding.

The core problems are:

1. **Error messages are cryptic**: SMT solver failures produce outputs like "VV : {VV : Bool | VV == True} not a subtype of VV : {VV : Bool | VV <=> Set_mem ?b (listElts ?a)}" that users cannot diagnose
2. **Compilation time grows significantly**: Verification of a 3,500-line library takes 30-60 seconds; incremental checking is essential but complex to implement
3. **Annotation burden is substantial**: Specification lines can reach 5-50% of code depending on property complexity
4. **Learning curve is steep**: Users must understand measures, qualifiers, abstract refinements, termination metrics, and SMT-decidable fragments

**Feasibility assessment**: A minimal viable refinement type system (simple scalar refinements, linear integer arithmetic, basic function contracts, Z3 integration) would require **3-6 months for a prototype**. A usable system with measures, inference, and improved error messages would add **6-12 months**. Given Indent's goals of Python readability and low learning curve, this investment is not justified for initial release.

**Alternative recommendation**: Implement **runtime-checked type predicates** as a lighter-weight alternative:

```python
type Port = int where 0 < _ < 65536  # Checked at type boundaries
type NonEmptyList[T] = list[T] where len(_) > 0

def bind_port(port: Port) -> Socket:  # Checked on function entry
    ...
```

This provides documentation value and catches bugs at runtime without SMT solver complexity. The predicate syntax establishes a migration path—if refinement types are added later, the syntax remains identical but checking becomes static.

## Typestate verification requires affine types but offers high value

The typestate pattern—tracking object state in the type system—provides **zero-cost runtime overhead** in Rust through phantom types and move semantics. The embedded systems community uses typestate extensively for GPIO pins, preventing reading from output pins or writing to input pins at compile time.

**Indent's memory model enables effective typestate**. Since Indent uses regions + ARC for automatic memory management, move semantics can be implemented: when a resource transitions state, the old type is consumed and inaccessible. This is the key requirement for compile-time typestate checking.

Recommended implementation:

```python
state Connection:
    Disconnected
    Connected { socket: Socket }
    Closed

impl Connection[Disconnected]:
    def connect(self, addr: Address) -> Connection[Connected]:
        socket = create_socket(addr)
        return Connection.Connected(socket)

impl Connection[Connected]:
    def send(self, data: bytes) -> None:
        self.socket.write(data)
    
    def close(self) -> Connection[Closed]:
        self.socket.shutdown()
        return Connection.Closed()

# Compile error: send() not available on Disconnected connections
conn = Connection.Disconnected()
conn.send(b"hello")  # Error!
```

For protocols that require runtime branching (state depends on external input), provide a fallback:

```python
def handle_response(conn: Connection[Connected]) -> Connection[Connected | Closed]:
    match conn.read():
        case KeepAlive:
            return conn  # Stay connected
        case Goodbye:
            return conn.close()  # Transition to closed
```

Session types—formal descriptions of communication protocols—are valuable for distributed systems but require careful ergonomic design. The recommended approach: provide session types as an **advanced library feature** rather than core language syntax, targeting teams building RPC frameworks or protocol implementations.

## Fuzzing deserves first-class language support

Go 1.18's built-in fuzzing demonstrates the value of language-integrated fuzzing. OSS-Fuzz has found **13,000+ vulnerabilities** and **50,000+ bugs** across 1,000+ projects, with 77% of bugs coming from recent changes—validating CI-integrated fuzzing.

Indent should follow Go's model with enhancements:

```python
fuzz test_parse_json(f: Fuzzer):
    f.seed("{}")
    f.seed('{"key": "value"}')
    
    f.run(|data: bytes|:
        result = parse_json(data)
        if result.is_ok():
            # Roundtrip property
            output = result.value.encode()
            assert parse_json(output) == result
    )
```

**Required compiler support**:

| Instrumentation | Purpose | Flag |
|----------------|---------|------|
| Edge coverage | Standard fuzzing guidance | `--fuzz-coverage=edge` |
| 8-bit counters | High-performance fuzzing | `--fuzz-coverage=inline` |
| Value profiling | Breaking through comparisons | `--fuzz-coverage=trace-cmp` |

The compiler should emit LLVM SanitizerCoverage-compatible instrumentation, enabling integration with existing fuzzing infrastructure (OSS-Fuzz, ClusterFuzzLite).

**Structure-aware fuzzing** via the `@Fuzzable` attribute enables grammar-aware mutation:

```python
@Fuzzable
struct HttpRequest:
    method: HttpMethod
    path: str
    headers: dict[str, str]
    body: bytes?

fuzz test_http_handler(f: Fuzzer):
    f.run(|request: HttpRequest|:
        handler.process(request)
    )
```

Build system integration should mirror Go's elegance:

```bash
indent test              # Run all tests including fuzz seeds
indent fuzz FuzzParse    # Fuzz specific test continuously
indent fuzz --time=1h    # Time-bounded fuzzing
indent fuzz --coverage   # Generate coverage report
```

## Invariant checking complements contracts for data integrity

Class invariants express properties that must hold throughout an object's lifetime. The checking semantics from Eiffel's 30-year experience:

- Invariants checked **after constructor completes** (not during initialization)
- Checked **before and after public method calls**
- May be **temporarily violated inside method bodies**
- Invariants are **inherited and AND-combined** with parent invariants

```python
struct BinaryTree[T: Ord]:
    root: Node[T]?
    size: int
    
    invariant:
        self.size == count_nodes(self.root)
        is_bst(self.root)  # Binary search tree property
```

**Loop invariants** support both static verification and runtime checking:

```python
def binary_search(arr: list[int], target: int) -> int?:
    low, high = 0, len(arr) - 1
    
    while low <= high:
        invariant 0 <= low <= len(arr)
        invariant -1 <= high < len(arr)
        decreases high - low + 1
        
        mid = (low + high) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return None
```

For initial implementation, **runtime-checked invariants** with tiered configuration provide the best value:

```python
# Module-level configuration
__invariants__ = "debug"  # Options: "none", "debug", "always"

# Or per-struct
@invariant(check="always")  # Critical data structures
struct BankAccount:
    ...
```

Performance overhead for invariant checking is typically **manageable** when checking is limited to public API boundaries. The XRP Ledger runs invariant checks on every transaction for reliability—demonstrating that even high-frequency systems can use runtime invariants when correctness justifies the cost.

## Undefined behavior must be eliminated by design

Indent's design decisions should make undefined behavior impossible in safe code—following Rust's proven model:

| UB Category | C/C++ | Recommended Indent Design |
|-------------|-------|---------------------------|
| Buffer overflow | UB | Bounds-checked indexing, panic on violation |
| Use-after-free | UB | Prevented by regions + ARC |
| Null dereference | UB | No null; use `Option[T]` |
| Data races | UB | Prevented by colorless async design |
| Uninitialized reads | UB | Initialization required by compiler |
| Signed overflow | UB | Debug: panic; Release: wrap (defined) |
| Double free | UB | Prevented by single ownership |

**Integer overflow handling** should follow Rust's RFC 560 model:

- **Debug builds**: Panic on overflow—catches bugs early
- **Release builds**: Two's complement wrap—defined behavior, never UB
- **Explicit APIs**: `wrapping_add()`, `checked_add()`, `saturating_add()` for intentional overflow handling

The `unsafe` escape hatch should be available but clearly marked:

```python
unsafe fn raw_pointer_access(ptr: *mut int) -> int:
    # Unchecked pointer dereference
    return *ptr
```

Sanitizer-compatible compilation should be supported for development:

```bash
indent build --sanitize=address,undefined  # ASan + UBSan
indent build --sanitize=thread             # TSan for concurrency bugs
```

## Adoption success requires excellent developer experience

The 2020 Expert Survey on Formal Methods (130 experts including Turing Award winners) found that **66.9%** believe formal methods are not properly integrated into industrial design lifecycles, and **63.8%** say they have a steep learning curve.

AWS's TLA+ adoption success provides the blueprint:

- Engineers achieved practical productivity in **2-3 weeks**
- Management calls it "exhaustively testable pseudo-code" (not "formal verification")
- But many engineers still struggled with TLA+'s mathematical notation—AWS later adopted P language for broader accessibility

**Error messages are critical**. Rust's philosophy should guide Indent:

1. Point to user's code, not abstract rules
2. Provide actionable fix suggestions
3. Use color and visual formatting for clarity
4. Offer extended explanations via `indent explain E0042`

**Gradual adoption** must be supported:

```python
# File-level strictness
__strict__ = "contracts"  # Enable contract checking for this file

# Or per-function opt-in
@strict
def critical_function(...):
    require:
        ...
```

The **recommended prioritization** for Indent's formal methods features:

| Priority | Feature | Rationale |
|----------|---------|-----------|
| **P0** | Runtime contracts (require/ensure) | Proven value, low learning curve |
| **P0** | Property-based testing | Highest adoption success rate |
| **P0** | Null safety (Option types) | Immediate, tangible value |
| **P1** | Built-in fuzzing | DevOps audience expects this |
| **P1** | Class/struct invariants | Complements contracts naturally |
| **P1** | Typestate for resources | High value for I/O, connections |
| **P2** | Loop invariants | Useful for algorithms |
| **P2** | Runtime type predicates | Bridge to refinement types |
| **P3** | Session types (library) | Advanced distributed systems |
| **Future** | Refinement types | Only after tooling matures |

## Implementation roadmap for formal methods in Indent

**Phase 1 (Core language release)**: Runtime contracts with `require`/`ensure` blocks, property-based testing framework with automatic shrinking, null safety via `Option[T]`, bounds-checked indexing, and defined integer overflow behavior.

**Phase 2 (Tooling maturity)**: Built-in fuzzing with compiler instrumentation, class invariants, typestate for resource management, and enhanced error messages with explanations.

**Phase 3 (Advanced features)**: Loop invariants with optional static verification, runtime type predicates, session types as library feature, and integration with external provers for critical code paths.

This phased approach ensures Indent delivers immediate value to backend developers while establishing a foundation for more advanced formal methods as the ecosystem matures. The key insight from 30 years of formal methods research: **usability trumps power**. A simple contract that developers actually write beats a sophisticated proof system that sits unused.