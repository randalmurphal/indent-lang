# Embedding and plugin architecture for Indent

Indent's unique characteristics—**regions + ARC memory management**, **colorless async**, and **sub-100ms script execution**—position it exceptionally well for embedding. The absence of garbage collection eliminates the coordination nightmares that plague Python and JavaScript embedding, while the compiled nature enables native-speed plugin execution. Based on research into successful embedding languages (Lua, QuickJS) and modern plugin systems (Wasmtime, HashiCorp go-plugin), this report recommends a **tiered plugin architecture** combining in-process native plugins for performance, WASM for untrusted code, and a capability-based security model native to the type system.

## What makes languages embeddable—and where Indent fits

Lua dominates game and infrastructure embedding (World of Warcraft, Redis, nginx, Neovim) through deliberate design constraints: **200KB binary**, **<1ms startup**, **no global state**, and a stack-based C API that decouples from type system internals. Every successful embeddable language shares these characteristics:

| Characteristic | Lua | QuickJS | V8 | Python | Indent (Target) |
|----------------|-----|---------|-----|--------|-----------------|
| Binary size | 200KB | 370KB | 74MB | 50MB+ | <500KB |
| Startup time | ~5ms | <300μs | 100ms+ | 50ms+ | <10ms |
| Global state | None | None | Per-isolate | Heavy | None (regions) |
| Memory model | GC | Ref counting | GC | GC | Regions + ARC |
| Predictable latency | No (GC) | Yes | No (GC) | No (GC) | **Yes** |

Indent's region-based memory model provides a **decisive advantage**: predictable latency without GC pauses. This mirrors Lua's appeal but with stronger guarantees. The compile-time ARC elimination means most reference counting happens at compile time, avoiding the runtime overhead that hampers Python embedding. QuickJS adopted reference counting specifically because it's more predictable for embedding than tracing GC—Indent's approach is even better.

**The critical insight**: Lua's C API design philosophy—"embeddability has a subtle influence; Lua favors mechanisms that can be represented naturally in the Lua-C API"—should guide Indent. Syntax features that can't be cleanly exposed via FFI should be reconsidered. Every language feature should have a natural embedding API representation.

## Three-tier plugin architecture recommendation

For a compiled, no-GC language targeting DevOps engineers, a single plugin mechanism cannot serve all needs. Research into real-world systems reveals three distinct use case clusters requiring different tradeoffs:

### Tier 1: Native plugins via C ABI for maximum performance

Native dynamic loading (.so/.dll) delivers zero-overhead integration but requires **strict ABI discipline**. The C ABI is the only universally stable binary interface—every compiler and language targets it. Indent should expose a C-compatible interface using `extern "C"` equivalents:

```
// Indent plugin interface (C ABI compatible)
pub extern "C" fn indent_plugin_init(host: *const HostInterface) -> *mut Plugin
pub extern "C" fn indent_plugin_call(plugin: *mut Plugin, fn_name: *const u8, args: *const Value) -> Result
pub extern "C" fn indent_plugin_free(plugin: *mut Plugin)
```

**Version compatibility strategy**: Follow Neovim's API level approach—each function has a `since` field indicating minimum API level, with deprecation warnings before removal. Never break ABI within a major version.

**When to use**: Performance-critical extensions, database drivers, system integrations where the 1.5-2x WASM overhead is unacceptable.

### Tier 2: WASM for untrusted and cross-language plugins

WebAssembly has matured into the **definitive sandbox format for 2025**. The Component Model (now stable in Wasmtime) solves the historic pain points of WASM plugins:

- **Rich type exchange** via WIT (WebAssembly Interface Types): strings, arrays, records cross the boundary without manual marshaling
- **Language-agnostic authoring**: Plugins written in Rust, Go, Python, or any language compiling to WASM
- **Strong isolation**: Memory bounds checking on every access, no ambient authority

**Performance reality check**: WASM incurs **45-55% overhead** versus native on compute-intensive benchmarks, with peak slowdowns of 2-2.5x. For Indent's "sub-100ms script execution" target, a 50ms native operation becomes 75-100ms in WASM—acceptable for user-defined automation but not for hot-path infrastructure.

**Extism integration**: Rather than building custom WASM infrastructure, adopt Extism's plugin model. It provides host SDKs for 16+ languages, handles memory management across the WASM boundary, and uses Wasmtime internally. Example:

```python
# Host loading Indent plugin compiled to WASM
manifest = {"wasm": [{"path": "user_plugin.wasm"}]}
with extism.Plugin(manifest, wasi=True) as plugin:
    result = plugin.call("process", input_data)
```

**When to use**: User-submitted automation scripts, marketplace plugins, any code from untrusted sources, cross-language integrations.

### Tier 3: gRPC subprocess for maximum isolation

HashiCorp's go-plugin pattern (used by Terraform, Vault, Nomad) provides **process isolation** via gRPC over subprocess. The plugin runs as a separate process—crashes cannot affect the host, memory is fully isolated by the OS, and plugins can be written in any language supporting gRPC.

**Latency cost**: IPC overhead adds 0.5-2ms per call. For plugins making thousands of calls per operation, this becomes prohibitive. But for plugins performing few high-level operations (provisioning infrastructure, managing secrets), the isolation guarantees outweigh latency.

**When to use**: Cloud provider integrations, security-sensitive operations (secrets, credentials), plugins that might hang or crash, long-running background operations.

### Architecture decision matrix

| Use Case | Tier 1 (Native) | Tier 2 (WASM) | Tier 3 (gRPC) |
|----------|-----------------|---------------|---------------|
| Database drivers | ✓ Best | Acceptable | Too slow |
| User automation scripts | Risky | ✓ Best | Overkill |
| Cloud provider SDKs | Acceptable | Good | ✓ Best |
| Template engines | ✓ Best | Acceptable | Too slow |
| Rule engines | ✓ Best | ✓ Best | Too slow |
| Marketplace plugins | Unsafe | ✓ Best | Good |

## Sandboxing design for embedded Indent code

When Indent code runs embedded in another application, or when Indent hosts untrusted plugins, a **capability-based security model** should be integral to the language, not bolted on.

### Capability-aware type system

Deno's permission model (`--allow-read=/path`, `--allow-net=hostname`) demonstrates how capabilities work at the CLI level. Indent should embed this in the **type system itself**:

```indent
// Function requires file system capability
fn read_config(path: Path) -> Config requires fs:read(path.parent):
    file = open(path)?
    return parse(file.read_all()?)

// Capability attenuation - pass only what's needed
fn main() requires fs:read("/etc"):
    // read_config can only read within /etc
    config = read_config("/etc/app/config.toml")
```

This approach, inspired by the object-capability model in Pony and Joe-E, makes capabilities **checkable at compile time**. A function requiring `net:connect` cannot be called from a context lacking that capability.

### Resource limits for embedded execution

Research into WASM runtimes and Redis Lua sandboxing reveals proven patterns:

**Fuel-based CPU limiting** (Wasmtime model): Compiled code is augmented to count operations. When fuel exhausts, execution traps. Deterministic but **~2x performance overhead**.

**Epoch-based interruption** (lower overhead): A global counter is incremented by the host; at function entries and loop headers, the guest checks if its deadline has passed. Only **~10% overhead** but non-deterministic.

For Indent's sub-100ms target, epoch-based is preferred for production, with fuel available for debugging/testing where determinism matters.

**Memory limiting**: Track allocations against a per-context budget. Regions simplify this—each region can have a maximum size, and region creation fails if the context's total allocation would exceed limits.

```indent
// Host creates bounded execution context
context = EmbedContext.new(
    memory_limit: 64.megabytes,
    cpu_deadline: 50.milliseconds,
    capabilities: [fs:read("/data"), net:connect("api.example.com:443")]
)
result = context.run(user_script)?
```

### Defense in depth for untrusted code

Cloudflare Workers demonstrates the gold standard: V8 isolate (memory isolation) + V8 sandbox (pointer compression, bounds checking) + memory protection keys + OS-level seccomp + network separation. For Indent:

1. **Language level**: No unsafe escape hatches in embedded code, no raw pointer arithmetic, validated regions
2. **Runtime level**: WASM sandbox with Component Model when running untrusted code
3. **Resource level**: Fuel or epoch-based CPU limits, memory caps per context
4. **Capability level**: Explicit grants for filesystem, network, environment access
5. **OS level**: Recommend seccomp profiles for production deployments

## Configuration language mode: competing with Dhall, CUE, Jsonnet

Indent's characteristics make it suitable as a **configuration language** for DevOps workflows. However, the landscape is crowded with specialized tools. Here's how Indent should position:

### Feature comparison for configuration use

| Feature | Dhall | CUE | Jsonnet | Nickel | Indent (Proposed) |
|---------|-------|-----|---------|--------|-------------------|
| Type safety | Full static | Constraints | None | Gradual | Gradual |
| Termination | Guaranteed | Guaranteed | Not guaranteed | Not guaranteed | Optional guarantee |
| Functions | First-class | Encoded | First-class | First-class | First-class |
| Import integrity | SHA256 hashes | In progress | None | Planned | **SHA256 hashes** |
| Evaluation | Strict | Lazy-ish | Lazy | Lazy | Strict |
| JSON superset | No | Yes | Yes | No | Yes (recommended) |
| IDE support | Good | Improving | Good | Good | Must have LSP |

### Recommendations for config mode

**Hash-verified imports** are essential. Dhall's approach—imports specify SHA256 hashes, mismatches are errors—prevents supply chain attacks and ensures reproducibility:

```indent
import "https://indent-hub.dev/kubernetes/v1.2.3" sha256:abc123...
```

**Termination analysis**: Offer a `--config-mode` that enables additional checks: no unbounded recursion, no infinite loops, guaranteed termination. This trades expressiveness for safety in the configuration use case.

**JSON superset syntax**: For adoption, ensure all valid JSON is valid Indent config. Users can migrate incrementally from YAML/JSON.

## API design for embedding Indent

### Stack-based vs handle-based embedding API

Lua's stack-based API achieves type agnosticism—functions like `push_value` and `get_value` work without knowing the specific type at compile time. This enables polyglot bindings (C, Rust, Go, Python hosts can all use the same API) but requires careful stack management.

V8's handle-based API (`Local<T>`, `Persistent<T>`) provides stronger type safety and clearer lifetime semantics but is more verbose.

**Recommendation for Indent**: A **hybrid approach** combining handle-based types with stack-like ergonomics:

```c
// Handle-based for type safety
indent_value_t value = indent_call(ctx, func_handle, args, num_args);
indent_string_t str = indent_to_string(ctx, value);  // Returns owned string
indent_free_string(str);  // Caller frees

// Arena-based for batch operations (Indent's regions)
indent_arena_t* arena = indent_arena_new(ctx, 64 * 1024);
for (int i = 0; i < 1000; i++) {
    indent_value_t v = indent_alloc_in(arena, ...);  // Allocated in arena
}
indent_arena_free(arena);  // Single deallocation
```

This leverages Indent's region model—the arena maps directly to a region, enabling efficient batch operations.

### Type marshaling strategy

For cross-boundary data:

| Data Type | Strategy | Performance |
|-----------|----------|-------------|
| Primitives (int, float, bool) | Direct pass | Zero overhead |
| Strings | Pointer + length pair | Zero-copy read, copy on write |
| Structs | Handle to region-allocated data | Zero-copy within region |
| Arrays | Pointer + length + element type | Zero-copy for primitive arrays |
| Complex nested | Serialize to MessagePack | Moderate overhead |

Indent's regions enable **zero-copy borrowing**: the host passes a region handle, embedded code allocates into that region, and both sides access the same memory without copying. The region's lifetime (controlled by the host) ensures safety.

### Error propagation

Indent's Result types with `?` operator should translate naturally across boundaries:

```c
// C API mirrors Indent's Result semantics
indent_result_t result = indent_call(ctx, func, args);
if (indent_is_err(result)) {
    indent_error_t err = indent_get_error(result);
    fprintf(stderr, "Error: %s\n", indent_error_message(err));
    indent_free_error(err);
    return -1;
}
indent_value_t value = indent_get_ok(result);
```

For panics: catch at FFI boundary, convert to error result, never allow panics to propagate into C code.

## Performance benchmarks and targets

### FFI call overhead reality

LuaJIT achieves the **fastest FFI** of any embeddable language—actually faster than C due to JIT-compiled direct calls avoiding PLT overhead. Benchmarks on 500 million calls:

| Language | Time (ms) | Relative to C |
|----------|-----------|---------------|
| LuaJIT | 900 | 0.76x (faster) |
| C/Rust/Zig | ~1,180 | 1.0x |
| Go | 38,000 | 32x slower |
| Node.js | 9,200 | 7.8x slower |
| Java (JNI) | 4,500 | 3.8x slower |

**Indent target**: Match C-level FFI overhead (~1,200ms for 500M calls). This is achievable with AOT compilation and no managed runtime coordination.

### Embedding overhead budget

For "sub-100ms script execution" goal:

| Phase | Budget | Notes |
|-------|--------|-------|
| Context creation | <1ms | Allocate region, initialize state |
| Script loading | <5ms | Parse + compile if not cached |
| Execution | <90ms | Actual script work |
| Cleanup | <1ms | Free regions |
| **Total** | <100ms | |

**AOT compilation** is essential—JIT warmup would blow the budget. Pre-compile scripts to native code; execution should be instantaneous.

### Memory footprint targets

| Component | Target | Comparison |
|-----------|--------|------------|
| Minimal runtime | <300KB | Lua: 200KB, QuickJS: 370KB |
| Per-context overhead | <100KB | Region allocator + state |
| Idle memory | <1MB | No background threads |

## Use case feasibility analysis

### Configuration language: highly feasible

Indent's Python-like syntax, type safety, and deterministic execution make it **excellent for configuration**. With termination guarantees in config mode and hash-verified imports, it would compete directly with Dhall (safer) and Jsonnet (more familiar). The sub-100ms target is perfect for configuration evaluation.

**Differentiator**: Unlike Dhall/CUE/Jsonnet, Indent configs can use the same tooling (LSP, debugger) as Indent application code.

### Game scripting: feasible with native plugins only

Game engines require **predictable latency** (no GC pauses) and **tight integration** (thousands of calls per frame). Indent's region model eliminates GC pauses; native plugin mode achieves required call overhead. WASM mode adds too much latency for hot paths.

**Challenge**: LuaJIT's FFI is actually faster than C for accessing game data structures—Indent cannot match this without JIT. Recommendation: position for turn-based/strategy games, not twitch-action games.

### Rule engines: highly feasible

Business rule evaluation requires type safety, fast execution, and hot-reloading. Indent can compile rules to native code, reload by replacing regions. The `?` error propagation maps naturally to rule evaluation (success/failure).

### Template engines: feasible with zero-copy strings

Indent's f-string syntax and region-based string allocation enable efficient template evaluation. Zero-copy string views avoid the allocation overhead that plagues template engines.

### User-defined automation: requires WASM tier

Untrusted user scripts **must** run in WASM sandbox. The 1.5-2x overhead is acceptable for automation (human-initiated, not hot-path). Capability restrictions prevent malicious scripts from accessing host resources.

## Development experience for embedders

### Documentation patterns

Following successful embedding APIs (Lua, Node N-API):

1. **Getting started guide**: Minimal working example in <20 lines
2. **API reference**: Every function documented with ownership semantics
3. **Cookbook**: Common patterns (callbacks, iteration, error handling)
4. **Migration guides**: From Lua, Python, JavaScript embedding

### Debugging embedded code

**Challenge**: Stack traces across language boundaries are fragmented. 

**Solution**: Unified trace format—Indent runtime captures its portion, provides hook for embedder to add native portion. Result:

```
Error: division by zero at user_script.indent:42
  in divide_numbers() at user_script.indent:42
  in process_data() at user_script.indent:28
  --- FFI boundary ---
  in native_call_indent() at host.c:156
  in main() at host.c:45
```

### Testing plugins

Provide mock host interface for unit testing plugins without embedding context:

```indent
// Plugin test using mock host
test "plugin processes data correctly":
    host = MockHost.new()
    host.mock_read_file("/config", "key=value")
    
    plugin = MyPlugin.new(host)
    result = plugin.process()?
    
    assert result.config["key"] == "value"
```

## Specific design recommendations for Indent

Given Indent's characteristics—compiled, no-GC, colorless async, Python-like syntax—here are concrete recommendations:

1. **Memory model advantage**: Market the region + ARC model explicitly as an embedding advantage. "Predictable performance without garbage collection pauses."

2. **Colorless async integration**: When embedded code calls a blocking host function, the region yields automatically. Host resumes when operation completes. No explicit async/await at FFI boundary—this is a major UX win over Node.js/Python embedding.

3. **Region-as-arena pattern**: Expose regions directly in embedding API. Host creates region, passes to embedded code, all allocations go there, single deallocation at end. This maps naturally to request-scoped operations.

4. **Configuration mode flag**: Add `--config` mode that enables termination checking, disables network/filesystem (unless explicitly granted), and outputs JSON/YAML. This positions Indent for the CUE/Dhall market.

5. **Capability types in stdlib**: Build capability awareness into standard library. `File.open` requires `fs:read` capability; `net.connect` requires `net:connect`. These propagate through call chains.

6. **WASM compilation target**: Ensure Indent compiles to WASM for when Indent plugins run in non-Indent hosts (nginx, Envoy, edge computing).

7. **"One obvious way" for embedding**: Provide exactly one recommended pattern for each use case. Don't offer three ways to register callbacks—offer one that works.

The combination of regions (predictable memory), colorless async (clean FFI), and capability types (built-in sandboxing) could make Indent the **first embedding-first compiled language** since Lua. Unlike Lua, it would offer static typing and native performance. Unlike Rust, it would be approachable for DevOps engineers. This is a significant opportunity in the language landscape.