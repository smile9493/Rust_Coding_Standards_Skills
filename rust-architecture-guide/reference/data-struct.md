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

## Related

- [traits.md](traits.md) — Default trait for initialization
- [borrowing.md](borrowing.md) — Mutability in construction
