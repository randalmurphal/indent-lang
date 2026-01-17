# Configuration and build system design for Indent

**Indent should adopt a pragmatic hybrid approach**: combining Python's secret-safe environment handling, Rust's declarative TOML-based configuration, Go's cross-compilation simplicity, and a streamlined profile system optimized for DevOps workflows. The unifying principle is **transparency over magic**—DevOps engineers replacing bash scripts need predictable, debuggable tooling that doesn't hide complexity behind opaque build systems.

## Environment configuration: safety-first API design

The most critical insight from analyzing Rust's envy, Go's envconfig, and Python's pydantic-settings is that **secret handling must be a first-class language feature**. DevOps scripts regularly handle credentials, API keys, and tokens—accidentally logging these causes real security incidents.

Python's `SecretStr` type stands out as the model to follow. It automatically masks values in string representations and logs, requiring explicit `.get_secret_value()` calls to access the underlying data. Neither Rust's envy nor Go's envconfig provide this natively, forcing developers to implement custom wrapper types. For a language targeting DevOps, built-in secret safety is non-negotiable.

The recommended Indent environment API combines three ecosystem strengths:

```indent
env Config {
    # Required with validation - fails fast at startup
    port: int @env("PORT") @required @range(1024, 65535)
    
    # Optional with sensible default
    host: string @default("0.0.0.0")
    
    # Secret type - automatically masked in logs and debug output
    db_password: secret @env("DB_PASSWORD") @required
    api_key: secret @env("API_KEY")
    
    # Duration parsing from human-readable strings
    timeout: duration @default("30s")
    
    # List from comma-separated values
    allowed_origins: string[] @delimiter(",")
    
    # Nested config with prefix for namespacing
    database: DatabaseConfig @prefix("DB_")
}

# Usage: single-line loading with aggregate validation
let config = Config.load()  # Validates all required vars, reports all errors
print(config.db_password)   # Output: **********
print(config.db_password.reveal())  # Actual value when needed
```

**Key design decisions**: The `@env()` annotation provides explicit variable naming (matching bash conventions), while `@required` and `@default()` make intent immediately readable. Validation attributes like `@range()` catch configuration errors at startup rather than runtime—the "fail fast" pattern that prevents half-initialized services.

For tooling integration, Indent should generate `.env.example` files automatically from config definitions via `indent env:template`, enabling both documentation and validation with `indent env:check`.

## Configuration format: TOML with layered precedence

TOML emerges as the clear winner for Indent's configuration format. The decision isn't about syntax preferences—it's about **safety and predictability** in production environments.

YAML's implicit type coercion creates real bugs: the country code `NO` parses as boolean `false`, version strings like `1.10` become floats, and `on`/`off` become booleans unexpectedly. The "Norway problem" has caused actual production incidents. YAML's 86-page specification and security vulnerabilities (billion laughs DoS, code execution in unsafe parsers) add complexity DevOps engineers shouldn't need to manage.

JSON lacks comments—a dealbreaker for configuration files that need inline documentation. TOML provides explicit typing (strings must be quoted, booleans are `true`/`false` literals), native datetime support, and a simple specification that enables reliable parsers.

| Feature | TOML | YAML | JSON |
|---------|------|------|------|
| **Comments** | ✓ `#` style | ✓ `#` style | ✗ None |
| **Type safety** | ✓ Explicit | ✗ Implicit coercion | ✓ Explicit |
| **Security** | ✓ Safe | ✗ Vulnerable | ✓ Safe |
| **Spec complexity** | Simple | 86 pages | Simple |
| **DevOps adoption** | Cargo, pyproject.toml | K8s, Ansible | package.json |

The layered configuration system should follow the standard precedence pattern with one addition—per-environment config files:

```
1. Built-in defaults (Indent stdlib)
2. System config (/etc/indent/config.toml)
3. User config (~/.config/indent/config.toml)
4. Project config (./indent.toml)
5. Environment-specific (./indent.production.toml)
6. Environment variables (INDENT_* prefix)
7. CLI flags (highest priority)
```

Environment variable mapping uses double underscores for nested keys: `INDENT_DATABASE__HOST` maps to `[database] host`. This convention, borrowed from Python's dynaconf and Rust's config crate, enables flat environment variables to override deeply nested configuration.

Hot reloading is essential for development (`indent run --watch`) but should be opt-in for production. The implementation must handle atomic updates (debounce file changes by **150-300ms**), validate before applying, and rollback on validation failure. Support `SIGHUP` for explicit reload triggering in production services.

## Build profiles: three-tier system with inheritance

Cargo provides the most sophisticated profile system, but its complexity isn't necessary for DevOps tooling. Go's approach (flags only) is too minimal for managing consistent build configurations. The right balance is **three built-in profiles with optional inheritance** for customization.

```toml
# indent.toml - Built-in profile defaults

[profile.dev]
optimize = false
debug-info = "full"
assertions = true
incremental = true  # Fast rebuilds during development

[profile.release]
optimize = "size"   # DevOps tools should be small for containers
debug-info = false
strip = true
static = true       # Self-contained binaries
assertions = false

[profile.test]
inherits = "dev"
coverage = true
```

The key insight for DevOps tools: **optimize for size in release builds**. Container image sizes directly impact deployment times and storage costs. Cargo defaults to `opt-level = 3` (maximum speed), but DevOps CLIs rarely need maximum performance—they need minimal footprint.

Custom profiles require explicit inheritance:

```toml
[profile.docker]
inherits = "release"
target = "linux-musl-amd64"  # Alpine-friendly

[profile.ci]
inherits = "release"
strip = false        # Keep symbols for debugging CI failures
debug-info = "line-tables"

[profile.profiling]
inherits = "release"
debug-info = "full"  # Enable profiler symbolication
strip = false
```

Dependency categorization follows Cargo's proven model:

```toml
[dependencies]
yaml = "1.0"
http = "2.0"

[dev-dependencies]     # Tests and benchmarks only
test-runner = "1.0"
mock-http = "0.5"

[build-dependencies]   # Build scripts only
embed = "0.5"
```

## Conditional compilation: hybrid runtime and compile-time

The fundamental question—compile-time conditionals like Rust's `#[cfg()]` versus runtime checks like Python's `sys.platform`—has no single right answer. The recommendation for Indent is a **hybrid approach** prioritizing simplicity.

**Runtime conditionals should be the default** for most DevOps use cases. Platform-specific command execution, path handling, and environment detection don't benefit from compile-time elimination—the overhead of a runtime branch is negligible, and the debugging experience is vastly simpler.

```indent
# Runtime checks - clear, debuggable, familiar to bash users
let package_manager = match system.os {
    "linux" => detect_linux_package_manager(),
    "darwin" => "brew",
    "windows" => "choco",
    _ => error("Unsupported OS: {system.os}")
}

# Available system variables
system.os        # "linux", "darwin", "windows"
system.arch      # "x86_64", "arm64"
system.hostname
system.username
```

**Compile-time conditionals are warranted** for three specific cases: removing unused platform implementations from binaries, eliminating optional feature code (cloud provider SDKs), and cross-compilation where target APIs differ fundamentally.

```indent
# File-level target selection (Go-style)
# network_linux.indent, network_darwin.indent, network_windows.indent
@target(os: "linux")
fn get_network_interfaces() -> []Interface {
    # Linux-specific implementation using /sys/class/net
}

# Feature flags for optional dependencies
@feature("aws")
fn deploy_to_s3(bucket: string, file: path) {
    # Only compiled when aws feature enabled
}
```

Feature declaration in the manifest:

```toml
[features]
default = []
aws = ["dep:aws-sdk"]
gcp = ["dep:gcp-sdk"]
kubernetes = ["dep:kubectl"]
enterprise = ["aws", "gcp"]  # Feature groups
```

**Testing strategy**: The combinatorial explosion of feature combinations makes exhaustive testing impractical (10 boolean flags = 1,024 combinations). The pragmatic approach is testing the default configuration thoroughly, each feature individually, and known high-risk combinations—not every permutation.

## Cross-compilation: Go's simplicity wins

Go's cross-compilation experience—`GOOS=linux GOARCH=amd64 go build`—represents the gold standard for DevOps tooling. Rust's target triple system (`x86_64-unknown-linux-musl`) is more precise but adds cognitive overhead that doesn't benefit most use cases.

Indent's cross-compilation should mirror Go's simplicity:

```bash
# Simple environment-style flags
indent build --os=linux --arch=amd64
indent build --os=darwin --arch=arm64

# Build all configured targets
indent build --all-targets

# List available targets
indent targets list
```

Target configuration in the manifest enables reproducible multi-platform builds:

```toml
[targets]
linux-amd64 = { os = "linux", arch = "amd64" }
linux-arm64 = { os = "linux", arch = "arm64" }
darwin-arm64 = { os = "darwin", arch = "arm64" }
windows-amd64 = { os = "windows", arch = "amd64" }

[target.linux-musl]
static = true
strip = true

[target.windows]
features = ["gui"]  # Windows-specific features
```

Per-target dependencies handle platform-specific libraries:

```toml
[target.'cfg(windows)'.dependencies]
windows-service = "1.0"

[target.'cfg(unix)'.dependencies]
daemonize = "0.5"
```

**Critical for DevOps**: Default to static linking in release builds. Dynamic dependencies cause deployment failures when target systems lack required libraries. The musl libc target (`linux-musl-*`) produces fully static binaries ideal for Alpine-based containers.

## Build scripts: explicit generation over implicit magic

The philosophical divide between Rust's `build.rs` (runs automatically on every build) and Go's `go generate` (explicit invocation by developer) has significant implications for DevOps tooling.

**Recommendation: Follow Go's explicit model.** Generated files should be committed to the repository, making builds reproducible without requiring generator tools on every developer's machine. This matters enormously for DevOps teams where build environments vary and debugging CI failures shouldn't require understanding code generation pipelines.

```indent
// Source file directive - explicit code generation
//indent:generate protoc --indent_out=. service.proto
//indent:generate indent-codegen openapi api.yaml > gen/client.indent
```

Or configured in the manifest:

```toml
[generate]
proto = "protoc --indent_out=gen/ proto/*.proto"
schema = "indent-codegen schema.json > gen/types.indent"
```

The `indent generate` command runs all configured generators:

```bash
indent generate        # Run all generators
indent generate proto  # Run specific generator
```

**Build hooks for advanced use cases** should be simple shell commands or Indent scripts:

```toml
[hooks]
pre-build = ["indent generate"]
post-build = ["strip $OUTPUT", "./sign-binary.sh"]
pre-test = ["./setup-test-db.sh"]
```

This keeps the build system transparent—every step is visible, every command is debuggable with standard tools.

## Design principles synthesized from three ecosystems

The analysis reveals consistent lessons across Rust, Go, and Python that should guide Indent's design:

**From Rust**: Declarative configuration (TOML), typed dependency management, profile inheritance, and the principle that configuration should be validated at load time. Cargo's design shows how rich tooling can remain approachable.

**From Go**: Cross-compilation simplicity, explicit code generation, single-binary distribution philosophy, and the value of "zero configuration" defaults. Go demonstrates that powerful tooling doesn't require complex setup.

**From Python**: Secret-safe types (SecretStr), runtime flexibility, optional dependency groups, and the importance of human-readable error messages. Pydantic shows how validation can be both rigorous and developer-friendly.

**Overarching principle for DevOps**: The build and configuration system should be **as transparent as a shell script**. DevOps engineers debug production systems at 3 AM—they need tools that expose what's happening, not tools that hide complexity behind abstractions. Every configuration option should have obvious effects. Every build step should be reproducible with standard commands. Every error message should point to actionable fixes.

The proposed design prioritizes production reliability over development convenience, explicit configuration over implicit defaults, and debuggability over brevity. For a language replacing bash scripts, these tradeoffs align with what DevOps engineers actually need.