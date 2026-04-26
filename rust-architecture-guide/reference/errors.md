# Error Handling Combinators

## Unwrap-or-Else for Lazy Evaluation

### Rule: Use unwrap_or_else for Expensive Defaults

When the default value is expensive to compute, use lazy evaluation.

```rust
// ❌ Eager: formats and allocates string every time
let name = opt_name.unwrap_or(format!("user_{}", generate_id()));

// ✅ Lazy: closure only executes if opt_name is None
let name = opt_name.unwrap_or_else(|| format!("user_{}", generate_id()));
```

### Decision Guide

| Scenario | Method |
|----------|--------|
| Default is simple literal | `.unwrap_or(default_value)` |
| Default requires computation | `.unwrap_or_else(\|\| compute())` |
| Need to log on error | `.unwrap_or_else(\|err\| { log(err); default })` |

## Map-Err and And-Then

### Chain Result Operations

Don't use `match` to unwrap and rewrap Results.

```rust
// ❌ Verbose match decomposition
let result = match parse_input(input) {
    Ok(val) => Ok(transform(val)),
    Err(e) => Err(ApiError::from(e)),
};

// ✅ Chain combinators
let result = parse_input(input)
    .map(transform)
    .map_err(ApiError::from);

// ✅ Chain multiple Result operations
let user = get_user_id(request)
    .and_then(fetch_user)
    .and_then(validate_user)
    .map_err(|e| ApiError::UserProcessing(e));
```

### Practical Combinations

```rust
// Provide friendly error context
let content = std::fs::read_to_string(path)
    .map_err(|e| format!("Failed to read {}: {}", path.display(), e))?;

// Convert error type while preserving info
let config: Config = raw_config
    .map_err(ConfigError::from)?;
```

## Option Combinators

### Map and Flatten

```rust
// Transform Option value
let upper_name = opt_name.map(|s| s.to_uppercase());

// Flatten nested Option
let id = opt_user.and_then(|u| u.id());

// Provide default
let name = opt_name.unwrap_or_else(|| "anonymous".to_string());
```

## Related

- [traits.md](traits.md) — From trait for error conversion
- [control-flow.md](control-flow.md) — Pattern matching vs combinators
