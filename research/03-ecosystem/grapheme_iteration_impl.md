# High-performance grapheme clusters for Indent's Rust runtime

**The optimal strategy combines unicode-segmentation for correctness, SIMD ASCII fast-paths for speed, and selective caching for repeated operations—avoiding full boundary caching due to its 5-10× overhead.** Modern Rust crates have achieved grapheme iteration at **1.5 GB/s** on ASCII text and **300-500 MB/s** on emoji-heavy content, making grapheme-cluster semantics viable for backend services. The key insight: full SIMD vectorization of grapheme detection is infeasible due to state machine dependencies, but hybrid approaches yield substantial wins.

This report addresses implementation strategy for Indent's grapheme-cluster-based string type, covering UAX #29 algorithm complexity, library selection, SIMD optimization opportunities, caching design, and Rust-specific recommendations.

## UAX #29 requires modest state machine but complex Unicode data

Unicode's grapheme cluster boundary detection (UAX #29) can be implemented as a deterministic finite automaton with **30-50 states** and **14 boundary rules** (GB1-GB999). The algorithm requires three Unicode properties: `Grapheme_Cluster_Break` (16 values), `Extended_Pictographic` (for emoji), and `InCB` (Indic conjunct handling, added in Unicode 15.1).

Memory requirements vary dramatically by optimization level:

| Approach | Memory | Lookup Time |
|----------|--------|-------------|
| Naive full-codepoint table | ~1.1 MB | O(1) |
| BMP direct + binary search | ~66 KB | O(1) BMP, O(log n) supplementary |
| Trie-based (ugrapheme) | **<11 KB** | O(1), L1 cache-friendly |

The **ugrapheme** library demonstrates that all necessary property tables fit within **11 KB**, small enough to remain L1-cache-resident. This makes per-character property lookup effectively O(1) with minimal memory pressure.

Edge cases have measurable but manageable performance impact. **Emoji ZWJ sequences** (like 👨‍👩‍👧‍👦, which contains 7 codepoints in 25 UTF-8 bytes) require tracking state through Extend* sequences. **Regional indicator pairs** for flags need odd/even parity counting, with naive implementations risking O(n²) on long RI runs—solved by tracking a single `ris_count` state variable. **Combining characters** (like é composed as e + ◌́) are the simplest case, requiring only GB9's "never break before Extend" rule.

**Unicode version updates** occur annually but rarely change rules. Property table regeneration is automated (minutes of work), while rule changes are rare—approximately every 2-3 versions. Unicode 15.1's addition of GB9c (Indic conjuncts) was the most significant recent change.

## Library comparison favors unicode-segmentation for pure grapheme iteration

For Indent's specific use case—grapheme iteration without locale-aware features—**unicode-segmentation** is the optimal choice. For full internationalization features, **icu4x** (the Rust rewrite of ICU) is production-ready and significantly outperforms ICU4C.

### unicode-segmentation (Recommended for Indent)

The `unicode-segmentation` crate provides zero-copy grapheme iteration with excellent performance characteristics:

```rust
use unicode_segmentation::UnicodeSegmentation;
let graphemes: Vec<&str> = "👨‍👩‍👧".graphemes(true).collect();
```

**Pros:** Lightweight (`no_std` compatible), zero-copy iteration yielding `&str` slices, **15-40% performance improvement** in v1.8.0 from inlining optimizations, ASCII special-case fast path (PR #79), and `GraphemeCursor` API for random access. **Cons:** Linear iteration only for standard API; cursor needed for random access.

### icu4x (ICU4X 2.1.0, October 2025)

ICU4X is now production-ready and powers **Firefox, Android, V8/SpiderMonkey, and Google Pixel Watch**. It dramatically outperforms ICU4C:

| Metric | ICU4X vs ICU4C |
|--------|----------------|
| Line segmenter | **19% faster** |
| Word segmenter | **52% faster** |
| Binary size | **50-90% smaller** |
| Segmentation data | **20% smaller** |

**Use icu4x when:** You need line breaking (UAX #14), locale-aware word/sentence segmentation, or full i18n support. **Don't use when:** You only need grapheme clusters—the data provider pattern adds initialization overhead unnecessary for simple grapheme iteration.

### Go and Swift approaches

**Go's standard library does not support grapheme clusters**—only rune (codepoint) iteration. Go developers use `github.com/rivo/uniseg` for UAX #29 support. A 2024 proposal to add `Graphemes()` to the standard library was closed as "not planned."

**Swift uses the operating system's ICU4C** for grapheme operations, with grapheme clusters being the default `Character` type. Swift 5 switched to UTF-8 internal encoding but still relies on ICU for Unicode operations, though Swift developers have expressed frustration with ICU's limited UTF-8 support.

Both ICU4C and ICU4X use the **Unicode License** (SPDX: Unicode-3.0), a permissive MIT-like license compatible with GPL and suitable for commercial use.

## SIMD optimization yields 2-4× gains via ASCII fast-paths

Full SIMD vectorization of grapheme boundary detection is **not feasible** due to the algorithm's sequential state machine nature—each boundary depends on previous grapheme break properties. However, hybrid approaches provide substantial performance wins.

### ASCII fast-path detection

The highest-impact optimization checks if the next 32-64 bytes are pure ASCII using SIMD:

```c
// AVX2: Load 32 bytes, check if any high bits set
__m256i data = _mm256_loadu_si256(input);
bool all_ascii = _mm256_movemask_epi8(data) == 0;
```

When true, each byte is exactly one grapheme—no state machine needed. This technique achieves **4× speedup on ASCII-only strings** and **2-3× on mostly-ASCII content** like JSON or HTML. The `simdutf8` Rust crate implements this pattern.

### SIMD UTF-8 validation

The **simdutf** library (used by Node.js, Safari, Chromium, Bun) validates UTF-8 at **13 GB/s**—approaching memcpy speed. The technique from Keiser & Lemire's paper "Validating UTF-8 In Less Than One Instruction Per Byte" uses vectorized classification via PSHUFB lookup tables.

For Indent, **pre-validating UTF-8 with SIMD** then skipping redundant validation in the grapheme iterator eliminates ~0.5-1 cycle per byte overhead. The `simdutf8` Rust crate provides a drop-in `from_utf8()` replacement that's **up to 23× faster** than std on valid non-ASCII.

### Expected performance by text composition

| Text Type | Technique | Expected Speedup |
|-----------|-----------|------------------|
| Pure ASCII | SIMD fast-path | **3-4×** |
| Mostly ASCII (JSON/HTML) | SIMD + scalar fallback | **2.5-3.5×** |
| Mixed Unicode (CJK) | Tiered property lookup | **1.5-2×** |
| Emoji-heavy | Optimized state machine | **1.2-1.5×** |

The **ugrapheme** library demonstrates achievable throughput: **~1.5 GB/s** on ASCII/Latin1, **~300-500 MB/s** on complex emoji text.

## Caching grapheme boundaries has hidden costs

The Ropey crate's experience is instructive: **earlier versions (0.5) had grapheme indexing but removed it due to 5-10× performance overhead** for keeping metadata up-to-date. For Indent's backend service workload, aggressive caching is likely counterproductive.

### When caching makes sense

Cache grapheme boundaries only when strings are large (>100KB), predominantly read-only, and accessed randomly by grapheme index—typical for text editors, not backend services. For typical backend workloads with many small strings and frequent mutations, **lazy computation beats eager caching**.

### Swift's clever index-embedded cache

Swift embeds a **6-bit grapheme cache** directly within string index values:

```
┌──────────┬────────────────╥────────────────┬───────╥───────┐
│ position │ transc. offset ║ grapheme cache │ rsvd  ║ flags │
└──────────┴────────────────╨────────────────┴───────╨───────┘
```

This enables fast forward traversal without separate allocation but doesn't help with random access or grapheme counting.

### Xi-editor's parallel rope approach

For large documents, xi-editor stores grapheme boundaries in a **separate "breaks rope"** aligned with the text rope, enabling O(log n) boundary lookup with **dirty range tracking** for incremental updates. This is the gold standard for text editor workloads but overkill for typical string operations.

### Recommended caching strategy for Indent

Store only the **grapheme count** lazily, with a **difficulty flag** to skip computation for ASCII:

```rust
struct IndentString {
    data: CompactString,
    cached_count: Option<u32>,
    flags: u8,  // ASCII_ONLY, COMPUTED_COUNT
}

impl IndentString {
    fn grapheme_len(&mut self) -> usize {
        if self.flags & ASCII_ONLY != 0 {
            return self.data.len();  // O(1) for ASCII
        }
        *self.cached_count.get_or_insert_with(|| 
            self.data.graphemes(true).count() as u32
        ) as usize
    }
}
```

Invalidation is trivial: any mutation clears `cached_count` and `flags`. This avoids the 5-10× overhead of full boundary caching while optimizing the common `len()` operation.

## Rust implementation should combine crates strategically

For Indent's Rust-based runtime, the recommended stack combines specialized crates:

| Purpose | Crate | Rationale |
|---------|-------|-----------|
| Grapheme iteration | `unicode-segmentation` | Zero-copy, fast, well-maintained |
| UTF-8 validation | `simdutf8` | 4-23× faster than std |
| Byte search | `memchr` | 5-6× faster than std |
| Small string optimization | `compact_str` | 24-byte inline, mutable |

### String type implementation

```rust
use compact_str::CompactString;
use unicode_segmentation::UnicodeSegmentation;

pub struct IndentString {
    inner: CompactString,
    grapheme_count: Option<u32>,
}

impl IndentString {
    pub fn grapheme_len(&mut self) -> usize {
        if self.inner.is_ascii() {
            return self.inner.len();  // Fast path
        }
        *self.grapheme_count.get_or_insert_with(|| 
            self.inner.graphemes(true).count() as u32
        ) as usize
    }
    
    pub fn graphemes(&self) -> impl DoubleEndedIterator<Item = &str> {
        self.inner.graphemes(true)
    }
}
```

### FFI exposure pattern

For exposing to Indent code, return byte ranges rather than copying strings:

```rust
#[repr(C)]
pub struct GraphemeSlice { pub start: u32, pub end: u32 }

pub fn precompute_boundaries(s: &str) -> Vec<u32> {
    std::iter::once(0)
        .chain(s.grapheme_indices(true).map(|(i, g)| (i + g.len()) as u32))
        .collect()
}
```

Pre-computing boundaries into a `Vec<u32>` enables O(1) indexed access when needed, paying the O(n) cost once rather than per-access.

## Conclusion

Indent's grapheme-cluster string semantics are achievable with excellent performance using a **layered optimization strategy**: SIMD ASCII detection as the first gate (3-4× speedup on ASCII), `unicode-segmentation` for correct grapheme iteration (~1.5 GB/s on Latin text), and lazy grapheme count caching only (avoiding the 5-10× overhead of full boundary caching).

**Key design decisions:**

- Use `unicode-segmentation` over icu4x—simpler API, sufficient for grapheme-only needs
- Implement ASCII fast-path checking (`is_ascii()`) before any grapheme computation
- Cache grapheme count lazily but **not** full boundary tables
- Invalidate cache on any mutation (simple, correct, fast enough)
- Expose iteration as byte-range pairs for FFI efficiency
- Consider `compact_str` for SSO if strings are typically small

The forbidden integer indexing decision is architecturally sound—grapheme indexing is inherently O(n), and pretending otherwise via caching introduces complexity disproportionate to the benefit. Iteration-based access with optional boundary pre-computation for indexed workloads is the right model for backend services processing internationalized text.