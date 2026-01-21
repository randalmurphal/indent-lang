# Extension methods and trait coherence for Indent

**Indent can achieve its goal of allowing trait implementations for external types by adopting scoped, import-controlled extensions with a Python-like syntax.** The recommended approach combines Kotlin-style static resolution with Swift-style conditional conformance, while using explicit `implements` declarations to upgrade structural satisfaction to coherent nominal traits. This avoids both Go's inflexibility and Rust's complex orphan rules.

## The fundamental coherence problem

The core challenge Indent faces is well-documented in recent academic research: **global uniqueness of trait implementations conflicts with modular extensibility**. A landmark 2025 paper from EPFL analyzing Swift, Rust, Scala, and Haskell concludes that "global model uniqueness is not the right approach for modular software design." Every language makes different trade-offs along this spectrum.

Consider the scenario where Library A defines trait `Serializable`, Library B defines type `Config`, and user code wants to use `Config` with functions requiring `Serializable`. This is the exact use case Indent wants to enable. The key insight is that **different coherence strategies** serve different programming contexts:

- **Rust's orphan rules** guarantee global coherence but block legitimate cross-library integration
- **Swift's retroactive conformance** allows flexibility but risks undefined behavior when conformances conflict at runtime  
- **Kotlin's scoped extensions** eliminate coherence issues entirely but cannot satisfy interface requirements
- **Haskell's open world** trusts programmers to coordinate, with mixed results in practice

For Indent's hybrid structural/nominal system, **import-controlled nominal extensions** provide the best balance: structural traits remain flexible and implicit, while explicit extension implementations are scoped to avoid conflicts.

## Recommended syntax for Python-like extensions

For a language with Python-like syntax, extension syntax should feel natural to Python developers while clearly signaling intent. Three syntactic forms address different use cases:

**Form 1: Adding methods without trait conformance**
```python
extend String:
    def truncate(self, max_len: int) -> String:
        if len(self) <= max_len:
            return self
        return self[:max_len - 3] + "..."
```

**Form 2: Implementing a trait for an external type**
```python
extend Config implements Serializable:
    def serialize(self) -> bytes:
        return json.dumps(self.__dict__).encode()
```

**Form 3: Conditional conformance (generic extensions)**
```python
extend List[T] implements Equatable where T implements Equatable:
    def __eq__(self, other: List[T]) -> bool:
        return len(self) == len(other) and all(a == b for a, b in zip(self, other))
```

The `extend` keyword is preferable to alternatives like `trait ... for ...` because it clearly communicates **adding capability to an existing type** rather than defining a new trait. The colon-based block syntax follows Python conventions for class and function definitions.

**Scoping recommendation**: Extensions should be **module-scoped by default** and require explicit import to use. This follows Kotlin's model where extensions are visible only where imported, eliminating global coherence conflicts entirely for method-only extensions.

```python
# In serialization_extensions.py
extend Config implements Serializable:
    def serialize(self) -> bytes: ...

# In user_code.py  
from serialization_extensions import *  # Extensions now available
config.serialize()  # Works
```

## Coherence rules specification

Indent's hybrid system requires two distinct coherence regimes, one for structural traits and one for nominal traits:

**Structural trait coherence (Go-style)**: Any type with matching method signatures automatically satisfies structural traits. No coherence checking is needed because there are no explicit implementations—just method matching. This enables maximum flexibility for duck-typing scenarios.

```python
structural trait Printable:
    def print(self) -> None

# Any type with a print() -> None method satisfies Printable
# No declaration needed, no coherence conflicts possible
```

**Nominal trait coherence (extension-based)**: When using `extend Type implements Trait`, Indent should enforce **import-scoped coherence**:

- Within any single compilation unit, at most one implementation of `Trait for Type` may be visible
- Multiple implementations may exist in different modules; conflicts are detected at the use-site when both are imported
- The compiler produces an error requiring explicit disambiguation when conflicts occur

This is more permissive than Rust's orphan rules but safer than Swift's global conformance model. Conflicts become explicit import-time decisions rather than silent runtime undefined behavior.

**Disambiguation syntax for conflicts**:
```python
from lib_a import extend Config implements Serializable as config_json
from lib_b import extend Config implements Serializable as config_xml

def save(cfg: Config, format: str):
    if format == "json":
        with using config_json:
            return cfg.serialize()
    else:
        with using config_xml:
            return cfg.serialize()
```

## How extensions interact with structural vs nominal traits

The interaction between extensions and Indent's hybrid trait system requires careful design. The key principle is that **structural satisfaction is automatic while nominal satisfaction requires declaration**.

**Scenario: Adding methods to satisfy a structural trait**

When `Serializable` is structural, adding a `serialize()` method via extension immediately satisfies the trait:

```python
structural trait Serializable:
    def serialize(self) -> bytes

extend Config:
    def serialize(self) -> bytes:
        return json.dumps(self.__dict__).encode()

# Config now satisfies Serializable in any scope where this extension is imported
def save(item: Serializable) -> None:
    storage.write(item.serialize())

save(Config())  # Works - structural match via extension method
```

**Scenario: Upgrading structural to nominal**

When code requires stronger guarantees, the `implements` keyword makes satisfaction explicit:

```python
trait Hashable:  # Nominal by default
    def hash(self) -> int

extend Config implements Hashable:  # Explicit implementation
    def hash(self) -> int:
        return hash(tuple(self.__dict__.items()))
```

**Scenario: Conflicting method signatures**

If `Config` already has a `serialize()` method with an incompatible signature, the extension cannot make it satisfy `Serializable`:

```python
class Config:
    def serialize(self) -> str:  # Returns str, not bytes
        return "..."

structural trait Serializable:
    def serialize(self) -> bytes  # Requires bytes

# Cannot extend - signature mismatch
extend Config implements Serializable:  # ERROR: Config.serialize returns str, expected bytes
    ...
```

Resolution options include wrapping, renaming via adapter, or using a newtype pattern.

## Backward compatibility and edge cases

Four critical scenarios require explicit handling in Indent's design:

**When a library adds an implementation you were extending**: This is the most challenging backward compatibility issue. If user code extends `Config implements Serializable`, then the library later adds its own implementation, Indent should:

1. **Detect the conflict at import time** when both the library's implementation and user's extension are in scope
2. **Require explicit resolution** via import qualification or `using` blocks  
3. **Warn at compile time** that the extension may conflict with library evolution

This follows Swift's `@retroactive` philosophy of making the risk explicit rather than silently breaking.

**Two extensions conflicting after separate compilation**: Unlike Rust's link-time guarantees, Indent's import-scoped model handles this gracefully. Two modules can define `extend Config implements Serializable` independently. Conflict only occurs if both are imported in the same file, at which point the compiler requires disambiguation.

**Extension methods shadowing existing methods**: Following Kotlin's proven semantics, **member methods always win** over extension methods with identical signatures. When a library adds a method matching an extension, the extension becomes shadowed. Indent should emit a warning (`EXTENSION_SHADOWED_BY_MEMBER`) rather than an error, allowing existing code to continue working while alerting developers.

```python
# Before: extension works
extend String:
    def normalize(self) -> String: ...

"hello".normalize()  # Calls extension

# After library adds String.normalize():
"hello".normalize()  # Now calls member, extension is shadowed
# Compiler warning: extension String.normalize is shadowed by member
```

**Generic trait coherence with type parameters**: Conditional conformance (`extend List[T] implements Eq where T implements Eq`) should follow Swift's model with **witness table accessor functions** computed at instantiation time. This incurs modest runtime overhead but enables powerful generic programming.

## Design trade-offs analysis

The recommended design makes explicit trade-offs between competing concerns:

| Concern | Indent's position | Alternative rejected |
|---------|------------------|---------------------|
| **Coherence** | Import-scoped (conflicts at use-site) | Global orphan rules (too restrictive) |
| **Extension power** | Can implement traits for external types | Method-only extensions (insufficient) |
| **Structural traits** | Automatic satisfaction, no coherence issues | Requiring explicit `implements` everywhere |
| **Nominal traits** | Explicit `implements` with scoped coherence | Global uniqueness (conflicts block compilation) |
| **Method resolution** | Static, member-wins | Dynamic dispatch (unpredictable) |

**Compared to Rust**: Indent's scoped extensions are more permissive—you can implement any trait for any type without orphan restrictions. The trade-off is that coherence is local to import scope rather than guaranteed globally. This matches Indent's goal of "allow trait implementations for external types."

**Compared to Kotlin**: Indent goes further by allowing extensions to satisfy trait requirements (nominal conformance), not just adding methods. This requires coherence rules Kotlin doesn't need.

**Compared to Go**: Indent achieves Go's structural flexibility while adding the extension capability Go explicitly forbids to maintain package-local coherence. The cost is more complex semantics and potential for scoped conflicts.

## Code examples for common patterns

**Pattern 1: Library interop (the motivating use case)**
```python
# User wants to use external Config with external ORM expecting Persistable
from orm import Persistable
from config_lib import Config

extend Config implements Persistable:
    def to_dict(self) -> dict:
        return self.__dict__
    
    def from_dict(cls, data: dict) -> Config:
        return Config(**data)

# Now Config works with ORM
db.save(my_config)
```

**Pattern 2: Newtype for domain safety**
```python
newtype Email(str):
    def domain(self) -> str:
        return self.split("@")[1]

# Email is distinct from str, can have own trait implementations
extend Email implements Validatable:
    def validate(self) -> bool:
        return "@" in self and "." in self.domain()
```

**Pattern 3: Conditional conformance**
```python
extend Optional[T] implements Hashable where T implements Hashable:
    def hash(self) -> int:
        match self:
            case None: return 0
            case Some(value): return 1 ^ value.hash()
```

**Pattern 4: Scoped disambiguation**
```python
from json_ext import extend Config implements Serializable as json_serial
from xml_ext import extend Config implements Serializable as xml_serial

def export(cfg: Config, format: Format) -> bytes:
    match format:
        case Format.JSON:
            with using json_serial:
                return cfg.serialize()
        case Format.XML:
            with using xml_serial:
                return cfg.serialize()
```

## Conclusion

Indent's extension system should embrace **import-scoped nominal extensions** alongside **automatic structural trait satisfaction**. This design achieves the stated goal of allowing trait implementations for external types while avoiding both Go's restrictive "must own the type" limitation and Rust's complex orphan rules. The key innovations are:

- **Scoped coherence** rather than global uniqueness—conflicts are import-time errors, not definition-time restrictions
- **Two-tier trait system** where structural traits require no coherence checking and nominal traits use explicit `implements` with import-scoped resolution
- **Python-familiar syntax** using `extend Type implements Trait:` blocks that mirror class definitions
- **Explicit disambiguation** through `using` blocks when multiple implementations are needed in the same context

This positions Indent between Kotlin's limited but safe extensions and Swift's powerful but potentially incoherent retroactive conformance, providing a pragmatic middle ground suited to its hybrid structural/nominal type system.