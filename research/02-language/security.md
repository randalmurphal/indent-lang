# Security models for untrusted code execution in backend languages

A new backend language targeting cloud-native deployment, multi-tenant safety, and plugin support requires a **defense-in-depth security architecture** that combines language-level capability enforcement, runtime sandboxing, and OS-level isolation. The optimal approach layers all three rather than choosing exclusively among them—each addresses different threat vectors and provides distinct guarantees. Given your language's features (Cranelift/LLVM compilation, structured concurrency, regions + ARC memory), you're uniquely positioned to implement compile-time capability checking with minimal runtime overhead.

The key insight from production systems is that **no single isolation mechanism is sufficient**. AWS Lambda layers Firecracker microVMs with seccomp and namespaces. Cloudflare Workers combines V8 isolates with memory protection keys, cordons, and process-level sandboxing. Roblox's Luau implements VM-level readonly tables, interrupt mechanisms, and removes dangerous metamethods entirely. Each discovered vulnerabilities in "simpler" approaches that required additional layers.

---

## The three security enforcement layers and when to use each

Security for untrusted code divides into **language-level** (compile-time capability tracking), **runtime-level** (sandbox enforcement during execution), and **OS-level** (kernel-mediated isolation). These layers are complementary, not competing alternatives.

**Language-level security** (Pony's reference capabilities, E's object capabilities) provides the strongest developer experience and can eliminate overhead through compile-time verification. Pony's six reference capabilities (`iso`, `trn`, `ref`, `val`, `box`, `tag`) prove at compile time that mutable data is never shared between actors, eliminating data races without runtime checks. The tradeoff is learning curve—developers report a maturity process of weeks to months before capabilities become intuitive. For your language, integrating capabilities with your region-based memory model creates a natural fit: regions could carry capability annotations that flow through bidirectional type inference.

**Runtime-level security** (Deno's permission system, WebAssembly's import/export boundary) enforces capabilities during execution for code that can't be statically verified. Deno checks permissions at the "op layer"—every call from JavaScript to native Rust code passes through `PermissionsContainer::check_*()` before syscalls execute. This design prevents JavaScript from bypassing security, but adds per-operation overhead (typically nanoseconds for hash set lookups). WASM's approach is more elegant: modules literally cannot express syscalls—all I/O requires explicit imports provided by the host, making capability enforcement zero-cost after linking.

**OS-level security** (seccomp-bpf, Landlock, namespaces) provides the final containment layer that survives runtime vulnerabilities. Chrome's sandbox uses Linux namespaces (PID, network, mount) plus seccomp-bpf filtering to contain compromised renderer processes. This layer cannot be bypassed by bugs in your language implementation—even if an attacker achieves arbitrary code execution, they're still confined by kernel-enforced limits. The overhead is minimal (seccomp adds ~50-100ns per syscall), but configuration complexity is significant.

---

## Deno's capability-based permission system sets the standard for runtime enforcement

Deno implements **deny-by-default capabilities** with fine-grained scoping. All system I/O is blocked unless explicitly granted via flags like `--allow-read=/specific/path` or `--allow-net=api.example.com:443`. The architecture places permission checks in Rust code at the lowest possible layer:

```rust
fn op_get_env(state: &mut OpState, key: String) -> Result<Option<String>, AnyError> {
    state.borrow_mut::<PermissionsContainer>().check_env(&key)?;
    // Only reaches actual env access if check passes
}
```

This design has three critical properties. First, **JavaScript cannot bypass checks**—there's no path from V8 to syscalls that doesn't pass through permission-checked ops. Second, **capabilities are hierarchical**—granting `/home/user` implicitly grants `/home/user/docs/file.txt` through the `stronger_than` method. Third, **capabilities cannot be elevated**—workers inherit parent permissions or receive fewer, never more.

However, Deno's CVE history reveals recurring vulnerability patterns your language should address proactively. **Symlink attacks** (CVE-2021-41641) bypassed deny rules via `/proc/self/root/etc/passwd`. **FFI escape** (CVE-2022-24783) allowed shell code execution because FFI ops didn't check permissions—native code loaded via `Deno.dlopen()` runs completely outside the sandbox. **Prompt bypasses** (CVE-2024-32477) used ANSI escape sequences and race conditions to trick users into granting permissions. The lesson: design your permission model to treat path canonicalization, native interop, and user interaction as attack surfaces requiring hardening.

For your language's design, the key recommendation is **permission checks at compilation boundaries**. Since you're using Cranelift for dev builds, you could emit runtime permission checks only for operations that can't be statically verified, while LLVM release builds verify capabilities at link time and eliminate checks entirely for proven-safe code paths.

---

## WebAssembly achieves zero-cost isolation through architectural constraints

WASM's security model is fundamentally different from Deno's: rather than checking permissions at runtime, **the bytecode format makes unsafe operations inexpressible**. A WASM module cannot emit syscalls, access memory outside its linear memory region, or call functions not explicitly imported. This provides capability security without per-operation overhead.

**Linear memory isolation** works through bounds checking on every memory access. Production runtimes like Wasmtime use **2GB guard regions** and **6GB total protection** via virtual memory—out-of-bounds accesses trigger page faults rather than requiring explicit bounds check instructions. This achieves near-native performance for memory-safe workloads. The type system ensures all memory references compute with infinite precision, preventing wraparound exploits.

**WASI (WebAssembly System Interface)** implements capability-based I/O through pre-opened file descriptors. The runtime explicitly grants directory access via preopens; modules can only access files within granted directories using relative paths. WASI Preview 2 (January 2024) extends this to network capabilities and introduces unforgeable resource handles. Preview 3 will add composable async I/O without function coloring—directly relevant to your colorless structured concurrency model.

The **component model** enables secure composition of WASM modules. Each component runs in a separate sandbox with isolated memory. Interface Types (WIT) define language-agnostic boundaries where capabilities are explicitly declared. A YAML parser component can be granted zero network access while a HTTP client component receives only specific endpoints—and they can safely compose without sharing mutable state.

For your language, WASM's architecture suggests designing your IR (whether compiled via Cranelift or LLVM) to enforce capability constraints at the bytecode level. If your IR cannot express raw syscalls—only high-level operations like `region.read()` or `net.connect()`—then your runtime only needs to implement those operations securely, with the compiler proving they're the only operations that can execute.

---

## Lua sandboxing in games reveals the limits of interpreter-level security

Roblox, Garry's Mod, and World of Warcraft have spent decades sandboxing Lua for millions of concurrent users running untrusted scripts. Their experience reveals that **bytecode is inherently dangerous** and **interpreter-level sandboxing is insufficient against sophisticated attackers**.

Roblox's Luau (a Lua 5.1 fork) implements comprehensive hardening. All standard libraries are marked **readonly at the VM level**—not just protected by metatables, but enforced by the bytecode interpreter itself. The `__gc` metamethod was completely removed because it runs during garbage collection in arbitrary thread context, creating uncontrollable timing. The `loadstring` function rejects bytecode input and `string.dump` was removed, because Lua bytecode can manipulate VM internals in ways impossible from source code (a 2013 exploit stole arbitrary function values including C functions by crafting bytecode that confused the VM's stack handling).

Luau's **interrupt mechanism** guarantees script termination: a global handler can be invoked from the host at any time, and scripts are guaranteed to call it "eventually" (at function calls, loop iterations). Roblox Studio uses a 10-second timeout; client shutdown triggers a 1-second interrupt. This architectural guarantee is essential—`debug.sethook` instruction counting has **10-20% CPU overhead** and doesn't work with LuaJIT (the JIT compiler bypasses bytecode interpretation entirely).

World of Warcraft's **taint system** provides an elegant model for capability propagation. All Blizzard code starts "secure"; all addon code is "tainted." Accessing tainted values taints the execution path; creating values inherits current execution taint. Protected functions (character movement, spellcasting) only execute from secure code paths with hardware event triggers. This prevents addons from automating gameplay while allowing rich UI customization. The model proves that capability tracking can be orthogonal to the type system—taint flows dynamically through data without explicit type annotations.

For your language, these lessons translate to: (1) never allow loading arbitrary bytecode—all executable code must come from your compiler; (2) remove or secure all reflective capabilities like debug APIs; (3) implement guaranteed termination through interrupt mechanisms integrated with your structured concurrency runtime; (4) consider taint-style dynamic capability tracking for multi-party code composition.

---

## Pony's reference capabilities prove compile-time security is practical

Object-capability security provides **architectural immunity** to confused deputy attacks by bundling designation with authorization. Instead of passing a filename (designation) to a function that uses its own authority, you pass a file capability that embeds both the identity of the resource and permission to access it. This eliminates entire vulnerability classes without runtime overhead.

Pony demonstrates that capability systems can be both **memory-safe and data-race-free** through compile-time verification. Its six reference capabilities create a lattice:

| Capability | Read | Write | Sendable (cross-actor) |
|------------|------|-------|------------------------|
| `iso` (isolated) | ✓ | ✓ | ✓ |
| `trn` (transition) | ✓ | ✓ | ✗ |
| `ref` (reference) | ✓ | ✓ | ✗ |
| `val` (value/immutable) | ✓ | ✗ | ✓ |
| `box` (read-only) | ✓ | ✗ | ✗ |
| `tag` (identity only) | ✗ | ✗ | ✓ |

The fundamental invariant is: **mutable data cannot be shared; shared data cannot be mutable**. `iso` references are unique—transferring requires `consume`, making the source unusable. `val` references are globally immutable and freely sharable. The `recover` block enables capability lifting by proving isolation within the block. The compiler enforces all invariants, eliminating runtime capability checks.

**E's object capability patterns** provide building blocks your language should consider:

- **Sealer/Unsealer** (rights amplification): Analogous to public/private key pairs. The sealer wraps data that only the matching unsealer can extract. Combining two capabilities yields authority neither has alone.
- **Caretaker** (revocation): A proxy that can be disabled. The revoker capability allows instant invalidation without modifying the underlying resource.
- **Membrane** (deep attenuation): Wraps entire object graphs, transitively wrapping all objects that pass through. Enables policy enforcement at trust boundaries.

For your language design, the integration opportunity is profound. Your **regions + ARC memory model** already tracks ownership; extending regions with capability annotations creates natural capability scoping. A region could be `iso` (exclusively owned, transferable), `val` (frozen, sharable), or scoped to specific actors. Your **bidirectional type inference** can propagate capability requirements, reducing annotation burden. Your **structured concurrency** provides natural capability boundaries—spawned tasks inherit parent capabilities or receive explicit subsets.

---

## Linux kernel sandboxing provides the unbypassable final layer

seccomp-bpf, Landlock, and namespaces form the **defense-in-depth backstop** that contains vulnerabilities in your language implementation. Even arbitrary code execution within a properly sandboxed process remains confined.

**seccomp-bpf** filters syscalls using BPF programs that inspect `struct seccomp_data`:

```c
struct seccomp_data {
    int nr;         // Syscall number
    __u32 arch;     // Architecture (CRITICAL: must validate)
    __u64 instruction_pointer;
    __u64 args[6];  // Raw argument values (cannot dereference)
};
```

The critical limitation is that **BPF cannot dereference pointers**—you can filter on syscall numbers and raw arguments, but not path strings or buffer contents. Chrome's sandbox whitelist approves ~60 syscalls; gVisor's Sentry makes only ~68 host syscalls. Filter overhead is **50-100 nanoseconds per syscall** with JIT compilation; Linux 5.11+ caches always-allow syscalls for near-zero overhead.

**Landlock** (Linux 5.13+) provides **semantic filesystem and network restrictions** that complement seccomp. Unlike seccomp, Landlock can restrict access to specific paths and TCP ports. Rules are stackable—each layer must grant access. ABI v4 added `LANDLOCK_ACCESS_NET_BIND_TCP` and `LANDLOCK_ACCESS_NET_CONNECT_TCP`. Current limitations: cannot restrict `stat()`, `chmod()`, `chown()`, or `chdir()`.

**Namespaces** isolate kernel resources without filtering. User namespaces enable unprivileged sandbox setup (maps UID 0 inside to unprivileged user outside). PID namespaces hide host processes. Mount namespaces isolate filesystem views. Network namespaces provide separate network stacks. Chrome's namespace sandbox creates new PID and network namespaces, chroots into an empty directory, and drops all capabilities.

The recommended **layering strategy** for your language's runtime:

```
Application Code
├── Landlock (filesystem paths, network ports)
├── seccomp-bpf (syscall whitelist)
├── Namespaces (user, pid, mount, network)
├── cgroups (CPU, memory, I/O limits)
└── Capabilities (privilege restriction)
```

Your runtime should apply these layers during initialization. For AOT-compiled code, you can analyze the syscall requirements at compile time and generate minimal seccomp profiles per workload. For plugin/script execution, apply maximally restrictive defaults that plugins can request relaxation from (with user approval).

---

## Cloud provider isolation reveals production-proven patterns at scale

AWS Lambda (Firecracker), Google Cloud Run (gVisor), and Cloudflare Workers (V8 isolates) represent three fundamentally different approaches, each proven at billions of invocations.

**Firecracker microVMs** provide hardware-virtualized isolation in ~50,000 lines of Rust (vs. QEMU's 1.4M lines). Each Lambda function runs in its own microVM with a full Linux kernel. The Jailer process applies defense-in-depth before VMM startup: fresh namespaces, cgroups, seccomp-bpf (~40 whitelisted syscalls), privilege dropping, chroot. Cold starts are **~125ms** with **<5 MiB memory overhead** per microVM. Boot rate exceeds 150 microVMs/second per server.

**gVisor's user-space kernel** intercepts syscalls without hardware virtualization. The Sentry (written in Go) implements **237 syscalls** for application compatibility while making only **~68 host syscalls** itself. The Gofer process mediates all filesystem access. The systrap platform uses `SECCOMP_RET_TRAP` for interception with instruction rewriting optimization that avoids full signal overhead. Overhead is **<3% for most applications** per Google's data, but syscall-heavy workloads see significant slowdown.

**Cloudflare Workers' V8 isolates** achieve **~5ms cold starts** with **~3MB memory per isolate**—orders of magnitude faster and smaller than containers. Multiple isolates run in a single OS process, relying on V8's battle-tested JavaScript sandboxing. Defense-in-depth includes:
- **Cordons**: Multiple runtime instances per machine; free tier never shares process with enterprise customers
- **Memory Protection Keys (MPK)**: Hardware protection for isolate heaps; 92% of cross-isolate access attempts hit hardware trap
- **V8 Sandbox**: 8 GiB virtual memory reservation with 32-bit offsets instead of pointers
- **Spectre mitigation**: `Date.now()` locked during execution; no multi-threading (prevents ad-hoc timers)

The performance comparison is stark:

| Technology | Cold Start | Memory/Instance |
|------------|------------|-----------------|
| V8 Isolates | ~5ms | ~3MB |
| WebAssembly (Fastly) | ~35μs | ~KB |
| gVisor | Fast (no kernel boot) | Variable |
| Firecracker | ~125ms | <5 MiB |

---

## Synthesis: recommended security architecture for your language

Given your language's features—Cranelift/LLVM compilation, structured concurrency with colorless functions, bidirectional type inference, regions + ARC with 95% RC elimination—here's a comprehensive security architecture:

### Layer 1: Compile-time capability verification

Integrate capabilities with your type system and region model. Define a capability algebra:

```
region iso[Capability] { ... }  // Isolated region with specific caps
region val { ... }              // Frozen, sharable region
```

Capabilities could include filesystem paths, network endpoints, environment variables, and spawn permissions. Your bidirectional type inference propagates capability requirements—functions that call `net.connect()` are inferred to require `cap::Net`. At compile time (especially LLVM release builds), prove capability safety and eliminate runtime checks for verified code paths.

This approach mirrors Pony's reference capabilities but extends to I/O. Wyvern's capability-based effects provide a model where `resource module` declarations require explicit capability imports.

### Layer 2: Runtime capability enforcement

For code that can't be statically verified (dynamic loading, eval-like features, user scripts):

- Implement Deno-style op-layer permission checks
- Use WASM's import/export model for plugin sandboxing—plugins can only access explicitly provided capabilities
- Provide interrupt mechanisms integrated with structured concurrency for guaranteed termination
- Support capability attenuation (give plugins reduced capabilities) and revocation (disable capabilities dynamically)

Your Cranelift dev builds could include verbose capability checking with detailed errors; LLVM release builds optimize away verified checks.

### Layer 3: OS-level containment

Ship default seccomp profiles generated from compile-time syscall analysis. Your runtime knows exactly which syscalls a compiled program can invoke. For multi-tenant deployments, layer:

1. User namespace (unprivileged operation)
2. PID namespace (process isolation)
3. Mount namespace (minimal filesystem)
4. Network namespace (controlled network)
5. Landlock rules (path-based access)
6. seccomp-bpf (syscall whitelist)
7. cgroups (resource limits)

Provide runtime APIs to query and configure isolation:

```
sandbox {
    allow_syscalls: [read, write, mmap, futex]
    allow_paths: ["/data/input": Read, "/data/output": Write]
    memory_limit: 256.megabytes
    cpu_limit: 50.percent
}
```

### Deployment guidance for different use cases

**User-provided scripts**: Use WASM-style sandbox with minimal imports. Compile user scripts to your IR with capability-stripped environment. Run in separate process with full OS isolation. Overhead budget: 10-50ms cold start acceptable.

**Plugin systems**: Define capability interfaces in your type system. Plugins declare required capabilities; loader verifies and grants minimal set. Use membrane pattern for capability attenuation across plugin boundaries. Overhead budget: <1ms hot path.

**Multi-tenant backend services**: Firecracker-style isolation for maximum security (tenants cannot affect each other even with exploits). For higher density with acceptable risk, use gVisor-style or V8-style isolation with trust-level separation (enterprise tenants don't share processes with free tier).

### Performance overhead summary

| Layer | Overhead | When to Use |
|-------|----------|-------------|
| Compile-time caps | Zero (eliminated checks) | All production code |
| Runtime cap checks | ~10-50ns per operation | Unverified code, plugins |
| seccomp-bpf | ~50-100ns per syscall | All production deployments |
| Landlock | Minimal per file op | Untrusted workloads |
| Namespaces | ~100-500μs creation | Per-tenant isolation |
| gVisor-style | 2-10% syscall overhead | Medium-trust multi-tenant |
| microVMs | ~125ms cold start | Zero-trust multi-tenant |

### Common vulnerabilities to prevent

Based on production CVE analysis:

1. **Path canonicalization**: Always resolve symlinks before permission checks. Treat `/proc/self/root` as an attack vector.
2. **FFI escape**: Native code runs outside your sandbox. Require explicit capability grants for FFI; consider running FFI code in separate process with OS isolation.
3. **Reflection/debug APIs**: Remove or heavily restrict. Lua's `debug` library, Python's `__builtins__`, and similar enable sandbox escapes.
4. **Bytecode attacks**: Never load untrusted bytecode. Your compiler should be the only bytecode source.
5. **Timing oracles**: For multi-tenant deployment, lock timer precision (like Cloudflare) or isolate tenants that need precise timing.
6. **Resource exhaustion**: Enforce memory limits at allocation time (not just cgroups); implement instruction counting or interrupt mechanisms.

### Developer experience tradeoffs

Capability systems require explicit capability passing, which initially feels verbose. Mitigation strategies:

- **Implicit capability contexts**: Like Scala implicits or Kotlin context receivers, capabilities can flow through call chains without explicit threading
- **Manifest files**: Declare package-level capability requirements (like WASM component model worlds)
- **IDE integration**: Show required capabilities in function signatures; suggest capability imports
- **Gradual adoption**: Allow `unsafe` blocks that opt out of capability checking for legacy code migration

The Pony community reports that once learned (weeks to months), capabilities become intuitive and make code easier to reason about. The compile-time guarantees eliminate entire categories of concurrency bugs and security vulnerabilities.

---

## Conclusion

For your new backend language, the optimal security architecture combines **compile-time capability verification** (integrated with your type system and region model), **runtime enforcement** (Deno-style permission checks for unverified code, WASM-style imports for plugins), and **OS-level containment** (seccomp + Landlock + namespaces for defense-in-depth).

Your existing features—Cranelift/LLVM compilation, structured concurrency, bidirectional type inference, regions + ARC—create unique opportunities. Regions can carry capability annotations. Type inference can propagate capability requirements. Structured concurrency provides natural capability boundaries. The 95% RC elimination philosophy extends to capability checks: statically verify 95% at compile time, emit runtime checks only for the remainder.

The production-proven lesson from Lambda, Cloud Run, and Workers is that defense-in-depth is non-negotiable. Every layer will eventually have vulnerabilities. Hardware isolation contains runtime bugs. Runtime sandboxing contains language bugs. Language capabilities prevent bugs in application code. Together, they create a security posture that degrades gracefully under attack rather than failing catastrophically.