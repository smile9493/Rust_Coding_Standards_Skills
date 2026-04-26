# Concurrency & Async

## Message Passing Over Shared State

### Priority: Channels

```rust
// ✅ Preferred: Actor model + MPSC channels
use tokio::sync::mpsc;

let (tx, rx) = mpsc::channel(100); // Bounded channel for backpressure
```

### Backpressure Requirement

**Must use bounded channels** to prevent out-of-memory errors:

- Bounded channels apply backpressure automatically
- Unbounded channels **only** when consumption >> production is strictly guaranteed

```rust
// ✅ Required: Bounded channel for backpressure
use tokio::sync::mpsc;
let (tx, rx) = mpsc::channel(100); // Bounded to 100 messages

// ⚠️ Allowed only when strictly guaranteed:
// let (tx, rx) = mpsc::unbounded_channel(); 
// Only if consumer is provably faster than producer
```

## Lock Strategies

### std::sync::Mutex (Synchronous - Minimal Critical Sections)

**Rule**: Keep critical sections minimal. Extract or clone needed scalars, then release lock immediately.

```rust
// ✅ Minimal critical section
{
    let guard = mutex.lock().unwrap();
    let value = guard.some_field; // Extract scalar or clone needed data
    drop(guard); // Release immediately via drop() or block scope
    // Expensive computation OUTSIDE lock
}

// ❌ Forbidden: await while holding sync lock
// This blocks the thread pool and can cause deadlocks
```

**Compromise**: For non-cloneable large resources, execute necessary operations inside the lock, but avoid time-consuming computations.

### tokio::sync::Mutex (Async)

```rust
// ✅ Only for indivisible "async transactions"
use tokio::sync::Mutex;

async fn transfer(account: Arc<Mutex<Account>>, amount: u64) {
    let mut guard = account.lock().await;
    guard.balance -= amount; // Allowed to await inside
}
```

## Cross-Await Lock Strategy

**Absolute Rule**: Never hold `std::sync::Mutex` across `.await` points.

**Compromise**: Use `tokio::sync::Mutex` only when:
1. Business logic requires indivisible async transaction
2. Deadlock risk is documented and reviewed
3. No alternative design exists

## Trade-offs

**Message passing vs Shared state**: Channels eliminate data races but add communication overhead.

**Sync vs Async locks**: Sync locks are faster but can't cross await points. Async locks are flexible but slower.

## Related

- [data-architecture.md](data-architecture.md) — Ownership in concurrent contexts
- [api-design.md](api-design.md) — Async API boundaries
