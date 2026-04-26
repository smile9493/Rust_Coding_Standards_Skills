# Error Handling Strategy

## Library vs Application Layering

### Library / Public Crate

**Must**: Define named, structured error enums.

```rust
// ✅ Required: Structured error enum
#[derive(thiserror::Error, Debug)]
pub enum DatabaseError {
    #[error("Connection failed: {0}")]
    Connection(String),
    #[error("Query failed: {0}")]
    Query(String),
}

// ❌ Forbidden: Exposing anyhow::Error in public API
pub fn query() -> anyhow::Result<Row> { } // Don't do this!
```

### Application / Binary

**Recommended**: Use `anyhow` for error aggregation.

```rust
use anyhow::{Context, Result};

async fn run() -> Result<()> {
    db.query().await.context("Failed to query database")?;
    Ok(())
}
```

## Panic Control

### Business Logic: Zero Panic

```rust
// ✅ Business logic: Return Option/Result
fn get_first(items: &[T]) -> Option<&T> {
    items.first()
}

// ✅ Proven safe precondition: Use expect with SAFETY comment
fn process_buffer(buf: &[u8]) {
    // SAFETY: Length verified on previous line
    let first = buf[0];
}
```

### Test Code: Allow unwrap

```rust
#[cfg(test)]
fn test_example() {
    let result = compute().unwrap(); // Allowed in tests
}
```

## Lazy Context Attachment

**Rule**: When context construction is expensive (string formatting, etc.), use lazy evaluation.

**Why**: Avoids unnecessary formatting when error is already present and prevents log redundancy from stacked contexts.

```rust
// ✅ Lazy evaluation for expensive context (must use lazy evaluation)
result.with_context(|| format!("Processing item {}", item.id))?;

// ❌ Avoid: Eager evaluation always formats (avoid eager evaluation)
result.with_context(format!("Processing item {}", item.id))?;
```

**Note**: Use `with_context` (lazy) instead of `context` (eager) from anyhow when formatting is involved.

## Trade-offs

**Structured errors vs Convenience**: Library users get better error handling, but library authors write more boilerplate.

**Context depth**: Too many context layers create noisy logs. Attach context at boundaries only.

## Related

- [api-design.md](api-design.md) — Error types in public APIs
- [toolchain.md](toolchain.md) — Clippy rules for error handling
