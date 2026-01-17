# Cloud-Native Features Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: MEDIUM - Differentiator for backend focus
**Key Question**: What cloud-native patterns should be built into the language?

## The Problem Space

We want:
- ✅ First-class Kubernetes support
- ✅ Easy health checks / readiness probes
- ✅ Graceful shutdown
- ✅ Configuration from environment
- ✅ Structured logging / observability
- ✅ Service mesh friendly

The insight: **These patterns are boilerplate in every language. Make them default.**

## What is "Cloud-Native"?

Cloud-native applications are designed for:
1. **Containerization**: Run in Docker/containerd
2. **Orchestration**: Managed by Kubernetes
3. **Microservices**: Decomposed, communicate over network
4. **Observability**: Metrics, logs, traces
5. **Resilience**: Handle failures gracefully

## Current Pain Points

### Go (Pretty Good)

```go
// Graceful shutdown in Go - boilerplate every time
func main() {
    srv := &http.Server{Addr: ":8080", Handler: handler}

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal(err)
    }
}
```

### Python

```python
# Even more boilerplate
import signal
import sys
from http.server import HTTPServer

server = HTTPServer(('', 8080), Handler)

def shutdown(signum, frame):
    server.shutdown()
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown)
signal.signal(signal.SIGINT, shutdown)

server.serve_forever()
```

### Our Goal

```
import http

fn main():
    server = http.Server(":8080")
    server.route("GET", "/", handler)
    server.run()  # Graceful shutdown built-in
```

## Built-in Cloud-Native Features

### 1. Graceful Shutdown

**Default behavior**: Servers handle SIGTERM/SIGINT gracefully.

```
import http

fn main():
    server = http.Server(":8080")
    server.route("GET", "/", handler)

    # On SIGTERM/SIGINT:
    # 1. Stop accepting new connections
    # 2. Wait for in-flight requests (default 30s)
    # 3. Exit cleanly
    server.run()

# Custom shutdown timeout
server.run(shutdown_timeout=60.seconds)

# Custom shutdown handler
server.on_shutdown(fn():
    db.close()
    cache.flush()
)
```

**Under the hood**:
```
# What the runtime does automatically
signal.handle([SIGTERM, SIGINT], fn():
    server.drain()
    server.wait_idle(timeout)
    exit(0)
)
```

### 2. Health Checks

Built-in health check endpoints:

```
import http

fn main():
    server = http.Server(":8080")

    # Automatic endpoints:
    # GET /healthz   - Liveness probe
    # GET /readyz    - Readiness probe

    # Custom health checks
    server.health.add_check("database", fn() -> bool:
        return db.ping()
    )

    server.health.add_check("cache", fn() -> bool:
        return cache.ping()
    )

    server.run()
```

**Kubernetes deployment**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 3. Configuration

Type-safe configuration from environment:

```
import config

# Define config schema
type Config:
    @env("DATABASE_URL")
    database_url: str

    @env("PORT")
    @default(8080)
    port: int

    @env("DEBUG")
    @default(false)
    debug: bool

    @env("ALLOWED_ORIGINS")
    @default([])
    allowed_origins: list[str]

# Load and validate
cfg = config.load[Config]()

# Errors are clear
# Error: DATABASE_URL is required but not set
# Error: PORT must be an integer, got "abc"
```

**Advanced patterns**:
```
import config

# From file with env override
cfg = config.load[Config](
    file="config.yaml",
    env_prefix="MYAPP_"
)

# ConfigMaps / Secrets
cfg = config.load[Config](
    file="/etc/config/settings.yaml",
    secrets_file="/etc/secrets/credentials.yaml"
)

# Watch for changes (ConfigMap reload)
config.watch(cfg, fn(new_cfg):
    update_settings(new_cfg)
)
```

### 4. Structured Logging

JSON logging by default, human-readable in development:

```
import log

# Automatic context
log.info("Request handled",
    method="GET",
    path="/users",
    duration=0.023,
    status=200
)

# Output (production):
# {"level":"info","msg":"Request handled","method":"GET","path":"/users","duration":0.023,"status":200,"time":"2024-01-15T10:30:00Z"}

# Output (development):
# 10:30:00 INFO Request handled method=GET path=/users duration=0.023 status=200

# Request context (automatic trace propagation)
fn handle_request(req: http.Request) -> http.Response:
    # Log automatically includes request_id, trace_id
    log.info("Processing request")
    ...
```

**Log levels**:
```
log.debug("Verbose debugging")
log.info("Normal operation")
log.warn("Something unexpected")
log.error("Something failed")

# Configure level from env
# LOG_LEVEL=debug
```

### 5. Metrics

Built-in Prometheus metrics:

```
import metrics

# Automatic metrics:
# - http_requests_total
# - http_request_duration_seconds
# - http_requests_in_flight

# Custom metrics
request_counter = metrics.counter(
    "myapp_requests_total",
    "Total requests processed",
    labels=["endpoint", "status"]
)

request_counter.inc(endpoint="/users", status="success")

# Gauges
active_connections = metrics.gauge(
    "myapp_connections_active",
    "Active connections"
)

active_connections.set(42)

# Histograms
response_time = metrics.histogram(
    "myapp_response_time_seconds",
    "Response time distribution",
    buckets=[0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

response_time.observe(0.023)

# Endpoint exposed automatically at /metrics
```

### 6. Distributed Tracing

OpenTelemetry integration:

```
import tracing

# Automatic HTTP tracing
fn handle_request(req: http.Request) -> http.Response:
    # Trace context extracted from headers automatically

    # Create child span
    tracing.span("fetch_user"):
        user = db.get_user(id)

    tracing.span("process_data"):
        result = process(user)

    return http.Response.json(result)
```

**Propagation**:
```
# Outgoing requests include trace context
response = http.get("http://other-service/api", headers=tracing.headers())

# Or automatic with traced client
client = http.Client(traced=true)
response = client.get("http://other-service/api")
```

### 7. Circuit Breaker

Built-in circuit breaker for resilience:

```
import resilience

# Create circuit breaker
breaker = resilience.circuit_breaker(
    name="external_api",
    failure_threshold=5,
    recovery_timeout=30.seconds
)

fn call_external_api() -> Data | Error:
    match breaker.call(fn(): http.get("http://api.example.com")):
        case Ok(response):
            return parse(response)
        case Err(CircuitOpen):
            return cached_fallback()
        case Err(e):
            return Err(e)

# Or with decorator
@resilience.circuit_breaker(failure_threshold=5)
fn call_external_api() -> Data | Error:
    response = http.get("http://api.example.com")?
    return parse(response)
```

### 8. Retry with Backoff

```
import resilience

# Retry with exponential backoff
result = resilience.retry(
    fn(): http.get(url),
    max_attempts=3,
    backoff=resilience.exponential(base=1.second, max=30.seconds)
)

# With decorator
@resilience.retry(max_attempts=3)
fn fetch_data() -> Data | Error:
    return http.get(url)?.json()
```

### 9. Context Propagation

Request context flows through the application:

```
import context

fn handle_request(req: http.Request) -> http.Response:
    # Context automatically includes:
    # - request_id
    # - trace_id
    # - user info (if authenticated)
    # - deadline (if timeout set)

    # Access context anywhere in call chain
    request_id = context.get("request_id")

    # Set custom values
    context.set("user_id", user.id)

    # Context flows to spawned tasks
    concurrent:
        spawn process_async()  # Has same context

    return response
```

### 10. Kubernetes Client

First-class Kubernetes support:

```
import k8s

fn main():
    # Automatic in-cluster config
    client = k8s.Client()

    # Or explicit config
    client = k8s.Client(config="~/.kube/config")

    # List pods
    pods = client.pods.list(namespace="default")
    for pod in pods:
        print(f"{pod.name}: {pod.status}")

    # Watch for changes
    for event in client.pods.watch(namespace="default"):
        match event:
            case Added(pod):
                print(f"Pod added: {pod.name}")
            case Modified(pod):
                print(f"Pod modified: {pod.name}")
            case Deleted(pod):
                print(f"Pod deleted: {pod.name}")

    # Create resources
    client.configmaps.create(k8s.ConfigMap(
        name="myconfig",
        data={"key": "value"}
    ))
```

## Application Lifecycle

### Standard Lifecycle Hooks

```
import app

# Define application with lifecycle
application = app.Application()

@application.on_startup
fn startup():
    db.connect()
    cache.connect()
    log.info("Application started")

@application.on_shutdown
fn shutdown():
    db.close()
    cache.close()
    log.info("Application stopped")

@application.on_ready
fn ready():
    # Called when ready to receive traffic
    log.info("Application ready")

fn main():
    server = http.Server(":8080")
    server.route("GET", "/", handler)
    application.run(server)
```

### Container-Aware Behavior

```
import runtime

# Detect environment
if runtime.in_kubernetes():
    # Configure for k8s
    log.set_format(log.JSON)
else:
    # Local development
    log.set_format(log.TEXT)

# Respect resource limits
available_memory = runtime.memory_limit()
pool_size = available_memory / per_connection_memory

# CPU limits
num_workers = runtime.cpu_limit()
```

## Service Mesh Integration

Works seamlessly with Istio, Linkerd, etc.:

```
# Standard headers respected:
# x-request-id
# x-b3-traceid (Zipkin)
# traceparent (W3C)

# Automatic propagation to outgoing requests
client = http.Client()
response = client.get("http://service-b/api")
# ^ Includes all trace headers
```

## 12-Factor App Compliance

| Factor | Our Support |
|--------|-------------|
| 1. Codebase | Standard, use git |
| 2. Dependencies | Package manifest |
| 3. Config | `config` module, env vars |
| 4. Backing services | Treat as attached resources |
| 5. Build, release, run | `build` vs `run` distinction |
| 6. Processes | Stateless by default |
| 7. Port binding | `http.Server(":8080")` |
| 8. Concurrency | Structured concurrency |
| 9. Disposability | Graceful shutdown built-in |
| 10. Dev/prod parity | Same binary everywhere |
| 11. Logs | Structured logging |
| 12. Admin processes | Run as one-off scripts |

## Example: Complete Cloud-Native App

```
import http
import log
import config
import metrics
import tracing
import k8s

type Config:
    @env("PORT")
    @default(8080)
    port: int

    @env("DATABASE_URL")
    database_url: str

    @env("LOG_LEVEL")
    @default("info")
    log_level: str

fn main():
    # Load config
    cfg = config.load[Config]()

    # Configure logging
    log.set_level(cfg.log_level)

    # Connect to database
    db = sql.connect(cfg.database_url)
    defer db.close()

    # Create server
    server = http.Server(f":{cfg.port}")

    # Health checks
    server.health.add_check("database", fn(): db.ping())

    # Routes
    server.route("GET", "/users", fn(req):
        tracing.span("list_users"):
            users = db.query("SELECT * FROM users")
            metrics.counter("users_listed").inc()
            return http.Response.json(users)
    )

    # Start with graceful shutdown
    log.info("Starting server", port=cfg.port)
    server.run()
```

## Open Questions

1. **How deep into k8s?** Just client, or also CRD generation?
2. **Service mesh specifics?** Istio/Linkerd-specific features?
3. **Secrets handling?** Vault integration? AWS Secrets Manager?
4. **Rate limiting?** Built-in or library?
5. **API gateway patterns?** Authentication middleware?

## Next Steps

1. [ ] Design health check API in detail
2. [ ] Design config loading semantics
3. [ ] Prototype metrics integration
4. [ ] Test with real k8s deployments
5. [ ] Document 12-factor compliance

---

## Research Links

- [12-Factor App](https://12factor.net/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Prometheus Exposition Format](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [Go Cloud Development Kit](https://gocloud.dev/)

---

*Last updated: 2024-01-17*
*Decision: Batteries-included for cloud-native (tentative)*
