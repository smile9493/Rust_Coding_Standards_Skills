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

## Related

- [data-struct.md](data-struct.md) — Mutability in construction
- [iterators.md](iterators.md) — Borrowing in iterator chains
