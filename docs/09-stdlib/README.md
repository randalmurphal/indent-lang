# Standard Library Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: MEDIUM - Defines what's "built-in"
**Key Question**: Batteries included (Go) or minimal core (Rust)?

## The Problem Space

We want:
- ✅ Enough built-in to be productive immediately
- ✅ Not so much that it's overwhelming
- ✅ High quality (no "this stdlib module is bad, use X instead")
- ✅ Consistent API design
- ✅ Good for backend/DevOps use cases

The tension: **Batteries included vs ecosystem flexibility**

## Philosophy Comparison

### Rust: Minimal Core

Rust stdlib provides:
- Primitives (Vec, String, HashMap, etc.)
- Core traits (Iterator, Display, etc.)
- Basic I/O
- Threading primitives

Everything else is ecosystem crates:
- HTTP: reqwest, hyper
- JSON: serde_json
- CLI: clap
- Async: tokio, async-std

**Pros**: Best-of-breed for each domain, evolves faster
**Cons**: Decision fatigue, dependency management, inconsistency

### Go: Batteries Included

Go stdlib provides:
- All of the above
- HTTP client AND server
- JSON, XML encoding
- Testing framework
- Crypto
- Templates
- Database/SQL interface

**Pros**: One answer, consistent, no deps for common tasks
**Cons**: Can't evolve independently, sometimes "good enough" not "great"

### Python: Everything and the Kitchen Sink

Python stdlib is huge:
- http, email, html, xml, json
- sqlite3, csv
- threading, multiprocessing
- tkinter (GUI!)
- unittest, logging
- And much more...

**Pros**: Incredibly complete
**Cons**: Some modules are outdated, quality varies

## Our Approach: **Targeted Batteries**

Include what backend developers need daily. Nothing more.

### Tier 1: Core (Always Available)

These are part of the language, not importable:

```
# Primitives
int, float, bool, str, rune

# Collections
list[T], dict[K, V], set[T], tuple

# Core types
Option[T], Result[T, E]

# Built-in functions
print, len, range, enumerate, zip
type, isinstance
```

### Tier 2: Standard Library (import required)

```
# Data formats
import json          # JSON encode/decode
import yaml          # YAML encode/decode
import csv           # CSV read/write
import toml          # TOML encode/decode

# Networking
import http          # HTTP client and server
import net           # Low-level networking
import url           # URL parsing

# File System
import fs            # File operations
import path          # Path manipulation
import glob          # Pattern matching

# Text
import strings       # String utilities
import regex         # Regular expressions
import fmt           # Formatting

# Time
import time          # Time and duration
import datetime      # Date/time handling

# Crypto
import crypto.hash   # SHA, MD5, etc.
import crypto.rand   # Secure random
import crypto.tls    # TLS configuration

# Encoding
import base64        # Base64 encode/decode
import hex           # Hex encode/decode

# System
import os            # OS interface
import env           # Environment variables
import process       # Process management
import signal        # Signal handling

# Concurrency (maybe core?)
import channel       # Channels
import sync          # Synchronization primitives

# Testing
import testing       # Test framework

# Logging
import log           # Structured logging

# CLI
import args          # Argument parsing
import cli           # CLI framework
```

### Tier 3: Extended (Separate packages, maintained by us)

```
# Databases
import sql           # SQL interface
import sql.postgres  # PostgreSQL driver
import sql.mysql     # MySQL driver
import sql.sqlite    # SQLite driver

# Cloud
import cloud.aws     # AWS SDK
import cloud.gcp     # GCP SDK
import cloud.azure   # Azure SDK

# Observability
import metrics       # Metrics collection
import tracing       # Distributed tracing

# Kubernetes
import k8s           # Kubernetes client
```

## Detailed Module Designs

### `http` Module

Most critical for backend work:

```
import http

# Client - Simple
response = http.get("https://api.example.com/users")
print(response.status)
print(response.json())

# Client - With options
response = http.get(
    "https://api.example.com/users",
    headers={"Authorization": f"Bearer {token}"},
    timeout=30.seconds
)

# Client - POST
response = http.post(
    "https://api.example.com/users",
    json={"name": "Alice", "email": "alice@example.com"}
)

# Client - Reusable with connection pooling
client = http.Client(
    base_url="https://api.example.com",
    timeout=30.seconds,
    headers={"User-Agent": "MyApp/1.0"}
)
users = client.get("/users").json()
client.close()

# Server
server = http.Server()

@server.route("GET", "/users")
fn list_users(req: http.Request) -> http.Response:
    users = db.get_users()
    return http.Response.json(users)

@server.route("POST", "/users")
fn create_user(req: http.Request) -> http.Response:
    data = req.json()
    user = db.create_user(data)
    return http.Response.json(user, status=201)

@server.route("GET", "/users/:id")
fn get_user(req: http.Request) -> http.Response:
    id = req.params["id"]
    match db.get_user(id):
        case Some(user):
            return http.Response.json(user)
        case None:
            return http.Response(status=404)

server.run(":8080")
```

### `json` Module

```
import json

# Parse
data = json.parse('{"name": "Alice", "age": 30}')
name = data["name"]

# Parse into type
type User:
    name: str
    age: int

user = json.parse[User]('{"name": "Alice", "age": 30}')

# Stringify
text = json.stringify({"name": "Alice", "age": 30})
text = json.stringify(user)
text = json.stringify(data, indent=2)  # Pretty print

# Streaming (large files)
for item in json.stream_array(file):
    process(item)
```

### `fs` Module

```
import fs

# Reading
content = fs.read("file.txt")                    # Read as string
bytes = fs.read_bytes("file.bin")                # Read as bytes
lines = fs.read_lines("file.txt")                # Iterator of lines

# Writing
fs.write("file.txt", "content")
fs.write_bytes("file.bin", bytes)
fs.append("log.txt", "new line\n")

# File info
exists = fs.exists("file.txt")
info = fs.stat("file.txt")
print(info.size, info.modified)

# Directories
files = fs.list_dir("path")
fs.mkdir("new_dir")
fs.mkdir_all("deep/nested/path")
fs.remove("file.txt")
fs.remove_all("directory")

# Temporary
tmp = fs.temp_file()
tmp_dir = fs.temp_dir()

# Watching
watcher = fs.watch("directory")
for event in watcher:
    match event:
        case Created(path): print(f"Created: {path}")
        case Modified(path): print(f"Modified: {path}")
        case Deleted(path): print(f"Deleted: {path}")
```

### `time` Module

```
import time

# Current time
now = time.now()
utc = time.now_utc()

# Duration
d = 5.seconds
d = 100.milliseconds
d = 2.hours + 30.minutes

# Sleep
time.sleep(1.second)

# Parsing/formatting
t = time.parse("2024-01-15T10:30:00Z", time.RFC3339)
s = t.format(time.RFC3339)
s = t.format("%Y-%m-%d %H:%M:%S")

# Arithmetic
tomorrow = now + 1.day
diff = end - start
print(f"Took {diff.seconds}s")

# Timers
timer = time.timer(5.seconds)
select:
    case from timer:
        print("Timer fired")

# Tickers (repeating)
ticker = time.ticker(1.second)
for _ in ticker:
    print("Tick")
```

### `log` Module

Structured logging by default:

```
import log

# Basic
log.info("Server started")
log.error("Connection failed")

# Structured
log.info("User logged in", user_id=123, ip="1.2.3.4")
log.error("Request failed", status=500, path="/api/users", duration=1.23)

# Output (JSON by default for machine parsing)
# {"level":"info","msg":"User logged in","user_id":123,"ip":"1.2.3.4","time":"..."}

# Configuration
log.set_level(log.DEBUG)
log.set_format(log.TEXT)  # Human-readable
log.set_output(file)

# With context (for request tracing)
logger = log.with_context(request_id="abc123")
logger.info("Processing request")
```

### `testing` Module

```
import testing

# Test function
@test
fn test_addition():
    assert 1 + 1 == 2
    assert_eq(2 + 2, 4)
    assert_ne(1, 2)

# Test with setup/teardown
@test
fn test_database():
    db = testing.setup(create_test_db)
    defer testing.teardown(db, cleanup_db)

    db.insert({"name": "Alice"})
    assert_eq(db.count(), 1)

# Table-driven tests
@test
fn test_parse_int():
    cases = [
        ("42", 42),
        ("-1", -1),
        ("0", 0),
    ]
    for input, expected in cases:
        assert_eq(parse_int(input), expected)

# Benchmarks
@bench
fn bench_json_parse():
    data = '{"name": "Alice", "age": 30}'
    for _ in testing.b:
        json.parse(data)

# Property-based testing
@property
fn prop_reverse_twice(items: list[int]):
    assert items.reverse().reverse() == items
```

### `args` Module

```
import args

# Simple
parser = args.Parser(
    name="myapp",
    description="Does something useful"
)
parser.add("--verbose", "-v", type=bool, help="Enable verbose output")
parser.add("--output", "-o", type=str, default="out.txt", help="Output file")
parser.add("input", type=str, help="Input file")

opts = parser.parse()
print(opts.verbose, opts.output, opts.input)

# Subcommands
parser = args.Parser(name="git")
add = parser.subcommand("add", help="Add files")
add.add("files", type=list[str], help="Files to add")

commit = parser.subcommand("commit", help="Commit changes")
commit.add("-m", "--message", type=str, required=true)

opts = parser.parse()
match opts.command:
    case "add":
        print(f"Adding {opts.files}")
    case "commit":
        print(f"Committing: {opts.message}")
```

## API Design Principles

### 1. Consistency

All modules follow same patterns:

```
# Creation
x = Module.new(...)
x = Module.from_string(...)
x = Module.from_bytes(...)

# Conversion
s = x.to_string()
b = x.to_bytes()

# Serialization
json.stringify(x)
json.parse[X](s)
```

### 2. Errors are Explicit

```
# Functions that can fail return Result
content = fs.read("file.txt")?

# Or Option for lookups
value = dict.get("key") else default
```

### 3. Sensible Defaults

```
# Works out of the box
http.get(url)

# Customizable when needed
http.get(url, timeout=30.seconds, headers={...})
```

### 4. Zero or Low Allocation in Hot Paths

```
# Buffer reuse
buffer = Buffer.new(4096)
for chunk in reader.read_into(buffer):
    process(chunk)

# String building
sb = strings.Builder()
for item in items:
    sb.write(item.to_string())
result = sb.build()
```

## What We Explicitly DON'T Include

| Omission | Reason | Use Instead |
|----------|--------|-------------|
| GUI | Not our domain | External libraries |
| Email | Complex, evolving | External package |
| Image processing | Large dependency | External package |
| Machine learning | Huge, specialized | Python interop |
| Compression | Except gzip | External for others |
| Full SQL drivers | Too many DBs | sql package ecosystem |

## Versioning Strategy

Standard library is versioned with the language:

```
lang 1.0 → stdlib 1.0
lang 1.1 → stdlib 1.1

# Stability guarantee
# stdlib 1.x will not break code from 1.0
```

For extended packages (tier 3), semantic versioning independently:

```
import sql.postgres  # version managed separately
```

## Open Questions

1. **Async in stdlib?** Are http/fs async-aware or blocking?
2. **Encoding/Decoding traits?** Standard way to serialize types?
3. **Iterator vs Stream?** Sync vs async iteration?
4. **Module organization?** Flat (http) or hierarchical (net.http)?
5. **How much crypto?** Just hashing or full TLS?

## Next Steps

1. [ ] Design core module APIs in detail
2. [ ] Prototype http module
3. [ ] Design encoding/decoding traits
4. [ ] Survey: what do backend devs need most?
5. [ ] Benchmark against Go stdlib

---

## Research Links

- [Go Standard Library](https://pkg.go.dev/std)
- [Rust Standard Library](https://doc.rust-lang.org/std/)
- [Python Standard Library](https://docs.python.org/3/library/)
- [Deno Standard Library](https://deno.land/std)

---

*Last updated: 2024-01-17*
*Decision: Targeted batteries (Go-lite) (tentative)*
