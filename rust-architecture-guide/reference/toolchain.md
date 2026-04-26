# Toolchain & CI

## Clippy Configuration

### CI Setup

```toml
# rust-toolchain.toml — Lock compiler version
[toolchain]
channel = "1.75.0"  # Prevents CI breaks from Clippy updates
```

```bash
# CI command
cargo clippy -- -D warnings
```

### Allowing Exceptions

```rust
// Must include explanatory comment
#[allow(clippy::too_many_arguments)] // Domain model requires many parameters
struct ComplexConfig { }
```

## Unsafe Guidelines (Strict Isolation)

### Requirements for Any `unsafe` Block (must satisfy before any unsafe block)

1. **Strictly encapsulated** in safe API wrapper
2. **SAFETY comment** explaining why memory safety is preserved
3. **Profiling evidence** that this is a bottleneck

```rust
fn unsafe_optimization(data: &[u8]) {
    // SAFETY: Bounds check performed on previous line via data.len() > 0
    // This memory operation preserves Rust's core invariants because...
    unsafe {
        *data.get_unchecked(0)
    }
}
```

### When to Use Unsafe

- FFI calls
- Extreme micro-optimizations eliminating bounds checks
- **Only after** profiling proves it's a bottleneck

**Rule**: Never use unsafe for convenience. Always preserve safe abstraction boundaries.

## Documentation Tests

### Requirements by Module Type

| Module Type | Doc-test Requirement |
|------------|---------------------|
| Public API of published crates | Must include executable doc-tests |
| Internal `pub` modules | Use `#[doc(hidden)]`, exempt from doc-tests |

```rust
/// Internal module, not for end users
#[doc(hidden)]
pub mod internal { }
```

**Why**: Avoids slowing down test suite execution.

## Cargo Workspace & Incremental Compilation Defense

### Pain Point

As projects grow, Rust compilation speed becomes a team efficiency killer. If all code is crammed into one Crate, changing one line can trigger minutes-long incremental compilation.

### Rule: Physical Isolation via Workspace

**Specification**: Must use Cargo Workspace. Split the system into logically clear independent Crates:

```toml
# Cargo.toml (workspace root)
[workspace]
members = [
    "crates/xxx-core",    # Pure business domain models & algorithms, no I/O
    "crates/xxx-proto",   # Protocol data structures & serialization
    "crates/xxx-infra",   # Database, cache, and other infrastructure
    "crates/xxx-api",     # Business routes & controllers
]
```

**Benefits**:
- Changing `xxx-api` only recompiles that crate and dependents, not `xxx-core`
- Parallel compilation across independent crates
- Clear dependency direction prevents circular coupling

### Rule: Dependency Slimming via Feature Flags

**Specification**: Don't make users pay for features they don't need.

```toml
# crates/xxx-core/Cargo.toml
[features]
default = []
serde = ["dep:serde"]      # Only pull serde when needed
tokio = ["dep:tokio"]      # Only pull tokio when needed
full = ["serde", "tokio"]

[dependencies]
serde = { version = "1", optional = true }
tokio = { version = "1", optional = true }
```

**Architecture Benefits**:
- Reduces final binary size
- Dramatically shortens compile time for modules only depending on core logic
- Downstream crates opt-in only to features they need

## Dependency Risk Management: `cargo deny`

### Pain Point

Rust's ecosystem is thriving but prone to "dependency hell". A seemingly simple library may indirectly pull in 200 unrelated Crates.

### Rule: Supply Chain Security Audit

**Specification**: Must regularly use `cargo deny` to check the dependency tree.

```toml
# deny.toml
[advisories]
vulnerability = "deny"      # No known vulnerabilities
unmaintained = "warn"       # Warn on unmaintained crates

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-2.0", "GPL-3.0"]  # Prevent GPL contamination

[bans]
multiple-versions = "warn"  # Warn on duplicate versions (binary bloat)
```

```bash
# CI command
cargo deny check
```

**Monitoring Targets**:
- **Vulnerabilities**: No crates with known security advisories
- **License compliance**: No GPL or other non-compliant licenses contaminating the project
- **Duplicate versions**: Avoid same library at different versions causing binary bloat

## Trade-offs

**Strictness vs Flexibility**: `-D warnings` keeps code clean but may block legitimate exceptions.

**Unsafe encapsulation**: More boilerplate but preserves safe abstraction boundaries.

**Workspace granularity**: Too many tiny crates increase maintenance overhead; too few lose incremental compilation benefits.

## Related

- [error-handling.md](error-handling.md) — Error types and unsafe error handling
- [review.md](review.md) — Code review checklist including toolchain compliance
- [api-design.md](api-design.md) — API evolution and forward compatibility
