# Idiomatic Traits

## From vs Into

### Rule: Implement From, Not Into

Always implement `From<T>` — the standard library provides `.into()` for free.

```rust
// ✅ Correct: Implement From
impl From<DatabaseError> for ApiError {
    fn from(err: DatabaseError) -> Self {
        ApiError::Database(err)
    }
}

// Usage:
let api_err: ApiError = db_err.into();

// ❌ Wrong: Don't implement Into
impl Into<ApiError> for DatabaseError {
    // Unnecessary! Standard library provides this
}
```

### Composition with ? Operator

```rust
// ✅ Elegant error conversion
fn query_user() -> Result<User, ApiError> {
    let row = db.query().await?;  // Auto calls From<DbError> for ApiError
    Ok(User::from(row))
}
```

## Default Over Parameterless new()

### Rule: Derive Default Instead of `new()`

If a struct has a reasonable "initial/empty" state, use `Default`.

```rust
// ❌ Redundant parameterless constructor
impl ServerConfig {
    pub fn new() -> Self {
        Self {
            port: 8080,
            host: "localhost".to_string(),
            timeout: Duration::from_secs(30),
            // ... more defaults
        }
    }
}

// ✅ Elegant approach
#[derive(Default)]
struct ServerConfig {
    port: u16,
    host: String,
    timeout: Duration,
}

// Use struct update syntax
let config = ServerConfig {
    port: 8080,
    ..ServerConfig::default()
};
```

### Advanced: Smart Defaults

```rust
impl Default for ServerConfig {
    fn default() -> Self {
        Self {
            port: 8080,
            host: env::var("HOST").unwrap_or_else(|_| "localhost".into()),
            timeout: Duration::from_secs(30),
        }
    }
}
```

## Other Idiomatic Traits

### Display for String Representation

```rust
impl fmt::Display for UserId {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "user_{}", self.0)
    }
}
```

### PartialEq for Comparisons

```rust
#[derive(PartialEq, Eq)]
struct Config {
    // Automatic equality comparison
}
```

## Zero-Cost Abstraction Boundaries

### Marker Traits: Compile-Time Constraint Enforcement

Use marker traits (traits with no methods) to constrain generic behavior and intercept illegal operations at compile time.

```rust
// ✅ Marker trait prevents misuse at compile time
trait Verified {} // No methods — purely a constraint marker

struct Document { content: String }

impl Verified for Document {} // Only explicitly verified documents

// Only verified documents can be published
fn publish(doc: impl Verified) { /* ... */ }

// ❌ Compile error: unverified document rejected
// publish(unverified_doc);
```

**Rule**: Prefer marker traits over runtime checks when the constraint is known at compile time. This shifts bug detection from production to the compiler.

### PhantomData: Use Cautiously

`PhantomData<T>` allows carrying type information without runtime data, but creates "type magic" that is hard to understand.

```rust
// ⚠️ Acceptable: Simple PhantomData for lifetime or type parameter
struct PhantomSlice<'a, T> {
    _marker: PhantomData<&'a T>,
}

// ❌ Avoid: Complex PhantomData chains that are hard to reason about
struct OverEngineered<A, B, C> {
    _a: PhantomData<fn(A) -> ()>,  // What does this mean?
    _b: PhantomData<fn() -> B>,    // Unclear invariant
    _c: PhantomData<*const C>,     // Why raw pointer semantics?
}
```

**Rule**: Use `PhantomData` only when necessary for well-understood patterns (unused type/lifetime parameters, variance control). Avoid creating deeply nested or cryptic `PhantomData` chains. If the type-level logic requires extensive explanation, prefer a simpler runtime approach.

### Monomorphization Explosion: Pragmatic Retreat

When generic monomorphization severely impacts compile speed or binary size, it is acceptable to retreat to runtime error checking.

```rust
// ❌ Problem: Excessive monomorphization
fn process<T: Processor>(data: T::Input) -> T::Output { /* ... */ }
// Generates N versions for N processor types → slow compilation, large binary

// ✅ Pragmatic compromise: Dynamic dispatch when monomorphization explodes
fn process(data: Box<dyn Processor>) -> Result<Output> { /* ... */ }
// Single version, faster compilation, tiny runtime cost

// ✅ Pragmatic compromise: Runtime validation when type system is overkill
fn process_by_name(name: &str, data: &[u8]) -> Result<Output> {
    let processor = match name {
        "fast" => process_fast(data),
        "accurate" => process_accurate(data),
        _ => return Err(Error::UnknownProcessor(name.into())),
    };
    processor
}
```

**Decision Flow**:
1. **Default**: Use generics for compile-time safety
2. **When compile time > 2x or binary bloat is significant**: Switch to `Box<dyn Trait>` (dynamic dispatch)
3. **When even dynamic dispatch is overkill**: Use runtime validation with concrete types

**Trade-off**: Sacrificing compile-time guarantees for engineering velocity. Only justified when the generic explosion is proven to hurt CI/CD (P2 > P0 for non-safety-critical internal code).

## Trait Implementation Priority

1. **Derive first**: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`
2. **Standard traits**: `Default`, `From`, `Display`
3. **Specialized traits**: `Iterator`, `IntoIterator`, `Deref` (carefully)

## Related

- [data-struct.md](22-data-struct.md) — Struct initialization with Default
- [errors.md](21-errors.md) — From trait for error conversion
- [api-design.md](13-api-design.md) — Generic vs dynamic dispatch in API boundaries
- [priority-pyramid.md](01-priority-pyramid.md) — P2 compile time trade-offs
