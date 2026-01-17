# Built-in observability design for Indent

**Observability as a first-class language feature fundamentally changes how developers instrument, debug, and operate cloud-native applications.** This specification outlines how Indent can embed tracing, logging, metrics, and debugging primitives directly into the language—achieving what currently requires dozens of third-party libraries and manual instrumentation. Drawing from Erlang/BEAM's production introspection, Rust's zero-cost abstractions, Go's pragmatic profiling, and OpenTelemetry's emerging standards, this design enables automatic instrumentation with less than 2% overhead while maintaining complete visibility into production systems.

## Automatic instrumentation through compiler injection

The compiler serves as the primary instrumentation point, eliminating boilerplate while preserving developer control. Rather than requiring manual span creation for every function, Indent's compiler can automatically inject observability code during compilation—similar to Go's Orchestrion project which manipulates the AST before compilation, but built into the language itself.

**Function-level tracing via attributes** provides opt-in automatic instrumentation:

```indent
@trace(level: Info, fields: {user_id: ctx.user_id})
fn process_order(ctx: Context, order: Order) -> Result<Receipt, OrderError> {
    // Compiler injects: span creation, timing, error capture, context propagation
    // Function arguments automatically recorded via Debug trait
    validate_order(order)?  // Errors automatically attached to span
    charge_payment(ctx, order.total)?
    Ok(generate_receipt(order))
}
```

The `@trace` attribute expands at compile time to inject span lifecycle management, argument recording, return value capture, and automatic error attachment. Unlike Java's runtime bytecode manipulation (which adds **19-52% overhead**), compile-time expansion has zero runtime overhead beyond the actual tracing operations.

**Automatic error context capture** embeds trace information directly into error types:

```indent
// Every error automatically captures trace context at creation point
struct TracedError {
    message: String,
    trace_id: TraceId,      // Automatically captured from current span
    span_id: SpanId,
    source_location: SourceLocation,
    cause: Option<Box<TracedError>>,
}

// Error messages automatically include correlation IDs
// "Order validation failed [trace_id=0af7651916cd43dd span_id=b7ad6b7169203331]"
```

**Memory allocation tracking** follows Go's sampling-based approach where the runtime automatically captures allocation stack traces:

```indent
// Runtime configuration
runtime.set_alloc_profile_rate(512 * 1024)  // Sample every 512KB

// Automatic exposure via built-in HTTP endpoints
// GET /debug/profile/heap -> allocation profiling
// GET /debug/profile/allocs -> all allocations since start
```

The BEAM VM demonstrates the gold standard for built-in tracing—every process spawn, message send, function call, and garbage collection event is traceable without code modification. Indent can adopt this model by making the runtime aware of tracing primitives, enabling `runtime.trace(process, [:calls, :messages, :gc])` to dynamically enable per-process tracing in production.

## Structured logging with compile-time type safety

Logs should be typed data structures, not interpolated strings. String-based logging costs **~100 nanoseconds** even when disabled due to concatenation, while parameterized logging costs only **~1.5 nanoseconds** with disabled-level checks.

**The `LogValue` discriminated union** avoids heap allocation for common types:

```indent
enum LogValue {
    String(str),
    Int64(i64),
    Uint64(u64),
    Float64(f64),
    Bool(bool),
    Duration(Duration),
    Time(Timestamp),
    Group(List<Attr>),
    Lazy(() -> LogValue),  // Deferred evaluation
}

struct Attr {
    key: str,
    value: LogValue,
}

// Type-safe constructors avoid allocation for small values
fn string(key: str, value: str) -> Attr
fn int(key: str, value: i64) -> Attr
fn duration(key: str, value: Duration) -> Attr
```

Go's slog found that **over 95% of log calls pass five or fewer attributes**, informing optimizations that store small attribute lists inline without heap allocation.

**Log levels as an enum enable compile-time filtering**:

```indent
enum Level { Trace = -8, Debug = -4, Info = 0, Warn = 4, Error = 8 }

// Cargo-style feature flags eliminate code at compile time
// Build with: indent build --max-log-level=info
// Result: all Trace and Debug logging code removed from binary

@compile_if(MAX_LOG_LEVEL >= Level::Debug)
macro log_debug(msg: str, attrs: ...Attr) {
    if logger.enabled(Level::Debug) {
        logger.log(Level::Debug, msg, attrs)
    }
}
```

Rust's tracing crate proves this approach works: with `max_level_info` enabled, trace-level instrumentation is **not even present in the resulting binary**—the expensive computation inside a `trace!()` call is completely eliminated by the compiler.

**Lazy evaluation prevents wasted computation**:

```indent
// LogValuer interface for expensive computations
trait LogValuer {
    fn log_value(self) -> LogValue
}

// Only computed if debug logging is enabled
log.debug("Processing complete", 
    "summary", LazyValue { compute_expensive_summary() })

// Closure-based alternative
Logger.debug(|| ("Expensive computation", data: compute_data()))
```

**The Handler abstraction** separates log production from consumption:

```indent
trait Handler {
    fn enabled(ctx: Context, level: Level) -> bool  // Early filtering
    fn handle(ctx: Context, record: Record) -> Result<()>
    fn with_attrs(attrs: List<Attr>) -> Handler  // Pre-format repeated attrs
    fn with_group(name: str) -> Handler  // Namespace attrs
}
```

## Context propagation and distributed tracing

Context propagation determines whether traces connect across service boundaries. Indent should make context a first-class runtime concept, automatically threaded through function calls while providing explicit APIs for cross-process propagation.

**Implicit context via execution unit attachment**:

```indent
// Context automatically attached to current fiber/task
let ctx = Context.current()
let span = Tracer.current_span()

// Explicit scoping for specific operations
Context.with(custom_ctx) {
    // All operations in this block use custom_ctx
    downstream_call()  // Automatically receives context
}
```

For async code, Rust's tracing demonstrates the critical pattern—**never hold span guards across await points**:

```indent
// WRONG: span context corrupted across await
async fn bad_example() {
    let _guard = span.enter()
    some_async_fn().await  // Guard held across await!
}

// CORRECT: use instrument combinator
async fn good_example() {
    some_async_fn()
        .instrument(info_span!("operation"))
        .await
}
```

**W3C Trace Context as the default propagation format**:

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             │   │                                │                │
             │   └─ trace-id (128-bit)            │                └─ flags (01=sampled)
             │                                    └─ parent-id (64-bit span ID)
             └─ version (00)
```

**Propagator interface for cross-process context**:

```indent
trait TextMapPropagator {
    fn inject(carrier: mut TextMap, context: Context)
    fn extract(carrier: TextMap, context: Context) -> Context
    fn fields() -> List<str>  // Header names used
}

// Built-in propagators
let propagator = CompositePropagator::new([
    W3CTraceContext::new(),      // Default
    W3CBaggage::new(),           // Application data propagation
    B3Propagator::new(),         // Zipkin compatibility
])
```

**Sampling strategies** balance overhead against data quality:

| Strategy | Decision Point | Overhead | Data Quality | Use Case |
|----------|---------------|----------|--------------|----------|
| Head-based | Trace start | Lowest | May miss anomalies | High-volume production |
| Tail-based | Trace completion | Medium (buffering) | Best for errors | Error-focused observability |
| Adaptive | Dynamic | Low-medium | Good balance | Variable traffic patterns |

Google's Dapper defaults to **1 in 1024** traces for high-throughput services, demonstrating that aggressive sampling still provides sufficient visibility while keeping overhead under **0.3% of CPU**.

**Baggage propagation** carries application data across services:

```indent
// Setting baggage (propagates in headers)
let baggage = Baggage::new([
    ("user_id", "alice"),
    ("tenant_id", "acme-corp"),
    ("feature_flags", "new_checkout"),
])
Context.with_baggage(baggage) {
    http_client.post(url, body)  // baggage header: user_id=alice,tenant_id=acme-corp,...
}

// W3C Baggage limits: max 8192 bytes, 180 entries
// NEVER put secrets, PII, or credentials in baggage
```

## Type-safe metrics primitives

Metrics should be language primitives with compile-time label safety, not string-based APIs prone to typos and cardinality explosions.

**Core metric types**:

```indent
// Counter: monotonically increasing, starts at 0
struct Counter<L: LabelSet> {
    fn inc()              // Increment by 1
    fn add(value: f64)    // Must be >= 0
}

// Gauge: can increase or decrease
struct Gauge<L: LabelSet> {
    fn set(value: f64)
    fn inc()
    fn dec()
    fn add(value: f64)    // Can be negative
}

// Histogram: distribution of values in buckets
struct Histogram<L: LabelSet> {
    buckets: List<f64>,   // Immutable after creation
    fn observe(value: f64)
    fn start_timer() -> Timer  // Timer.stop() calls observe()
}
```

**Type-safe labels prevent instrumentation errors**:

```indent
// Labels as struct with derive macro
#[derive(LabelSet)]
struct HttpLabels {
    method: HttpMethod,    // Enum: GET, POST, PUT, DELETE
    status: StatusClass,   // Enum: 2xx, 3xx, 4xx, 5xx
    path: str,
}

let requests = Counter::<HttpLabels>::new(
    name: "http_requests_total",
    help: "Total HTTP requests processed",
)

// Type-checked at compile time
requests.with(HttpLabels { 
    method: HttpMethod::GET, 
    status: StatusClass::Success2xx,
    path: "/api/users"
}).inc()

// Compile error: wrong type
requests.with(HttpLabels { method: "GET", ... })  // Error: expected HttpMethod
```

**Global registry with file-static registration** (Prometheus recommendation):

```indent
// File-level metric declaration
static requests: Counter<HttpLabels> = Counter::new(
    name: "http_requests_total",
    help: "Total HTTP requests",
)

fn init() {
    registry.must_register(requests)
}

// Alternative: explicit registry for testing
let test_registry = Registry::new()
let counter = Counter::new(opts)
test_registry.register(counter)
```

**Cardinality management** prevents the primary metrics failure mode:

```indent
// SDK-enforced cardinality limits
MeterProvider::new(
    cardinality_limit: 2000,  // Per metric
    views: [
        View::new(
            instrument: "http_request_duration",
            cardinality_limit: 1000,  // Override for specific metric
        )
    ]
)

// Guidelines: Never use high-cardinality labels
// BAD: user_id, request_id, session_id
// GOOD: method, status_class, endpoint_category
```

**Aggregation at source** (OpenTelemetry model) provides predictable behavior:

```indent
// Pre-aggregation in SDK before export
// Memory preallocated during initialization
// Constant data volume regardless of measurement volume

Instrument → Pre-Aggregation → Metrics → MetricReader → Collector
```

## Production debugging without restarts

The BEAM VM demonstrates the gold standard: every process, message, and function call is traceable in production without code changes. Indent should provide similar capabilities as language primitives.

**Built-in profiling endpoints** (following Go's net/http/pprof pattern):

```indent
// Automatic exposure when debug server enabled
indent.debug_server.start(port: 6060)

// Endpoints automatically available:
// GET /debug/profile/cpu?seconds=30  -> CPU profile
// GET /debug/profile/heap            -> Memory snapshot  
// GET /debug/profile/goroutines      -> All fiber/task stacks
// GET /debug/profile/block           -> Blocking profile
// GET /debug/profile/mutex           -> Lock contention
// GET /debug/snapshot/heap           -> Heap dump
```

**Dynamic log level changes** without restart:

```indent
// HTTP API endpoint
POST /debug/logger/com.example.MyModule
{ "level": "debug" }

// Programmatic API
runtime.set_log_level("com.example", Level::Debug)

// Atomic level for thread-safe updates
let level = AtomicLevel::new(Level::Info)
level.serve_http(handler)  // Expose as HTTP endpoint
```

**Process/fiber introspection** following Erlang's model:

```indent
// Inspect any running process
let info = runtime.process_info(pid, [:memory, :message_queue_len, :current_function])

// List top memory consumers
let top_memory = runtime.proc_count(:memory, 10)

// Safe production tracing with automatic limits
runtime.trace_calls(
    module: "payment",
    function: "*",
    limit: 100,  // Stop after 100 trace messages
)

// OTP-style process state inspection
sys.get_state(registered_process)
```

**Signal-based debug triggers** (Unix pattern):

```indent
// Standard signal handlers
SIGUSR1 -> Write heap snapshot
SIGUSR2 -> Dump all fiber stacks
SIGHUP  -> Toggle debug logging
```

**Hot code reloading** for stateful services (Erlang model):

```indent
// Runtime supports two concurrent code versions
code.load_module("payment_processor")  // Load new version
code.soft_purge("payment_processor")   // Fail if processes running old code

// State migration callback for GenServer-style processes
trait HotReloadable {
    fn code_change(old_vsn: Version, state: State, extra: Any) -> Result<State>
}
```

## Health check primitives for Kubernetes

Health checks should be built-in with first-class support for the liveness/readiness/startup probe model.

```indent
// Core health types
enum HealthStatus { Up, Down, Degraded }

trait HealthCheck: Send + Sync {
    fn name() -> str
    fn check() -> HealthResult
    fn timeout() -> Duration { Duration::seconds(1) }
}

struct HealthResult {
    status: HealthStatus,
    message: Option<str>,
    data: Map<str, Value>,
    duration: Duration,
}
```

**Dependency aggregation with critical/non-critical distinction**:

```indent
let health = HealthRegistry::new()

// Critical dependencies: failure = DOWN
health.register_critical("database", DatabaseHealthCheck::new(pool))
health.register_critical("cache", RedisHealthCheck::new(client))

// Non-critical: failure = DEGRADED
health.register_non_critical("recommendation_service", HttpHealthCheck::new(url))

// Automatic aggregation logic:
// - Any critical DOWN -> aggregate DOWN
// - Any non-critical DOWN -> aggregate DEGRADED  
// - All UP -> aggregate UP
```

**Kubernetes-compatible endpoints**:

```indent
// Automatic HTTP endpoints
health.serve_http(router)

// Exposes:
// GET /health/live   -> Liveness (internal state only)
// GET /health/ready  -> Readiness (includes dependencies)
// GET /health/started -> Startup (initialization complete)

// Response format:
{
    "status": "UP",
    "checks": [
        {"name": "database", "status": "UP", "data": {"pool_size": 10}},
        {"name": "cache", "status": "UP", "data": {"latency_ms": 2}}
    ]
}
```

## Performance overhead and zero-cost patterns

A 2024 study measured distributed tracing overhead across frameworks: **Node.js showed 52-80% throughput reduction**, while Java and Go showed **19-39%**. These numbers are unacceptable for a language with built-in observability.

**Target overhead budgets**:

| Mode | CPU Overhead | Memory Overhead | Latency Impact |
|------|-------------|-----------------|----------------|
| Always-on tracing | <2% | <50MB per service | <5% P99 |
| Sampling enabled (10%) | <5% | <100MB | <10% P99 |
| Full tracing (debug) | <20% | <500MB | <50% P99 |
| Disabled | 0% | 0 | 0 |

**Compile-time elimination** achieves true zero cost:

```indent
// Feature flag approach (like Rust's cargo features)
// Build with: indent build --features=tracing

#[cfg(feature = "tracing")]
mod tracing_impl {
    pub fn trace_event(name: str) {
        fastrace::event!(name)
    }
}

#[cfg(not(feature = "tracing"))]
mod tracing_impl {
    #[inline(always)]
    pub fn trace_event(_name: str) {}  // Completely optimized away
}

// Without tracing feature: zero instructions, zero memory, zero binary size
```

**Level-based compile-time filtering**:

```indent
// Build configuration
[build]
max_log_level = "info"           // Development
release_max_log_level = "warn"   // Production

// Effect: all trace!() and debug!() calls removed from release binary
// Not just skipped at runtime—not present in compiled code
```

**Go 1.21's frame pointer improvements** reduced tracing overhead from **10-20%** to **<1%** by using frame pointer unwinding instead of runtime stack walking. Indent should adopt similar techniques from the start.

**Early enabled checks** prevent argument evaluation:

```indent
// Pattern: check before any work
if logger.enabled(Level::Debug) {
    let expensive_data = compute_expensive_summary()
    logger.debug("Summary", data: expensive_data)
}

// Better: lazy evaluation built into API
logger.debug("Summary", data: Lazy { compute_expensive_summary() })
```

**Sampling reduces overhead proportionally**:

```indent
// Head-based sampling at 1% = ~99% reduction in tracing overhead
let sampler = TraceIdRatioSampler::new(0.01)

// Tail-based sampling catches important traces without full overhead
let processor = TailSampling::new(
    policies: [
        StatusCodePolicy::new([Error]),         // Keep all errors
        LatencyPolicy::new(threshold_ms: 500),  // Keep slow requests
        ProbabilisticPolicy::new(rate: 0.01),   // 1% baseline
    ]
)
```

## OpenTelemetry and standards compatibility

Indent must produce observability data consumable by the existing ecosystem. OpenTelemetry compatibility is non-negotiable.

**Required exporters**:

| Signal | Required | Optional |
|--------|----------|----------|
| Traces | OTLP, Stdout | Zipkin, Jaeger |
| Metrics | OTLP, Prometheus | StatsD |
| Logs | OTLP, Stdout | JSON file, Loki |

**OTLP protocol support**:

```indent
// Configuration via environment variables (OTel standard)
OTEL_SERVICE_NAME=my-service
OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc  // or http/protobuf
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1

// Endpoints
// gRPC: collector:4317
// HTTP: collector:4318/v1/{traces,metrics,logs}
```

**Prometheus exposition format** for metrics scraping:

```
# TYPE http_requests_total counter
# HELP http_requests_total Total HTTP requests
http_requests_total{method="GET",status="200"} 1234

# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 100
http_request_duration_seconds_bucket{le="0.5"} 200
http_request_duration_seconds_bucket{le="+Inf"} 250
http_request_duration_seconds_sum 50.5
http_request_duration_seconds_count 250
# EOF
```

**Structured log format** (Elastic Common Schema compatible):

```json
{
    "@timestamp": "2026-01-17T10:30:00.000Z",
    "log.level": "INFO",
    "message": "Request processed",
    "service.name": "my-service",
    "trace.id": "0af7651916cd43dd8448eb211c80319c",
    "span.id": "b7ad6b7169203331",
    "http.request.method": "GET",
    "http.response.status_code": 200
}
```

## Recommended API surface

The complete observability API for Indent consolidates these primitives:

```indent
// Module: indent.telemetry

// === Tracing ===
@trace(level: Level, fields: Map<str, Value>)  // Function attribute
struct Span { ... }
struct Tracer { ... }
struct TracerProvider { ... }

// === Logging ===
enum Level { Trace, Debug, Info, Warn, Error }
struct Logger { ... }
trait Handler { ... }
macro trace!(msg, ...attrs)
macro debug!(msg, ...attrs)
macro info!(msg, ...attrs)
macro warn!(msg, ...attrs)
macro error!(msg, ...attrs)

// === Metrics ===
struct Counter<L: LabelSet> { ... }
struct Gauge<L: LabelSet> { ... }
struct Histogram<L: LabelSet> { ... }
struct MeterProvider { ... }
trait LabelSet { ... }

// === Health ===
enum HealthStatus { Up, Down, Degraded }
trait HealthCheck { ... }
struct HealthRegistry { ... }

// === Context ===
struct Context { ... }
struct Baggage { ... }
trait TextMapPropagator { ... }

// === Runtime Debugging ===
module runtime {
    fn process_info(pid: ProcessId, keys: List<Symbol>) -> ProcessInfo
    fn set_log_level(module: str, level: Level)
    fn trace_calls(module: str, function: str, limit: u32)
    fn heap_snapshot() -> HeapSnapshot
    fn cpu_profile(duration: Duration) -> CpuProfile
}
```

## Conclusion

Building observability into a programming language rather than bolting it on through libraries offers three fundamental advantages: **zero overhead when disabled** through compile-time elimination, **automatic instrumentation** without developer burden, and **consistent semantics** across all code in the ecosystem. The research across Go, Rust, Erlang/BEAM, and Java demonstrates that each approach has tradeoffs—Erlang's VM-level introspection provides the deepest visibility, Rust's macro system achieves true zero-cost abstractions, Go's pragmatic HTTP endpoints enable production debugging, and Java's bytecode manipulation offers maximum flexibility.

For Indent, the optimal design combines BEAM's philosophy (observability as a runtime primitive), Rust's execution (compile-time elimination and type safety), and Go's ergonomics (simple HTTP endpoints and sampling-based profiling). The key insight is that **observability overhead is acceptable when it's predictable and controllable**—targeting <2% for always-on tracing with compile-time opt-out makes observability a feature teams enable by default rather than disable for performance.