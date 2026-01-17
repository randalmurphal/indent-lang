# FFI Implementation for Modern Systems Languages

A new systems programming language can achieve **zero-overhead C calls** and **safe Python embedding** by combining Zig's compile-time header translation with Rust's ownership-aware type system and Swift's trampoline patterns. The optimal approach uses **direct ABI-compatible types** for the hot path while generating **safe wrappers** through compile-time code generation.

## The core design tradeoff

Every FFI system must balance three competing concerns: **runtime overhead**, **developer ergonomics**, and **memory safety**. Zig proves that zero-overhead is achievable through ABI-compatible types and compile-time translation. Rust demonstrates that safety wrappers can be generated automatically without sacrificing performance for simple cases. The winning strategy for a new language: **make the unsafe path ergonomic for experts, then layer safety on top**.

The recommended architecture uses a two-tier system. Tier 1 provides zero-overhead `extern` declarations with ABI-compatible types—no wrappers, no checks, direct function calls indistinguishable from C. Tier 2 generates safe wrappers automatically where ownership and error semantics can be inferred, with explicit annotations for ambiguous cases.

## C ABI mechanics every FFI must implement

The System V AMD64 ABI passes the first six integer arguments in **RDI, RSI, RDX, RCX, R8, R9** and the first eight floating-point arguments in **XMM0-XMM7**. Return values use RAX/RDX for integers and XMM0/XMM1 for floats. ARM64 AAPCS uses X0-X7 for integers and V0-V7 for floats, with **X8** reserved for indirect return value pointers.

Structs ≤16 bytes pass in registers on both architectures; larger structs pass by hidden pointer. The classification algorithm analyzes each 8-byte chunk: if any field is INTEGER class, the whole chunk is INTEGER; otherwise SSE. A `struct { int a; float b; }` passes in a single general-purpose register because INTEGER wins the merge.

**Struct layout replication is critical**. Each member aligns to its natural alignment, with padding inserted as needed:

```
struct Example {
    char a;      // offset 0
    // 3 bytes padding
    int b;       // offset 4
    char c;      // offset 8  
    // 7 bytes padding
    long d;      // offset 16
};              // Total: 24 bytes
```

Varargs require a register save area on AMD64 (304 bytes for all potential argument registers). Apple Silicon simplifies this—variadic arguments always go on the stack, eliminating the save area complexity. A new language should detect the target platform and generate the appropriate calling sequence.

## Zig's header translation as a model

Zig's `@cImport` achieves zero-overhead C interop through compile-time header translation. The compiler parses C headers using **Aro** (a C compiler written in Zig, replacing the earlier libclang dependency), converts C AST to Zig AST, and imports the result. The generated code uses ABI-compatible types like `c_int`, `c_long`, and the special **`[*c]T` pointer type** that can be null and coerces to both single-item and many-item pointers.

```zig
const c = @cImport({
    @cInclude("SDL2/SDL.h");
});

// Direct zero-overhead call - identical to C
_ = c.SDL_Init(c.SDL_INIT_VIDEO);
```

Function-like macros translate to inline generic functions:
```zig
// C: #define ADD(a, b) ((a) + (b))
// Becomes:
pub inline fn ADD(a: anytype, b: anytype) @TypeOf(a + b) {
    return a + b;
}
```

**Known limitations** that a new language should address: token pasting (`##`) cannot translate, `do { } while(0)` idioms fail, macros calling extern functions become problematic comptime calls, and GCC inline assembly doesn't translate. Zig plans to move `@cImport` entirely to the build system, which is the right architectural choice—separating binding generation from core language semantics.

## Rust bindgen's lessons on maintenance burden

Bindgen uses **libclang** to parse C headers, traverse the AST, and generate Rust code with `#[repr(C)]` structs and `extern "C"` function declarations. The generated code includes automatic layout tests that verify struct sizes and alignments match at test time.

**What works well**: structs, unions, enums, function pointers, typedefs. **What fails**: function-like macros (completely unsupported), inline functions (no linkable symbol), complex preprocessor conditionals, and **bitfields** (generates getter/setter functions but with known bugs around unions and packed structs).

The maintenance burden splits into two schools. **Regenerate at build time** ensures bindings always match the target architecture and library version, but requires libclang on every build machine. **Commit bindings** eliminates build dependencies but risks architecture mismatches. The hybrid approach—commit bindings with an optional regeneration feature—works best for libraries.

Critical lesson for a new language: **bindings are architecture-dependent**. `c_char` is `i8` on x86 but `u8` on ARM. Pre-generated bindings silently break on different architectures. Either regenerate per-target or use conditional compilation.

For C++, bindgen is essentially inadequate. The **cxx** crate provides bidirectional, safe interop but requires manual bridge declarations. **autocxx** combines cxx's safety with automatic generation. A new language should explicitly choose: either support C++ properly (expensive) or don't pretend to (Zig's honest approach).

## Memory ownership patterns across FFI boundaries

The golden rule: **whoever allocates memory must provide a way to free it**. Memory allocated by Rust must be freed by Rust; memory allocated by C must be freed by C. Mixing allocators causes silent corruption.

**Rust's ownership transfer pattern**:
```rust
// Transfer ownership TO C
let obj = Box::new(MyStruct::new());
let ptr: *mut MyStruct = Box::into_raw(obj);
// ptr now owns the memory; Box destructor will NOT run

// Transfer ownership FROM C (must be same pointer)
let obj = unsafe { Box::from_raw(ptr) };
// obj now owns memory; freed when dropped
```

**Swift's Unmanaged<T>** bridges ARC with manual memory management:
```swift
let retained = Unmanaged.passRetained(object)  // +1 ref count
let pointer = retained.toOpaque()               // Get raw pointer
// Later:
let object = Unmanaged<MyClass>.fromOpaque(pointer).takeRetainedValue()  // -1
```

The **opaque handle pattern** is the safest FFI design. Hide internal structure, expose only handles:
```rust
pub struct ZipCodeDatabase { /* fields hidden */ }

#[no_mangle]
pub extern "C" fn zip_code_database_new() -> *mut ZipCodeDatabase {
    Box::into_raw(Box::new(ZipCodeDatabase::new()))
}

#[no_mangle]
pub extern "C" fn zip_code_database_free(ptr: *mut ZipCodeDatabase) {
    if !ptr.is_null() { unsafe { Box::from_raw(ptr); } }
}
```

**ManuallyDrop is preferred over mem::forget** for ownership transfer because it's more panic-safe—disable the destructor before extracting raw parts, so even if a panic occurs, no double-free happens.

RAII wrappers should encapsulate all C resources:
```rust
pub struct OpenSSL { ctx: *mut ffi::SSL_CTX }

impl Drop for OpenSSL {
    fn drop(&mut self) {
        unsafe { ffi::SSL_CTX_free(self.ctx) }
    }
}
```

For arena-based FFI patterns, **bumpalo** (heterogeneous bump allocator) or **typed_arena** (single-type, runs destructors) work well. When many allocations share a lifetime (per-request, per-frame), batch deallocation eliminates individual free overhead.

## Python embedding: choosing the right approach

| Approach | Performance | Safety | Ergonomics | Maintenance |
|----------|-------------|--------|------------|-------------|
| CPython C API | Baseline | Manual refcounting | Verbose | Stable ABI available |
| PyO3 (Rust) | Near C | Compile-time GIL tracking | Excellent | Maturin tooling |
| pybind11 (C++) | Good | Template-based | Good | Header-only |
| cffi | API mode fast | Moderate | Pythonic | Pure Python setup |
| nanobind | **10× faster than pybind11** | Excellent | Clean | C++17 required |

**For a new systems language embedding Python, PyO3's approach is the model to follow**: compile-time GIL tracking via a `Python<'py>` token, automatic type conversions, and explicit ownership semantics.

**Critical initialization sequence**:
```c
PyConfig config;
PyConfig_InitPythonConfig(&config);
PyConfig_SetString(&config, &config.home, L"/path/to/venv");
Py_InitializeFromConfig(&config);
```

For **PyTorch/NumPy interop**, use the **DLPack protocol** for zero-copy tensor exchange:
```python
import torch
t = torch.arange(4)
t2 = torch.from_dlpack(t)  # Zero-copy, memory shared
```

The **buffer protocol** enables direct pointer access to NumPy arrays without copying. NumPy's `__array_interface__` provides the raw pointer, shape, strides, and dtype. For GPU tensors, `__cuda_array_interface__` enables zero-copy sharing between PyTorch, CuPy, and other CUDA frameworks.

**Stable ABI considerations**: Define `Py_LIMITED_API` to target a minimum Python version (e.g., `0x03080000` for 3.8). Extensions use `.abi3.so` naming and work across Python 3.x versions. Trade-off: **18-33% performance loss** in some benchmarks, and not all APIs are available.

## GIL management in multi-threaded contexts

The GIL protects CPython's reference counting—without it, concurrent threads modifying reference counts cause memory corruption. Only one thread executes Python bytecode at a time.

**Standard pattern for calling Python from C threads**:
```c
PyGILState_STATE gstate = PyGILState_Ensure();

/* Python operations here */
PyObject* result = PyObject_CallObject(callback, NULL);
Py_XDECREF(result);

PyGILState_Release(gstate);
```

**Releasing GIL for parallelism** (when not using Python objects):
```c
Py_BEGIN_ALLOW_THREADS
/* Long-running C computation - other Python threads can run */
expensive_native_computation();
Py_END_ALLOW_THREADS
```

NumPy releases the GIL for most numeric operations, enabling true parallelism for array math. Operations on `dtype=object` arrays don't release the GIL.

**Subinterpreters (PEP 684, Python 3.12+)** provide per-interpreter GIL:
```c
PyInterpreterConfig config = {
    .check_multi_interp_extensions = 1,
    .gil = PyInterpreterConfig_OWN_GIL,
};
PyThreadState *tstate = NULL;
Py_NewInterpreterFromConfig(&tstate, &config);
```

Limitation: No direct object sharing between interpreters. Extensions must opt-in via `Py_mod_multiple_interpreters` slot.

**Free-threaded Python (PEP 703)** is now officially supported in Python 3.14. Uses biased reference counting, mimalloc allocator, and stop-the-world garbage collection. Single-threaded overhead reduced to **~5-10%** (improved from 40% in 3.13). A new language should design FFI to be thread-safe from the start, as the GIL is disappearing.

## Callback safety and trampoline techniques

Closures can't directly become C function pointers because closures consist of **two pointers** (code + captured data) while C function pointers are single pointers. Only non-capturing closures convert directly.

**The standard solution: static trampoline with user_data**:
```rust
unsafe fn unpack_closure<F>(closure: &mut F) -> (*mut c_void, extern "C" fn(*mut c_void, c_int))
where F: FnMut(c_int)
{
    extern "C" fn trampoline<F>(data: *mut c_void, n: c_int)
    where F: FnMut(c_int)
    {
        let closure: &mut F = unsafe { &mut *(data as *mut F) };
        (*closure)(n);
    }
    (closure as *mut F as *mut c_void, trampoline::<F>)
}
```

Rust monomorphizes `trampoline<F>`, creating a unique function for each closure type. The closure passes as `void*` user_data, cast back inside the trampoline.

**Swift uses Unmanaged for context management**:
```swift
let context = Unmanaged.passRetained(self).toOpaque()
c_register_callback(myCallback, context)

// In callback:
let innerSelf = Unmanaged<MyClass>.fromOpaque(info!)
    .takeRetainedValue()  // Balances passRetained
```

**libffi's closure API** creates executable trampolines at runtime—useful when C APIs don't provide user_data parameters, but has security implications (requires writable+executable memory).

**Thread-local storage** is the fallback when C APIs lack user_data:
```rust
thread_local! {
    static CALLBACK_CONTEXT: RefCell<Option<Box<dyn FnMut(i32)>>> = RefCell::new(None);
}
```

**Lifetime safety**: Callbacks stored by C must use `'static` lifetime and `Box::into_raw()` to prevent Rust from dropping the data. RAII guards should unregister callbacks before the owning object is destroyed:
```rust
impl Drop for CallbackHandle {
    fn drop(&mut self) {
        unsafe { c_unregister_callback(self.id); }
    }
}
```

## Error handling across language boundaries

C returns error codes; modern languages use Result/Option types. The bridge requires explicit conversion.

**Rust pattern for wrapping C errors**:
```rust
pub fn safe_open(path: &str) -> Result<File> {
    let c_path = CString::new(path)?;
    let fd = unsafe { libc::open(c_path.as_ptr(), libc::O_RDONLY) };
    if fd < 0 {
        Err(std::io::Error::last_os_error())  // Reads errno
    } else {
        Ok(unsafe { File::from_raw_fd(fd) })
    }
}
```

**Thread-local error storage** for FFI-exported functions:
```rust
thread_local! {
    static LAST_ERROR: RefCell<Option<Box<dyn Error>>> = RefCell::new(None);
}

#[no_mangle]
pub extern "C" fn last_error_length() -> i32 {
    LAST_ERROR.with(|e| {
        e.borrow().as_ref().map(|e| e.to_string().len() as i32 + 1).unwrap_or(0)
    })
}
```

**Zig's error unions** provide the cleanest FFI error model:
```zig
fn wrapCCall(result: c_int) !void {
    if (result == -1) {
        const errno = std.c.getErrno();
        return switch (errno) {
            .EINVAL => error.InvalidArgument,
            .ENOMEM => error.OutOfMemory,
            else => error.Unexpected,
        };
    }
}
```

**Panic safety at FFI boundaries**: Use `catch_unwind` to convert panics to error codes:
```rust
#[no_mangle]
pub extern "C" fn safe_callback(data: *mut c_void) -> i32 {
    match catch_unwind(AssertUnwindSafe(|| do_work(data))) {
        Ok(value) => value,
        Err(_) => { update_last_error("panic occurred"); -1 }
    }
}
```

With `extern "C"`, panics abort the process. With `extern "C-unwind"` (Rust 1.71+), panics can unwind through C frames, allowing proper cleanup.

## Implementation recommendations for a new language

**Zero-overhead C calls** require ABI-compatible primitive types (`c_int`, `c_long`, etc.) that change size per target, and `extern` struct declarations that replicate C layout rules exactly. Follow Zig's approach: make FFI types first-class, not an afterthought.

**Automatic safe wrapper generation** should use a compile-time translation tool (like Zig's Aro or Rust's bindgen) that emits both raw bindings and safe wrappers. Annotate ownership semantics directly in binding declarations:

```
// Hypothetical syntax
@ffi("header.h") {
    @returns_owned fn create_object() -> *Object;
    @borrows(self) fn get_name(obj: *Object) -> *const char;
    @consumes(self) fn destroy_object(obj: *Object);
}
```

**Python embedding** should follow PyO3's model: compile-time GIL state tracking, automatic type conversions with explicit opt-out, and DLPack support for tensor interop. Design for free-threaded Python from the start—the GIL is deprecated.

**Memory safety** requires clear ownership documentation at every FFI boundary. The language should make it easy to express "this function transfers ownership" vs "this function borrows." Consider requiring explicit `@owned` or `@borrowed` annotations.

The minimum viable FFI implementation:
1. ABI-compatible primitive types with target-specific sizes
2. `extern struct` with C layout replication
3. `extern fn` with correct calling convention
4. Compile-time header translation for C interop
5. Thread-local error storage for error bridging
6. Trampoline infrastructure for callbacks with user_data

The advanced implementation adds:
7. Automatic safe wrapper generation from ownership annotations
8. RAII wrapper generation for C resources
9. Python embedding with GIL state tracking
10. Arena allocators for FFI-heavy workloads

The key insight: **don't make developers choose between safety and performance**. The unsafe primitives should be as ergonomic as Zig's, while the safe wrappers should be as automatic as Rust's bindgen with clear, documented escape hatches.