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

## RwLock Strategy

Use `RwLock` when the access pattern is **read-heavy** (many readers, few writers):

```rust
use std::sync::RwLock;

let cache: RwLock<HashMap<String, Data>> = RwLock::new(HashMap::new());

// ✅ Multiple readers can hold read lock simultaneously
let data = cache.read().unwrap().get("key").cloned();

// ✅ Write lock is exclusive — only when mutation needed
cache.write().unwrap().insert("key".into(), new_data);
```

### Decision: Mutex vs RwLock

| Scenario | Choice | Reason |
|----------|--------|--------|
| Read-heavy (>80% reads) | `RwLock` | Concurrent reads, no contention |
| Write-heavy or balanced | `Mutex` | `RwLock` write overhead > `Mutex` |
| Tiny critical section | `Mutex` | Simpler, lower overhead |
| Read-heavy + high contention | `DashMap` | Sharded, lock-free reads |

### parking_lot Alternatives

For production systems, prefer `parking_lot` over `std::sync`:

```toml
[dependencies]
parking_lot = "0.12"
```

| Feature | `std::sync` | `parking_lot` |
|---------|------------|---------------|
| Lock size | 40 bytes (Mutex) | 1 byte (Mutex) |
| Poisoning | Yes (panic on lock) | No (faster, no panic) |
| Fairness | Not guaranteed | Fair FIFO |
| Performance | Good | Better (smaller, faster) |

```rust
// ✅ parking_lot: smaller, faster, no poisoning
use parking_lot::Mutex;
let guard = mutex.lock(); // No .unwrap() needed — no poisoning
```

## Deadlock Prevention

### Rules

1. **Never hold more than one lock at a time** — if you must, always acquire in the same order
2. **Use `std::sync::Mutex` with timeout** — `lock().unwrap()` can deadlock; consider `try_lock` with retry
3. **Prefer channels over multiple locks** — eliminates deadlock by construction
4. **Test with `loom`** — model check all concurrent code paths

```rust
// ❌ Deadlock: two locks acquired in different order
fn transfer(a: &Mutex<Account>, b: &Mutex<Account>) {
    let mut ga = a.lock().unwrap();
    let mut gb = b.lock().unwrap(); // May deadlock if another thread does transfer(b, a)
}

// ✅ Channel-based: deadlock-free by design
enum Command {
    Transfer { from: Id, to: Id, amount: u64 },
}
// Single actor processes commands sequentially — no deadlock possible
```

## Synchronization Primitives

| Primitive | Use When | Example |
|-----------|----------|---------|
| `Barrier` | Wait for N threads to reach a point | Parallel initialization |
| `Once` / `OnceLock` | One-time initialization | Global config, logger setup |
| `Condvar` | Wait + signal pattern | Producer-consumer without channels |
| `Atomic` | Lock-free counters/flags | Metrics, sequence numbers |

```rust
use std::sync::atomic::{AtomicU64, Ordering};

// ✅ Lock-free counter with precise memory ordering
static REQUEST_COUNT: AtomicU64 = AtomicU64::new(0);

fn handle_request() {
    REQUEST_COUNT.fetch_add(1, Ordering::Relaxed); // No synchronization needed
}
```

## Trade-offs

**Message passing vs Shared state**: Channels eliminate data races but add communication overhead.

**Sync vs Async locks**: Sync locks are faster but can't cross await points. Async locks are flexible but slower.

**Mutex vs RwLock**: RwLock enables concurrent reads but has higher write overhead. Use only when reads >> writes.

**std::sync vs parking_lot**: parking_lot is smaller and faster but removes poisoning (a safety net for panic-during-lock).

## Related

- [data-architecture.md](data-architecture.md) — Ownership in concurrent contexts
- [api-design.md](api-design.md) — Async API boundaries
- [borrowing.md](borrowing.md) — Interior mutability patterns
- [performance-tuning.md](performance-tuning.md) — Lock-free concurrency and atomics
