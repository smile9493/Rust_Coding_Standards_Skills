# Control Flow & Patterns

## Let-Else for Early Return

### When to Use

Use `let else` when destructuring `Option` or `Result` and needing immediate return on failure.

### Pattern

```rust
// ❌ Bloated (indentation hell)
if let Some(user) = get_user() {
    if let Ok(config) = load_config(&user) {
        // ... core logic
    }
}

// ✅ Elegant (flat structure)
let Some(user) = get_user() else { return };
let Ok(config) = load_config(&user) else { return };
// ... core logic
```

### Benefits

- Eliminates nested indentation
- Keeps core logic at base indentation level
- Makes failure paths explicit and immediate

## Matches! Macro

### When to Use

Use `matches!` when checking if an enum matches specific variants without extracting data.

### Pattern

```rust
// ❌ Verbose
let is_valid = match state {
    State::Active | State::Pending => true,
    _ => false,
};

// ✅ Concise
let is_valid = matches!(state, State::Active | State::Pending);
```

### Advanced Usage

```rust
// Check specific variant
if matches!(result, Err(Error::NetworkError)) {
    retry_connection();
}

// Complex conditions
let should_process = matches!(
    status,
    Status::Ready | Status::Processing { retry: true }
);
```

## Pattern Matching Best Practices

### Destructure in let Bindings

```rust
// ❌ Unnecessary match
match point {
    Point { x, y } => println!("{}, {}", x, y),
}

// ✅ Destructure directly
let Point { x, y } = point;
println!("{}, {}", x, y);
```

### Guard Clauses with Patterns

```rust
// ✅ Combine patterns with guards
match value {
    Some(x) if x > 0 => process_positive(x),
    Some(x) => process_non_positive(x),
    None => handle_error(),
}
```

## Related

- [iterators.md](iterators.md) — Functional control flow with iterators
- [errors.md](errors.md) — Error handling patterns
