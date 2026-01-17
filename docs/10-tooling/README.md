# Tooling Exploration

**Status**: 🔴 Deep exploration needed
**Priority**: HIGH - Tooling makes or breaks adoption
**Key Question**: What tools do we ship from day one?

## The Problem Space

We want:
- ✅ Single binary distribution (like Go)
- ✅ Fast, reliable tools
- ✅ Great editor experience (LSP)
- ✅ Opinionated formatting (no debates)
- ✅ Easy dependency management
- ✅ Good debugging experience

The insight: **Tooling is 50% of language UX**.

## Tooling Philosophy

### Go's Approach (Inspiration)

Go ships one binary with everything:
```bash
go build      # Compile
go run        # Compile and run
go test       # Test
go fmt        # Format
go vet        # Static analysis
go mod        # Dependency management
go doc        # Documentation
```

**Pros**:
- One thing to install
- Consistent experience
- Tools know about each other

### Rust's Approach (cargo)

Cargo is the tool:
```bash
cargo build   # Compile
cargo run     # Compile and run
cargo test    # Test
cargo fmt     # Format (via rustfmt)
cargo clippy  # Lint
cargo doc     # Documentation
cargo add     # Add dependency
```

**Pros**:
- Unified UX
- Extensible (cargo-xxx)
- Great dependency management

### Our Approach: Single Tool

```bash
ourlang build     # Compile to binary
ourlang run       # Compile and run
ourlang test      # Run tests
ourlang fmt       # Format code
ourlang lint      # Static analysis
ourlang check     # Type check without building
ourlang doc       # Generate documentation
ourlang new       # Create new project
ourlang add       # Add dependency
ourlang repl      # Interactive mode
```

## Core Tools

### 1. Compiler (`ourlang build`)

```bash
# Build current project
$ ourlang build
Building myproject v0.1.0
   Compiled in 0.3s

# Build specific file
$ ourlang build main.ol

# Build with optimization
$ ourlang build --release

# Cross-compile
$ ourlang build --target linux-amd64
$ ourlang build --target darwin-arm64

# Output options
$ ourlang build -o myapp
$ ourlang build --lib  # Build as library
```

**Compiler modes**:
| Mode | Backend | Speed | Output Quality |
|------|---------|-------|----------------|
| dev (default) | Cranelift | Fast | Good |
| release | LLVM | Slow | Excellent |
| check | None | Very fast | N/A |

### 2. Runner (`ourlang run`)

```bash
# Run with compilation
$ ourlang run main.ol
$ ourlang run main.ol -- --arg1 --arg2

# Run project
$ ourlang run

# Run with release optimizations
$ ourlang run --release
```

### 3. Formatter (`ourlang fmt`)

**Non-negotiable**: One style, always.

```bash
# Format file
$ ourlang fmt file.ol

# Format directory
$ ourlang fmt src/

# Format entire project
$ ourlang fmt

# Check formatting (CI)
$ ourlang fmt --check
```

**Formatting rules**:
- 4 space indentation
- No tabs
- 100 char line limit (soft)
- Imports sorted alphabetically
- Trailing commas in multiline

No configuration. Ever.

### 4. Linter (`ourlang lint`)

Static analysis beyond type checking:

```bash
$ ourlang lint
warning: unused variable 'x' at src/main.ol:10
warning: unreachable code at src/main.ol:25
error: possible nil dereference at src/main.ol:30
```

**Lint categories**:
- Correctness (likely bugs)
- Style (non-idiomatic code)
- Performance (inefficient patterns)
- Security (unsafe operations)

```bash
# All lints (default)
$ ourlang lint

# Specific category
$ ourlang lint --correctness

# Treat warnings as errors (CI)
$ ourlang lint --strict
```

### 5. Test Runner (`ourlang test`)

```bash
# Run all tests
$ ourlang test

# Run specific test
$ ourlang test test_user_creation

# Run tests matching pattern
$ ourlang test user_*

# Run with verbose output
$ ourlang test -v

# Run benchmarks
$ ourlang test --bench

# Generate coverage
$ ourlang test --coverage
```

**Output**:
```
$ ourlang test
Running tests in myproject

  user_tests
    ✓ test_user_creation (2ms)
    ✓ test_user_update (1ms)
    ✗ test_user_delete (3ms)
      Assertion failed: expected 0, got 1
      at src/user_test.ol:45

  api_tests
    ✓ test_get_users (15ms)
    ✓ test_create_user (12ms)

Results: 4 passed, 1 failed, 0 skipped
Time: 0.033s
```

### 6. Package Manager (`ourlang add`, etc.)

```bash
# Initialize new project
$ ourlang new myproject
$ ourlang new myproject --lib

# Add dependency
$ ourlang add http
$ ourlang add http@1.2.3
$ ourlang add github.com/user/package

# Remove dependency
$ ourlang remove http

# Update dependencies
$ ourlang update
$ ourlang update http

# Show dependency tree
$ ourlang deps
$ ourlang deps --tree
```

**Package manifest** (`package.ol`):
```
name = "myproject"
version = "0.1.0"
authors = ["Alice <alice@example.com>"]

[dependencies]
http = "1.0"
json = "2.0"
log = { version = "1.0", features = ["json"] }

[dev-dependencies]
testing = "1.0"

[build]
target = "binary"
entry = "src/main.ol"
```

### 7. Documentation (`ourlang doc`)

```bash
# Generate docs
$ ourlang doc

# Generate and open in browser
$ ourlang doc --open

# Generate for dependency
$ ourlang doc http
```

**Doc comments**:
```
### Fetches a user by ID.
###
### Args:
###   id: The user's unique identifier
###
### Returns:
###   The user if found, or NotFoundError
###
### Example:
###   user = get_user(123)?
###   print(user.name)
fn get_user(id: int) -> User | NotFoundError:
    ...
```

### 8. REPL (`ourlang repl`)

```bash
$ ourlang repl
>>> 1 + 2
3
>>> import http
>>> http.get("https://example.com").status
200
```

## Editor Integration

### Language Server Protocol (LSP)

Ship LSP server from day one:

```bash
$ ourlang lsp
# Runs LSP server on stdin/stdout
```

**Features**:
- Diagnostics (errors, warnings)
- Completions
- Hover information
- Go to definition
- Find references
- Rename symbol
- Code formatting
- Code actions (quick fixes)
- Inlay hints (type annotations)

### Editor Support Priority

1. **VS Code** - Extension with bundled LSP
2. **Neovim** - LSP config + tree-sitter grammar
3. **JetBrains** - Plugin
4. **Helix** - LSP config + tree-sitter
5. **Emacs** - LSP mode config

### VS Code Extension

```
ourlang-vscode/
  syntaxes/
    ourlang.tmLanguage.json    # Syntax highlighting
  snippets/
    ourlang.json               # Code snippets
  language-configuration.json   # Bracket matching, etc.
  extension.js                  # Extension entry
```

**Minimal first release**:
- Syntax highlighting
- LSP integration
- Format on save
- Run/test commands

### Tree-sitter Grammar

For Neovim, Helix, etc:

```javascript
// tree-sitter-ourlang/grammar.js
module.exports = grammar({
  name: 'ourlang',

  rules: {
    source_file: $ => repeat($._statement),

    _statement: $ => choice(
      $.function_definition,
      $.type_definition,
      $.assignment,
      // ...
    ),

    function_definition: $ => seq(
      'fn',
      $.identifier,
      $.parameter_list,
      optional(seq('->', $.type)),
      ':',
      $.block
    ),
    // ...
  }
});
```

## Debugging

### Debug Build

```bash
$ ourlang build --debug
# Includes full debug info
```

### Debugger Support

Support standard debuggers:
- **LLDB** (macOS, Linux)
- **GDB** (Linux)
- **VS Code debugger** (via DAP)

```bash
$ lldb ./myapp
(lldb) break set -f main.ol -l 10
(lldb) run
(lldb) print variable_name
```

### Debug Adapter Protocol (DAP)

For VS Code integration:

```bash
$ ourlang dap
# Runs DAP server
```

**Debug features**:
- Breakpoints
- Step in/out/over
- Variable inspection
- Call stack
- Watch expressions

### Built-in Profiler

```bash
$ ourlang run --profile myapp
$ ourlang profile report

Top functions by time:
  1. process_data     45%  (2.3s)
  2. parse_json       20%  (1.0s)
  3. http_get         15%  (0.8s)
  ...

Allocation hotspots:
  1. build_response   100MB  (40%)
  2. parse_json       50MB   (20%)
  ...
```

## CI/CD Integration

### GitHub Actions

```yaml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ourlang/setup-ourlang@v1
        with:
          version: '1.0'
      - run: ourlang fmt --check
      - run: ourlang lint --strict
      - run: ourlang test
      - run: ourlang build --release
```

### Docker

```dockerfile
# Multi-stage build
FROM ourlang:1.0 AS builder
WORKDIR /app
COPY . .
RUN ourlang build --release

FROM alpine:latest
COPY --from=builder /app/myapp /usr/local/bin/
CMD ["myapp"]
```

### Pre-built Binaries

```bash
# Install script
$ curl -fsSL https://ourlang.dev/install.sh | sh

# Or package managers
$ brew install ourlang
$ apt install ourlang
$ choco install ourlang
```

## Project Structure

Standard layout:

```
myproject/
  package.ol          # Package manifest
  src/
    main.ol           # Entry point (binary)
    lib.ol            # Library entry (library)
    module1.ol
    module2/
      mod.ol
      helper.ol
  tests/
    main_test.ol
    module1_test.ol
  examples/
    basic.ol
  docs/
    README.md
  .gitignore
```

**Conventions**:
- `src/` for source code
- `tests/` for test files (or `*_test.ol` in `src/`)
- `examples/` for example code
- `docs/` for documentation

## Error Messages

High quality error messages are tooling:

```
error: Type mismatch
 --> src/main.ol:15:10
   |
15 |     x = "hello"
   |         ^^^^^^^ expected int, found str
   |
   = note: x was declared as int on line 10
   = help: try converting with parse_int()?
```

**Error message requirements**:
1. Point to exact location
2. Explain what's wrong
3. Show relevant context
4. Suggest fixes when possible
5. Link to docs for complex issues

## Performance Targets

| Operation | Target |
|-----------|--------|
| Type check (10K lines) | < 1s |
| Full build (10K lines, dev) | < 2s |
| Full build (10K lines, release) | < 10s |
| Incremental build | < 500ms |
| Format (10K lines) | < 500ms |
| Lint (10K lines) | < 2s |
| LSP response | < 100ms |

## Open Questions

1. **Package registry?** Central (crates.io style) or decentralized (Go)?
2. **Vendoring support?** Copy deps into project?
3. **Workspace support?** Multiple packages in one repo?
4. **Plugin system?** Allow extending the toolchain?

## Implementation Priority

### Phase 1: MVP
- [ ] Compiler (build, run, check)
- [ ] Formatter
- [ ] Basic test runner
- [ ] REPL

### Phase 2: Usable
- [ ] Package manager
- [ ] Linter
- [ ] LSP (basic)
- [ ] VS Code extension

### Phase 3: Complete
- [ ] Full LSP
- [ ] Documentation generator
- [ ] Debugger integration
- [ ] Profiler
- [ ] Coverage

## Next Steps

1. [ ] Design CLI interface in detail
2. [ ] Prototype formatter
3. [ ] Start LSP implementation
4. [ ] Create VS Code extension skeleton
5. [ ] Design error message format

---

## Research Links

- [Rust Analyzer](https://rust-analyzer.github.io/)
- [gopls (Go LSP)](https://pkg.go.dev/golang.org/x/tools/gopls)
- [LSP Specification](https://microsoft.github.io/language-server-protocol/)
- [Tree-sitter](https://tree-sitter.github.io/tree-sitter/)
- [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/)

---

*Last updated: 2024-01-17*
*Decision: Single tool distribution (Go-style) (tentative)*
