# Scripting Mode Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: MEDIUM - Important for DevOps use case
**Key Question**: How do we make a compiled language feel like a scripting language?

## The Problem Space

We want:
- ✅ Instant startup for small scripts (< 100ms)
- ✅ No explicit compile step for simple use
- ✅ REPL for exploration
- ✅ Can grow from script to application seamlessly
- ✅ Same language, not a subset

The goal: **Replace bash scripts with something maintainable and fast**.

## Use Cases

### DevOps Scripts

```bash
# Current: bash script for deployment
#!/bin/bash
kubectl get pods -n prod | grep -v Running | awk '{print $1}' | xargs kubectl delete pod
```

```
# Desired: our language
#!/usr/bin/env ourlang
pods = k8s.get_pods(namespace="prod")
failing = [p for p in pods if p.status != "Running"]
for pod in failing:
    k8s.delete_pod(pod.name)
    print(f"Deleted {pod.name}")
```

### One-off Data Processing

```
#!/usr/bin/env ourlang
# Process a CSV file
import csv

data = csv.read("input.csv")
filtered = [row for row in data if row["status"] == "active"]
csv.write("output.csv", filtered)
print(f"Wrote {len(filtered)} rows")
```

### Quick API Testing

```
#!/usr/bin/env ourlang
import http
import json

response = http.get("https://api.example.com/users")
users = json.parse(response.body)
for user in users[:5]:
    print(f"{user['name']}: {user['email']}")
```

## Execution Models

### 1. AOT (Ahead-of-Time) Compilation

**How it works**: Compile to binary, then run.

```bash
$ ourlang build script.ol
$ ./script
```

**Pros**:
- Best runtime performance
- No runtime dependency
- Can distribute binary

**Cons**:
- Compile step adds latency
- Poor for edit-run cycles
- Overkill for small scripts

### 2. JIT (Just-in-Time) Compilation

**How it works**: Compile to machine code at runtime, cache the result.

```bash
$ ourlang run script.ol  # Compiles + runs
```

Behind the scenes:
1. Check if cached compiled version exists
2. If not (or source changed), compile to native
3. Execute compiled code
4. Cache for next time

**Pros**:
- Fast repeat execution (cached)
- No explicit build step
- Optimal runtime performance

**Cons**:
- First run has compilation latency
- Caching adds complexity
- Still noticeable latency for small scripts

### 3. Bytecode Interpreter

**How it works**: Compile to bytecode, interpret.

```bash
$ ourlang run script.ol  # Compile to bytecode, interpret
```

**Pros**:
- Fast startup (bytecode compiles quickly)
- Simple implementation
- Portable bytecode

**Cons**:
- Slower execution than native
- Another compilation target to maintain
- Not our performance goal

### 4. Tiered Compilation

**How it works**: Start interpreted/bytecode, compile hot paths.

```
First run:    Quick bytecode compilation, interpret
Hot paths:    Detected, compiled to native
Later runs:   Native code cached
```

**Pros**:
- Fast startup
- Achieves native performance eventually
- Smart resource usage

**Cons**:
- Complex implementation
- Inconsistent performance during warmup
- Memory overhead for multiple representations

### 5. Fast AOT with Caching

**How it works**: Very fast compilation (Cranelift), aggressive caching.

```bash
$ ourlang run script.ol
# 1. Hash source file
# 2. Check cache for compiled version
# 3. If miss: compile with Cranelift (fast), cache
# 4. Run compiled code
```

**Target latency**:
- Cache hit: < 10ms
- Cache miss (small script): < 100ms
- Cache miss (medium script): < 500ms

**Pros**:
- Always native performance
- Single compilation pipeline
- Simple mental model

**Cons**:
- First run of new script has latency
- Cache management needed

## Our Recommended Approach

**Fast AOT with aggressive caching** (option 5)

Rationale:
1. Cranelift can compile fast enough for scripting
2. Same code path as "build" mode
3. Native performance always
4. Caching makes repeat runs instant
5. No interpreter to maintain

### Implementation Details

```
~/.cache/ourlang/
  scripts/
    a1b2c3d4/           # Hash of script source + dependencies
      script.native     # Compiled binary
      metadata.json     # Compile time, deps, etc.
```

```
fn run_script(path: str):
    source = read_file(path)
    deps = resolve_dependencies(source)
    cache_key = hash(source, deps, compiler_version)

    if cached = cache.get(cache_key):
        exec(cached)
    else:
        binary = compile_fast(source)  # Cranelift
        cache.put(cache_key, binary)
        exec(binary)
```

### Cache Invalidation

- Source file changed → recompile
- Dependency changed → recompile
- Compiler version changed → recompile
- Cache entry older than N days → clean up

## REPL (Read-Eval-Print Loop)

Essential for exploration and learning.

### Basic REPL

```
$ ourlang repl
>>> x = 1 + 2
3
>>> items = [1, 2, 3]
[1, 2, 3]
>>> items.map(x -> x * 2)
[2, 4, 6]
>>> fn double(x): return x * 2
<function double>
>>> double(5)
10
```

### Implementation Challenges

1. **Incremental compilation**: Each line needs to compile incrementally
2. **State persistence**: Variables survive across lines
3. **Type inference**: Harder without full context
4. **Multiline input**: Detect incomplete statements

### REPL Features

```
>>> # Tab completion
>>> items.<TAB>
items.append    items.clear    items.len    items.map

>>> # Help
>>> ?items.map
list.map(f: fn(T) -> U) -> list[U]
  Apply function to each element, return new list.

>>> # Load file
>>> :load myfile.ol
Loaded myfile.ol

>>> # Show type
>>> :type items
list[int]

>>> # Timing
>>> :time expensive_operation()
Result: ...
Time: 1.234s
```

## Shebang Support

Scripts can be executed directly:

```
#!/usr/bin/env ourlang run

# This is a script
print("Hello from script!")
```

```bash
$ chmod +x script.ol
$ ./script.ol
Hello from script!
```

## Script-Specific Features

### Implicit Main

No `fn main()` required for scripts:

```
# This is a complete, runnable script
import http

response = http.get("https://example.com")
print(response.status)
```

Equivalent to:

```
import http

fn main():
    response = http.get("https://example.com")
    print(response.status)
```

### Command-Line Arguments

```
#!/usr/bin/env ourlang run

# args is automatically available
print(f"Script: {args[0]}")
print(f"Arguments: {args[1:]}")

# Or use argument parser
import argparse

parser = argparse.Parser(description="My script")
parser.add("--verbose", "-v", type=bool, default=false)
parser.add("file", type=str)

options = parser.parse(args)
```

### Environment Variables

```
import env

db_url = env.get("DATABASE_URL") else "localhost:5432"
debug = env.get_bool("DEBUG") else false
port = env.get_int("PORT") else 8080
```

### Inline Dependencies

For truly standalone scripts:

```
#!/usr/bin/env ourlang run
#! require http >= 1.0
#! require json >= 2.0

import http
import json

# ... script code
```

The runtime fetches/caches dependencies automatically.

## Comparison: Script vs Application Mode

| Aspect | Script Mode | Application Mode |
|--------|-------------|------------------|
| Entry point | Top-level code | `fn main()` |
| Compilation | On-demand, cached | Explicit `build` |
| Output | Run immediately | Binary file |
| Dependencies | Inline or file | Package manifest |
| Optimization | Cranelift (fast) | LLVM (optimized) |
| Use case | DevOps, one-offs | Production servers |

## Transition: Script to Application

As scripts grow, they become applications:

```
# scripts/deploy.ol (started as script)
#!/usr/bin/env ourlang run
import k8s
# ... 500 lines later
```

Becomes:

```
# deploy/
#   package.ol          # Package manifest
#   src/
#     main.ol           # Entry point
#     k8s_helpers.ol    # Extracted module
#     config.ol         # Configuration
```

```
$ ourlang build deploy/
$ ./deploy --env production
```

The transition should be gradual:
1. Script works as-is
2. Extract functions to modules
3. Add package manifest
4. Build as application when ready

## Shell Integration

### Output Formatting

```
# Automatic pretty printing
>>> users
┌────┬─────────┬─────────────────┐
│ id │ name    │ email           │
├────┼─────────┼─────────────────┤
│ 1  │ Alice   │ alice@test.com  │
│ 2  │ Bob     │ bob@test.com    │
└────┴─────────┴─────────────────┘

# Or JSON
>>> :format json
>>> users
[{"id": 1, "name": "Alice", ...}, ...]
```

### Shell Commands

For true bash replacement:

```
import shell

# Run shell command
result = shell.run("ls -la")
print(result.stdout)

# With pipe
files = shell.run("ls").pipe("grep .ol").stdout

# Or native equivalents
files = glob("*.ol")
```

### Process Management

```
import process

# Background process
proc = process.spawn("long-running-server")
proc.wait()

# With output capture
proc = process.spawn("command", capture=true)
for line in proc.stdout_lines():
    print(f"Got: {line}")
```

## Open Questions

1. **How fast is "fast enough"?** What's our target for cold-start?
2. **Global cache or per-project?** Security implications?
3. **Network-fetched dependencies?** Like Deno? Security?
4. **Windows support?** Shebang doesn't work, what instead?
5. **Interactive debugging in REPL?** Breakpoints?

## Performance Targets

| Scenario | Target | Notes |
|----------|--------|-------|
| REPL startup | < 50ms | Including stdlib |
| Script (cache hit) | < 10ms | Hash check + exec |
| Script (cache miss, tiny) | < 100ms | < 100 LOC |
| Script (cache miss, small) | < 500ms | < 1000 LOC |
| Script (cache miss, medium) | < 2s | < 10000 LOC |

## Next Steps

1. [ ] Prototype Cranelift compilation speed
2. [ ] Design cache format and invalidation
3. [ ] Implement basic REPL
4. [ ] Test with real DevOps scripts
5. [ ] Benchmark against Python/bash

---

## Research Links

- [Deno (fast TS runtime)](https://deno.land/)
- [Nushell (modern shell)](https://www.nushell.sh/)
- [Julia REPL](https://docs.julialang.org/en/v1/stdlib/REPL/)
- [Python startup time](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONSTARTUP)

---

*Last updated: 2024-01-17*
*Decision: Fast AOT with caching (tentative)*
