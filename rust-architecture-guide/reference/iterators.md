# Iterators & Functional Streams

## Eliminate Manual Loops

### Rule: No Manual `for` + `push`

When transforming, filtering, and collecting, use iterator chains.

```rust
// ❌ Mutable state everywhere
let mut active_users = Vec::new();
for user in users {
    if user.is_active {
        active_users.push(user.name);
    }
}

// ✅ Declarative stream, no extra mutability
let active_users: Vec<_> = users.into_iter()
    .filter(|u| u.is_active)
    .map(|u| u.name)
    .collect();
```

### Benefits

- Zero-cost abstraction (compiles to same code as C loops)
- No manual mutable state management
- Higher signal-to-noise ratio

## Filter-Map Mastery

### When to Use

Use `filter_map` when mapping might fail and you want to discard failures.

### Pattern

```rust
// ❌ Verbose filter + map
let valid_ids: Vec<_> = items
    .into_iter()
    .filter(|x| x.is_valid())
    .map(|x| x.id())
    .collect();

// ✅ Concise filter_map
let valid_ids: Vec<_> = items
    .into_iter()
    .filter_map(|x| x.is_valid().then(|| x.id()))
    .collect();

// ✅ For parse operations
let numbers: Vec<i32> = strings
    .into_iter()
    .filter_map(|s| s.parse().ok())
    .collect();
```

## Common Iterator Patterns

### Flatten Nested Collections

```rust
let all_items: Vec<_> = matrix
    .iter()
    .flatten()
    .collect();
```

### Take While Condition Holds

```rust
let prefix_sum: Vec<_> = numbers
    .iter()
    .take_while(|&&x| x > 0)
    .sum();
```

### Chain Multiple Operations

```rust
let result: Vec<_> = items
    .into_iter()
    .filter(|x| x.is_valid())
    .map(|x| x.transform())
    .filter_map(|x| x.maybe_convert())
    .take(10)
    .collect();
```

## Performance Notes

- Iterator chains are **zero-cost**: compiled code equals handwritten loops
- Use `.into_iter()` for ownership transfer
- Use `.iter()` for borrowing
- Use `.iter_mut()` for mutable borrowing

## Related

- [control-flow.md](control-flow.md) — Pattern matching alternatives
- [traits.md](traits.md) — Implementing Iterator trait
