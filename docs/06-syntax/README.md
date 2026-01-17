# Syntax Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: MEDIUM - Syntax should fit semantics, not drive them
**Key Question**: How do we make code read like pseudocode while being unambiguous?

## The Problem Space

We want:
- ✅ Reads like pseudocode (Python-level readability)
- ✅ Unambiguous (one parse, no context-dependent parsing)
- ✅ Fast to parse (affects compile times)
- ✅ Easy to learn (familiar to Python/Go developers)
- ✅ Not verbose (Go's weak point)
- ✅ Good for tooling (formatter, LSP)

The insight: **Syntax is the UI of a language**. It should be designed last, after semantics.

## Syntax Design Philosophy

### 1. Optimize for Reading

Code is read 10x more than written. Optimize for comprehension.

**Bad**: Perl's symbol soup
```perl
my @sorted = sort { $a->{age} <=> $b->{age} } @people;
```

**Good**: Python's clarity
```python
sorted_people = sorted(people, key=lambda p: p.age)
```

### 2. One Way to Do Things

Multiple syntaxes for the same thing = cognitive overhead.

**Bad**: Ruby's many ways
```ruby
[1,2,3].map { |x| x * 2 }
[1,2,3].map do |x| x * 2 end
[1,2,3].map(&:to_s)
```

**Good**: Python's one way
```python
[x * 2 for x in [1, 2, 3]]
```

### 3. Familiar but Better

Don't be novel for novelty's sake. Use familiar syntax where it doesn't conflict with our goals.

### 4. Significant Structure

Whitespace/indentation as structure (like Python) vs braces (like C):

| Indentation | Braces |
|-------------|--------|
| + Cleaner look | + Explicit boundaries |
| + Forces formatting | + Copy-paste friendlier |
| - Tab/space issues | - More visual noise |
| - Harder to minify | - Needs formatter anyway |

**Recommendation**: **Indentation-based** (like Python). Our users (backend devs, DevOps) are familiar with Python. Formatters solve the consistency issue.

## Lexical Structure

### Keywords

Minimal keyword set. Each keyword should be essential.

**Control Flow**:
```
if, else, match, case
for, while, break, continue
return
```

**Definitions**:
```
fn, type, trait, impl
let, const
```

**Concurrency**:
```
concurrent, spawn, detach
channel, select
```

**Other**:
```
import, from
true, false
and, or, not
in, is, as
```

**Notable exclusions**:
- No `class` (we use `type`)
- No `try/except` (we use Result types)
- No `async/await` (concurrency is transparent)
- No `public/private` (use conventions or explicit visibility)

### Identifiers

```
# Snake case for variables and functions
user_name = "alice"
fn calculate_total():

# Pascal case for types
type UserAccount:
type HttpResponse:

# Screaming snake for constants
MAX_CONNECTIONS = 100
DEFAULT_TIMEOUT = 30

# Single underscore for unused
for _ in range(10):
```

### Literals

```
# Numbers
42              # int
3.14            # float
1_000_000       # underscores for readability
0xFF            # hex
0b1010          # binary
0o755           # octal

# Strings
"hello"         # regular string
'c'             # character (rune)
f"Hello {name}" # interpolated
r"\d+\.\d+"     # raw (no escapes)
"""             # multiline
  Multi
  line
"""

# Collections
[1, 2, 3]       # list
{"a": 1, "b": 2} # dict
{1, 2, 3}       # set
(1, "a", true)  # tuple

# Boolean
true
false
```

### Comments

```
# Single line comment

##
Multi-line comment
Can span multiple lines
##

# Doc comment (for documentation generation)
### This function does X
### Args:
###   x: The input value
### Returns:
###   The processed result
fn process(x: int) -> int:
    ...
```

## Statement Syntax

### Variable Declaration

```
# Type inferred
name = "alice"
count = 42
items = [1, 2, 3]

# Type explicit
name: str = "alice"
count: int = 42
items: list[int] = []

# Constants (compile-time)
const MAX_SIZE = 1024
const API_URL = "https://api.example.com"

# Mutable vs immutable (if we go this route)
x = 5           # immutable by default
mut y = 5       # explicitly mutable
y = 10          # OK
x = 10          # ERROR: x is immutable
```

**Decision needed**: Mutable by default (Python) or immutable by default (functional)?

Arguments for immutable default:
- Safer
- Better for concurrency
- Clear intent when mutating

Arguments for mutable default:
- Familiar to Python/Go devs
- Less annotation for common case

**Recommendation**: **Mutable by default** for familiarity, but encourage immutability in style guide.

### Function Definition

```
# Basic function
fn greet(name: str) -> str:
    return f"Hello, {name}!"

# With default arguments
fn connect(host: str, port: int = 8080, timeout: int = 30):
    ...

# Multiple returns (tuple)
fn divmod(a: int, b: int) -> (int, int):
    return (a // b, a % b)

# With error return
fn read_file(path: str) -> str | IOError:
    ...

# Generic function
fn first[T](items: list[T]) -> T | IndexError:
    if len(items) == 0:
        return Err(IndexError("empty list"))
    return items[0]

# Method (on a type)
type Circle:
    radius: float

    fn area(self) -> float:
        return 3.14159 * self.radius ** 2

# Short lambda
double = x -> x * 2
add = (a, b) -> a + b

# Multi-line lambda (if needed)
process = item -> {
    validated = validate(item)
    transform(validated)
}
```

### Control Flow

```
# If/else
if condition:
    do_something()
else if other_condition:
    do_other()
else:
    do_default()

# If as expression
result = if x > 0: "positive" else: "non-positive"

# Match (pattern matching)
match value:
    case 0:
        print("zero")
    case 1..10:
        print("small")
    case n if n < 0:
        print("negative")
    case _:
        print("other")

# Match on types
match response:
    case Success(data):
        process(data)
    case Error(code, msg):
        log(code, msg)

# For loops (only one kind)
for item in items:
    process(item)

for i, item in enumerate(items):
    print(f"{i}: {item}")

for key, value in mapping.items():
    print(f"{key} = {value}")

# While loop (rarely needed)
while condition:
    do_something()
    condition = check()

# Loop control
for item in items:
    if should_skip(item):
        continue
    if should_stop(item):
        break
    process(item)
```

### Type Definitions

```
# Simple type (struct-like)
type Point:
    x: int
    y: int

# With defaults
type Config:
    host: str = "localhost"
    port: int = 8080
    debug: bool = false

# With methods
type Rectangle:
    width: float
    height: float

    fn area(self) -> float:
        return self.width * self.height

    fn scale(self, factor: float) -> Rectangle:
        return Rectangle(
            width=self.width * factor,
            height=self.height * factor
        )

# Enum/Sum type
type Color:
    Red
    Green
    Blue
    Custom(r: int, g: int, b: int)

# Generic type
type Box[T]:
    value: T

    fn map[U](self, f: fn(T) -> U) -> Box[U]:
        return Box(value=f(self.value))

# Type alias
type UserId = str
type Handler = fn(Request) -> Response

# Newtype (distinct type)
newtype Meters(float)
newtype Feet(float)
```

### Traits/Interfaces

```
# Trait definition
trait Printable:
    fn to_string(self) -> str

# With default implementation
trait Comparable:
    fn compare(self, other: Self) -> Ordering

    fn less_than(self, other: Self) -> bool:
        return self.compare(other) == Ordering.Less

# Implementing trait
impl Printable for Point:
    fn to_string(self) -> str:
        return f"({self.x}, {self.y})"

# Or inline in type definition (if structural)
type Point:
    x: int
    y: int

    fn to_string(self) -> str:  # Satisfies Printable
        return f"({self.x}, {self.y})"
```

### Concurrency

```
# Concurrent block
concurrent:
    a = spawn fetch_a()
    b = spawn fetch_b()
# Both complete before continuing

# With timeout
concurrent timeout(5.seconds):
    result = spawn long_operation()

# Channels
ch = channel[int](buffer=10)

concurrent:
    spawn:
        for i in range(100):
            ch.send(i)
        ch.close()

    spawn:
        for value in ch:
            process(value)

# Select
select:
    case msg from ch1:
        handle_msg(msg)
    case msg from ch2:
        handle_other(msg)
    case timeout(1.second):
        handle_timeout()
```

### Modules/Imports

```
# Import entire module
import http

http.get("url")

# Import specific items
from http import get, post

get("url")
post("url", data)

# Aliased import
import http.client as client

client.get("url")

# Relative imports (within package)
from .utils import helper
from ..common import shared
```

## Operator Precedence

Clear, predictable, minimal surprises:

```
# Highest to lowest
1. () [] . ?             # Grouping, access, optional chain
2. **                     # Exponentiation
3. - not ~               # Unary
4. * / // %              # Multiplicative
5. + -                   # Additive
6. << >> & | ^           # Bitwise
7. < <= > >= == != in is # Comparison
8. and                   # Logical and
9. or                    # Logical or
10. = += -= etc          # Assignment
```

**Key decisions**:
- `and`/`or` instead of `&&`/`||` (readability)
- `**` for exponentiation (like Python)
- `//` for integer division (like Python)
- Comparison chaining: `1 < x < 10` works

## Expression vs Statement

**Expressions** return values. **Statements** don't.

```
# Expression: if-else
result = if x > 0: x else: -x

# Expression: match
name = match status:
    case 200: "OK"
    case 404: "Not Found"
    case _: "Unknown"

# Expression: block (last expression is value)
result = {
    a = compute_a()
    b = compute_b()
    a + b  # This is the value
}

# Statement: for loop (no value)
for item in items:
    process(item)

# Statement: assignment
x = 5
```

## Sample Program

Putting it all together:

```
### A simple HTTP server that fetches and caches user data

import http
from json import parse, stringify
from time import duration

type User:
    id: int
    name: str
    email: str

type Cache:
    data: dict[int, User]
    ttl: duration

    fn get(self, id: int) -> User?:
        return self.data.get(id)

    fn set(self, user: User):
        self.data[user.id] = user

cache = Cache(data={}, ttl=5.minutes)

fn fetch_user(id: int) -> User | HttpError | ParseError:
    # Check cache first
    if cached = cache.get(id):
        return cached

    # Fetch from API
    response = http.get(f"https://api.example.com/users/{id}")?
    user = parse[User](response.body)?

    cache.set(user)
    return user

fn handle_user_request(req: http.Request) -> http.Response:
    id = req.params.get("id") else:
        return http.Response(status=400, body="Missing id")

    match fetch_user(parse_int(id)?):
        case Ok(user):
            return http.Response(
                status=200,
                body=stringify(user),
                headers={"Content-Type": "application/json"}
            )
        case Err(HttpError.NotFound):
            return http.Response(status=404, body="User not found")
        case Err(e):
            log.error(f"Failed to fetch user: {e}")
            return http.Response(status=500, body="Internal error")

fn main():
    server = http.Server(port=8080)

    server.route("GET", "/users/:id", handle_user_request)

    print("Starting server on :8080")
    server.run()
```

## Comparison with Other Languages

### vs Python
```python
# Python
def greet(name: str) -> str:
    return f"Hello, {name}"

# Us (nearly identical)
fn greet(name: str) -> str:
    return f"Hello, {name}"
```

Main differences:
- `fn` instead of `def`
- Return type after `->` is not optional
- Errors in signature

### vs Go
```go
// Go
func greet(name string) string {
    return fmt.Sprintf("Hello, %s", name)
}

// Us
fn greet(name: str) -> str:
    return f"Hello, {name}"
```

Main differences:
- No braces (indentation)
- Type annotations after colon
- String interpolation built-in

### vs Rust
```rust
// Rust
fn greet(name: &str) -> String {
    format!("Hello, {}", name)
}

// Us
fn greet(name: str) -> str:
    return f"Hello, {name}"
```

Main differences:
- No explicit borrowing syntax (usually)
- String interpolation built-in
- No semicolons

## Open Questions

1. **Mutable by default?** Python-like or functional-like?
2. **Semicolons?** Never (Python) or optional (Go)?
3. **Trailing commas?** Required, optional, or forbidden?
4. **Method call syntax?** `obj.method()` only or also `method(obj)`?
5. **Operator overloading?** Allow or forbid?
6. **String interpolation syntax?** `f"..."` or `"...{}"` or `$"..."`?

## Next Steps

1. [ ] Write formal grammar (BNF/PEG)
2. [ ] Implement lexer prototype
3. [ ] Implement parser prototype
4. [ ] Test with sample programs
5. [ ] Gather feedback on readability

---

## Research Links

- [Python Grammar](https://docs.python.org/3/reference/grammar.html)
- [Go Spec](https://go.dev/ref/spec)
- [Rust Reference](https://doc.rust-lang.org/reference/)
- [Language Design Patterns](https://www.hillside.net/plop/plop2001/accepted_submissions/PLoP2001/jnewkirk0/PLoP2001_jnewkirk0_1.pdf)

---

*Last updated: 2024-01-17*
*Decision: Python-like with explicit types (tentative)*
