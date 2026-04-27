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

## Custom Iterator Implementation

When built-in combinators aren't enough, implement `Iterator` trait:

```rust
struct FibIter {
    prev: u64,
    curr: u64,
}

impl FibIter {
    fn new() -> Self {
        Self { prev: 0, curr: 1 }
    }
}

impl Iterator for FibIter {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let result = self.prev;
        let next = self.prev.checked_add(self.curr)?;
        self.prev = self.curr;
        self.curr = next;
        Some(result)
    }
}

// Usage: composable with all iterator methods
let first_10: Vec<u64> = FibIter::new().take(10).collect();
```

**Rule**: Custom iterators should be small and focused. If the state is complex, consider returning `impl Iterator` from a function instead.

## Advanced Combinators

| Combinator | Use When | Example |
|-----------|----------|---------|
| `peekable` | Need to look ahead without consuming | Parsing: peek next token before committing |
| `scan` | Stateful transformation | Running total, state machines |
| `partition` | Split into two collections by predicate | Even/odd, valid/invalid |
| `zip` | Process two iterators in lockstep | Index + value pairs |
| `enumerate` | Need index alongside value | `for (i, val) in iter.enumerate()` |
| `fold` | Custom accumulation | Building a map from key-value pairs |
| `try_fold` | Fallible accumulation | Early exit on error |

```rust
// ✅ peekable: look ahead in parser
let mut iter = tokens.iter().peekable();
while let Some(token) = iter.next() {
    if iter.peek() == Some(&&Token::Equals) {
        // Handle assignment
    }
}

// ✅ partition: split into two collections
let (even, odd): (Vec<_>, Vec<_>) = numbers.into_iter().partition(|n| n % 2 == 0);

// ✅ scan: running sum
let running_sums: Vec<_> = numbers.iter().scan(0, |state, &x| {
    *state += x;
    Some(*state)
}).collect();
```

## Related

- [control-flow.md](18-control-flow.md) — Pattern matching alternatives
- [traits.md](20-traits.md) — Implementing Iterator trait
- [performance-tuning.md](25-performance-tuning.md) — Iterator performance pitfalls
