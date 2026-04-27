# Performance Tuning & Low-Level Optimization

## Core Philosophy: Hardware Sympathy

**Break through the safe abstractions at the language surface and optimize directly for CPU caches, instruction sets, and memory buses at the system level.**

### Absolute Prerequisite

**Any unconventional performance tuning must be based on micro-benchmarking or flamegraph profiling results. Blind premature optimization is strictly prohibited.**

```bash
# Required: Profiling before optimization
flamegraph ./target/release/myapp --benchmark
cargo bench --profile=profiling
```

**Rule**: No profiling data = No optimization.

---

## 1. Memory Allocation & Allocator Strategy

Heap memory allocation is the core source of system latency jitter. Reducing allocation frequency is the first priority of performance optimization.

### 1.1 Global Allocator Replacement

**Specification**: In highly concurrent long-running server applications, never use the system default allocator. Must introduce jemalloc (suitable for multi-threaded fragmentation prevention) or mimalloc (excellent throughput) based on workload characteristics.

```rust
// ✅ Required: Replace system allocator for server applications
// Option 1: jemalloc (suitable for multi-threaded fragmentation prevention)
use tikv_jemallocator::Jemalloc;

// Option 2: mimalloc (excellent throughput)
use mimalloc::MiMalloc;

#[global_allocator]
static GLOBAL: MiMalloc = MiMalloc;
```

**Benefits**:
- Reduced memory fragmentation
- Improved multi-threaded concurrent performance
- Reduced allocation latency jitter

### 1.2 Arena Allocation

**Specification**: For batches of small objects with identical lifetimes (such as AST nodes, single request contexts), use arena allocators (such as bumpalo or typed-arena).

```rust
// ✅ Arena allocation for batch objects with same lifetime
use bumpalo::Bump;

let arena = Bump::new();

// Bump allocation: O(1) allocation, O(1) bulk deallocation
let obj = arena.alloc(MyStruct { /* ... */ });

// When arena goes out of scope, all objects freed at once
```

**Applicable Scenarios**:
- AST node parsing
- Single request contexts
- Batch computation intermediate results

**Benefits**:
- Reduces tens of thousands of malloc/free overhead to a single pointer offset (Bump Allocation)
- Achieves O(1) bulk destruction
- Completely eliminates memory fragmentation

### 1.3 Pre-allocation & Capacity Awareness

**Specification**: For any dynamic collection construction involving Vec, HashMap, String, etc., if size can be predicted or estimated, must call `with_capacity(N)`. Absolutely avoid implicit memory reallocation and data copying triggered during collection expansion.

```rust
// ❌ Avoid: Implicit reallocations
let mut vec = Vec::new();
for i in 0..1000 {
    vec.push(i); // May trigger multiple reallocations
}

// ✅ Required: Pre-allocate with known capacity
let mut vec = Vec::with_capacity(1000);
for i in 0..1000 {
    vec.push(i); // Zero reallocations
}

// ✅ For HashMap
let mut map = HashMap::with_capacity(1000);

// ✅ For String
let mut s = String::with_capacity(estimated_len);
```

**Why**: Avoids implicit memory reallocation and data copying.

---

## 2. Cache Locality & Data Layout

The bottleneck of modern CPUs lies not in computation but in memory access latency. Arranging data to match CPU L1/L2 cache line extraction patterns yields benefits far exceeding fine-tuning algorithm complexity.

### 2.1 SoA (Struct of Arrays) Architecture Design

**Specification**: When traversing large numbers of entities and frequently updating a specific field, abandon traditional AoS (Array of Structs) and adopt SoA (Struct of Arrays).

```rust
// ❌ AoS (Array of Structs) - Cache pollution
struct User {
    id: u64,
    name: String,
    is_active: bool,
}

let users: Vec<User> = // ...
// Traversing is_active pollutes cache with name data

// ✅ SoA (Struct of Arrays) - Cache friendly
struct Users {
    ids: Vec<u64>,
    names: Vec<String>,
    is_actives: Vec<bool>,
}

// Iterating is_actives: cache hit rate ≈ 100%
for &active in &users.is_actives {
    if active { /* ... */ }
}
```

**Benefits**: Cache hit rate improves from ~30% to nearly 100%.

### 2.2 Field Reordering & Memory Alignment

**Specification**: Struct fields must be arranged from largest to smallest (Rust compiler defaults to automatic reordering, but manual control is needed when using `#[repr(C)]`).

**Advanced**: For variables independently updated in multi-threaded contexts, use `#[repr(align(64))]` (align to 64-byte cache line size) to completely eliminate CPU bus lock contention caused by false sharing.

```rust
// ✅ Manual field reordering (largest to smallest)
struct Optimized {
    data: [u8; 64],  // 64 bytes
    id: u64,         // 8 bytes
    flags: u32,      // 4 bytes
    active: bool,    // 1 byte
}

// ✅ Cache line alignment to prevent false sharing
use std::sync::atomic::{AtomicU64, Ordering};

#[repr(align(64))]
struct AlignedCounter {
    counter: AtomicU64,
}

// Each counter occupies separate cache line
// No false sharing between threads
```

**Why**: 64-byte alignment to cache line size eliminates CPU bus lock contention.

### 2.3 Flattening Indirection

**Specification**: Never use `Vec<Box<T>>` or deeply nested Rc/Arc in hot paths. Data should be compactly laid out in memory (such as Vec<T>), simplifying extremely expensive heap pointer dereferencing (Pointer Chasing) into continuous memory page reads.

```rust
// ❌ Forbidden in hot path: Pointer chasing
let nodes: Vec<Box<Node>> = // ...
let value = nodes[0].next.as_ref().unwrap().value;

// ✅ Required: Flat, contiguous memory
let nodes: Vec<Node> = // ...
let value = nodes[0].value; // Direct memory access
```

**Benefits**: Extremely expensive heap pointer dereferencing → continuous memory page reads.

---

## 3. Extreme Lock-Free Concurrency

In modern architectures with many cores, synchronization locks (Mutex) are the absolute culprits strangling throughput.

### 3.1 Sharded Locks & Lock Granularity Refinement

**Specification**: When handling globally shared high-frequency read/write collections, never use global RwLock<HashMap<K, V>>. Must use sharded hash tables (such as dashmap) to reduce lock contention probability by N times (N = number of shards).

```rust
// ❌ Forbidden: Global lock bottleneck
use std::sync::RwLock;
let map: RwLock<HashMap<K, V>> = // ...

// ✅ Required: Sharded data structure
use dashmap::DashMap;
let map: DashMap<K, V> = DashMap::new();

// Lock contention reduced by N shards
```

**Benefits**: Lock contention probability reduced by N times (N = number of shards).

### 3.2 Atomic Operations & Memory Ordering

**Specification**: Scalar data such as counters and status flags must use `std::sync::atomic`.

**Deep Dive**: Refuse to blindly use Ordering::SeqCst (sequential consistency, high performance cost). In accumulation scenarios without logical dependencies, must downgrade to Ordering::Relaxed; when implementing lock-free data structures, precisely use Acquire and Release semantics.

```rust
use std::sync::atomic::{AtomicU64, Ordering};

// ✅ Use atomics for counters
let counter = AtomicU64::new(0);

// ❌ Avoid: SeqCst (sequential consistency) - high overhead
counter.fetch_add(1, Ordering::SeqCst);

// ✅ Preferred: Relaxed for simple accumulation
counter.fetch_add(1, Ordering::Relaxed);

// ✅ Use Acquire/Release for lock-free data structures
counter.store(value, Ordering::Release);
let val = counter.load(Ordering::Acquire);
```

**Memory Ordering Selection Guide**:

| Scenario | Memory Ordering |
|----------|----------------|
| Simple accumulation, no logical dependency | `Relaxed` |
| Lock-free data structure publication | `Release` |
| Lock-free data structure reading | `Acquire` |
| Global order guarantee required | `SeqCst` (last resort) |

### 3.3 Thread Local Storage (TLS)

**Specification**: For high-frequency metric aggregation or random number generators, utilize ThreadLocal variables for lock-free local accumulation, aggregating to global state only when reporting, achieving zero contention overhead.

```rust
use thread_local::ThreadLocal;

// ✅ TLS for metrics aggregation
let metrics: ThreadLocal<MetricCollector> = ThreadLocal::new();

// Each thread accumulates locally, zero contention
metrics.get_or(|| MetricCollector::new()).record(value);

// Aggregate only when reporting
let total = metrics.iter().map(|m| m.sum()).sum();
```

**Benefits**: Local lock-free accumulation, zero contention overhead.

---

## 4. Compile-Time Magic & Engineering Configuration

Squeeze the last drop of performance potential from rustc and LLVM.

### 4.1 Const Generics & Stack Memory

**Specification**: For small buffers with known length not exceeding 4KB, never use Vec<u8> for heap allocation. Utilize const generics (struct Buffer<const N: usize>([u8; N])) to allocate and process data directly on the stack.

```rust
// ❌ Avoid: Heap allocation for small buffers
let buffer: Vec<u8> = vec![0; 256];

// ✅ Preferred: Stack allocation with const generics
struct Buffer<const N: usize>([u8; N]);

impl<const N: usize> Buffer<N> {
    fn new() -> Self {
        Self([0; N]) // Stack allocated
    }
}

let buffer: Buffer<256> = Buffer::new();
```

**Why**: Stack allocation is 1-2 orders of magnitude faster than heap allocation.

### 4.2 Precise Inline Instruction Placement (`#[inline]`)

**Specification**: For extremely small functions on high-frequency call paths (such as state extraction, mathematical calculations), explicitly annotate with `#[inline]` to promote cross-crate inlining. Never use `#[inline(always)]` on large functions to prevent instruction cache (I-Cache) misses causing performance cliffs.

```rust
// ✅ Use #[inline] for small, hot functions
#[inline]
fn extract_status(code: u32) -> u8 {
    (code >> 24) as u8
}

// ❌ Avoid: #[inline(always)] for large functions
// Causes I-Cache misses and performance cliff
```

**Rules**:
- ✅ Small functions (< 10 lines): `#[inline]`
- ❌ Large functions: Avoid `#[inline(always)]`
- ⚠️ Cross-crate calls: Consider `#[inline]`

### 4.3 Ultimate Build Configuration (Cargo Profile)

**Specification**: For Release builds in production environments, must enable Link-Time Optimization (LTO) and adjust Codegen Units in Cargo.toml.

```toml
# Cargo.toml
[profile.release]
lto = "fat"           # Full link-time optimization
codegen-units = 1     # Single code generation unit for ultimate runtime optimization
panic = "abort"       # Eliminate extra instructions and overhead from panic unwinding

# Optional: Profile-guided optimization (PGO)
# Requires clang for llvm-profdata
```

**Benefits**:
- LTO: Cross-crate inlining and optimization
- codegen-units=1: Global optimization opportunities
- panic=abort: Reduced binary size and instruction overhead

---

## 5. Profiling & Observability

**Core Principle**: Without measurement, there is no optimization.

### 5.1 Flamegraph — Macro Hotspot Location

**Purpose**: Quickly identify macro distribution of CPU time consumption and find function call stacks that consume the most time.

```bash
# Step 1: Install flamegraph tool
cargo install flamegraph

# Step 2: Generate flamegraph (automatically compiles and runs)
flamegraph --bin myapp -- --benchmark-args

# Step 3: View generated SVG file
# Open flamegraph.svg, deeper red indicates longer time consumption

# Async program flamegraph
flamegraph --async --bin myapp
```

**Interpretation Tips**:
- Look for **wide flat tops** — indicates the function consumes the most CPU time
- Focus on **call stack depth** — identify unnecessary abstraction layers
- Compare before and after optimization flamegraphs — verify optimization effects

### 5.2 perf (Linux) — Hardware Event Analysis

**Purpose**: Analyze CPU cache hit rates, branch prediction failures, instruction-level parallelism, and other hardware events.

```bash
# Basic performance analysis
perf record -g ./target/release/myapp

# View detailed report
perf report

# Analyze cache hit rate (Cache misses)
perf stat -e cache-references,cache-misses ./target/release/myapp

# Analyze branch prediction failures
perf stat -e branches,branch-misses ./target/release/myapp

# Analyze CPU cycles
perf stat -e cycles,instructions ./target/release/myapp
```

**Key Metrics**:
- **Cache miss rate** > 5% → Need to optimize data layout (SoA, alignment)
- **Branch miss rate** > 10% → Need to optimize branch prediction (reduce conditional branches)
- **IPC (Instructions Per Cycle)** < 1 → Memory bottleneck or pipeline stalls exist

### 5.3 Valgrind/Callgrind — Detailed Call Graph Analysis

**Purpose**: Instruction-level performance analysis, suitable for analyzing complex call relationships.

```bash
# Collect data using Callgrind
valgrind --tool=callgrind --callgrind-out-file=callgrind.out \
  ./target/release/myapp

# View call graph (requires kcachegrind or qcachegrind)
kcachegrind callgrind.out

# View hotspot functions
callgrind_annotate callgrind.out | head -n 100
```

**Advantages**:
- Precision to every instruction overhead
- Visualized call graph (Call Graph)
- Identify indirect call overhead

### 5.4 Criterion — Nanosecond-level Micro-benchmarking

**Purpose**: Precisely measure function execution time and perform statistical significance analysis.

```toml
# Cargo.toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn benchmark_algorithm(c: &mut Criterion) {
    c.bench_function("algorithm 1024", |b| {
        b.iter(|| {
            my_algorithm(black_box(1024))
        })
    });
}

criterion_group!(benches, benchmark_algorithm);
criterion_main!(benches);
```

```bash
# Run benchmark tests
cargo bench

# Generate detailed reports (including statistical distribution, confidence intervals)
# View target/criterion/reports/index.html
```

**Key Features**:
- **Statistical significance testing** — Distinguish real performance improvements from random fluctuations
- **Outlier detection** — Identify system noise interference
- **Baseline comparison** — `cargo bench --baseline new` to compare different versions

### 5.5 Other Performance Tools

| Tool | Purpose | Platform |
|------|---------|----------|
| `cargo flamegraph` | Flamegraph generation | Cross-platform |
| `perf` | CPU hardware events | Linux |
| `Instruments` | macOS performance analysis | macOS |
| `VTune` | Intel performance analysis | Cross-platform |
| `heaptrack` | Memory allocation analysis | Linux |
| `tracy` | Real-time performance analysis | Cross-platform |

---

## 6. Unsafe & Low-Level Interventions

In extreme hotspots where no further optimization is possible within safe boundaries, open Pandora's box in a controlled manner.

### 6.1 Bounds Check Elimination

**Specification**: For extremely high-frequency array slice access within loop bodies, if external code has absolutely proven index validity, use `unsafe { slice.get_unchecked(i) }` instead of direct indexing `slice[i]` to eliminate branch prediction overhead.

```rust
// ❌ Standard: Bounds check on every access
for i in 0..vec.len() {
    let val = vec[i]; // Runtime bounds check
}

// ✅ Unsafe: Eliminate bounds check
// SAFETY: i is guaranteed to be in bounds by loop condition
for i in 0..vec.len() {
    let val = unsafe { *vec.get_unchecked(i) };
}
```

**Prerequisite**: External code has absolutely proven index validity.

### 6.2 Vectorization & SIMD

**Specification**: For tasks with data parallelism characteristics such as graphics processing, cryptographic algorithms, batch string searches, etc., should use std::simd (portable) or directly summon platform-specific AVX2/AVX-512 vectorization instructions via core::arch to process multiple sets of data in one clock cycle.

```rust
// ✅ Portable SIMD (stable)
use std::simd::{f32x8, SimdFloat};

let a = f32x8::from_array([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0]);
let b = f32x8::from_array([1.0; 8]);
let c = a.mul(b); // Single instruction, 8 floats

// ✅ Platform-specific (AVX2, AVX-512)
use core::arch::x86_64::*;

unsafe {
    let result = _mm256_add_ps(a, b); // AVX2: 8 floats in parallel
}
```

**Applicable Scenarios**:
- Graphics processing
- Cryptographic algorithms
- Batch string searches
- Numerical computations

### 6.3 Raw Operation of Uninitialized Memory (MaybeUninit)

**Specification**: When constructing extremely high-performance network I/O buffers, abandon vec![0; N] (contains meaningless zeroing overhead). Must use MaybeUninit<u8> to directly read OS kernel data into uninitialized memory, then safely convert to initialized slices after completion.

```rust
use std::mem::MaybeUninit;

// ❌ Avoid: Unnecessary zero-initialization
let mut buffer = vec![0; N]; // O(N) zeroing overhead
file.read_exact(&mut buffer)?;

// ✅ Preferred: MaybeUninit for direct kernel read
let mut buffer: Vec<MaybeUninit<u8>> = Vec::with_capacity(N);
let len = file.read(unsafe {
    std::slice::from_raw_parts_mut(
        buffer.as_mut_ptr() as *mut u8,
        N,
    )
})?;
unsafe { buffer.set_len(len); }

// Now buffer is initialized by kernel data
```

**Benefits**: Eliminates meaningless zeroing overhead, directly reads into uninitialized memory.

---

## Performance Checklist

In performance-critical code, check in sequence:

### Memory
- [ ] Using appropriate allocator (jemalloc/mimalloc)
- [ ] Arena allocation for batch objects
- [ ] Pre-allocated capacity for dynamic collections
- [ ] Cache-friendly data layout (SoA)
- [ ] Avoided Vec<Box<T>> pointer chasing

### Concurrency
- [ ] Sharded locks for high-frequency shared data
- [ ] Atomic operations for counters (Relaxed/Acquire-Release)
- [ ] Thread-local storage for TLS data

### Compilation
- [ ] `#[inline]` for small functions
- [ ] Const generics for small buffers (stack allocation)
- [ ] Release configuration with LTO, codegen-units=1

### Unsafe (Last Resort)
- [ ] Bounds check elimination in hot loops
- [ ] SIMD for data parallel tasks
- [ ] MaybeUninit for I/O buffers

### Profiling (Required First Step)
- [ ] Flamegraph generated and hotspots identified
- [ ] Cache miss rate analyzed (perf stat)
- [ ] Micro-benchmarking performed (Criterion)
- [ ] Optimization effects data-supported

---

## Quick Reference: Profiling Workflow

```
1. Macro Location: flamegraph → Find hotspot functions
   ↓
2. Hardware Events: perf stat → Analyze cache miss, branch miss
   ↓
3. Detailed Analysis: perf record + report or valgrind → Call graph analysis
   ↓
4. Micro-benchmark: criterion → Precisely measure optimization effects
   ↓
5. Implement Optimization: Choose optimization strategy based on data
   ↓
6. Verify Effects: Repeat steps 1-4, compare before and after optimization
```

**Remember**: Optimization without profiling is blind; optimization without benchmarking is self-deception.

## 7. Stack-Allocated Small Collections

### `smallvec` / `smallstring` — Avoid Heap for Small Sizes

When most instances are small but you need to handle large cases too:

```rust
use smallvec::SmallVec;

// ✅ Stack-allocated for ≤ 4 items, heap only when exceeded
let items: SmallVec<[Item; 4]> = SmallVec::new();
items.push(item1);  // Stack — no allocation
items.push(item5);  // Still stack
items.push(item5);  // Exceeds 4 → spills to heap automatically
```

### Decision: Vec vs SmallVec vs ArrayVec

| Type | Stack | Heap | Use When |
|------|-------|------|----------|
| `Vec<T>` | No | Always | Unknown size, large collections |
| `SmallVec<[T; N]>` | ≤ N items | > N items | Most instances ≤ N, rare large cases |
| `ArrayVec<T, N>` | Always | Never | Size strictly ≤ N, never exceeds |
| `[T; N]` | Always | Never | Fixed size known at compile time |

**Rule**: Use `SmallVec` when profiling shows heap allocation in hot paths for collections that are usually small (e.g., function arguments, small buffers, AST node children).

## 8. Profile-Guided Optimization (PGO)

PGO uses runtime profiling data to guide the compiler's optimization decisions:

```bash
# Step 1: Build instrumented binary
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Step 2: Run representative workload
./target/release/myapp --benchmark-workload

# Step 3: Build optimized binary with profile data
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

### When PGO Helps

| Scenario | Expected Improvement |
|----------|---------------------|
| Hot branch prediction | 5-15% |
| Inlining decisions | 3-10% |
| Function layout (icache) | 2-8% |

**Rule**: Use PGO for latency-critical services where 5-15% improvement matters. Not worth the CI complexity for most applications.

## Related

- [priority-pyramid.md](01-priority-pyramid.md) — P3 (Runtime Performance) guidelines
- [trade-offs.md](04-trade-offs.md) — When performance optimization is justified
- [toolchain.md](17-toolchain.md) — Profiling and benchmarking setup
- [data-struct.md](22-data-struct.md) — Memory layout and repr annotations
