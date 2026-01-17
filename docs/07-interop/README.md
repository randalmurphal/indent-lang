# Interop Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Must call existing libraries
**Key Question**: How do we seamlessly use C, Python, and Go libraries?

## The Problem Space

We want:
- ✅ Call C libraries (SSL, compression, databases, OS APIs)
- ✅ Embed/call Python (ML libraries, existing codebases)
- ✅ Possibly call Go (existing Go services, libraries)
- ✅ Be called FROM Python/other languages
- ✅ Low overhead (can't afford FFI tax on hot paths)
- ✅ Safe (don't crash because of FFI bugs)

The reality: **No new language succeeds without good C interop**.

## C Interop (Most Critical)

### Why C Interop Matters

Everything is written in C or has C bindings:
- Operating system APIs
- OpenSSL, libcurl, zlib
- SQLite, PostgreSQL client, MySQL client
- Image processing (libjpeg, libpng)
- Almost any native library

### Approaches

#### 1. Same ABI as C (Zig approach)

**How it works**: Compile to the same ABI, call C directly.

```zig
// Zig can directly import C headers
const c = @cImport({
    @cInclude("stdio.h");
});

pub fn main() void {
    c.printf("Hello, C!\n");
}
```

**Pros**:
- Zero overhead
- No FFI layer
- Can use C headers directly (with tooling)

**Cons**:
- Need C compiler in toolchain
- Must match C ABI exactly
- Unsafe by nature

#### 2. FFI Declaration (Rust approach)

**How it works**: Declare C functions/types in your language, link at compile time.

```rust
extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    unsafe {
        let len = strlen(b"hello\0".as_ptr() as *const c_char);
    }
}
```

**Pros**:
- Clear boundary between safe/unsafe
- Can wrap in safe interface
- Explicit about what's foreign

**Cons**:
- Must write declarations manually (or generate)
- `unsafe` required
- Type mismatches possible

#### 3. Generated Bindings (bindgen approach)

**How it works**: Parse C headers, generate bindings automatically.

```bash
# Generate bindings from C header
bindgen sqlite3.h -o sqlite_bindings.rs
```

**Pros**:
- Automated, less manual work
- Less error-prone
- Can track header changes

**Cons**:
- Complex build process
- Generated code can be hard to use directly
- May need manual wrappers anyway

### Our Recommended Approach

**Hybrid**: FFI declaration with optional binding generation

```
# Declare C functions explicitly
extern "C":
    fn strlen(s: *c_char) -> c_size
    fn malloc(size: c_size) -> *void
    fn free(ptr: *void)

# Use them (in unsafe block)
unsafe:
    ptr = malloc(100)
    # ... use ptr
    free(ptr)

# Typical pattern: wrap in safe interface
type Buffer:
    ptr: *void
    size: int

    fn new(size: int) -> Buffer:
        unsafe:
            return Buffer(ptr=malloc(size), size=size)

    fn drop(self):
        unsafe:
            free(self.ptr)
```

### C Type Mapping

```
# C Type       -> Our Type
# char         -> i8
# unsigned char-> u8
# short        -> i16
# int          -> i32 (usually)
# long         -> i64 (on 64-bit)
# long long    -> i64
# float        -> f32
# double       -> f64
# void*        -> *void
# char*        -> *c_char
# size_t       -> c_size
# struct X     -> type X (with same layout)
```

### Structs and Layout

```
# C struct
# struct Point { int x; int y; };

# Our equivalent (explicit C layout)
@c_repr
type Point:
    x: i32
    y: i32

# Can pass directly to C functions
extern "C":
    fn draw_point(p: Point)

draw_point(Point(x=10, y=20))
```

### Callbacks

C libraries often use callbacks:

```
# C: void qsort(void*, size_t, size_t, int(*)(const void*, const void*))

extern "C":
    fn qsort(
        base: *void,
        num: c_size,
        size: c_size,
        compare: fn(*void, *void) -> i32
    )

# Use with our callback
fn compare_ints(a: *void, b: *void) -> i32:
    unsafe:
        ia = cast[*i32](a)^
        ib = cast[*i32](b)^
        return ia - ib

# Sort an array
arr = [3, 1, 4, 1, 5, 9]
unsafe:
    qsort(arr.data_ptr(), len(arr), sizeof(i32), compare_ints)
```

### String Handling

C strings are null-terminated, ours are length-prefixed:

```
fn to_c_string(s: str) -> *c_char:
    # Allocate with null terminator
    unsafe:
        ptr = malloc(len(s) + 1)
        copy(s.data_ptr(), ptr, len(s))
        cast[*u8](ptr)[len(s)] = 0
        return cast[*c_char](ptr)

fn from_c_string(s: *c_char) -> str:
    unsafe:
        length = strlen(s)
        return str.from_ptr(cast[*u8](s), length)
```

Or provide wrappers:

```
# High-level wrapper
import c.strings

c_str = c.strings.to_c("hello")
defer c.strings.free(c_str)
result = c_function(c_str)
our_str = c.strings.from_c(result)
```

---

## Python Interop

### Why Python Interop Matters

- Machine learning (PyTorch, TensorFlow, scikit-learn)
- Scientific computing (NumPy, SciPy, Pandas)
- Existing Python codebases
- Gradual migration path

### Approaches

#### 1. Embed Python Runtime

**How it works**: Link Python interpreter, call Python from our code.

```
import python

python.init()
defer python.finalize()

# Call Python code
result = python.eval("2 + 2")

# Import Python module
np = python.import("numpy")
arr = np.array([1, 2, 3, 4])
mean = arr.mean()  # Returns Python object

# Convert to our types
mean_value: float = python.to_native(mean)
```

**Pros**:
- Full Python compatibility
- Access to all Python packages
- Familiar to Python users

**Cons**:
- Python runtime dependency
- GIL limits parallelism
- Overhead for crossing boundary
- Memory management complexity

#### 2. Call Python via Subprocess

**How it works**: Run Python as subprocess, communicate via stdio/IPC.

```
fn call_python_ml(data: list[float]) -> float:
    result = subprocess.run(
        ["python", "ml_model.py"],
        input=json.dumps(data),
        capture_output=true
    )
    return json.parse(result.stdout)
```

**Pros**:
- No runtime dependency
- Clean separation
- Python can crash without affecting us

**Cons**:
- High overhead (process startup)
- Serialization costs
- Awkward for interactive use

#### 3. Compile to Python Extension

**How it works**: Our language compiles to Python C extension format.

```
# Our code compiles to a .so that Python can import
@python_export
fn fast_compute(data: list[float]) -> float:
    # ... fast implementation
```

Python side:
```python
import our_module
result = our_module.fast_compute([1.0, 2.0, 3.0])
```

**Pros**:
- Python users can use our code
- No Python runtime in our process
- Good for accelerating Python bottlenecks

**Cons**:
- One-way (Python calling us)
- Need to match Python ABI
- Complex to implement

### Our Recommended Approach

**Multiple options for different use cases**:

1. **Embed Python** for calling ML libraries
2. **Python extension** for being called from Python
3. **Subprocess** for simple integration

```
# Calling Python (embedded)
import python

fn predict(data: list[float]) -> float:
    python.with_context:
        torch = python.import("torch")
        model = torch.load("model.pt")
        tensor = torch.tensor(data)
        result = model(tensor)
        return result.item()

# Being called from Python (export)
@python_export
fn fast_process(data: list[float]) -> list[float]:
    return [x * 2 for x in data]
```

---

## Go Interop (Harder)

### Why Go Interop is Hard

1. Go has its own runtime (goroutines, GC)
2. Go calling convention is non-standard
3. `cgo` exists but has overhead

### Options

#### 1. cgo (Go's C FFI)

Our code exposes C interface, Go calls via cgo:

```
# Our code, compiled to C-compatible shared library
@c_export
fn process_data(ptr: *u8, len: i32) -> i32:
    data = from_c_buffer(ptr, len)
    return compute(data)
```

Go side:
```go
// #cgo LDFLAGS: -lourlib
// #include "ourlib.h"
import "C"

func ProcessData(data []byte) int {
    return int(C.process_data((*C.uchar)(&data[0]), C.int(len(data))))
}
```

**Pros**:
- Works
- Bidirectional possible

**Cons**:
- cgo overhead (~100ns per call)
- Complexity
- Thread/runtime coordination issues

#### 2. gRPC/HTTP

Communicate over network/IPC:

```
# Our service
fn main():
    server = grpc.Server()
    server.register(OurService())
    server.run(":50051")

# Go calls us via gRPC
```

**Pros**:
- Clean separation
- No runtime conflicts
- Works across machines

**Cons**:
- Network overhead
- Serialization
- More infrastructure

### Our Recommendation

**gRPC/HTTP for Go interop**. The complexity of cgo + Go runtime coordination isn't worth it for most cases.

---

## Being Called From Other Languages

### C ABI Export

Any language that can call C can call us:

```
@c_export
fn our_function(input: i32) -> i32:
    return input * 2

# Generates:
# int our_function(int input);
```

### Python Extension (as above)

### WebAssembly Export

```
@wasm_export
fn compute(a: i32, b: i32) -> i32:
    return a + b
```

Can be called from JavaScript, other WASM hosts.

---

## Safety Considerations

### Memory Safety Across Boundaries

C doesn't have our memory model. Rules:

1. **Data passed to C must outlive the call** (or be copied)
2. **Data from C must be copied** (or carefully managed)
3. **Callbacks must not capture short-lived references**

```
# WRONG: dangling pointer
fn bad():
    local_str = "hello"
    c_function(local_str.as_c_ptr())  # Might use ptr after return!

# RIGHT: ensure lifetime
fn good():
    local_str = "hello"
    c_str = to_c_string(local_str)  # Copy to C memory
    defer c.free(c_str)
    c_function(c_str)
```

### Type Safety

C types don't have our guarantees:

```
extern "C":
    # This C function might return null, but we don't know
    fn c_get_data() -> *Data

# Safe wrapper
fn get_data() -> Data | NullError:
    unsafe:
        ptr = c_get_data()
        if ptr == null:
            return Err(NullError())
        return Ok(ptr^)  # Dereference
```

### Threading

C libraries may not be thread-safe:

```
# Mark as single-threaded
@not_thread_safe
extern "C":
    fn sqlite3_exec(...)

# Compiler ensures calls are serialized
```

---

## Binding Generation

### Automatic Binding Generator

Parse C headers, generate our bindings:

```bash
# Generate bindings
our-bindgen sqlite3.h -o sqlite_bindings.ol

# Use in code
from sqlite_bindings import sqlite3_open, sqlite3_exec
```

### Binding Files

```
# sqlite.bindings - declarative binding spec
library: "sqlite3"
header: "sqlite3.h"

types:
  sqlite3: opaque
  sqlite3_stmt: opaque

functions:
  sqlite3_open:
    params: [path: *c_char, db: **sqlite3]
    returns: i32

  sqlite3_exec:
    params: [db: *sqlite3, sql: *c_char, callback: *, arg: *, err: **c_char]
    returns: i32
```

---

## Summary

| Target | Mechanism | Overhead | Complexity | Use Case |
|--------|-----------|----------|------------|----------|
| C | FFI | Zero | Low | System libs, perf-critical |
| Python (call) | Embed | Medium | Medium | ML libraries |
| Python (be called) | Extension | Low | Medium | Accelerate Python |
| Go | gRPC | Medium | Low | Microservices |
| JavaScript | WASM | Low | Medium | Web deployment |

## Open Questions

1. **How automatic should binding generation be?** Full auto vs declarative spec?
2. **Runtime or compile-time Python linking?** Embed vs dlopen?
3. **WASM priority?** Important for edge computing use case?
4. **JNI for Java?** Worth supporting?

## Next Steps

1. [ ] Prototype C FFI layer
2. [ ] Test with common libraries (sqlite, OpenSSL)
3. [ ] Design Python embedding API
4. [ ] Create bindgen tool prototype
5. [ ] Document ABI stability guarantees

---

## Research Links

- [Rust FFI](https://doc.rust-lang.org/nomicon/ffi.html)
- [Zig C Interop](https://ziglang.org/documentation/master/#Interoperate-with-C-Code)
- [Python C Extensions](https://docs.python.org/3/extending/extending.html)
- [cgo](https://pkg.go.dev/cmd/cgo)
- [PyO3 (Rust-Python)](https://pyo3.rs/)

---

*Last updated: 2024-01-17*
*Decision: C FFI + Python embed + gRPC for Go (tentative)*
