# Type System Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Affects user experience, safety, and performance
**Key Question**: How do we get Python's feel with static type safety?

## The Problem Space

We want:
- ✅ Type safety (catch errors at compile time)
- ✅ Feels like Python (not verbose like Java)
- ✅ Good inference (don't write types everywhere)
- ✅ Generics (can't have a modern language without them)
- ✅ Fast compile times (limits how sophisticated inference can be)
- ✅ Clear error messages (Rust's are good, C++'s are awful)

The tension: **Inference power vs compile time vs error message quality**

## Type System Landscape

### Static vs Dynamic

| | Static | Dynamic |
|---|--------|---------|
| Errors caught | Compile time | Runtime |
| Verbosity | Higher (usually) | Lower |
| Tooling | Better (autocomplete, refactoring) | Harder |
| Performance | Better (no runtime checks) | Runtime overhead |
| Examples | Rust, Go, Java, TypeScript | Python, Ruby, JS |

**We want**: Static with dynamic's feel.

### Nominal vs Structural

**Nominal**: Types are distinct by name
```
type Meters = int
type Feet = int
# Meters and Feet are different types, can't mix
```

**Structural**: Types are compatible by structure
```
type A { x: int, y: int }
type B { x: int, y: int }
# A and B are compatible (same structure)
```

**Go's interfaces are structural**:
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
// Any type with Read method satisfies Reader
```

**Trade-offs**:
- Nominal: Prevents accidental compatibility, clearer intent
- Structural: More flexible, less boilerplate

**Our direction**: Mostly nominal for types, structural for interfaces/traits.

## Type Inference

### Levels of Inference

| Level | What's Inferred | Example |
|-------|----------------|---------|
| None | Nothing | `int x = 5` (Java style) |
| Local | Variables | `x = 5` (x is int) |
| Bidirectional | Variables + some generics | `list = [1, 2, 3]` (list[int]) |
| Hindley-Milner | Nearly everything | ML, Haskell, Rust (partial) |
| Global | Across function boundaries | Can infer function signatures |

**Trade-offs**:
- More inference = less typing
- More inference = slower compilation
- More inference = harder error messages

**Go's approach**: Local inference only. Function signatures always explicit.
```go
func add(a int, b int) int {  // explicit
    return a + b
}
x := add(1, 2)  // x inferred as int
```

**Rust's approach**: Local + bidirectional + some global
```rust
fn add(a: i32, b: i32) -> i32 {  // explicit
    a + b
}
let x = add(1, 2);  // x inferred
let v: Vec<_> = (0..10).collect();  // partial inference
```

**Our sweet spot**: Bidirectional inference
- Local variables always inferred
- Function parameters explicit
- Return types can often be inferred
- Generics inferred from context

```
fn add(a: int, b: int) -> int:
    return a + b

x = add(1, 2)  # x inferred as int

items = [1, 2, 3]  # list[int] inferred

# Return type can be inferred
fn double(x: int):
    return x * 2  # returns int, inferred
```

## Generics

### Monomorphization vs Type Erasure

**Monomorphization (Rust, C++)**:
```rust
fn print<T: Display>(x: T) {
    println!("{}", x);
}
// Compiler generates:
// fn print_i32(x: i32) { ... }
// fn print_string(x: String) { ... }
// etc.
```

**Pros**: Zero runtime overhead, can inline
**Cons**: Code bloat, slow compilation

**Type Erasure (Go, Java)**:
```go
func Print(x any) {
    fmt.Println(x)
}
// Single function, uses interface boxing
```

**Pros**: Fast compilation, small binaries
**Cons**: Runtime overhead (boxing), less optimization

**Hybrid approach** (possible for us):
```
# Monomorphize for primitives (hot paths)
fn sum[T: Numeric](items: list[T]) -> T:
    ...

sum([1, 2, 3])  # Monomorphized for int

# Type-erase for complex types
fn process[T](items: list[T]) -> list[T]:
    ...

process(objects)  # Type-erased
```

### Constraints/Bounds

How do we constrain generic types?

**Go**: No constraints (until Go 1.18, now has type constraints)
```go
func Sum[T int | float64](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}
```

**Rust**: Traits
```rust
fn sum<T: Add<Output = T> + Default>(values: &[T]) -> T {
    values.iter().cloned().fold(T::default(), |a, b| a + b)
}
```

**Our approach**: Simple constraints
```
# Constraint with trait
fn sum[T: Numeric](items: list[T]) -> T:
    total = T.zero()
    for item in items:
        total += item
    return total

# Multiple constraints
fn compare[T: Comparable + Printable](a: T, b: T):
    if a < b:
        print(a)
```

## Traits/Interfaces

### How Polymorphism Works

**Go interfaces** (structural):
```go
type Stringer interface {
    String() string
}
// Any type with String() method works
```

**Rust traits** (nominal, explicit impl):
```rust
trait Display {
    fn display(&self) -> String;
}

impl Display for MyType {
    fn display(&self) -> String { ... }
}
```

**Our approach**: Go-style structural with optional explicit impl

```
# Trait definition
trait Printable:
    fn print(self)

# Implicit: any type with print() satisfies Printable
type User:
    name: str

    fn print(self):
        echo(self.name)

fn log[T: Printable](item: T):
    item.print()

log(User("alice"))  # Works: User has print()

# Explicit impl also allowed (for external types)
impl Printable for ExternalType:
    fn print(self):
        echo(str(self))
```

### Default Methods

Traits can have default implementations:

```
trait Comparable:
    fn compare(self, other: Self) -> Ordering

    # Default implementations
    fn less_than(self, other: Self) -> bool:
        return self.compare(other) == Ordering.Less

    fn greater_than(self, other: Self) -> bool:
        return self.compare(other) == Ordering.Greater
```

## Algebraic Data Types

### Sum Types (Enums with Data)

Critical for error handling and representing variants:

```
type Result[T, E]:
    Ok(value: T)
    Err(error: E)

type Option[T]:
    Some(value: T)
    None

type HttpResponse:
    Success(status: int, body: str)
    Redirect(url: str)
    Error(code: int, message: str)
```

### Pattern Matching

Sum types need pattern matching:

```
fn handle(response: HttpResponse):
    match response:
        case Success(status, body):
            print(f"Got {status}: {body}")
        case Redirect(url):
            fetch(url)
        case Error(code, msg):
            log_error(code, msg)
```

**Exhaustiveness checking**: Compiler ensures all variants handled.

```
fn incomplete(opt: Option[int]):
    match opt:
        case Some(x):
            print(x)
    # COMPILE ERROR: missing case for None
```

## Null Handling

### The Billion Dollar Mistake

Tony Hoare (inventor of null): "I call it my billion-dollar mistake."

**Options**:

| Approach | Example | Safety |
|----------|---------|--------|
| Nullable by default | Java, C#, Go | ❌ NPE |
| Optional types | Rust, Swift, Kotlin | ✅ Safe |
| Sentinel values | Go's zero values | ⚠️ Subtle bugs |

**Our approach**: No null. Use Option type.

```
type User:
    name: str
    email: Option[str]  # Explicitly optional

fn get_email(user: User) -> str:
    match user.email:
        case Some(email):
            return email
        case None:
            return "no email"

# Sugar for common patterns
fn get_email_short(user: User) -> str:
    return user.email else "no email"

# Safe navigation
display_name = user.profile?.display_name else user.name
```

## Primitive Types

### Numeric Types

**Go approach**: Fixed sizes, explicit
```go
int8, int16, int32, int64
uint8, uint16, uint32, uint64
float32, float64
```

**Python approach**: Arbitrary precision
```python
x = 10**100  # Just works
```

**Our approach**: Default to platform int, explicit when needed

```
# Default types (most code)
x: int = 42          # Platform integer (64-bit on modern systems)
y: float = 3.14      # 64-bit float
z: bool = true

# Explicit sizes when needed (FFI, binary protocols)
a: i32 = 42
b: u8 = 255
c: f32 = 3.14

# Literals
decimal = 1_000_000
hex = 0xFF
binary = 0b1010
octal = 0o755
```

### Strings

**One string type**: UTF-8, owned, immutable by default

```
name: str = "hello"
char: rune = 'a'     # Unicode code point

# String operations
length = len(name)   # Byte length
chars = name.chars() # Iterator of runes
upper = name.upper()

# Interpolation
greeting = f"Hello, {name}!"

# Raw strings (no escapes)
regex = r"\d+\.\d+"

# Multiline
doc = """
    This is a
    multiline string
"""
```

### Collections

**Built-in, generic**:

```
# List (dynamic array)
items: list[int] = [1, 2, 3]
items.append(4)
first = items[0]

# Dict (hash map)
ages: dict[str, int] = {"alice": 30, "bob": 25}
ages["charlie"] = 35
age = ages.get("alice") else 0

# Set
unique: set[int] = {1, 2, 3}
unique.add(4)

# Tuple (fixed size, heterogeneous)
point: (int, int) = (10, 20)
x, y = point
```

## User-Defined Types

### Struct-like Types

```
type Point:
    x: int
    y: int

type User:
    name: str
    email: str
    age: int = 0  # Default value

# Construction
p = Point(x=10, y=20)
u = User(name="alice", email="a@b.com")

# With methods
type Circle:
    center: Point
    radius: float

    fn area(self) -> float:
        return 3.14159 * self.radius ** 2

    fn contains(self, point: Point) -> bool:
        dx = point.x - self.center.x
        dy = point.y - self.center.y
        return (dx*dx + dy*dy) <= self.radius**2
```

### Type Aliases

```
type UserId = str
type Timestamp = int
type Handler = fn(Request) -> Response
```

### Newtype Pattern

For type safety without runtime overhead:

```
newtype Meters(int)
newtype Feet(int)

fn calculate(m: Meters):
    ...

m = Meters(100)
f = Feet(328)

calculate(m)  # OK
calculate(f)  # COMPILE ERROR: Feet is not Meters
```

## Type Inference Examples

Show what gets inferred:

```
# Variable types inferred
x = 42                    # int
y = 3.14                  # float
name = "hello"            # str
items = [1, 2, 3]         # list[int]
mapping = {"a": 1}        # dict[str, int]

# Function return inferred
fn double(x: int):
    return x * 2          # returns int

# Generic type inferred
fn first[T](items: list[T]) -> T:
    return items[0]

x = first([1, 2, 3])      # T = int, x: int

# Lambda types inferred
squared = items.map(x -> x * x)  # list[int]
```

## Compilation Speed Considerations

Type system complexity affects compile time:

| Feature | Compile Cost | Value |
|---------|-------------|-------|
| Local inference | Low | High |
| Bidirectional inference | Low-Medium | High |
| Full HM inference | High | Medium |
| Monomorphization | High | High (perf) |
| Type erasure | Low | Medium |
| Trait resolution | Medium | High |

**Our strategy**:
1. Local + bidirectional inference (fast, good UX)
2. Explicit function signatures (fast, good errors)
3. Monomorphize primitives, erase complex types (balance)
4. Simple trait resolution (no associated types initially)

## Error Messages

Good error messages are crucial. Rust is the gold standard.

**Bad (C++ templates)**:
```
error: no matching function for call to 'sort(std::vector<Foo>::iterator, std::vector<Foo>::iterator)'
note: candidate template ignored: substitution failure [with _RandomAccessIterator = __gnu_cxx::__normal_iterator<Foo*, std::vector<Foo>>]: no type named 'type' in 'std::enable_if<false, void>'
```

**Good (Rust)**:
```
error[E0277]: `Foo` doesn't implement `Ord`
 --> src/main.rs:5:5
  |
5 |     v.sort();
  |       ^^^^ the trait `Ord` is not implemented for `Foo`
  |
  = help: the following implementations were found:
            <i32 as Ord>
            <String as Ord>
  = note: required by `slice::<impl [T]>::sort`
```

**Our goal**: Rust-quality error messages
- Point to exact location
- Explain what's wrong
- Suggest fixes
- Show relevant context

## Summary: Proposed Type System

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Static typing | Yes | Safety, tooling |
| Type inference | Bidirectional | Balance of convenience and compile speed |
| Generics | Yes, constrained | Necessary for collections, reusable code |
| Monomorphization | Primitives only | Balance compile time and performance |
| Traits | Structural (Go-like) | Flexibility, less boilerplate |
| Null | No null, use Option | Safety |
| Pattern matching | Yes, exhaustive | Work with sum types |
| Default types | int, float, str, bool | Simple for most code |

## Open Questions

1. **Associated types in traits?** Complex but powerful. Maybe later.
2. **Higher-kinded types?** Probably not. Too complex.
3. **Variance (covariance/contravariance)?** Need to decide for generics.
4. **Reflection?** Limited runtime type info? None?
5. **Type narrowing in conditionals?** TypeScript-style flow typing?

## Next Steps

1. [ ] Define complete primitive type set
2. [ ] Design trait syntax and resolution
3. [ ] Prototype type checker
4. [ ] Create error message catalog
5. [ ] Test inference with real code patterns

---

## Research Links

- [Rust Type System](https://doc.rust-lang.org/book/ch10-00-generics.html)
- [Go Type Parameters](https://go.dev/doc/tutorial/generics)
- [Swift Type System](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/types/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [Hindley-Milner Type Inference](https://en.wikipedia.org/wiki/Hindley%E2%80%93Milner_type_system)

---

*Last updated: 2024-01-17*
*Decision: Bidirectional inference + structural traits (tentative)*
