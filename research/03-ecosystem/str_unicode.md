# String and Unicode design for the Indent language

**UTF-8 with grapheme awareness emerges as the optimal foundation for Indent's string system.** A systems language with Python-like syntax should pair Rust's memory-safe ownership model with Swift's Unicode correctness philosophy, avoiding the mistakes of UTF-16 legacy languages while maintaining the ergonomics Python developers expect. The recommended design uses UTF-8 internally with small string optimization, distinguishes owned from borrowed strings, defaults to grapheme-cluster semantics for length and iteration, and forbids integer indexing in favor of explicit position types.

## UTF-8 wins the representation debate decisively

The choice of UTF-8 as Indent's internal representation rests on overwhelming technical and practical advantages. UTF-8 uses **50% less memory** than UTF-16 for ASCII-heavy text—which dominates most code, URLs, log messages, and markup. It provides natural C interoperability, requires no byte-order marks, and has become the web's universal encoding. Rust, Go, and Swift (since version 5) all standardized on UTF-8 internally.

The historical argument for UTF-16—that Java, JavaScript, and Windows adopted it—reflects 1990s assumptions that 16 bits would suffice for all characters. When Unicode expanded beyond the Basic Multilingual Plane, UTF-16 gained surrogate pairs, creating a worst-of-both-worlds encoding: variable-width like UTF-8 but without its space efficiency for ASCII. JavaScript's `"𝌆".length === 2` exemplifies the confusion this causes.

**Small string optimization (SSO)** should store strings up to **22-23 bytes** inline within the string object itself, avoiding heap allocation. The libc++ implementation demonstrates this threshold works well: a 24-byte string object can hold 22 characters plus a null terminator, with a single byte distinguishing inline from heap-allocated modes. Benchmarks show SSO eliminates heap allocations for the vast majority of strings in typical programs—identifiers, short messages, and configuration keys all fit inline.

| Encoding | ASCII efficiency | CJK efficiency | Fixed-width | C interop |
|----------|-----------------|----------------|-------------|-----------|
| UTF-8    | Excellent (1 byte) | Poor (3 bytes) | No | Excellent |
| UTF-16   | Poor (2 bytes) | Good (2 bytes) | No (surrogates) | Windows only |
| UTF-32   | Poor (4 bytes) | Poor (4 bytes) | Yes | Poor |

## Owned and borrowed strings solve the systems programming problem

Indent should provide two primary string types: an owned `String` and a borrowed `str` slice. This distinction, proven by Rust, enables zero-copy operations and explicit memory management without garbage collection. The owned `String` controls its heap allocation; the borrowed `str` references string data without ownership, enabling efficient parameter passing and substring operations.

```
# Owned string - heap allocated, growable
let greeting: String = "Hello, world!"

# Borrowed slice - references existing data, no allocation
fn greet(name: str):
    print(f"Hello, {name}!")

# Both types work with the same function
greet(greeting)  # String coerces to str
greet("literal")  # String literal is str
```

A **`Cow[str]`** (clone-on-write) type should handle cases where code might or might not need to modify a string. This proves essential for APIs that usually return unchanged input but occasionally must transform it—URL normalization, for instance. Swift demonstrates that copy-on-write semantics work well when the language controls reference counting, and Indent's ownership system can provide similar guarantees.

A separate **`bytes`** type handles raw binary data. Python 3's strict separation between `str` (text) and `bytes` (binary) eliminated an entire class of encoding bugs. Indent should follow this model: strings guarantee valid UTF-8, while bytes hold arbitrary octets. Explicit encoding/decoding converts between them.

## Grapheme clusters define what users perceive as characters

The question of what `len(s)` returns reveals a fundamental tension. Bytes (Rust, Go) give O(1) performance but confuse users: `len("Здравствуйте")` returning 24 instead of 12 violates expectations. Code points (Python) improve intuition but still fail: `len("👨‍👩‍👧‍👦")` returns 7 for what users see as one emoji. Only grapheme clusters (Swift) align with human perception.

**Indent should default to grapheme clusters** for `len()` and iteration, following Swift's lead. This means:

- `"👨‍👩‍👧‍👦".len()` returns **1** (one family emoji)
- `"é".len()` returns **1** regardless of normalization form
- `"🇸🇪".len()` returns **1** (Swedish flag, composed from two regional indicators)

This O(n) operation cost is acceptable because string length queries are rare compared to iteration, and the semantic correctness prevents internationalization bugs. Explicit accessors provide O(1) alternatives when needed:

```
s.len()           # Grapheme count, O(n), what users expect
s.byte_len()      # Byte count, O(1), for allocation/IO
s.codepoint_len() # Code point count, O(n), for protocol compliance
```

## Direct integer indexing should be forbidden

Swift and Rust effectively forbid `s[5]` because it implies O(1) access that UTF-8 cannot provide without risking split characters. Indent should follow this pattern. Instead of integer indexing, provide explicit position types and iterators:

```
# COMPILE ERROR: integer indexing not allowed
let char = name[5]

# Instead: iterate over graphemes
for grapheme in name.graphemes():
    process(grapheme)

# Or: use explicit indices for byte-level access
let idx = name.byte_index(5)
let slice = name[idx..idx+3]  # Must be valid UTF-8 boundaries
```

The iteration API should provide three views with graphemes as the default:

```
for g in s:              # Sugar for s.graphemes()
for g in s.graphemes():  # Extended grapheme clusters (UAX #29)
for cp in s.codepoints(): # Unicode scalar values
for b in s.bytes():      # Raw bytes
```

This design makes correct code easy and incorrect code impossible at compile time.

## String operations should feel Pythonic but compile safely

**Concatenation** requires care. The naive `+` operator creates quadratic performance when used in loops. Indent should optimize simple cases (`a + b + c` folds into single allocation) while providing a `StringBuilder` for explicit bulk concatenation:

```
# Simple concatenation - compiler optimizes
let full = first + " " + last

# Bulk concatenation - explicit builder
let mut builder = StringBuilder.with_capacity(1000)
for item in items:
    builder.push(item)
let result = builder.to_string()

# Collection joining - most efficient for lists
let csv = ",".join(values)
```

**String interpolation** should use Python's f-string syntax with compile-time checking:

```
let message = f"Hello, {name}! You have {count} messages."
let formatted = f"Price: {price:.2f}"
let debug = f"Object: {obj:?}"  # Debug representation
```

Format specifiers follow Python's mini-language (`:.2f` for two decimal places, `:>10` for right-aligned width 10). The compiler validates format strings at compile time, catching type mismatches before runtime.

**Regular expressions** belong in the standard library, not built into syntax. The `/pattern/` literal syntax of Perl and JavaScript couples too tightly to implementation details. Instead, use raw strings and a library:

```
import re

let pattern = re.compile(r"^\d{4}-\d{2}-\d{2}$")
if pattern.matches(date_str):
    ...
```

The regex engine should guarantee **linear-time matching** (RE2 semantics), preventing catastrophic backtracking on untrusted input. This means no backreferences or lookahead—features that enable exponential-time patterns—but the safety guarantee proves worth the trade-off for a systems language.

## Normalization requires explicit programmer control

Unicode defines four normalization forms, but automatic normalization would be a mistake. NFC and NFD preserve meaning but can surprise users expecting exact round-trips. NFKC and NFKD are lossy, converting "ﬁ" (ligature) to "fi" and "²" to "2". Automatic application destroys information.

Instead, provide explicit methods and normalize identifiers at parse time:

```
# Explicit normalization
let normalized = text.nfc()  # Canonical composition
let folded = text.nfkc()     # Compatibility composition (lossy!)

# Comparison with normalization
if text1.equals_normalized(text2, form=NFC):
    ...

# Case-insensitive comparison (Unicode case folding)
if a.equals_ignore_case(b):
    ...
```

**Identifier normalization**: The parser should apply NFKC normalization to identifiers, following Unicode recommendations for identifier comparison. This ensures that visually identical identifiers are treated as equal, preventing security issues from lookalike characters.

## Memory strategies enable high-performance string handling

**String interning** stores one copy of each distinct string value. Indent should automatically intern short strings (≤40 bytes) and provide explicit interning for longer strings:

```
# Automatic for literals and short strings
let symbol = "status"  # Interned automatically

# Explicit for dynamic strings
let key = user_input.intern()  # Force interning
if key is "expected":  # Pointer comparison, O(1)
    ...
```

**Arena allocation** enables zero-allocation parsing patterns. Following Zig's approach, functions accepting strings should optionally accept an allocator parameter:

```
fn parse_json(input: str, allocator: Allocator = default_allocator) -> JsonValue:
    ...

# Use arena for request-scoped parsing
let arena = Arena.new()
defer arena.free()
let data = parse_json(request.body, arena.allocator())
# All strings freed when arena drops
```

**Zero-copy string views** enable efficient parsing without allocation:

```
struct HttpHeader:
    name: str   # Borrows from input buffer
    value: str  # Borrows from input buffer

fn parse_headers(input: str) -> List[HttpHeader]:
    # Returns views into input, no string copying
    ...
```

The lifetime system ensures these borrowed references cannot outlive their source, catching use-after-free at compile time rather than runtime.

## Comparison with existing language approaches

| Aspect | Indent (recommended) | Python | Rust | Go | Swift |
|--------|---------------------|--------|------|-----|-------|
| Internal encoding | UTF-8 | Adaptive (1/2/4 byte) | UTF-8 | UTF-8 | UTF-8 |
| `len()` returns | Graphemes | Code points | Bytes | Bytes | Graphemes |
| Integer indexing | Forbidden | Code points | Forbidden | Bytes | Forbidden |
| String types | String, str, bytes | str, bytes | String, &str, &[u8] | string, []byte | String, Substring |
| Ownership model | Owned/borrowed | GC | Owned/borrowed | GC | COW |
| Interpolation | f-strings | f-strings | format! macro | fmt.Sprintf | \() syntax |
| Normalization | Explicit | Explicit | Explicit | Explicit | Implicit for comparison |

**Python's influence** appears in Indent's syntax (f-strings, str/bytes separation) and ergonomics (len() meaning what users expect). **Rust's influence** shapes the ownership model (String vs str distinction, lifetime-based borrowing). **Swift's influence** drives Unicode correctness (grapheme-first semantics, no integer indexing). **Go's influence** warns against excessive simplicity (treating strings as byte sequences causes real bugs).

## Recommended type hierarchy and API summary

```
# Core types
String      # Owned, heap-allocated (or SSO), mutable capacity, immutable contents
str         # Borrowed slice, immutable, validates UTF-8 boundaries
bytes       # Raw byte sequence, no UTF-8 guarantee
ByteString  # Owned byte sequence

# Conversion
String.from_utf8(bytes) -> Result[String, Utf8Error]
String.from_utf8_lossy(bytes) -> String  # Replaces invalid with �
string.encode("utf-16") -> bytes
bytes.decode("utf-8") -> Result[String, DecodeError]

# Length and iteration
string.len() -> int           # Grapheme clusters
string.byte_len() -> int      # Bytes
string.codepoint_len() -> int # Code points
for g in string: ...          # Iterates graphemes
string.graphemes() -> Iterator[str]
string.codepoints() -> Iterator[Codepoint]
string.bytes() -> Iterator[byte]

# Manipulation
string.to_uppercase() -> String
string.to_uppercase(locale=Locale.TR) -> String
string.case_fold() -> String  # For comparison
string.trim() -> str
string.split(sep) -> Iterator[str]
string.replace(old, new) -> String

# Normalization
string.nfc() -> String
string.nfd() -> String
string.equals_normalized(other, form=NFC) -> bool

# Building
StringBuilder.new() -> StringBuilder
builder.push(str) -> None
builder.to_string() -> String
",".join(iterable) -> String
f"interpolation {expr}" -> String
```

This design balances Python's ergonomics with Rust's safety and Swift's Unicode correctness, providing a foundation for string handling that is both performant and resistant to internationalization bugs. The explicit ownership model enables zero-copy optimizations critical for systems programming, while the grapheme-first semantics prevent the class of bugs that plague UTF-16 languages when handling emoji, combining characters, and non-Latin scripts.

---

## Implementation Strategy (Rust Runtime)

**See also:** `research/03-ecosystem/grapheme_iteration_impl.md` for detailed implementation research.

The Indent runtime implementation should combine specialized Rust crates for optimal performance:

| Purpose | Crate | Rationale |
|---------|-------|-----------|
| Grapheme iteration | `unicode-segmentation` | Zero-copy, fast, well-maintained |
| UTF-8 validation | `simdutf8` | 4-23× faster than std |
| Byte search | `memchr` | 5-6× faster than std |
| Small string optimization | `compact_str` | 24-byte inline, mutable |

**Performance expectations:**
- **Pure ASCII:** 3-4× speedup via SIMD fast-path detection
- **Mostly ASCII (JSON/HTML):** 2.5-3.5× speedup
- **Mixed Unicode (CJK):** 1.5-2× speedup
- **Emoji-heavy:** 1.2-1.5× speedup

**Key implementation decisions:**
1. Use `unicode-segmentation` over icu4x—simpler API, sufficient for grapheme-only needs
2. Implement ASCII fast-path checking (`is_ascii()`) before any grapheme computation
3. Cache grapheme count lazily but **not** full boundary tables (avoids 5-10× overhead)
4. Invalidate cache on any mutation (simple, correct, fast enough)
5. Expose iteration as byte-range pairs for FFI efficiency