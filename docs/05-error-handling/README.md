# Error Handling Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Errors are half of real-world code
**Key Question**: How do we make error handling explicit but not annoying?

## The Problem Space

We want:
- ✅ Errors can't be ignored accidentally (unlike C, Go's `_`)
- ✅ Not verbose (unlike Go's `if err != nil`)
- ✅ Errors are values (not exceptions with hidden control flow)
- ✅ Type-safe (know what errors can happen)
- ✅ Composable (errors propagate cleanly)
- ✅ Good error messages (context, stack traces when needed)

The tension: **Explicitness vs Ergonomics**

## Existing Approaches

### 1. Exceptions (Java, Python, C++)

**How it works**: Errors thrown up the stack until caught.

```python
try:
    result = risky_operation()
except ValueError as e:
    handle_error(e)
except:
    handle_unknown()
```

**Pros**:
- Doesn't clutter happy path
- Can't forget to handle (crashes if uncaught)
- Stack traces built-in

**Cons**:
- Hidden control flow (any line might throw)
- Function signature doesn't show what can be thrown
- Expensive (stack unwinding)
- Easy to catch too broadly
- Hard to reason about

**Example of the problem**:
```python
def transfer(from_acc, to_acc, amount):
    from_acc.withdraw(amount)    # Might throw
    to_acc.deposit(amount)       # Might throw - but withdraw already happened!
```

**Verdict**: No. Hidden control flow is contrary to our "explicit" goal.

---

### 2. Return Codes (C)

**How it works**: Functions return error codes, caller checks.

```c
int result = do_something();
if (result != 0) {
    // handle error
}
```

**Pros**:
- Explicit
- No hidden control flow
- Zero overhead

**Cons**:
- Easy to forget to check
- Can't return both value and error elegantly
- No type safety on error codes
- Error info is limited

**Verdict**: No. Too easy to ignore.

---

### 3. Multiple Returns (Go)

**How it works**: Functions return `(value, error)` tuple.

```go
func ReadFile(name string) ([]byte, error) {
    f, err := os.Open(name)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    data, err := io.ReadAll(f)
    if err != nil {
        return nil, err
    }
    return data, nil
}
```

**Pros**:
- Explicit
- Can't ignore (without explicit `_`)
- Simple to understand
- No hidden control flow

**Cons**:
- **Extremely verbose** (`if err != nil` everywhere)
- Easy to use `_` and ignore
- Errors are untyped (`error` interface is broad)
- Repetitive (same pattern hundreds of times)

**The verbosity problem**:
```go
// This pattern appears 3-10 times per function
a, err := stepA()
if err != nil {
    return nil, err
}
b, err := stepB(a)
if err != nil {
    return nil, err
}
c, err := stepC(b)
if err != nil {
    return nil, err
}
return c, nil
```

**Verdict**: Right idea (explicit, values), wrong syntax.

---

### 4. Result Types (Rust, Swift)

**How it works**: Functions return `Result<T, E>` or `Option<T>`.

```rust
fn read_file(path: &str) -> Result<String, io::Error> {
    let mut file = File::open(path)?;  // ? propagates error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

fn main() {
    match read_file("foo.txt") {
        Ok(contents) => println!("{}", contents),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

**Pros**:
- Type-safe (signature shows exact error types)
- Can't ignore (must unwrap or handle)
- `?` operator reduces verbosity
- Composable with combinators (`.map()`, `.and_then()`)
- No hidden control flow

**Cons**:
- Can still use `.unwrap()` which panics
- Error type conversion can be tedious
- Learning curve for combinators

**The `?` operator is key**:
```rust
// Without ?
let file = match File::open(path) {
    Ok(f) => f,
    Err(e) => return Err(e),
};

// With ?
let file = File::open(path)?;
```

**Verdict**: Strong contender. Type-safe + `?` operator.

---

### 5. Algebraic Effects (Koka, Eff)

**How it works**: Effects are part of the type system, can be handled at different levels.

```koka
fun read-file(path: string): <io,exn> string
  // io effect: file operations
  // exn effect: can throw

fun safe-read(path: string): io string
  with handler
    ctl throw(e) -> "default"  // Handle exn effect
  read-file(path)
```

**Pros**:
- Very expressive
- Composable
- Can implement exceptions, async, etc. as effects

**Cons**:
- Complex to understand
- Novel (less familiar)
- Implementation complexity

**Verdict**: Interesting for research, too complex for our goals.

---

## Our Design Goals

1. **Errors are values**: `Result[T, E]` type
2. **Type-safe**: Know what errors can happen from signature
3. **Propagation is easy**: `?`-like operator
4. **Can't ignore**: Must handle or propagate
5. **Good defaults**: Common patterns are concise
6. **Context**: Easy to add context to errors
7. **Debugging**: Stack traces available when needed

## Proposed Error Handling

### Core Types

```
# Result type (like Rust)
type Result[T, E]:
    Ok(value: T)
    Err(error: E)

# Option type (for nullable-like cases)
type Option[T]:
    Some(value: T)
    None
```

### Function Signatures

```
# Explicit error types in signature
fn read_file(path: str) -> str | IOError:
    ...

# Multiple error types
fn fetch_user(id: int) -> User | NotFoundError | NetworkError:
    ...

# No errors (infallible)
fn add(a: int, b: int) -> int:
    return a + b
```

The `|` syntax is sugar for `Result[T, E]`:
```
fn read_file(path: str) -> str | IOError
# Equivalent to:
fn read_file(path: str) -> Result[str, IOError]
```

### Propagation Operator

The `?` operator propagates errors:

```
fn process_file(path: str) -> Data | IOError:
    content = read_file(path)?    # Propagates IOError
    return parse(content)

# Without ?
fn process_file_verbose(path: str) -> Data | IOError:
    match read_file(path):
        case Ok(content):
            return parse(content)
        case Err(e):
            return Err(e)
```

### Error Handling Patterns

#### Pattern 1: Propagate

```
fn fetch_all(urls: list[str]) -> list[Response] | NetworkError:
    results = []
    for url in urls:
        results.append(fetch(url)?)  # Propagate on first error
    return results
```

#### Pattern 2: Handle with Match

```
fn safe_fetch(url: str) -> Response:
    match fetch(url):
        case Ok(response):
            return response
        case Err(NetworkError.Timeout):
            return Response.empty()
        case Err(e):
            log_error(e)
            return Response.error()
```

#### Pattern 3: Default Value

```
fn get_config(key: str) -> str:
    return read_config(key) else "default"

# More complex default
fn get_port() -> int:
    return parse_int(env("PORT")) else 8080
```

#### Pattern 4: Map/Transform

```
fn get_user_name(id: int) -> str | NotFoundError:
    user = find_user(id)?
    return user.name

# With map
fn get_user_name(id: int) -> str | NotFoundError:
    return find_user(id).map(u -> u.name)
```

#### Pattern 5: Collect Errors

```
fn validate_all(items: list[Item]) -> list[Error]:
    errors = []
    for item in items:
        match validate(item):
            case Err(e):
                errors.append(e)
            case Ok(_):
                pass
    return errors
```

### Adding Context

Errors should carry context:

```
fn load_user_data(user_id: int) -> UserData | Error:
    path = get_data_path(user_id)?
    content = read_file(path).context(f"loading data for user {user_id}")?
    return parse_json(content).context("parsing user data")?

# Error output:
# Error: parsing user data
#   caused by: invalid JSON at line 5
#   while: loading data for user 42
```

### Error Type Definition

```
# Simple error
type NotFoundError:
    message: str

# Error with data
type ValidationError:
    field: str
    message: str
    value: any

# Enum of errors
type DatabaseError:
    ConnectionFailed(host: str, port: int)
    QueryFailed(query: str, reason: str)
    Timeout(duration: duration)

# With trait for common behavior
trait Error:
    fn message(self) -> str
    fn source(self) -> Option[Error]

impl Error for DatabaseError:
    fn message(self) -> str:
        match self:
            case ConnectionFailed(host, port):
                return f"Failed to connect to {host}:{port}"
            case QueryFailed(query, reason):
                return f"Query failed: {reason}"
            case Timeout(d):
                return f"Operation timed out after {d}"

    fn source(self) -> Option[Error]:
        return None
```

### Panic for Bugs

Errors are for expected failure modes. Panics are for bugs:

```
# Errors: expected, handled
fn find_user(id: int) -> User | NotFoundError:
    ...

# Panic: bug, should never happen
fn get_index(list: list[T], i: int) -> T:
    if i >= len(list):
        panic(f"Index {i} out of bounds for list of length {len(list)}")
    return list[i]

# Or use assert
fn process(data: Data):
    assert data.valid, "process() called with invalid data"
    ...
```

**Panics**:
- Unwind the stack (or abort)
- Print stack trace
- Not caught in normal code
- Used for programmer errors, not user errors

### Comparison: Go vs Us

**Go**:
```go
func processFile(path string) (*Data, error) {
    content, err := readFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading file: %w", err)
    }

    data, err := parse(content)
    if err != nil {
        return nil, fmt.Errorf("parsing: %w", err)
    }

    result, err := transform(data)
    if err != nil {
        return nil, fmt.Errorf("transforming: %w", err)
    }

    return result, nil
}
```

**Us**:
```
fn process_file(path: str) -> Data | IOError | ParseError | TransformError:
    content = read_file(path).context("reading file")?
    data = parse(content).context("parsing")?
    result = transform(data).context("transforming")?
    return result
```

Same explicitness, ~70% less code.

## Error Unification

What if a function can return different error types?

### Option 1: Error Enums

```
type ProcessError:
    IO(IOError)
    Parse(ParseError)
    Transform(TransformError)

fn process_file(path: str) -> Data | ProcessError:
    content = read_file(path).map_err(ProcessError.IO)?
    data = parse(content).map_err(ProcessError.Parse)?
    return transform(data).map_err(ProcessError.Transform)
```

### Option 2: Automatic Union

```
# Compiler automatically creates union type
fn process_file(path: str) -> Data | IOError | ParseError | TransformError:
    content = read_file(path)?    # IOError
    data = parse(content)?        # ParseError
    return transform(data)        # TransformError
```

### Option 3: Trait Object (Erased)

```
fn process_file(path: str) -> Data | Error:
    # All errors implement Error trait
    content = read_file(path)?
    data = parse(content)?
    return transform(data)
```

**Recommendation**: Support all three. Explicit enum for libraries, auto-union for application code, trait object when you don't care about types.

## Try Blocks

For grouping operations that might fail:

```
fn complex_operation() -> Result | Error:
    try:
        a = step_a()?
        b = step_b(a)?
        c = step_c(b)?
        return Ok(c)
    catch IOError as e:
        log_io_error(e)
        return Err(e)
    catch ParseError:
        return Err(Error.InvalidInput)
```

This is **not exceptions** - it's sugar for matching on the combined result of a block.

## Summary: Proposed Model

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Errors are values | Yes | Explicit, type-safe |
| Result type | `T \| E` syntax | Clean, familiar |
| Propagation | `?` operator | Concise, explicit |
| Context | `.context()` method | Easy debugging |
| Can't ignore | Compile error if unhandled | Safety |
| Panics | For bugs only | Separate from errors |
| Error traits | `Error` trait | Polymorphism |

## Comparison Matrix

| Feature | Go | Rust | Exceptions | Our Proposal |
|---------|-----|------|------------|--------------|
| Explicit | ✅ | ✅ | ❌ | ✅ |
| Type-safe | ⚠️ | ✅ | ⚠️ | ✅ |
| Concise | ❌ | ✅ | ✅ | ✅ |
| Must handle | ⚠️ | ✅ | ⚠️ | ✅ |
| Context | Manual | Good | Good | Good |
| Composable | ⚠️ | ✅ | ⚠️ | ✅ |

## Open Questions

1. **Stack traces?** When to capture, how expensive?
2. **Error chaining syntax?** How to chain multiple `.context()`?
3. **Try-catch blocks?** Sugar or anti-pattern?
4. **Async errors?** How do errors work with concurrency?
5. **Error recovery?** Can you "resume" from an error?

## Next Steps

1. [ ] Finalize `Result` type semantics
2. [ ] Design error trait hierarchy
3. [ ] Prototype `?` operator in parser
4. [ ] Test with real-world code patterns
5. [ ] Design error formatting/display

---

## Research Links

- [Rust Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
- [Go Error Handling](https://go.dev/blog/error-handling-and-go)
- [Swift Error Handling](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/errorhandling/)
- [Error Handling Survey (research paper)](https://homepages.inf.ed.ac.uk/wadler/papers/handler/handler.pdf)

---

*Last updated: 2024-01-17*
*Decision: Result types with ? operator (tentative)*
