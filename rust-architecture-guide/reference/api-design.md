# API Design & Boundaries

## Builder Pattern

### Required Fields in `new()`, Optional in Builder

```rust
// ✅ Required fields in constructor
struct Config {
    required_field: String,
    optional_timeout: Option<Duration>,
}

impl Config {
    fn new(required_field: String) -> Self {
        Self {
            required_field,
            optional_timeout: None,
        }
    }

    fn with_timeout(mut self, timeout: Duration) -> Self {
        self.optional_timeout = Some(timeout);
        self
    }
}
```

**Why**: Prevents runtime errors from missing required fields at `build()` time.

## Generic Parameters (Monomorphization Trade-offs)

### Decision Matrix

| API Type | Strategy | Rationale |
|---------|---------|-----------|
| Public API for external developers | Use `impl Into<String>` or `impl AsRef<Path>` | Ergonomics for callers |
| Internal modules, high-frequency helpers | Use concrete types (`&str`, `&Path`) | Limit monomorphization & compile time |

```rust
// ✅ Public API: Generic for ergonomics (better caller experience)
pub fn load_file(path: impl AsRef<Path>) -> Result<String> { }

// ✅ Internal: Concrete for compile speed (prefer concrete types for internal functions)
fn internal_helper(path: &Path) { }
```

**Trade-off**: Generics increase monomorphization and compile times. Limit over-generalization in internal code.

## Return Types: Concrete vs Dynamic Dispatch

### Priority: Concrete or `impl Trait`

```rust
// ✅ Preferred: Static dispatch for inlining
fn get_items() -> impl Iterator<Item = u32> { }
```

### Use `Box<dyn Trait>` When:

- Heterogeneous collections
- Deep recursion
- Complex conditional branches returning different iterators

```rust
// ✅ Don't fight static dispatch: Use Box<dyn Trait>
fn get_processor(config: &Config) -> Box<dyn Processor> {
    match config.algorithm {
        Algorithm::Fast => Box::new(FastProcessor),
        Algorithm::Accurate => Box::new(AccurateProcessor),
    }
}
```

**Trade-off**: Tiny heap allocation cost for clear API and faster compile times.

## Forward Compatibility: `#[non_exhaustive]`

### Pain Point

In large systems, downstream modules are often developed by different teams. If you add a variant to a public Enum or a field to a public Struct in v1.1, all downstream code breaks compilation due to "non-exhaustive patterns" or "missing field".

### Rule: Mark Evolving Public Types

**Specification**: For all externally exposed Enums and Structs that may evolve in the future, must mark as `#[non_exhaustive]`.

```rust
// ✅ Forward-compatible enum
#[non_exhaustive]
pub enum ServerEvent {
    Connected(ConnectionInfo),
    Disconnected(DisconnectReason),
    // Future variants can be added without breaking downstream
}

// Downstream MUST provide wildcard branch
match event {
    ServerEvent::Connected(info) => { /* ... */ },
    ServerEvent::Disconnected(reason) => { /* ... */ },
    _ => { /* handle future variants */ },
}
```

```rust
// ✅ Forward-compatible struct
#[non_exhaustive]
pub struct Config {
    pub port: u16,
    pub host: String,
    // Future fields can be added without breaking downstream
}

// Downstream MUST use Builder or struct update syntax, cannot construct directly
let config = Config {
    port: 8080,
    host: "localhost".to_string(),
    ..Config::default()
};
```

**Benefit**: Allows adding fields/variants without breaking downstream compilation. Downstream is forced to handle unknown cases.

## Sealed Trait Pattern

### Rule: Prevent External Implementations

**Specification**: If you define a Trait for internal dynamic dispatch but don't want users to implement it, must use the "sealed trait" pattern.

```rust
// ✅ Sealed trait: users cannot implement, only use
mod private {
    pub trait Sealed {}
}

pub trait Processor: private::Sealed {
    fn process(&self, data: &[u8]) -> Result<Output>;
}

// Only your crate can implement
impl private::Sealed for FastProcessor {}
impl Processor for FastProcessor {
    fn process(&self, data: &[u8]) -> Result<Output> { /* ... */ }
}

// ❌ Downstream CANNOT implement:
// impl Processor for CustomProcessor { } // Error: trait `Sealed` is private
```

**Architecture Benefits**:
- Prevents users from implementing custom logic that breaks your system invariants
- Preserves your right to add methods to the Trait without breaking compatibility
- Clear boundary: "this Trait is for internal dispatch, not for extension"

## Trade-offs

**Ergonomics vs Compile time**: Generic APIs are pleasant to use but slow compilation.

**Static vs Dynamic dispatch**: Static is faster but can lead to complex type signatures.

**Stability vs Flexibility**: `#[non_exhaustive]` adds minor boilerplate for downstream but guarantees forward compatibility.

## Related

- [error-handling.md](error-handling.md) — Error types in API boundaries
- [newtype.md](newtype.md) — Type-safe parameters in APIs
- [toolchain.md](toolchain.md) — Workspace and dependency management
