# Mutability & Borrowing

## Reduce Mutability Scope

### Technique: Shadowing to Remove Mut

If a variable is only mutable during initialization, rebind to remove `mut`.

```rust
// ✅ Elegant mutability management
let mut data = fetch_data();
data.sort();
data.dedup();

// Remove mutability, prevent accidental modification
let data = data;

// Subsequent code cannot modify data

// ❌ Always mutable - risk of accidental modification
let mut data = fetch_data();
data.sort();
data.dedup();
// ... data still mutable, could be accidentally modified
```

### Advanced: Phased Construction

```rust
// Build in phases with different mutability
let mut builder = RequestBuilder::new();
builder.set_url(url);
builder.set_method(method);

// Convert to immutable state
let request = builder.build();
// request is now immutable, safe to share across threads
```

## Slices Over Vectors

### Rule: Accept Borrowed Slices as Parameters

Always accept `&str` or `&[T]`, not `&String` or `&Vec<T>`.

```rust
// ❌ Restrictive, caller must own Vec/String
fn process_names(names: &Vec<String>) { }
fn greet(name: &String) { }

// ✅ Flexible, accepts any borrowable type
fn process_names(names: &[String]) { }
fn greet(name: &str) { }

// Usage examples:
let vec = vec!["Alice".to_string(), "Bob".to_string()];
process_names(&vec);  // ✅ Vec<String>

let array = ["Alice", "Bob"];
process_names(&array.iter().map(|s| s.to_string()).collect::<Vec<_>>());  // ✅ Array

greet("Hello");  // ✅ &str
greet(&String::from("Hello"));  // ✅ String
```

### Universal Rule

| Parameter Type | Prefer | Reason |
|---------------|--------|--------|
| `&Vec<T>` | `&[T]` | Accepts Vec, array, slice |
| `&String` | `&str` | Accepts String, &str, literal |
| `&PathBuf` | `&Path` | Accepts PathBuf, Path, &str |

## Borrowing in Loops

### Avoid Unnecessary Cloning

```rust
// ❌ Unnecessary clone in loop
for item in items.clone() {
    process(item);
}

// ✅ Borrow instead
for item in &items {
    process(item);
}
```

### Mutable Iteration

```rust
// ✅ Mutable reference in loop
for item in &mut items {
    item.process();
}
```

## Interior Mutability

When you need mutation behind a shared reference, use the standard interior mutability types:

| Type | Thread-Safe | Use When | Overhead |
|------|------------|----------|----------|
| `Cell<T>` | No | `T: Copy`, small values (counters, flags) | Zero (no borrow checking) |
| `RefCell<T>` | No | Single-threaded, dynamic borrow checking | Runtime borrow check (panic on conflict) |
| `OnceCell<T>` | No | Write-once, read-many (lazy statics, config) | None after first write |
| `Mutex<T>` | Yes | Multi-threaded, exclusive access | OS lock + poisoning |
| `RwLock<T>` | Yes | Multi-threaded, read-heavy | OS lock + poisoning |
| `AtomicU*` | Yes | Simple counters/flags | Lock-free, hardware atomic |

```rust
use std::cell::{Cell, RefCell, OnceCell};

// ✅ Cell: Copy types, zero-cost mutation
struct Counter {
    count: Cell<usize>,
}
impl Counter {
    fn increment(&self) {  // &self, not &mut self!
        self.count.set(self.count.get() + 1);
    }
}

// ✅ RefCell: dynamic borrow checking (single-threaded only!)
struct Logger {
    entries: RefCell<Vec<String>>,
}
impl Logger {
    fn log(&self, msg: &str) {  // &self
        self.entries.borrow_mut().push(msg.to_string());
    }
}

// ✅ OnceCell: write-once, read-many
struct Config {
    db_url: OnceCell<String>,
}
impl Config {
    fn db_url(&self) -> &str {
        self.db_url.get_or_init(|| read_from_env("DATABASE_URL"))
    }
}
```

### Rules

- **Never use `RefCell` in multi-threaded code** — use `Mutex`/`RwLock` instead
- **Prefer `Cell<T>` over `RefCell<T>`** when `T: Copy` — zero overhead, no panic risk
- **Prefer `OnceCell`** for lazy initialization — no `Option` + `unwrap` pattern
- **`RefCell::borrow_mut` panics on double borrow** — this is a *debugging feature*, not a production failure mode; fix the logic

## Split Borrowing

Rust allows borrowing disjoint fields simultaneously:

```rust
struct Player {
    name: String,
    score: u32,
    health: u32,
}

// ✅ Borrow different fields simultaneously
fn update(player: &mut Player) {
    let score = &mut player.score;   // mutable borrow of score
    let health = &player.health;     // immutable borrow of health
    // Both valid — disjoint fields
    *score += 10;
    println!("health: {}", health);
}
```

### Rule: Prefer Field-Level Borrows Over Struct-Level

```rust
// ❌ Borrows entire struct, blocks all other borrows
let p = &mut player;
p.score += 10;
p.health -= 5;

// ✅ Borrows only what's needed
player.score += 10;
player.health -= 5;
```

## Related

- [data-struct.md](data-struct.md) — Mutability in construction
- [iterators.md](iterators.md) — Borrowing in iterator chains
- [concurrency.md](concurrency.md) — Mutex/RwLock in concurrent contexts
