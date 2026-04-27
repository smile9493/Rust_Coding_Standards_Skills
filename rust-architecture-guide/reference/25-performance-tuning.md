# Performance Tuning & Low-Level Optimization: Mechanical Sympathy

> **Core Philosophy — Mechanical Sympathy**: Ultimate performance comes not from clever tricks, but from deep resonance between code logic and underlying physical hardware — CPU cache lines, pipelines, memory buses, and the kernel I/O stack.

---

## 1. Memory Strategy: Eliminating Allocation Redundancy

> **Economy of Motion — Reduce interaction with the operating system.**

### 1.1 Arena Architecture: Intercepting Scattered Allocation

**Rule**: For small objects sharing the same lifetime (request contexts, AST nodes), **prohibit** the global heap allocator. Must use `bumpalo` or `typed-arena`.

**Benefit**: Reduces thousands of `malloc`/`free` calls to a single pointer offset — O(1) allocation efficiency.

```rust
use bumpalo::Bump;

let arena = Bump::new();
let obj = arena.alloc(MyStruct { /* ... */ });
// When arena goes out of scope, all objects freed at once
```

### 1.2 Capacity First: Pre-allocation Determinism

**Rule**: Any construction involving `Vec`, `HashMap`, `String` must declare `with_capacity()` at initialization. Prohibit implicit reallocation on hot paths.

```rust
// ❌ Implicit reallocations
let mut vec = Vec::new();

// ✅ Pre-allocate with known capacity
let mut vec = Vec::with_capacity(1000);
```

### 1.3 Custom Allocator Physical Isolation

**Rule**: In high-performance server applications, evaluate replacing the global allocator with `tikv-jemallocator` (reduces fragmentation) or `mimalloc` (improves concurrent throughput).

```rust
use tikv_jemallocator::Jemalloc;

#[global_allocator]
static GLOBAL: Jemalloc = Jemalloc;
```

---

## 2. Data Layout: Aligning with Physics

> **Conform to cache laws, reduce memory bus oscillation.**

### 2.1 AoS to SoA Evolution (Struct of Arrays)

**Rule**: When processing large-scale data entities with high-frequency access to specific fields, must restructure `Vec<Struct>` into `Struct<Vec>`.

**Benefit**: CPU loads cache lines in a continuous, predictable pattern — maximizing L1 Cache utilization.

```rust
// ❌ AoS — Cache pollution
struct User { id: u64, name: String, is_active: bool }
let users: Vec<User> = /* ... */;

// ✅ SoA — Cache resonance
struct Users { ids: Vec<u64>, names: Vec<String>, is_actives: Vec<bool> }
```

### 2.2 Field Reordering & Alignment (Padding & Align)

**Rule**: Core data structure fields should be arranged from largest to smallest to minimize padding.

**Advanced**: For fields frequently updated by multiple threads, use `#[repr(align(64))]` to isolate them to independent cache lines — eliminating **False Sharing**.

```rust
#[repr(align(64))]
struct AlignedCounter {
    counter: AtomicU64,
}
```

### 2.3 Flatten Indirection (Eliminate Pointer Chasing)

**Rule**: Prohibit `Vec<Box<T>>` or `Vec<Rc<T>>` on hot paths. Data should be contiguously laid out in memory (`Vec<T>`), reducing pointer dereference (Pointer Chasing) that causes random memory access.

```rust
// ❌ Pointer chasing on hot path
let nodes: Vec<Box<Node>> = /* ... */;

// ✅ Flat, contiguous memory
let nodes: Vec<Node> = /* ... */;
```

---

## 3. Extreme Concurrency: Flow Like Water

> **Eliminate synchronization locks, leverage hardware atomic energy.**

### 3.1 Sharded Locks

**Rule**: Globally shared collections must not use a single `RwLock`. Must adopt sharding (e.g., `DashMap`) to distribute contention pressure.

```rust
// ❌ Global lock bottleneck
let map: RwLock<HashMap<K, V>> = /* ... */;

// ✅ Sharded data structure
let map: DashMap<K, V> = DashMap::new();
```

### 3.2 Atomic Operations: Lightweight Force

**Rule**: Counters and status flags must use `Atomic*` series.

**Memory Ordering Strategy**: Prohibit blind `Ordering::SeqCst` in non-logical-dependency scenarios. Use `Relaxed` for accumulation, `Acquire`/`Release` to protect critical sections.

```rust
// ❌ Avoid: SeqCst — high overhead
counter.fetch_add(1, Ordering::SeqCst);

// ✅ Preferred: Relaxed for simple accumulation
counter.fetch_add(1, Ordering::Relaxed);

// ✅ Acquire/Release for lock-free data structures
counter.store(value, Ordering::Release);
let val = counter.load(Ordering::Acquire);
```

### 3.3 Thread-per-Core (TPC) Architecture

**Rule**: For 10M+ concurrent gateways, evaluate abandoning traditional work-stealing scheduling in favor of `monoio` — fixed-core, lock-free interaction runtimes achieving physical core-level linear scaling.

---

## 4. Low-Level Interventions: Intercepting Overhead

> **Penetrate abstractions, reach machine instructions directly.**

### 4.1 Bounds Check Elimination (BCE)

**Rule**: In profiler-verified extremely hot loops, if index safety is proven externally, use `get_unchecked()` or `get_unchecked_mut()` to eliminate CPU branch prediction pressure.

```rust
// SAFETY: i is guaranteed to be in bounds by loop condition
for i in 0..vec.len() {
    let val = unsafe { *vec.get_unchecked(i) };
}
```

### 4.2 SIMD Vectorization

**Rule**: For large-scale numerical computation, string search, or protocol parsing, explicitly invoke `std::simd` or platform-specific instruction sets (AVX-512) — single instruction, multiple data parallel burst.

```rust
use std::simd::{f32x8, SimdFloat};
let a = f32x8::from_array([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);
let b = f32x8::from_array([1.0; 8]);
let c = a.mul(b); // Single instruction, 8 floats
```

### 4.3 Prefetching Instructions

**Rule**: When processing non-contiguous access to large indexes (e.g., LSM-Tree lookups), use explicit prefetch intrinsics to load next-stage data into L1 cache ahead of time.

```rust
use std::arch::x86_64::_mm_prefetch;

unsafe { _mm_prefetch(next_node_ptr as *const i8, _MM_HINT_T0); }
```

---

## 5. Diagnostic Toolkit

> **Data-driven. Prohibit intuition-only optimization.**

1. **Criterion**: All optimizations must be accompanied by micro-benchmark reports.
2. **Flamegraph**: Use `cargo-flamegraph` to locate CPU time consumption.
3. **Miri/Valgrind**: Detect memory leaks and undefined behavior in unsafe blocks.
4. **Perf/bpftrace**: Directly sample hardware performance counters (L1 Cache Misses, Branch Misses).

---

## 6. Agent Performance Checklist

1. **Hot path `format!` or `.to_string()`?** (Implicit allocation risk)
2. **Critical loops inlined (`#[inline]`)?**
3. **Data structure Cache Miss from excessive pointer nesting?**
4. **Can compute at compile time (`const fn`)?**
5. **Async Task holding oversized state across `.await`?** (Future state machine bloat risk)
