# Designing a "targeted batteries" standard library for backend developers

Backend developers need **immediate productivity without overwhelm**—the research reveals clear patterns across Go, Rust, Python, and emerging languages about what works. The most successful approach combines Go's simplicity with Rust's performance principles, prioritizing HTTP, JSON, database access, logging, and CLI parsing while avoiding scope creep into GUI, email, or specialized domains.

## What backend developers actually need on day one

Survey data from Stack Overflow (49,000+ developers) and JetBrains (24,534 developers) combined with package download statistics reveal a remarkably consistent pattern across ecosystems. **HTTP client/server**, **JSON serialization**, **database access**, **logging**, and **configuration management** appear in virtually every backend project regardless of language.

The numbers are striking: in Python, `urllib3` and `requests` exceed **500 million** monthly downloads combined. In Rust, `serde` has **711 million** total downloads—it's essentially mandatory for any serious project. Go's `net/http` and `encoding/json` packages are imported in nearly every Go backend service. The "must have on day one" list isn't controversial; it's the same everywhere.

Package download data reveals this priority order for backend work:

- **HTTP handling** (95%+ of backend projects): Express (25-30M weekly npm), reqwest (38M crates.io), requests (200M+ monthly PyPI)
- **JSON serialization** (90%+): Native in JS, serde_json in Rust (711M+), encoding/json standard in Go
- **Database connectivity** (90%+): PostgreSQL now dominates (58% developer usage, surpassing MySQL)
- **Logging** (85%+): winston/pino in Node, zap/slog in Go, tracing in Rust
- **Environment/config** (80%+): dotenv packages exceed 25M weekly downloads across ecosystems

Docker usage hit **73.8%** among professional developers (up 17 points), making containerization nearly universal—stdlib design should assume containerized deployment patterns like stdout logging.

## HTTP library design requires safe defaults and layered APIs

Go's `net/http` demonstrates both the best and worst of standard library design. The `http.Handler` interface—just one method, `ServeHTTP(ResponseWriter, *Request)`—is praised universally for enabling composition. Automatic HTTP/2 support since Go 1.6 and the goroutine-per-connection model handle concurrency naturally. Go 1.22 finally added method matching and path parameters (`GET /users/{id}`), features that previously required third-party routers.

However, Go's HTTP library has a **critical design flaw**: no default timeouts. Cloudflare's engineering team documented that `http.ListenAndServe()` and `http.Get()` are "unfit for public Internet servers" because the default client waits forever for slow servers. One Cloudflare blog post states: "Timeouts are amongst the easiest and most subtle things to get wrong." Streaming servers particularly suffer—there's no way to cancel a blocked `ResponseWriter.Write` or implement proper idle timeouts.

Rust's hyper takes the opposite approach: low-level, async-first, and explicitly not ergonomic. It's designed as a building block, not a complete solution. This works because the ecosystem layers on top: **reqwest** provides the ergonomic wrapper (automatic connection pooling, redirects, cookies), while **axum** provides the server framework. The pattern is instructive—separate protocol correctness from developer convenience.

Deno's approach deserves attention for a new language. Using web platform standard `Request`/`Response` objects means developers who know the Fetch API already know Deno's HTTP. The server setup is minimal: `Deno.serve((req) => new Response("Hello"))`. Automatic compression, WebSocket upgrades, and HTTP/2 are built-in without configuration.

For a new language targeting backend developers, the HTTP design should follow these principles:

1. **Default timeouts of 30 seconds** for clients—never infinite
2. **Single-line server startup** for basic cases, escape hatches for advanced configuration
3. **Method + path + parameters in stdlib routing**—Go 1.22 proves this is expected baseline
4. **Layered design**: low-level protocol implementation separated from ergonomic API
5. **Web standards alignment** where practical (`Request`/`Response` concepts)

## JSON serialization should use compile-time code generation

The performance difference between compile-time and reflection-based JSON is substantial. Benchmarks show Rust's serde_json achieving **300-580 MB/s** throughput versus Go's encoding/json at **24-89 MB/s**—serde is **3-10x faster** depending on workload. Go's json-iterator (using code generation) achieves 4x the decode speed of encoding/json, demonstrating that the gap is methodology, not language.

Go's reflection approach has ergonomic problems beyond performance. Struct tags are unvalidated strings—a missing close quote in `json:"field` compiles but fails silently at runtime. Case-insensitive unmarshaling by default causes surprising behavior. The distinction between missing fields and zero values is impossible without pointer types, adding nil-check boilerplate throughout codebases.

Serde's derive macro approach (`#[derive(Serialize, Deserialize)]`) provides compile-time validation, rich customization attributes (`rename`, `skip`, `default`, `flatten`), and format-agnostic design—the same struct works with JSON, YAML, TOML, and binary formats. The cost is compile time (proc macros add overhead) and larger binaries. Developer feedback is overwhelmingly positive on ergonomics.

**Miniserde**, created by serde's author dtolnay, demonstrates an interesting middle ground: 12x less code than full serde, uses trait objects instead of monomorphization (reducing binary size), and avoids recursion (preventing stack overflow on deeply nested data). The tradeoff is fewer features.

For a new language, the recommended approach combines lessons from both:

- **Compile-time code generation as default** via derive macros—catches type mismatches at compile time, eliminates reflection overhead
- **Trait-based design** separating Serialize/Deserialize from format-specific serializers
- **Rich attribute system with compile-time validation**—unlike Go's string-based struct tags
- **Strict defaults**: reject invalid UTF-8, duplicate keys, require case-sensitive matching (Go's json/v2 proposal validates these choices)
- **Streaming support from the start**—don't buffer entire values in memory

## Database interfaces need the Go pattern with better null handling

Go's `database/sql` implements a successful abstraction: the user-facing `sql.DB` type handles connection pooling, retries, and query management, while the `driver` interface defines the minimal contract for backend implementations. Driver authors implement a small interface; shared functionality lives in the abstraction layer once.

The pattern works because pooling complexity is hidden—developers get configurable `MaxOpenConns`, `MaxIdleConns`, and `ConnMaxLifetime` without implementing any of it. Optional capability interfaces (`Pinger`, `SessionResetter`, `Validator`) let drivers expose advanced features without breaking the core interface.

But Go's null handling is a major pain point. Scanning NULL into non-pointer types fails. The `sql.NullString`, `sql.NullInt64` wrapper types produce ugly JSON (`{"Valid": true, "Value": 5}`) and add boilerplate everywhere. GitHub issues proposing fixes have hundreds of comments. A new language must **design for null ergonomically from day one**, likely using `Option<T>` types with good serialization behavior.

Rust intentionally lacks a standard database abstraction, preferring specialized libraries: **Diesel** for compile-time query validation via Rust's type system, **SQLx** for raw SQL with compile-time verification against actual databases during build, **SeaORM** for a higher-level ORM. This diversity serves different needs but fragments the ecosystem.

Python's DB-API 2.0 (PEP 249) demonstrates the specification approach: define interfaces without providing code, letting implementations vary. It achieved wide adoption (sqlite3, psycopg2, mysql-connector all implement it) but lacks shared types for static analysis and leaves connection pooling to external libraries.

For database interface design:

- **Split user API from driver interface** (Go pattern)—connection pooling at abstraction layer, not driver responsibility
- **Ergonomic null handling**—`Option<T>` types that serialize cleanly
- **Minimal driver interface** with optional capability extensions
- **Scanner/Valuer pattern** for custom type conversion
- **Clear error hierarchy** (Python DB-API's exception tree is well-designed)
- **Support both sync and async** without forcing async on backends that don't benefit (SQLite is synchronous)

## Logging design should follow Go's slog model

Go's `slog` package, added in Go 1.21, represents the current state of the art in stdlib logging design. Its architecture separates the **Logger** (frontend providing `Info`, `Error`, etc.) from the **Handler** (backend interface for output formatting). This separation means any logging frontend can output to any handler—enabling ecosystem standardization on handlers while maintaining frontend flexibility.

The Handler interface includes `Enabled(context.Context, Level) bool`, called before argument processing to skip expensive work for disabled levels. The `WithAttrs` and `WithGroup` methods allow pre-formatting persistent attributes for performance. Value types are optimized to store small values without allocation using unsafe (fitting any value in 3 machine words).

Slog's level design is intentional: DEBUG=-4, INFO=0, WARN=4, ERROR=8. The gaps of 4 match OpenTelemetry's mapping, INFO at 0 makes it the zero-value default, and negative values support verbosity-style logging (like glog). Context integration is built-in for trace correlation.

Python's `logging` module serves as a cautionary tale: hierarchical loggers, complex configuration, no native structured output, and global state issues make it frustrating for library authors and application developers alike. Rust's ecosystem split between `log` (simple facade) and `tracing` (spans/structured) shows value in keeping basic logging separate from distributed tracing infrastructure.

Honeycomb's "wide events" philosophy offers a perspective shift: emit one structured event per request per service with all relevant context, rather than scattered log lines. Initialize an event blob at request start, accumulate throughout the request lifecycle, emit at completion.

For a new language's logging:

**Stdlib should include:**
- Severity levels (DEBUG, INFO, WARN, ERROR) with intentional numeric spacing
- Structured key-value API as default (`log.info("message", key=value)`)
- Handler interface for pluggable backends
- Built-in text and JSON handlers
- Context integration for trace ID propagation
- Lazy evaluation mechanism (LogValuer pattern)
- `Enabled()` early bailout for disabled levels

**Libraries should handle:**
- Tracing/spans (complex, evolving OpenTelemetry spec)
- Network exporters (OTLP, vendor-specific)
- Advanced sampling algorithms
- Development pretty-printing

## Standard library versioning requires layered compatibility mechanisms

Go's compatibility promise—"programs will continue to compile and run correctly"—has enabled decade-long stability while allowing evolution. The secret is the **GODEBUG mechanism**: when behavior changes, a setting allows opting back to old behavior for minimum 2 years (4 releases). The `go` line in `go.mod` determines default GODEBUG settings, separating "upgrade toolchain" from "adopt new behavior."

Despite the promise, Go averages approximately one breaking change per year for large projects. Examples include `net.ParseIP` rejecting addresses with leading zeros (Go 1.17), HTTP/2 default enabling breaking connections to buggy servers, and `panic(nil)` becoming a runtime error (Go 1.21). The GODEBUG mechanism provides escape hatches.

Rust's edition system (2015, 2018, 2021, 2024) enables syntax-breaking changes opt-in per crate, with **cross-edition interoperability mandatory**. All Rust code compiles to the same internal representation regardless of edition. Critically, **editions cannot change standard library APIs**—std is shared across all editions. The system works for new keywords (`async`, `await`) and lint defaults, not API evolution.

Java uses multi-level deprecation: ordinary deprecation (`@Deprecated`) for discouraged-but-staying APIs, terminal deprecation (`@Deprecated(forRemoval=true)`) for scheduled removal. The jdeprscan tool enables static analysis. Java 9's module system encapsulates internal APIs, enabling systematic migration away from `sun.misc.Unsafe`.

Python requires minimum 2-year deprecation periods between warning and removal. The `__future__` mechanism enables opt-in to future behavior before it becomes mandatory. "Soft deprecation" covers features that should be avoided but won't be removed.

Deno's current approach uses **independent semver versioning per stdlib package** (@std/fs, @std/path, etc.). Packages at v1.0.0+ are considered stable with strict semver. This reduces "update anxiety" for large libraries and allows experimental packages to iterate freely.

For a new language's versioning strategy:

1. **Project manifest version** controls default behaviors (like Go's go.mod)
2. **Runtime toggles** for behavior changes (GODEBUG pattern) with minimum 2-year support
3. **Multi-level deprecation**: soft deprecation → developer warnings → user warnings → removal
4. **Modular stdlib** with independent versioning per package where practical
5. **Ecosystem testing infrastructure** (like Rust's Crater) from early days
6. **Edition markers** reserved for rare syntax-breaking changes, not API evolution
7. **Automated migration tooling** (`cargo fix --edition` pattern)

## Concrete stdlib scope for a backend-focused language

Based on this research, a "targeted batteries" stdlib for backend developers should include these packages at 1.0:

**Core (must have day one):**
- `http` - Client and server with safe defaults, method+path routing, middleware composition
- `json` - Compile-time serialization with derive macros, streaming support
- `yaml`, `toml` - Configuration file formats (same serialization traits)
- `sql` - Database abstraction with pooling, null ergonomics, driver interface
- `log` - Structured logging with slog-style Handler interface, JSON/text handlers
- `fs` - File system operations
- `time` - Time handling, duration, formatting (learned from Go's `time` success)
- `env` - Environment variables and configuration
- `cli` - Argument parsing (clap-style ergonomics)
- `test` - Testing framework built-in

**Important but can follow shortly:**
- `crypto` - Hashing, encryption (security-sensitive, benefits from careful design)
- `regex` - Regular expressions
- `uuid` - UUID generation (ubiquitous in backend work)
- `url` - URL parsing and manipulation

**Explicitly NOT in stdlib:**
- GUI, graphics, image processing
- Email, SMTP
- Machine learning, numerical computing
- Specific database drivers (PostgreSQL, MySQL)—these implement the `sql` driver interface
- Full HTTP frameworks with routing, middleware, etc.—provide building blocks
- OpenTelemetry exporters—library territory

This scope provides immediate backend productivity while leaving room for ecosystem libraries to provide specialized functionality. The stdlib establishes interfaces and conventions; the ecosystem provides diversity and innovation.

## Key design principles synthesized from research

**Default to safety over convenience.** Go's infinite HTTP timeouts are a footgun used in production. Safe defaults with explicit opt-out for power users serves everyone better.

**Compile-time over runtime where practical.** Serde's 3-10x performance advantage over reflection-based JSON isn't marginal. Compile-time code generation catches errors earlier and runs faster.

**Layered APIs with escape hatches.** Provide ergonomic high-level APIs for common cases, expose low-level primitives for customization. Rust's hyper/reqwest pattern works.

**Design for null from day one.** Go's sql.Null* types demonstrate how painful retrofitting null handling becomes. Build Option types with good serialization behavior into the language.

**Structured data as default.** Structured logging, typed serialization, rich error types—assume machine-parseable output everywhere.

**Version-aware behavior.** The GODEBUG pattern separates toolchain upgrades from behavior changes, enabling gradual migration. Build this mechanism before you need it.

**Modular stability.** Independent package versioning allows mature packages to stabilize while experimental ones iterate. Don't couple everything to a single version number.

The research consistently shows that **getting the defaults right matters more than providing every option**. Backend developers want to be productive immediately with patterns that scale to production. A "targeted batteries" stdlib achieves this by being opinionated about the 80% case while providing escape hatches for the rest.