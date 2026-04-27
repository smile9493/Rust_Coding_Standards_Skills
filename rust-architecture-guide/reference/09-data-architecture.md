# Data Architecture & Ownership

## Default Strategy: Flat + Index

For global entities and wide-scope data associations, use data-oriented design (DOD) with flat structures.

### ✅ Default Pattern

```rust
// ✅ Global entities: flat storage with indexing
struct EntityManager {
    entities: Vec<Entity>,
    index: HashMap<EntityId, usize>,
}

// Use Generational Arena to prevent dangling references
```

### ⚠️ Compromise: Smart Pointers for Local Subgraphs

For local, tightly-scoped subgraphs with consistent lifetimes (GUI trees, AST):

```rust
// ✅ Allowed: Local subgraphs with smart pointers
struct TreeNode {
    children: Vec<Rc<RefCell<TreeNode>>>,
}
```

## Lifetime Annotations

### Rule: Confine Complex Lifetimes to Low-Level Libraries

**Low-level libraries** (zero-copy parsers, network protocol frame handlers):
```rust
// ✅ Appropriate: Complex lifetimes in low-level code
fn parse_frame<'a>(data: &'a [u8]) -> Result<Frame<'a>> { }
```

**Business logic layer** (API routes, database CRUD):
```rust
// ✅ Critical: Avoid complex lifetimes in business logic
// Use owned types to prevent lifetime contagion
async fn get_user(id: UserId) -> Result<User> { }
```

**Why**: Complex lifetime annotations (`<'a, 'b>`) in business logic create cognitive overhead and make refactoring difficult.

## Pragmatic Cloning

### Decision Flow

```rust
// ✅ Non-hot path + small data → Bold cloning
fn load_config(path: &str) -> Config {
    let content = std::fs::read_to_string(path)?;
    Config::parse(content.clone()) // Small string, clone boldly
}

// ✅ Large text/collections + possibly read-only → Use Cow
use std::borrow::Cow;

fn process_text(input: &str) -> Cow<str> {
    if needs_modification(input) {
        Cow::Owned(input.replace("old", "new"))
    } else {
        Cow::Borrowed(input)
    }
}
```

## Trade-offs

**Performance vs Developer Experience**: Cloning small data in non-hot paths is negligible cost for massive simplicity gains.

**Memory layout**: Flat structures improve cache locality but may require more complex indexing logic.

## Related

- [concurrency.md](11-concurrency.md) — Sharing data across threads
- [api-design.md](13-api-design.md) — Ownership in API boundaries
