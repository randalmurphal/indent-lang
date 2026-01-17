# Indent Language Specification

**Version**: 0.1.0 (Draft)
**Date**: 2026-01-17
**Status**: Pre-implementation specification

This document provides the formal specification for the Indent programming language. It serves as the authoritative reference for language implementers.

---

## 1. Lexical Structure

### 1.1 Source Encoding

Source files must be valid UTF-8. No BOM is permitted.

### 1.2 Line Structure

Indent uses significant whitespace. Lines are terminated by:
- Line Feed (U+000A)
- Carriage Return + Line Feed (U+000D U+000A)

### 1.3 Indentation

- Indentation must use spaces only (no tabs)
- Standard indentation is 4 spaces per level
- Inconsistent indentation within a block is a syntax error

### 1.4 Comments

```
# Single-line comment (only style supported)
```

Block comments are not supported to prevent commented-out code.

### 1.5 Keywords

```
and         as          assert      async       await
break       case        comptime    concurrent  const
continue    defer       depends_on  detach      else
ensure      enum        extern      false       for
from        func        if          implement   import
in          interface   internal    is          linear
match       mod         mut         not         or
pub         require     return      scope       self
spawn       state       static      struct      true
type        unsafe      val         var         with
while
```

### 1.6 Reserved Keywords (Future Use)

```
abstract    become      box         do          dyn
final       fn          let         loop        macro
move        override    priv        region      super
trait       typeof      unsized     virtual     where
yield
```

### 1.7 Operators and Punctuation

```
# Arithmetic
+    -    *    /    //   %    **

# Bitwise
&    |    ^    ~    <<   >>

# Comparison
==   !=   <    >    <=   >=

# Logical
and  or   not

# Assignment
=    +=   -=   *=   /=   //=  %=   **=
&=   |=   ^=   <<=  >>=

# Special Arithmetic
+%   -%   *%   /%     # Wrapping
+|   -|   *|   /|     # Saturating

# Other
->   |>   ?.   ??   ?    ...  @

# Delimiters
(    )    [    ]    {    }
:    ,    .    ;    #
```

### 1.8 Literals

#### Integer Literals

```
42              # Decimal
0x2A            # Hexadecimal
0o52            # Octal
0b101010        # Binary
1_000_000       # Underscores for readability
```

#### Floating-Point Literals

```
3.14            # Decimal
3.14e10         # Scientific notation
3.14e-10        # Negative exponent
```

#### String Literals

```
"hello"              # Basic string
"line1\nline2"       # Escape sequences
f"Hello, {name}!"    # Interpolated string
r"raw\nstring"       # Raw string (no escapes)
b"bytes"             # Byte string
"""
multiline
string
"""
```

#### Escape Sequences

| Sequence | Meaning |
|----------|---------|
| `\\` | Backslash |
| `\'` | Single quote |
| `\"` | Double quote |
| `\n` | Newline |
| `\r` | Carriage return |
| `\t` | Tab |
| `\0` | Null |
| `\xNN` | Hex byte |
| `\u{NNNN}` | Unicode scalar |

---

## 2. Types

### 2.1 Primitive Types

| Type | Description | Size |
|------|-------------|------|
| `bool` | Boolean | 1 byte |
| `i8`, `i16`, `i32`, `i64`, `i128` | Signed integers | 1, 2, 4, 8, 16 bytes |
| `u8`, `u16`, `u32`, `u64`, `u128` | Unsigned integers | 1, 2, 4, 8, 16 bytes |
| `isize`, `usize` | Pointer-sized integers | Platform-dependent |
| `f32`, `f64` | IEEE 754 floats | 4, 8 bytes |
| `char` | Unicode scalar value | 4 bytes |
| `()` | Unit type | 0 bytes |

### 2.2 String Types

| Type | Description |
|------|-------------|
| `str` | Borrowed string slice (UTF-8) |
| `String` | Owned, growable string (UTF-8) |

### 2.3 Compound Types

#### Tuples

```
val point: (int, int) = (10, 20)
val (x, y) = point  # Destructuring
```

#### Arrays (Fixed Size)

```
val arr: [int; 5] = [1, 2, 3, 4, 5]
val zeros: [int; 100] = [0; 100]  # Fill with 0
```

#### Slices (Dynamic Size)

```
val slice: [int] = arr[1..4]
```

### 2.4 Collection Types

```
List[T]         # Dynamic array
Dict[K, V]      # Hash map (insertion-ordered)
Set[T]          # Hash set
Deque[T]        # Double-ended queue
Sorted[T]       # B-tree based ordered collection
```

### 2.5 Option and Result

```
enum Option[T]:
    Some(T)
    None

enum Result[T, E]:
    Ok(T)
    Err(E)
```

### 2.6 User-Defined Types

#### Structs

```
struct Point:
    x: f64
    y: f64

struct Person:
    name: String
    age: int = 0  # Default value
```

#### Enums

```
enum Color:
    Red
    Green
    Blue
    Rgb(u8, u8, u8)
    Named { name: String, hex: u32 }
```

#### Newtypes

```
newtype UserId(u64)
newtype Email(String)
```

### 2.7 Type Aliases

```
type Id = u64
type Result[T] = Result[T, Error]  # Partial application
```

### 2.8 Generic Types

```
struct Container[T]:
    value: T

func identity[T](x: T) -> T:
    return x
```

---

## 3. Declarations

### 3.1 Variable Declarations

```
val x = 42                    # Immutable, inferred type
var y = 42                    # Mutable
val z: i64 = 42               # Explicit type, immutable
const MAX: int = 100          # Compile-time constant
static COUNTER: int = 0       # Static variable
```

### 3.2 Function Declarations

```
func add(a: int, b: int) -> int:
    return a + b

func greet(name: str, greeting: str = "Hello") -> str:
    return f"{greeting}, {name}!"

func sum(...numbers: int) -> int:
    return numbers.reduce((a, b) -> a + b, 0)
```

### 3.3 Struct Declarations

```
struct Rectangle:
    width: f64
    height: f64

    func area(self) -> f64:
        return self.width * self.height

    func scale(var self, factor: f64):
        self.width *= factor
        self.height *= factor
```

### 3.4 Interface Declarations

```
interface Printable:
    func to_string(self) -> String

interface Iterator[T]:
    type Item = T
    func next(var self) -> Option[T]
```

### 3.5 Implement Blocks

```
implement Rectangle:
    func new(width: f64, height: f64) -> Rectangle:
        return Rectangle { width, height }

implement Printable for Rectangle:
    func to_string(self) -> String:
        return f"Rectangle({self.width}, {self.height})"
```

---

## 4. Expressions

### 4.1 Arithmetic Expressions

| Operator | Description |
|----------|-------------|
| `a + b` | Addition (checked) |
| `a - b` | Subtraction (checked) |
| `a * b` | Multiplication (checked) |
| `a / b` | Division |
| `a // b` | Integer division |
| `a % b` | Remainder |
| `a ** b` | Exponentiation |
| `-a` | Negation |
| `a +% b` | Wrapping addition |
| `a +\| b` | Saturating addition |

### 4.2 Comparison Expressions

```
a == b    # Equality
a != b    # Inequality
a < b     # Less than
a <= b    # Less than or equal
a > b     # Greater than
a >= b    # Greater than or equal
a < b < c # Chained comparison
```

### 4.3 Logical Expressions

```
a and b   # Logical AND (short-circuit)
a or b    # Logical OR (short-circuit)
not a     # Logical NOT
```

### 4.4 Conditional Expressions

```
val value = if condition: x else: y
```

### 4.5 Match Expressions

```
val result = match value:
    case 0: "zero"
    case n if n < 0: "negative"
    case _: "positive"
```

### 4.6 Lambda Expressions

```
val double = x -> x * 2
val add = (a, b) -> a + b
val complex = x ->
    val y = process(x)
    y.finalize()
```

### 4.7 Pipeline Expressions

```
data |> transform |> filter |> format
# Equivalent to: format(filter(transform(data)))
```

### 4.8 Optional Chaining

```
user?.address?.city ?? "Unknown"
```

### 4.9 Block Expressions

```
val result = {
    val a = compute()
    val b = transform(a)
    b.finalize()  # Last expression is the value
}
```

---

## 5. Statements

### 5.1 Expression Statements

Any expression followed by a newline.

### 5.2 If Statements

```
if condition:
    body
elif other_condition:
    other_body
else:
    fallback_body
```

### 5.3 Match Statements

```
match value:
    case Pattern1:
        body1
    case Pattern2 if guard:
        body2
    case Pattern3 as binding:
        body3
    case _:
        default_body
```

### 5.4 For Loops

```
for item in collection:
    process(item)

for i in 0..10:
    print(i)

for key, value in dict:
    print(f"{key}: {value}")
```

### 5.5 While Loops

```
while condition:
    body

# With else (executes if loop completes without break)
while condition:
    body
else:
    no_break_body
```

### 5.6 Return Statements

```
return value
return  # Returns unit
```

### 5.7 Break and Continue

```
break           # Exit innermost loop
break 'outer    # Exit labeled loop
continue        # Skip to next iteration
continue 'outer # Skip in labeled loop
```

---

## 6. Patterns

### 6.1 Literal Patterns

```
case 42: ...
case "hello": ...
case true: ...
```

### 6.2 Variable Patterns

```
case x: ...           # Binds x
case _: ...           # Wildcard
case x as alias: ...  # Named binding
```

### 6.3 Tuple Patterns

```
case (x, y): ...
case (0, _): ...
case (a, b, ...rest): ...
```

### 6.4 Struct Patterns

```
case Point { x, y }: ...
case Point { x: 0, y }: ...
case Point { x, .. }: ...  # Ignore other fields
```

### 6.5 Enum Patterns

```
case Some(value): ...
case None: ...
case Color.Rgb(r, g, b): ...
```

### 6.6 List Patterns

```
case []: ...
case [first]: ...
case [first, second]: ...
case [first, ...rest]: ...
case [...init, last]: ...
```

### 6.7 Guard Patterns

```
case n if n > 0: ...
case s if s.is_empty(): ...
```

---

## 7. Modules

### 7.1 Module Declaration

```
# In project.mod
module github.com/user/myproject
version 1.0.0

require (
    github.com/lib/http v1.2.0
    github.com/lib/json v2.1.0
)
```

### 7.2 Imports

```
import http
import http.{Server, Client}
import http as web
import "./local_module"
```

### 7.3 Visibility

```
func private_function():        # Default: private to module
    pass

internal func shared_function(): # Visible within package
    pass

pub func public_function():      # Public API
    pass
```

### 7.4 File Structure

```
myproject/
├── project.mod
├── main.idt
├── lib.idt            # Library root
├── handlers/
│   ├── mod.idt        # handlers module
│   └── users.idt
└── internal/          # Never exported
    └── helpers.idt
```

---

## 8. Memory Model

### 8.1 Ownership

Values have a single owner. When the owner goes out of scope, the value is dropped.

### 8.2 Borrowing

```
func read(data: str):       # Immutable borrow (default)
    print(data)

func modify(data: var str): # Mutable borrow
    data.push_str("!")
```

### 8.3 Reference Counting

```
val shared: Ref[Data] = Ref.new(Data { ... })
val clone = shared.clone()  # Increment reference count
```

### 8.4 Scopes

```
scope request:
    val buffer = allocate(1024)
    process(buffer)
# All allocations freed at scope exit
```

### 8.5 Resource Cleanup

```
interface Resource:
    func close(var self) -> Result[(), CloseError]

with file = File.open("data.txt"):
    file.write("content")
# file.close() called automatically
```

---

## 9. Concurrency

### 9.1 Structured Concurrency

```
concurrent:
    val result1 = spawn fetch_data()
    val result2 = spawn process_batch()
# Both tasks complete before continuing
```

### 9.2 Detached Tasks

```
detach:
    background_work()
# Task runs independently (explicit opt-in)
```

### 9.3 Channels

```
val (tx, rx) = channel[Message]()

spawn:
    tx.send(Message { ... })

match rx.recv():
    case Some(msg): process(msg)
    case None: done()
```

### 9.4 Select

```
select:
    case msg = rx1.recv():
        handle_msg(msg)
    case data = rx2.recv():
        handle_data(data)
    case timeout(Duration.seconds(5)):
        handle_timeout()
```

### 9.5 Blocking FFI

```
@blocking
func call_c_library(data: bytes) -> bytes:
    return c_lib.process(data)
```

---

## 10. Error Handling

### 10.1 Result Type

```
func divide(a: int, b: int) -> Result[int, DivError]:
    if b == 0:
        return Err(DivError.DivisionByZero)
    return Ok(a / b)
```

### 10.2 Propagation Operator

```
func process() -> Result[Output, Error]:
    val data = read_file(path)?  # Propagates error
    val parsed = parse(data)?
    return Ok(transform(parsed))
```

### 10.3 Anonymous Sum Types

```
func load() -> Config | IOError | ParseError:
    val content = read_file(path)?
    return parse(content)?
```

### 10.4 Error Context

```
func process() -> Result[Data, Error]:
    val file = open(path).context("opening config file")?
    return parse(file).context("parsing config")?
```

---

## 11. Attributes

### 11.1 Derive Attributes

```
@derive(Debug, Clone, Eq, Hash, Serialize, Deserialize)
struct User:
    id: UserId
    name: String
```

### 11.2 Function Attributes

```
@test
fn test_addition():
    assert 2 + 2 == 4

@trace(level: Info)
fn process_request(req: Request) -> Response:
    ...

@blocking
fn read_file_sync(path: str) -> bytes:
    ...
```

### 11.3 Conditional Compilation

```
@cfg(target_os: "linux")
fn linux_specific():
    ...

@cfg(feature: "debug")
fn debug_only():
    ...
```

---

## 12. Contracts

### 12.1 Preconditions

```
func withdraw(account: Account, amount: Money) -> Result[(), Error]:
    require:
        amount > 0, "amount must be positive"
        amount <= account.balance, "insufficient funds"
    ...
```

### 12.2 Postconditions

```
func deposit(account: Account, amount: Money):
    require:
        amount > 0
    ensure:
        account.balance == old(account.balance) + amount
    ...
```

### 12.3 Invariants

```
struct BankAccount:
    balance: Money

    invariant:
        self.balance >= 0
```

---

## 13. Testing

### 13.1 Test Functions

```
@test
func test_user_creation():
    val user = User.new("alice")
    assert user.name == "alice"
```

### 13.2 Table-Driven Tests

```
@cases([
    ("empty", "", false),
    ("valid", "test@example.com", true),
    ("no_at", "invalid", false),
])
func test_email_validation(name: str, email: str, expected: bool):
    assert validate_email(email) == expected
```

### 13.3 Property-Based Tests

```
@property_test(iterations: 1000)
func test_sort_preserves_length(xs: list[int]):
    assert len(sort(xs)) == len(xs)
```

### 13.4 Fixtures

```
@fixture
func test_db() -> TestDatabase:
    return TestDatabase.new()

@test
func test_with_db(db: TestDatabase):
    val user = db.insert(User.new("test"))
    assert db.find(user.id).is_some()
```

---

## 14. Standard Library Overview

### 14.1 Core Modules

| Module | Description |
|--------|-------------|
| `std.io` | File I/O, streams |
| `std.fs` | Filesystem operations |
| `std.net` | Networking primitives |
| `std.http` | HTTP client and server |
| `std.json` | JSON serialization |
| `std.time` | Time and duration |
| `std.log` | Structured logging |
| `std.env` | Environment variables |
| `std.process` | Process management |
| `std.sync` | Synchronization primitives |

### 14.2 Collection Modules

| Module | Description |
|--------|-------------|
| `std.collections` | List, Dict, Set, Deque |
| `std.iter` | Iterator adapters |

### 14.3 String Modules

| Module | Description |
|--------|-------------|
| `std.str` | String operations |
| `std.regex` | Regular expressions (RE2) |
| `std.unicode` | Unicode utilities |

---

## 15. Grammar Summary (EBNF)

```ebnf
program = { statement } ;

statement = expression_stmt
          | var_decl
          | func_decl
          | struct_decl
          | enum_decl
          | interface_decl
          | implement_block
          | if_stmt
          | match_stmt
          | for_stmt
          | while_stmt
          | return_stmt
          ;

var_decl = ( "val" | "var" ) IDENT [ ":" type ] "=" expression ;

func_decl = [ visibility ] "func" IDENT [ generics ] "(" [ params ] ")" [ "->" type ] ":" block ;

struct_decl = [ visibility ] "struct" IDENT [ generics ] ":" INDENT { field_decl } DEDENT ;

expression = literal
           | IDENT
           | expression "." IDENT
           | expression "(" [ args ] ")"
           | expression binary_op expression
           | unary_op expression
           | "if" expression ":" expression "else" ":" expression
           | "match" expression ":" INDENT { case_arm } DEDENT
           | lambda
           | block
           ;

lambda = params "->" expression
       | params "->" INDENT block DEDENT ;

block = INDENT { statement } DEDENT ;

type = IDENT [ "[" type_args "]" ]
     | "(" [ types ] ")"
     | "[" type "]"
     | "[" type ";" INTEGER "]"
     | type "|" type
     ;
```

---

## Appendix A: Operator Precedence

| Precedence | Operators | Associativity |
|------------|-----------|---------------|
| 1 (highest) | `()` `[]` `.` `?.` | Left |
| 2 | `**` | Right |
| 3 | `not` `-` (unary) `~` | Right |
| 4 | `*` `/` `//` `%` | Left |
| 5 | `+` `-` | Left |
| 6 | `<<` `>>` | Left |
| 7 | `&` | Left |
| 8 | `^` | Left |
| 9 | `\|` | Left |
| 10 | `in` `not in` `is` `is not` `<` `>` `<=` `>=` `==` `!=` | Left |
| 11 | `and` | Left |
| 12 | `or` | Left |
| 13 | `??` | Left |
| 14 | `\|>` | Left |
| 15 (lowest) | `=` `+=` `-=` etc. | Right |

---

## Appendix B: Type Conversion Rules

### Implicit Conversions

None. All numeric conversions must be explicit.

### Explicit Conversions

```
let i: i64 = i32_value.into()      # Widening (always safe)
let s: i32 = i64_value.try_into()? # Narrowing (may fail)
let b: u8 = value as u8            # Truncating cast (unchecked)
```

---

## Appendix C: Memory Layout

### Struct Layout

Structs use C-compatible layout by default:
- Fields ordered as declared
- Alignment based on field types
- Padding added for alignment

### Enum Layout

Tagged union:
- Tag: smallest integer type that fits variants
- Payload: size of largest variant
- Niche optimization when possible

---

*This specification is a living document and will be updated as the language evolves.*
