# Designing a production-grade serialization system for Indent

A modern backend serialization system must balance **performance** (speed and size), **safety** (security and type correctness), and **ergonomics** (developer experience and format independence). This report synthesizes best practices from Protocol Buffers, serde, FlatBuffers, and production systems at Google, Netflix, and Cloudflare to provide concrete recommendations for Indent's serialization layer.

## The core architecture should mirror serde's format-agnostic design

The most successful serialization systems share a critical insight: **separate type mapping from format encoding**. Serde demonstrates this through a 29-type data model that sits between Rust types and wire formats. This creates clean boundaries—data structures implement `Serialize/Deserialize` to map to the data model, while formats implement `Serializer/Deserializer` to handle encoding.

For Indent, adopt this layered architecture:

**The data model should include** 14 primitive types (bool, i8–i64, u8–u64, f32, f64, char), strings, byte arrays, optionals, unit types, sequences, tuples, maps, structs, and enum variants (unit, newtype, tuple, struct). This covers Rust's type system while remaining implementable across formats. The model deliberately excludes language-specific constructs—no raw pointers, no Box types on the wire.

**Achieve zero-cost through monomorphization**. Serde generates specialized code for each type/format combination at compile time. No reflection, no vtables, no runtime type information. The visitor pattern threads state between components while compiling down to code as efficient as hand-written serializers. This matters: benchmarks show serde implementations within 5% of manual parsing.

**Derive macros eliminate boilerplate**. The `#[derive(Serialize, Deserialize)]` pattern auto-generates implementations, with attributes like `#[serde(rename)]`, `#[serde(skip)]`, `#[serde(default)]` handling common customizations. The derive macro parses container attributes, processes fields, applies rename rules, computes trait bounds automatically, and generates visitor code. Custom serialization escapes via `serialize_with` and `deserialize_with` attributes handle edge cases without abandoning the framework.

## Format selection depends on context—support three tiers

After benchmarking JSON, MessagePack, Protocol Buffers, FlatBuffers, CBOR, and Cap'n Proto, clear patterns emerge for different use cases.

**Tier 1 (native support): JSON and MessagePack.** JSON remains essential for public APIs, debugging, and browser clients. MessagePack provides a **4x faster** drop-in replacement with **30% smaller** payloads for internal services. Redis caching benchmarks show MessagePack + zlib achieving **70% reduction** in fetch+deserialize time versus JSON. Support these formats out of the box.

**Tier 2 (optional support): Protocol Buffers.** For gRPC integration and strict schema evolution requirements, Protobuf offers the best balance of size, speed, and tooling maturity. Benchmarks show **6x faster** serialization than JSON with **60-70% smaller** payloads. However, it requires schema files and code generation—make this an optional feature.

**Tier 3 (specialized): FlatBuffers or rkyv for zero-copy.** When deserialization latency is critical (game engines, memory-mapped files, high-frequency trading), zero-copy formats eliminate parsing entirely. FlatBuffers achieves **62x faster deserialization** than Protobuf by accessing data directly via offset arithmetic. Reserve this for performance-critical paths where complexity is justified.

| Use Case | Recommended Format | Why |
|----------|-------------------|-----|
| Public REST APIs | JSON | Universal support, debuggable |
| Internal microservices | MessagePack | Fast, compact, schema-flexible |
| gRPC services | Protocol Buffers | Schema evolution, compact |
| Redis/cache | MessagePack | 70% faster than JSON with compression |
| Kafka/message queues | Protobuf + Schema Registry | Compatibility checking |
| Memory-mapped files | FlatBuffers/rkyv | Zero-copy random access |
| Configuration | JSON/TOML | Human-readable |

## Schema evolution requires discipline around field identity

Schema evolution is the art of modifying data structures while maintaining compatibility between different software versions. Protocol Buffers, Avro, and Cap'n Proto each handle this differently, but share fundamental principles.

**Field numbers are sacred—never reuse them.** In tagged formats like Protobuf, field numbers (not names) identify data on the wire. Deleting field 3 and later adding a new field 3 corrupts data silently. Always use `reserved` statements:

```protobuf
message User {
  reserved 2, 3, 15 to 20;
  reserved "old_field", "deleted_field";
  string name = 1;
  string email = 4;
}
```

**Adding fields is safe when optional with defaults.** New fields must be optional (proto3's default) or have explicit default values. Old readers ignore unknown fields; new readers use defaults for missing fields. This enables backward compatibility (new code reads old data) and forward compatibility (old code reads new data).

**Renaming fields is format-dependent.** Binary formats identify fields by number, so renaming is safe. JSON and text formats use names—renaming breaks existing data. Use aliases during transitions:

```rust
#[serde(rename = "new_name", alias = "old_name")]
name: String,
```

**Type changes are mostly forbidden.** Only specific "widening" conversions work safely: int32 ↔ int64, sint32 ↔ sint64, fixed32 ↔ sfixed32. Changing between signed/unsigned encoding, or between entirely different types, corrupts data.

**Enum evolution follows strict rules.** Always include an unknown/unspecified value at position 0. Never reuse enum values. Adding values is backward-compatible; removing values breaks forward compatibility. Reserve deleted values:

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;
  ACTIVE = 1;
  INACTIVE = 2;
  reserved 3;  // DELETED was here
}
```

**Avro differs fundamentally**: it identifies fields by name (not number) and requires the writer's schema for reading. This enables richer evolution semantics but demands a schema registry. Use transitive compatibility checking (FULL_TRANSITIVE) for long-term storage systems.

## Zero-copy deserialization trades complexity for speed

Zero-copy eliminates CPU copying during parsing by directly referencing bytes in the serialized buffer. Instead of allocating heap objects, the deserialized view points into the original buffer using offset arithmetic.

**FlatBuffers uses vtables for field lookup.** The buffer contains a root offset, vtables with field offsets, and inline data. Access works by: finding the table location via root offset, subtracting to find vtable, looking up field offset (0 means absent, use default), adding offset to get field location. This enables **O(1) random access** to any field without parsing the entire message.

**Cap'n Proto uses fixed struct layouts.** Fields live at compile-time-known offsets, like C structs. Pointers store type information enabling recursive copying without schema knowledge. Cap'n Proto's wire format equals its in-memory format—serialization literally does nothing.

**Rust's rkyv provides total zero-copy for Rust types.** It serializes data laid out exactly as in-memory representation, using relative pointers instead of absolute ones. Deserialization becomes a pointer cast. This requires "Archived" type variants (ArchivedString, ArchivedHashMap) that use offset-based pointers.

**The trade-offs are significant:**

- **Arena allocation leaks memory** on modification—you can't free individual objects
- **Alignment requirements** (u64 at 8-byte boundaries) make buffers larger
- **Lifetime complexity** propagates through the codebase when borrowing from buffers
- **Verification overhead** for untrusted input requires O(n) validation passes

**Use zero-copy when**: large files are accessed partially via mmap, memory allocation must be minimized, or data is loaded once and accessed many times. **Avoid when**: messages are small (setup overhead dominates), frequent mutations are needed, or cross-platform portability matters.

## Streaming serialization handles data exceeding memory

For large datasets, streaming avoids loading everything into memory. The key is separating **framing** (detecting message boundaries) from **parsing** (interpreting content).

**Length-prefixed framing is the standard approach:**
```
[4 bytes: length][N bytes: message][4 bytes: length][M bytes: message]...
```

This enables incremental processing: read length, buffer until complete, parse, repeat. Protocol Buffers uses base-128 varints for internal lengths but requires application-level framing for message streams.

**NDJSON (newline-delimited JSON) offers simplicity.** One complete JSON object per line enables trivial framing (split on newline), works with Unix tools, and is append-friendly. Use for log files, data exports, and streaming APIs.

**Backpressure integration prevents memory exhaustion.** When consumers can't keep up with producers, systems must signal slowdown. Reactive Streams standardizes this: subscribers request N items, publishers send only N items. For gRPC, HTTP/2 provides protocol-level flow control. For TCP, use bounded buffers that trigger throttling when full.

**simdjson's On-Demand API demonstrates efficient partial parsing.** Stage 1 uses SIMD to find all structural characters; Stage 2 walks structure with a forward-only cursor. Only accessed values are materialized—parsing a single field from a large document is O(1) in the accessed field count.

## Enum serialization strategies affect cross-language compatibility

Rust enums (sum types with data) have no direct equivalent in JSON or most formats. Four strategies exist:

**Externally tagged (default):** Tag is map key, content is value.
```json
{"Request": {"id": "123"}}
```
Works in no-alloc environments, best for binary formats. Use as default.

**Internally tagged:** Tag inside content object.
```json
{"type": "Request", "id": "123"}
```
Common in Java APIs, more readable. Requires heap allocation for deserialization. Doesn't work with tuple variants.

**Adjacently tagged:** Tag and content are siblings.
```json
{"t": "Request", "c": {"id": "123"}}
```
Common in Haskell (Aeson). Enables consistent structure regardless of content type.

**Untagged:** No tag, deserializer tries variants in order.
```json
"hello"  // or 42
```
Useful for flexible parsing but produces poor error messages and has performance cost. Use sparingly.

**Recommendation:** Default to externally tagged for binary formats, internally tagged for JSON APIs, and always include an `#[serde(other)]` catch-all variant for forward compatibility.

## Security requires explicit limits and hardened defaults

Serialization vulnerabilities caused the **2017 Equifax breach** (Java deserialization via Apache Struts) and remain OWASP Top 10. Design for untrusted input from day one.

**Implement mandatory limits:**

| Parameter | Recommended Default | Rationale |
|-----------|---------------------|-----------|
| Max depth | 64 | Prevents stack overflow from nested structures |
| Max string length | 20 MB | Balances legitimate use with allocation bombs |
| Max payload size | 10 MB | Matches gRPC default, prevents memory exhaustion |
| Max array elements | 100,000 | Application-dependent but needs cap |
| Max field count | 10,000 | Prevents HashDoS on struct deserialization |

**Prevent hash collision attacks.** Use randomized hashing (SipHash) for all hash maps populated from untrusted input. Java 8+ mitigates via TreeNode fallback when buckets exceed threshold; Rust's HashMap uses SipHash by default. Never use predictable hash functions (SDBM, FNV) for user-controlled keys.

**Handle Billion Laughs variants.** XML entity expansion can turn 1KB into 3GB. Disable DTD processing entirely for XML. For nested structures in any format, use iterative parsing with depth counters—never recursive functions on untrusted input.

**Validate before trusting.** Zero-copy formats especially require validation—reading past buffer boundaries causes crashes or worse. FlatBuffers and rkyv provide optional verification passes; always verify untrusted input before access. Cap'n Proto validates during access, catching violations lazily.

**Specific CVEs to learn from:**
- CVE-2024-7254: Protobuf-java stack overflow via nested groups
- CVE-2023: Protobuf ~500KB payload causes 3GB+ RAM allocation
- SNYK-JS-PROTOBUFJS-2441248: Prototype pollution in protobufjs

## Integration patterns vary by context

**API request/response:** Use JSON with RFC 7807 Problem Details for errors. Support content negotiation via Accept header. Version APIs via URL path (`/v1/users`) for simplicity or Accept header (`application/vnd.api.v1+json`) for RESTful purity.

**Database storage:** PostgreSQL JSONB provides native querying and indexing—prefer it over serializing to TEXT. Store schema version alongside data for future migrations. Extract frequently-queried fields into dedicated columns.

**Redis caching:** Use Redis Hashes for object storage with individual field access—**70% less memory** than separate keys. MessagePack outperforms JSON significantly. Always set TTL with random jitter (TTL + random(0, TTL*0.1)) to prevent cache stampede. Version key names (`user:123:v1`) for zero-downtime migrations.

**Kafka/message queues:** Use Schema Registry for Protobuf/Avro messages. The wire format includes magic byte, schema ID, and binary payload. Disable auto-register in production (`auto.register.schemas=false`). Configure compatibility checking (BACKWARD_TRANSITIVE for maximum safety).

**Configuration files:** TOML for simple configs (Rust ecosystem standard), YAML for complex hierarchies (Kubernetes), JSON for programmatic generation. Never commit secrets—use environment variable overlay: load base config from file, override with env vars.

## Recommended trait design for Indent

Based on this research, Indent's serialization system should:

1. **Define a finite data model** covering primitives, strings, bytes, optionals, sequences, maps, structs, and enum variants (unit/newtype/tuple/struct). Approximately 25-30 types.

2. **Separate Serialize/Deserialize traits** (for types) from Serializer/Deserializer traits (for formats). This enables format independence through the data model abstraction layer.

3. **Support derive macros** with attributes for rename, skip, default, flatten, serialize_with, deserialize_with. Use `#[serde(rename_all = "camelCase")]` patterns for case conversion.

4. **Implement the visitor pattern** for deserialization. Deserializer parses input and calls Visitor methods; Visitor constructs target types. This enables flexible type coercion.

5. **Provide format hints** via `is_human_readable()` for format-appropriate representations (DateTime as ISO string vs Unix timestamp).

6. **Support borrowing for zero-copy** via lifetime parameters. Allow `&'de str` and `&'de [u8]` to borrow from input buffers.

7. **Build in security limits** as non-optional configuration with sane defaults. Reject inputs exceeding max depth, size, or string length.

8. **Ship with JSON and MessagePack** as native formats. Provide optional Protocol Buffers support for gRPC integration. Document how to implement custom formats.

9. **Document schema evolution rules** prominently: field addition/removal safety, enum evolution, type change restrictions. Provide migration tooling for breaking changes.

10. **Validate untrusted input** by default. Require explicit opt-out (`#[serde(unchecked)]`) for skipping validation in trusted contexts.

This architecture balances Rust ecosystem conventions (serde compatibility), production requirements (security, schema evolution), and performance needs (zero-copy capability, efficient binary formats). The result is a serialization system that handles the 80% case ergonomically while providing escape hatches for specialized requirements.