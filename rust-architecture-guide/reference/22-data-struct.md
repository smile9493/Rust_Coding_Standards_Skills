# Data Structures & Initialization

## Field Initialization Shorthand

### Rule: Never Repeat Variable Names

When variable name matches field name, use shorthand.

```rust
let email = "admin@test.com".to_string();
let age = 30;

// ❌ Verbose
let user = User { email: email, age: age };

// ✅ Elegant
let user = User { email, age };
```

### Mixed Usage

```rust
// ✅ Combine shorthand with explicit values
let user = User {
    email,  // Shorthand
    age,    // Shorthand
    role: Role::Admin,  // Explicit different value
    created_at: Utc::now(),
};
```

## Avoid Type Stuttering

### Rule: Use Module Scope for Naming

Don't repeat module name in type name.

```rust
// ❌ Stuttering
use user::UserConfig;
let config = UserConfig::new();

// ✅ Clean
use user::Config;
let config = user::Config::default();  // Path provides context
```

### Module Organization

```rust
// Well-organized module structure
mod user {
    pub struct Config { }
    pub struct User { }
    pub struct Service { }
}

// Usage is clear without repetition
use user::{Config, User, Service};
let config = Config::default();
let user = Service::create(config)?;
```

## Struct Update Syntax

### Use with Default

```rust
// ✅ Override only needed fields
let config = ServerConfig {
    port: 8080,
    ..ServerConfig::default()
};

// ✅ Chain multiple overrides
let dev_config = ServerConfig {
    port: 3000,
    host: "localhost".to_string(),
    ..ServerConfig::default()
};
```

## Tuple Structs

### For Newtype Pattern

```rust
// ✅ Semantic isolation
struct UserId(u64);
struct OrderId(u64);

// Usage
let id = UserId(123);
let raw: u64 = id.0;  // Explicit access
```

## Memory Layout: `#[repr]` Annotations

### Rule: Use `#[repr]` Only When Required

| Annotation | Use When | Effect |
|-----------|----------|--------|
| `#[repr(C)]` | FFI boundaries, deterministic layout | C-compatible field ordering and alignment |
| `#[repr(transparent)]` | Newtype wrapping a single field | Same layout as inner type (e.g., `#[repr(transparent)] struct NonZeroU32(u32)`) |
| `#[repr(u8)]` / `#[repr(i32)]` | C-compatible enum discriminant | Fixed-size discriminant for FFI or serialization |
| `#[repr(align(64))]` | Cache line alignment, prevent false sharing | Forces minimum alignment to N bytes |
| `#[repr(packed)]` | **Avoid** — only for parsing binary formats with unaligned fields | Removes padding — causes unaligned access UB on some platforms |

```rust
// ✅ FFI-compatible struct
#[repr(C)]
pub struct FfiHeader {
    pub magic: u32,
    pub version: u16,
    pub flags: u8,
    pub _reserved: u8,  // Explicit padding for C compat
}

// ✅ Transparent newtype — identical layout to inner type
#[repr(transparent)]
pub struct Timestamp(u64);

// ✅ Fixed-size enum for serialization
#[repr(u8)]
pub enum Status {
    Ok = 0,
    Error = 1,
    Timeout = 2,
}
```

### Rule: Never `#[repr(packed)]` for Performance

`#[repr(packed)]` causes **undefined behavior** on unaligned access. Use it only for parsing wire formats, and access fields via `copy_nonoverlapping` or by copying to a local.

## Enum Layout & Size Optimization

Rust reorders enum variants for size optimization by default. Understanding this helps avoid surprises:

```rust
// Rust auto-optimizes: discriminant + largest variant
enum Message {
    Quit,                        // 0 bytes data
    Move { x: i32, y: i32 },    // 8 bytes data
    Write(String),               // 24 bytes data
}

// Size: 1 (discriminant) + 24 (largest variant) = 25 bytes, aligned to 32
```

### Rule: Prefer Small Variants for Hot Paths

```rust
// ❌ Large enum — all variants pay for the largest
enum Event {
    Click(ClickData),        // 8 bytes
    KeyPress(KeyData),       // 4 bytes
    Network(NetworkPacket),  // 1024 bytes — inflates ALL variants
}

// ✅ Box the rare large variant
enum Event {
    Click(ClickData),
    KeyPress(KeyData),
    Network(Box<NetworkPacket>),  // 8 bytes pointer — keeps enum small
}
```

**Rule**: If one enum variant is much larger than others and rarely used, `Box` it to keep the enum size small. This improves cache locality for the common variants.

## Derive Best Practices

### Rule: Derive Order Matters for Readability

```rust
// ✅ Standard derive order: Debug first, then comparison, then ops
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct UserId(u64);

// ✅ Full standard order for complex types
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Priority(u8);
```

Standard order: `Debug` → `Clone`/`Copy` → `PartialEq`/`Eq` → `PartialOrd`/`Ord` → `Hash` → `Default`

### Rule: Don't Blindly Derive Everything

- **`Copy`**: Only for small, cheap-to-copy types (< 32 bytes). Never for types with heap data.
- **`Ord`**: Only when ordering is semantically meaningful. Don't derive for `f32`/`f64` (NaN).
- **`Hash`**: Only when `Eq` is consistent with `Hash` (equal values must hash equal).

## Related

- [traits.md](20-traits.md) — Default trait for initialization
- [borrowing.md](23-borrowing.md) — Mutability in construction
- [ffi-interop.md](15-ffi-interop.md) — FFI-compatible struct layout
- [performance-tuning.md](25-performance-tuning.md) — Cache locality and SoA layout
