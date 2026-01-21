# Syntax design guide for the Indent programming language

The Indent language faces a defining challenge: creating syntax that feels familiar to Python developers while embracing expression-oriented semantics and enforcing the "one obvious way" philosophy more rigorously than Python itself. This comprehensive guide synthesizes research across **seven major syntax domains**, drawing lessons from Rust, Kotlin, Swift, Elixir, Haskell, and cautionary tales from CoffeeScript and Boo.

**Core design principles** emerging from this research: prefer explicit keywords over symbols when readability matters, use Python conventions where they work, borrow from Rust/Kotlin where Python falls short, and ruthlessly eliminate syntax variations that provide equivalent functionality.

---

## 1. Lambda and anonymous function syntax

The lambda syntax decision has outsized impact on code readability because anonymous functions appear in nearly every data transformation. Python's `lambda x: x * 2` is explicit but limited to single expressions; Rust's `|x| x * 2` is compact but the pipe symbols conflict with Indent's `or` keyword semantics.

### Recommended syntax

```
# Single parameter, single expression
x -> x * 2

# Multiple parameters
(x, y) -> x + y

# With type annotations
(x: Int) -> x * 2
(x: Int, y: Int) -> Int: x + y

# Multi-line with indentation
items.map x ->
    let doubled = x * 2
    doubled + 1
```

**Rationale**: The arrow `->` is mathematical ("maps to"), familiar from Haskell/Kotlin/Swift, and doesn't conflict with Indent's boolean keywords. Parentheses are required only for multiple parameters, keeping simple cases minimal. Multi-line bodies use indentation naturally—no braces needed.

### Trade-offs comparison

| Syntax | Verbosity | Type annotation support | Multi-line support | Conflicts |
|--------|-----------|------------------------|-------------------|-----------|
| `lambda x: expr` (Python) | High | None | No | None |
| `\|x\| expr` (Rust) | Low | Yes | Yes | Conflicts with `or` visually |
| `{ x -> expr }` (Kotlin) | Medium | Yes | Yes | Conflicts with dict literals |
| `x -> expr` (**Recommended**) | Low | Yes | Yes | None |
| `\x -> expr` (Haskell) | Low | External | Yes | Escape character confusion |

### Closure capture semantics

Python's **late binding** causes one of the most common bugs in the language:

```python
# Python bug: all functions return 8
funcs = [lambda: i * 2 for i in range(5)]
[f() for f in funcs]  # [8, 8, 8, 8, 8]
```

**Recommendation for Indent**: Use **early binding by default**—capture the value at lambda creation time. For explicit reference capture, provide optional capture syntax:

```
# Early binding (default) - captures current value of i
for i in range(5):
    funcs.append(-> i * 2)  # Each captures its own i

# Explicit capture list for control
[&x, y] (a) -> a + x + y  # x by reference, y by value
```

### Features to reject

- **Implicit single parameter** (`it` in Kotlin, `$0` in Swift): Violates "explicit is better than implicit" and creates confusion in nested lambdas
- **Underscore placeholders** (`_ * 2` in Scala): Too magical, hard to understand for newcomers
- **Two block syntaxes** (Ruby's `{ }` vs `do...end`): Violates "one obvious way"

---

## 2. Function arguments

Function signatures are the API contract. The syntax must balance flexibility for library authors against simplicity for everyday use.

### Default values

```
def greet(name, greeting = "Hello"):
    f"{greeting}, {name}!"
```

Use `=` with spaces (Python/JavaScript style). Type annotations optional: `def greet(name: String, greeting: String = "Hello")`.

### Named/keyword arguments

```
# Any parameter usable as keyword at call site
greet("Alice", greeting = "Hi")

# Keyword-only (after *)
def http_get(url, *, timeout = 30, verify_ssl = True):
    ...

http_get("/api", timeout = 60)  # Must use keyword for timeout

# Positional-only (before /)
def len(obj, /):  # Can't call as len(obj=x)
    ...
```

**Rationale**: Python's `/` and `*` separators are proven patterns for API design control. Library authors can prevent parameter name changes from being breaking changes.

### Variadic arguments

```
def printf(format, ...args):  # Positional variadics
    ...

def configure(**options):  # Keyword variadics
    ...
```

Use `...args` (JavaScript/TypeScript style) for positional—more visually distinct than Python's `*args`. Keep `**kwargs` for keyword variadics (Python-compatible).

### Destructuring in parameters

```
# Tuple destructuring
def distance((x1, y1), (x2, y2)):
    sqrt((x2 - x1)**2 + (y2 - y1)**2)

# Object/record destructuring
def greet({name, age = 0}):
    f"Hello {name}, age {age}"
```

**Limit to one level of nesting**. Deep destructuring in signatures becomes unreadable—encourage explicit unpacking in the function body instead.

### Trade-offs comparison

| Feature | Python | Kotlin | Swift | Rust | **Recommendation** |
|---------|--------|--------|-------|------|-------------------|
| Default values | `= val` | `= val` | `= val` | None (builders) | `= val` |
| Named at call site | Universal | Universal | Labels required | None | Universal |
| Keyword-only marker | `*` | None | N/A | None | `*` |
| Positional-only marker | `/` | None | `_` | None | `/` |
| Variadics | `*args` | `vararg` | `...` | None | `...args` |

### Argument order convention

**Data-last** for pipeline compatibility:

```
# Standard library functions put data last
def filter(predicate, data): ...
def map(transform, data): ...

# Enables natural piping
data
    |> filter(x -> x > 0)
    |> map(x -> x * 2)
```

---

## 3. Comprehensions versus method chaining

This is the most consequential decision for Indent's "one obvious way" philosophy. Python has both comprehensions and `map`/`filter`, which creates choice paralysis.

### The core tension

**Comprehensions excel at**:
- Simple filter-map operations: `[x * 2 for x in items if x > 0]`
- Multiple input sources: `[(x, y) for x in xs for y in ys]`
- Inline transformations without defining functions

**Method chaining excels at**:
- Long processing pipelines (3+ operations)
- When functions already exist (no lambda needed)
- Debugging (can insert logging between steps)
- Consistent mental model regardless of complexity

### Recommendation: pipelines as primary, limited comprehensions

```
# Simple cases: comprehension is clearer
let evens = [x for x in items if x % 2 == 0]

# Complex pipelines: method chaining
let result = data
    .filter(is_valid)
    .map(transform)
    .group_by(category)
    .take(10)

# Or with pipe operator
let result = data
    |> filter(is_valid)
    |> map(transform)
    |> group_by(category)
    |> take(10)
```

**Constraints for "one obvious way"**:
- Limit comprehensions to **2 generators maximum**
- Beyond 2 generators, use explicit loops or chaining
- Provide a lint warning when comprehension exceeds readable complexity

### Pipeline operator design

Use **implicit first-argument insertion** (Elixir style):

```
# Pipeline passes value as FIRST argument to each function
x |> f |> g |> h  # Equivalent to h(g(f(x)))

# With partial application
data
    |> filter(x -> x > 0)    # filter(x -> x > 0, data)
    |> map(x -> x * 2)       # map(x -> x * 2, result)
```

For cases needing explicit placeholder, support `_`:

```
# When value isn't first argument
data |> some_function(config, _)
```

### Lazy versus eager evaluation

```
# Eager by default (predictable)
let results = items.map(x -> x * 2)  # Computed immediately

# Explicit lazy conversion
let lazy_results = items.lazy().map(x -> x * 2)  # Computed on demand
```

Kotlin's explicit `.asSequence()` conversion is clearer than Python's subtle `[...]` vs `(...)` distinction.

### Trade-offs matrix

| Aspect | Comprehension | Method chaining | Pipeline operator |
|--------|--------------|-----------------|-------------------|
| Simple transforms | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| Complex chains | ★★☆☆☆ | ★★★★★ | ★★★★★ |
| Debugging | ★☆☆☆☆ | ★★★★☆ | ★★★★★ |
| Learning curve | ★★★★☆ | ★★★☆☆ | ★★★☆☆ |
| Nested data sources | ★★★★★ | ★★☆☆☆ | ★★☆☆☆ |

---

## 4. Pattern matching details

Pattern matching is central to expression-oriented programming. Every design choice here affects daily code writing.

### Core syntax

```
match value:
    case Some(n) if n > 0: handle_positive(n)
    case Some(n): handle_other(n)
    case None: handle_none()
```

**Keyword choices**:
- `match` (not `switch`): Signals pattern matching, not C-style value switching
- `case` for branches: Nearly universal, familiar
- `if` for guards: Used by Rust, Python, Scala—most familiar to target audience

### Guard syntax: use `if`

```
match temperature:
    case n if n < 0: "freezing"
    case n if n < 20: "cold"
    case n if n < 30: "comfortable"
    case _: "hot"
```

**Rejected alternatives**:
- `when` (Elixir/OCaml): Less familiar, introduces unnecessary keyword
- `where` (Swift): Conflicts with potential `where` clause for local bindings

### Binding patterns: use `as`

```
match response:
    case [first, *rest] as whole: process(whole, first)
    case Point(x, y) as p if x > 0: transform(p)
```

**Rejected**: `@` (Rust/Haskell)—conflicts with decorator syntax associations, less readable for Python audience.

### Or-patterns with `|`

```
match status_code:
    case 200 | 201 | 204: "success"
    case 400 | 404 | 405: "client error"
    case _: "unknown"

# With bindings (must be identical in all alternatives)
match point:
    case Point(x, 0) | Point(0, x): on_axis(x)
```

### Range patterns

```
match age:
    case ..0: "invalid"
    case 0..13: "child"
    case 13..20: "teenager"
    case 20..65: "adult"
    case 65..: "senior"
```

Use `..` for ranges (consistent with slice syntax). Consider `..=` for explicit inclusive if needed.

### Exhaustiveness checking

**Match expressions must be exhaustive** (return a value, so all cases must be covered):

```
# Expression - compiler error if not exhaustive
let category = match score:
    case 90..100: "A"
    case 80..90: "B"
    case _: "other"  # Required!

# Statement - warning only
match command:
    case "start": start()
    case "stop": stop()
    # Warning: non-exhaustive, consider case _
```

### Pattern matching comparison

| Feature | Rust | Python 3.10 | Scala | **Indent** |
|---------|------|-------------|-------|-----------|
| Guard keyword | `if` | `if` | `if` | `if` |
| Binding syntax | `@` | `as` | `@` | `as` |
| Or-patterns | `\|` | `\|` | `\|` | `\|` |
| Range patterns | `..=` | None | None | `..` |
| Exhaustiveness | Required | None | Warning | Required for expressions |
| Returns value | Yes | No | Yes | Yes |

### Features to reject

- **Multi-clause function definitions** (Elixir/Haskell style): Multiple `def` statements with different patterns violates "one obvious way"—use explicit match in body instead
- **Deep nested patterns** beyond 2-3 levels: Encourage guard conditions or helper functions

---

## 5. Slice syntax

Slice syntax must be consistent with range literals and for-loop syntax—any inconsistency creates cognitive load.

### Recommended syntax

```
# Basic slicing (exclusive end, like Python)
list[start:end]
list[start:end:step]

# Omitted bounds
list[:5]      # First 5 elements
list[5:]      # From index 5 to end
list[::2]     # Every other element
list[::-1]    # Reversed

# Negative indices supported
list[-1]      # Last element
list[-3:]     # Last 3 elements
```

### Exclusive versus inclusive end bounds

**Use exclusive end bounds** (Python style):

- `len(list[:n])` equals `n`—intuitive for "first n elements"
- `list[:i] + list[i:]` reconstructs original with no overlap
- Matches mathematical half-open interval notation `[a, b)`
- Enables empty ranges when `start == end`

### Negative indices: support with safety

Negative indexing is highly ergonomic (`-1` for last element), but computed indices that accidentally go negative cause bugs. **Recommendation**:

- Support negative literals: `list[-1]`, `list[-3:]`
- Runtime error with clear message for out-of-bounds
- Lint warning for computed indices: `list[x - y]` should warn about potential negative result

### Consistency with range/for loops

**Critical**: Slice and range syntax must use identical semantics:

```
# These must be equivalent
for i in 0:5:
    print(list[i])

for item in list[0:5]:
    print(item)
```

### Slice syntax comparison

| Language | Syntax | End bound | Negative indices | Step |
|----------|--------|-----------|------------------|------|
| Python | `[a:b:c]` | Exclusive | Yes | Yes |
| Rust | `[a..b]` / `[a..=b]` | Both options | No | No |
| Go | `[a:b]` | Exclusive | No | No |
| Julia | `[a:b]` | Inclusive | Yes | Via range |
| **Indent** | `[a:b:c]` | Exclusive | Yes | Yes |

---

## 6. Method chaining ergonomics

Beyond the comprehension versus chaining decision, several features affect chaining ergonomics.

### Trailing lambda syntax

When the last argument is a lambda, allow it outside parentheses (Kotlin style):

```
# Standard call
items.map(x -> x * 2)

# Trailing lambda for multi-line
items.map x ->
    let processed = complex_operation(x)
    processed.finalize()

# Multiple parameters with trailing lambda
dialog.show("Title", "Message") result ->
    handle_result(result)
```

**This enables DSL-like APIs**:

```
html:
    div(class = "container"):
        h1: "Welcome"
        p: "Hello world"
```

### Pipeline operator interaction with async

```
# Async pipelines work naturally
await fetch_data()
    |> process
    |> await save_to_db

# Or with await in pipeline
data
    |> await fetch_additional(?)
    |> transform
    |> await persist(?)
```

### Extension methods for chaining

Allow adding methods to existing types via protocols:

```
protocol Stringable for Int:
    def to_binary(self) -> String:
        format(self, "b")

42.to_binary()  # "101010"
```

This enables chaining without monkey-patching core types.

### Debugging long chains

Provide a tap/debug operator:

```
data
    |> filter(is_valid)
    |> tap(x -> print(f"After filter: {len(x)}"))
    |> map(transform)
    |> tap(x -> print(f"After map: {len(x)}"))
    |> take(10)
```

---

## 7. Operators

Operator design affects both readability and correctness. Each choice here has been the subject of language design regrets elsewhere.

### Integer division: use `//`

```
7 // 2   # → 3 (floor division)
7 / 2    # → 3.5 (true division, always float)
-7 // 2  # → -4 (floor toward negative infinity)
```

**Rationale**: Python's approach eliminates the "mixed meaning" problem where `/` behaves differently for integers versus floats. Floor division makes `%` modulo always return non-negative when divisor is positive.

### Exponentiation: use `**` with mandatory parentheses for negatives

```
2 ** 3        # → 8
-2 ** 2       # SYNTAX ERROR: ambiguous
(-2) ** 2     # → 4
-(2 ** 2)     # → -4
```

**Rationale**: JavaScript's approach of requiring parentheses for `-x**y` prevents confusion. Python's `-2**2 = -4` surprises many developers.

### Null/Option handling operators

> **DECISION NOTE:** The final design uses `.else()` and `.or()` methods instead of the `??` operator.
> See `docs/SYNTHESIS.md` "Null Coalescing and Dict Access" for rationale. The `??` operator was
> rejected because method syntax chains naturally and is self-documenting.

```
# Original research suggested:
let name = user.name ?? "Anonymous"

# Final decision - method syntax:
let name = user.name.else("Anonymous")      # Null coalescing
let name = user.name.or("Anonymous")        # Falsy coalescing
let name = user.name.else_do(fetch_default) # Lazy null coalescing

# Optional chaining (unchanged)
let city = user?.address?.city

# Combined
let city = user?.address?.city.else("Unknown")

# Error/None propagation (Rust-style, unchanged)
def get_user_city(id) -> Option[String]:
    let user = find_user(id)?
    let address = user.address?
    Some(address.city)
```

### String concatenation

```
# Use + (explicit types required, no coercion)
"Hello " + name            # Works
"Value: " + 42             # TYPE ERROR
"Value: " + str(42)        # Explicit conversion required

# Prefer interpolation
f"Value: {42}"             # Recommended way
```

**Reject JavaScript's implicit coercion**. Require explicit conversion to prevent `"5" + 3 = "53"` bugs.

### Comparison chaining

```
# Natural mathematical chaining
0 <= x < 100               # Works as expected
a < b <= c < d             # All comparisons evaluated

# Short-circuit evaluation
0 <= expensive_call() < 100  # expensive_call() evaluated only once
```

**This is a major ergonomic win** over languages where `a < b < c` compares a boolean to `c`.

### Operator comparison table

| Operator | Python | JavaScript | Rust | **Indent** |
|----------|--------|------------|------|-----------|
| Integer div | `//` | None | `/` (truncates) | `//` (floor) |
| True div | `/` | `/` | `/` (for floats) | `/` (always float) |
| Exponent | `**` | `**` | `.pow()` | `**` (parens required for negatives) |
| Null coalesce | None | `??` | `.unwrap_or()` | `.else()` / `.else_do()` |
| Falsy coalesce | `or` | `\|\|` | None | `.or()` / `.or_do()` / `or` |
| Optional chain | None | `?.` | None | `?.` |
| String concat | `+` | `+` (coerces) | `format!` | `+` (explicit types) |
| Comparison chain | Yes | No | No | Yes |

### Operator overloading

**Allow via traits, with a limited operator set**:

```
protocol Addable:
    def __add__(self, other: Self) -> Self

struct Vector2:
    x: Float
    y: Float

    implements Addable:
        def __add__(self, other: Vector2) -> Vector2:
            Vector2(self.x + other.x, self.y + other.y)
```

**Reject custom operators** (Scala/Haskell style `+++`): They harm readability and violate "one obvious way."

---

## Lessons from similar languages

### Nim: what worked

Nim successfully combined Python-like whitespace with systems programming:

- **Disallowing tabs** (only spaces) eliminates mixed-indentation bugs—Indent should follow this
- **Uniform Function Call Syntax** (`x.f()` = `f(x)`) enables chaining on any function
- **`and`/`or`/`not` keywords** are readable and natural
- **ARC/ORC memory management** provides deterministic performance without GC pauses

### CoffeeScript: cautionary tale

CoffeeScript's decline offers critical warnings:

- **Ambiguous optional parentheses** caused bugs: `func1 1, func2 2, 3`—which function gets `3`?
- **Whitespace sensitivity in expressions**: `a + b` versus `a +b` had different meanings
- **When the platform catches up, the abstraction loses value**—ES6 absorbed CoffeeScript's best features

### Boo: ecosystem matters

Boo (Python syntax for .NET) failed despite technical merit:

- Small community meant limited documentation and resources
- Competing with the platform's native language (C#) is difficult
- Maintenance burden without critical mass is unsustainable

### Go's minimalism lesson

Go's **enforced single formatting style** via `gofmt` is universally praised:

> "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."

**Indent should ship with an opinionated formatter from day one.**

---

## Interaction matrix: how features connect

| Feature A | Feature B | Interaction |
|-----------|-----------|-------------|
| Lambdas `x -> expr` | Pipelines `\|>` | Pipeline needs data-last functions for clean `\|> map(x -> x*2)` |
| Trailing lambdas | Method chaining | Enables `items.map x -> body` without parentheses |
| Pattern matching | Lambdas | Match in lambda body needs clear indentation rules |
| Comprehensions | Pipelines | Must pick one as primary; both creates choice paralysis |
| Slice `[a:b]` | Range `0:10` | Must use identical semantics for consistency |
| `.else()` method | Optional chaining `?.` | Compose naturally: `x?.y.else(default)` |
| Comparison chaining | `and` keyword | `0 < x < 10 and y > 0` reads naturally |

---

## Complete syntax example

```
# Indent language - comprehensive example

import http
import json

struct User:
    name: String
    email: String  
    age: Int = 0

protocol Serializable:
    def to_json(self) -> String

implements Serializable for User:
    def to_json(self) -> String:
        json.dumps({"name": self.name, "email": self.email, "age": self.age})

def fetch_users(url, /, *, timeout = 30, verify_ssl = True) -> Result[List[User], Error]:
    let response = http.get(url, timeout = timeout, verify_ssl = verify_ssl)?
    let data = json.parse(response.body)?
    
    let users = [User(name = u["name"], email = u["email"]) for u in data["users"]]
    Ok(users)

def process_users(users: List[User]) -> List[String]:
    users
        |> filter(u -> u.age >= 18)
        |> map(u -> u.email)
        |> filter(e -> e.ends_with("@company.com"))

def categorize(user: User) -> String:
    match user.age:
        case ..0: "invalid"
        case 0..13: "child"
        case 13..20 as age: f"teenager ({age})"
        case 20..65: "adult"
        case 65..: "senior"

def main():
    let result = fetch_users("https://api.example.com/users", timeout = 60)
    
    match result:
        case Ok(users):
            let emails = process_users(users)
            for email in emails:
                print(email)
        case Err(e):
            print(f"Error: {e}")
```

---

## Summary of recommendations

| Feature | Recommendation | Rationale |
|---------|---------------|-----------|
| **Lambda** | `x -> expr` | Mathematical, no conflicts, multi-line via indent |
| **Closure capture** | Early binding default | Avoid Python's loop closure bug |
| **Default args** | `param = value` | Python-compatible, familiar |
| **Variadics** | `...args`, `**kwargs` | Best of JS and Python |
| **Primary iteration** | Pipelines + limited comprehensions | "One obvious way" with flexibility |
| **Pipeline** | `\|>` with implicit first-arg | Elixir proven pattern |
| **Pattern match** | `match`/`case` with `if` guards | Rust/Python consensus |
| **Binding** | `as` keyword | More readable than `@` |
| **Exhaustiveness** | Required for expressions | Catches bugs at compile time |
| **Slice** | `[a:b:c]` exclusive end | Python-compatible, consistent with range |
| **Negative indices** | Supported | Ergonomic, with runtime bounds checking |
| **Integer division** | `//` floor, `/` float | Clear semantics, no type ambiguity |
| **Exponentiation** | `**` with paren requirement | Prevents `-2**2` confusion |
| **Null handling** | `.else()`, `.or()`, `?.` | Method-based coalescing + optional chaining |
| **Comparison chain** | Supported | Natural mathematical notation |
| **Formatter** | Built-in, opinionated | Eliminates style debates |

This design provides Python's readability, Rust's safety patterns, and Kotlin's expressiveness while maintaining the "one obvious way" philosophy more consistently than any of its inspirations.