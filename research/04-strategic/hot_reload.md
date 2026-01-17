# Hot Code Reload Feasibility Assessment for Indent

**Implementing hot code reload in a new language is technically achievable but requires foundational design decisions made early—retrofitting it later is nearly impossible.** The most successful implementations (Erlang/OTP, ClojureScript/Figwheel) share a common architecture: late binding, explicit state management, and runtime module replacement. The JVM's struggles demonstrate what happens when performance optimizations assume immutable classes. For Indent, the decision comes down to prioritizing developer experience and operational flexibility over raw performance—a tradeoff that companies like WhatsApp and Discord have proven worthwhile at massive scale.

## What makes hot reload work: the three pillars

The research reveals that successful hot reload systems rest on three architectural foundations that Indent must embrace from the start.

**First, late binding is non-negotiable.** When Erlang calls `Module:function()`, the lookup happens at runtime through the code server—there's no compiled-in address that becomes stale. Clojure achieves this through Vars (mutable containers holding functions), and Smalltalk through dictionary-based method dispatch. The JVM's fundamental limitation is that JIT compilation assumes classes are frozen forever; the moment a method call is inlined, no amount of clever engineering can update that code path without deoptimization overhead. Indent should default to indirect function calls through a runtime registry, with an opt-in mechanism for inlining in performance-critical paths.

**Second, state and behavior must be separable.** ClojureScript's Figwheel works beautifully because functional components describe *what should render* given state, not imperative mutations. When a function is redefined, simply re-invoking it with existing state produces the correct new output. Erlang's GenServer codifies this: processes hold state, receive messages, and expose a `code_change/3` callback that transforms old state shapes into new ones during upgrades. The pattern `defonce state (atom {})` in ClojureScript prevents state reinitialization on reload. Indent needs explicit constructs for reload-stable state—either atoms/refs that survive reloads, or automatic state migration hooks integrated into its module system.

**Third, the runtime must support module versioning.** BEAM runs exactly two versions of each module simultaneously: "current" and "old." Fully-qualified calls (`Module:function()`) route to current; local calls stay in old until the process exits the function and calls remotely. A third version load kills processes still running the oldest code—a hard deadline for migration. This two-version architecture enables graceful transitions without coordination complexity. Indent should adopt similar semantics: modules are first-class runtime entities with version metadata, and the runtime maintains an old-new pair with configurable purge behavior.

## Language features that enable versus prevent hot reload

The research clearly identifies which language features help versus hurt.

**Features that enable hot reload:**
- **Late binding and dynamic dispatch** allow new function definitions to take effect immediately for future calls without relinking
- **First-class functions with indirection** (like Clojure Vars) mean function references update atomically across the codebase
- **Immutable data structures** eliminate aliasing bugs—old code safely operates on old data while new code uses new data
- **Runtime code evaluation** (eval) enables defining new code in a running context
- **Reflection and introspection** allow the runtime to enumerate and update live objects

**Features that prevent or complicate hot reload:**
- **Aggressive inlining** copies function bodies into call sites, eliminating the indirection needed for replacement. GCC's `-O2` breaks hot reload in C++; the JVM's JIT inlining is the primary obstacle to enhanced HotSwap
- **Closure capture semantics** create orphaned code. Once a closure captures a function, redefining the source doesn't update existing closures—they're independent objects. Flutter's documentation acknowledges "reloading closures is complicated"
- **Global mutable state** without lifecycle management becomes orphaned between reloads. Clojure's tools.namespace `refresh` destroys Vars, even `defonce` ones, requiring explicit lifecycle frameworks (Component, Integrant)
- **AOT compilation** as default eliminates runtime flexibility. Clojure explicitly warns that reloading doesn't work with AOT-compiled namespaces
- **Static type systems with structural rigidity** prevent adding fields or changing hierarchies. The JVM can't add instance variables because object layout is fixed at class load time

**Indent design implications:** The language should default to late-bound calls with opt-in inlining. Static typing is compatible with hot reload (see Erlang's dialyzer) but must not constrain runtime structure. A `reload-stable` annotation should mark state that survives reloads, with mandatory migration functions for shape changes.

## State management is the hardest problem

Preserving application state across code changes requires deliberate architecture. The research identifies four proven patterns.

**Pattern 1: In-process state transformation (Erlang's code_change/3).** Processes are suspended, the callback transforms state from old to new format, and they resume. This is the fastest approach—no serialization—but requires developers to write migration logic for every state shape change.

**Pattern 2: External state storage (ETS, atoms, stores).** State lives outside the code being reloaded. ClojureScript's `defonce` ensures atoms aren't reinitialized. React Fast Refresh preserves hooks state by tracking "signatures" and only resetting state when hooks order changes. This pattern works well for UI but requires careful design to avoid stale references.

**Pattern 3: State hydration on restart.** Discord's guild processes (GenServers) hydrate state from Postgres on restart. The process is a "living breathing cache where the persistence layer is merely used as a backup." Combined with fast startup times (**0.5ms** for actor read/write vs **10ms** database read), this makes restarts nearly invisible.

**Pattern 4: Managed lifecycle frameworks.** Clojure's Component and Integrant formalize start/stop sequences. Reload becomes: stop system → refresh code → start system. This is the most explicit but requires buying into the framework.

**Connection preservation** is critical for production use. Discord's Phoenix LiveView automatically re-renders after upgrade without dropping WebSocket connections. Slack keeps old processes running for hours during migrations, draining connections naturally. Envoy's hot restart copies socket file descriptors between parent and child processes.

**Indent recommendations:** Build a lifecycle system into the standard library. Connections (WebSocket, database, file handles) should live in supervised processes separate from application logic. Provide `before_reload` and `after_reload` hooks at module level. State containers should have explicit versioning and migration functions.

## The JVM lesson: what not to do

The JVM's hot swap limitations illustrate exactly what Indent should avoid. Standard HotSwap, introduced in Java 1.4, only supports **method body changes**—no adding methods, fields, changing signatures, or modifying hierarchies. This makes it "practically useless for real-world development," as JRebel's documentation states.

**Why is full hot reload so hard on JVM?**

| Challenge | Technical Cause |
|-----------|-----------------|
| JIT inlining | Call sites contain copied function bodies with no indirection to update |
| Object layout | Fields compiled to fixed byte offsets; adding fields breaks all existing instances |
| Class identity | Class = name + classloader; true replacement requires new classloader and object migration |
| Static state | Third-party code holds references to old class versions, causing ClassCastException |
| Multiple JIT compilers | C1, C2, Graal all optimize differently; enhanced hot swap must work with all |

**Workarounds exist but have tradeoffs.** DCEVM patches HotSpot to allow structural changes but requires running a non-standard JVM. JRebel uses bytecode instrumentation to create proxy classes that can be versioned, but it's commercial ($550/year) and proprietary. Spring DevTools uses fast restart with dual classloaders—not true hot swap, but much faster than cold restart. GraalVM's Espresso (Java on Truffle) offers the most complete hot swap but runs in an interpreter with speculation-based JIT.

**Key insight for Indent:** The fundamental tension is between performance optimizations (inlining, fixed object layout) and runtime flexibility. A language designed for hot reload must choose the flexibility side for development mode, with optional production optimizations that trade away some reload capability.

## Frontend patterns worth adopting

Frontend tooling has achieved remarkable developer experience through HMR, with patterns directly applicable to Indent.

**Vite/Webpack HMR boundaries** are explicit opt-in points. A module calling `import.meta.hot.accept()` creates a boundary; changes propagate upward through importers until hitting a boundary (HMR works) or the entry point (full reload). This gives developers control over reload granularity and fallback behavior.

**React Fast Refresh** preserves component state through signature tracking. The Babel plugin instruments components with `$RefreshReg$` (registers components) and `$RefreshSig$` (tracks hook usage). When a component is redefined, React maps old fiber tree nodes to new component definitions, preserving `useState` and `useRef` values if the hooks signature matches. **Critically, `useEffect` always re-runs during Fast Refresh**, ignoring dependency arrays—this ensures side effects reflect new code.

**Figwheel's validation before loading** is elegant: it refuses to load code with compiler warnings, keeping the running application stable. Errors are displayed without corrupting state. This "fail-safe before fail-fast" approach should be standard.

**Adoption for Indent:**
- Implement module-level accept boundaries with explicit `hot.accept()` equivalent
- Track function signatures for state preservation decisions
- Run side effects unconditionally on reload (or provide explicit opt-out)
- Validate new code before loading; keep old version on error

## Production hot upgrade: it works at scale

Real-world deployments prove hot code upgrade is viable for mission-critical systems.

**WhatsApp** ran 2+ million connections per server on Erlang, achieving "extremely long uptimes" through hot code loading. Their approach was pragmatic: simple module patches propagated via RPC across the cluster, avoiding complex OTP relup tooling. The combination of vertical scaling (fewer, larger servers) and hot swap meant minimal coordination overhead.

**Discord** scaled to 5 million concurrent users on Elixir, developing custom libraries (Manifold for distributed fanout, FastGlobal for hot data lookup at **0.3μs**) to make the actor model performant. Their guild processes are GenServers that hydrate from Postgres on restart—enabling both hot upgrade and crash recovery with the same mechanism.

**Ericsson** pioneered the approach for telecom switches requiring "five nines" (99.999%) availability. Their key insight: "divisions of Ericsson that use relups spend as much time testing them as they do testing their applications themselves." Hot upgrade works, but the testing investment is substantial.

**However, most teams prefer simpler approaches.** From the Erlang community: "If it is possible to upgrade your application in ways that do not require relups, I would recommend doing so." Rolling restarts with load balancers, blue-green deployments, and canary releases are operationally simpler and well-understood. Hot upgrade should be a capability for when you truly need zero-downtime connection preservation—not the default deployment strategy.

## Implementation roadmap for Indent

Based on this analysis, here's a phased implementation approach:

### Phase 1: Foundation (must ship with v1.0)

Build these capabilities into the language and runtime from day one—they cannot be retrofitted:

- **Late-bound function calls by default** through a runtime registry/module table
- **Two-version module coexistence** with current/old semantics like BEAM
- **Soft purge primitive** that only succeeds if no processes are executing old code
- **Reload-stable state annotation** (`@stable` or similar) for data that survives reloads
- **Module lifecycle hooks** (`before_reload`, `after_reload`) at the module level
- **State migration callback** (`migrate_state(old_state, old_version) -> new_state`) invoked automatically during reload

### Phase 2: Development experience (v1.1)

Optimize for fast iteration during development:

- **File watcher with dependency analysis** for automatic reload of changed modules in dependency order
- **REPL with full system access** including breakloops on error (not just print-and-exit)
- **Validation before load** rejecting code with errors/warnings while preserving running version
- **Error overlay** displaying compile errors without breaking application state
- **Test runner integration** that hot-reloads test code without restarting the test process

### Phase 3: Production capabilities (v1.2)

Enable zero-downtime deployment for high-availability use cases:

- **Appup/relup equivalent** for coordinated multi-module upgrades with dependency ordering
- **Distributed reload coordination** across cluster nodes
- **Connection-preserving upgrade protocol** for WebSockets and long-lived connections
- **Metrics and health monitoring** during upgrade with automatic rollback on failure
- **Canary upgrade support** running multiple code versions with traffic splitting

### Phase 4: Advanced features (v2.0)

Nice-to-have capabilities that can wait:

- **More than two versions** for complex upgrade sequences
- **Automatic state migration inference** using structural typing
- **Profile-guided inlining** that tracks call sites for deoptimization on reload
- **Time-travel debugging** leveraging state snapshots between reloads

## Critical design decisions for Indent

The research points to several non-obvious decisions:

**Actor model provides the best foundation.** Process isolation, message-based communication, and supervision trees make hot reload tractable. Erlang's success isn't despite its actor model—it's because of it. Indent should either adopt actors natively or provide actor-like abstractions (isolated state, explicit message passing, supervision).

**Development mode should differ from production mode.** Disable inlining and enable full hot reload for development. In production, offer a spectrum: "fast restart" mode (sub-second process restart with state hydration), "module reload" mode (BEAM-style), and "live upgrade" mode (full relup support). Let teams choose their availability/complexity tradeoff.

**The two-version limit is a feature, not a bug.** It forces developers to complete migrations promptly and prevents combinatorial explosion of version compatibility testing. Adopt it.

**Closures are the hardest problem.** Document clearly that closures stored in data structures (event handlers, middleware stacks, callbacks) become stale after reload. Encourage patterns that store function references by name rather than capturing function values.

## Conclusion: feasible with commitment

Hot code reload is feasible for Indent, but only if the commitment is made at language design time. The successful implementations—Erlang/BEAM, Smalltalk, ClojureScript—designed for liveness from the beginning. The failed attempts—standard JVM HotSwap, most static languages—tried to bolt it on later and hit fundamental architectural barriers.

The key requirements are late binding by default, explicit state management with migration hooks, runtime module versioning with two-version coexistence, and a supervision/lifecycle system for managing connections and resources. The actor model provides an elegant foundation for all of these.

For development workflow, hot reload dramatically improves feedback loops—seeing changes in under a second versus 10-30 second restarts. For production, the capability exists and works at WhatsApp/Discord scale, but most teams should start with simpler deployment strategies and reserve true hot upgrade for high-availability requirements where connection preservation is critical.

**The feasibility verdict:** Implement it. Design for it now. The developer experience benefits justify the architectural constraints, and the production capabilities—while complex—are proven at the largest scales.