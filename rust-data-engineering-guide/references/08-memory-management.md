# 08 — Memory Management for Analytics

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Rust allocator model, `Arc`/`Box` lifetimes, memory mapping (`mmap`) |
| Aligned Crates | `bumpalo`, `tempfile`, `string_cache`, `lasso`, `snmalloc-rs` |

---

## Core Architecture

Analytics workloads have unique memory patterns: large short-lived allocations (sort buffers, join hash tables), repeated string allocations (dimension tables), and the constant risk of OOM at TB scale. The design must handle three axes: allocation speed, lifetime management, and overflow to disk.

### Arena per Query — Bump Allocation

A bump allocator (`bumpalo`) allocates memory with a simple pointer increment and reclaims everything at query completion:

```rust
use bumpalo::Bump;

struct QueryArena {
    arena: Bump,
}

impl QueryArena {
    fn new() -> Self {
        QueryArena { arena: Bump::new() }
    }

    fn alloc_i64_slice(&self, len: usize) -> &mut [i64] {
        self.arena.alloc_slice_fill_iter((0..len).map(|_| 0i64))
    }

    fn alloc_string(&self, s: &str) -> &mut str {
        self.arena.alloc_str(s)
    }
}

fn execute_query() -> Vec<RecordBatch> {
    let arena = QueryArena::new();

    // All intermediate results go into the arena
    let temp_buffer = arena.alloc_i64_slice(1_000_000);
    let temp_strings: Vec<&str> = (0..1000)
        .map(|i| arena.alloc_str(&format!("key_{}", i)))
        .collect();

    // ... process query using temp_buffer, temp_strings ...

    // All arena memory reclaimed at once when arena drops
    vec![]
}
```

**Performance**: Bump allocation is ~10× faster than the global allocator (no `free()`, no fragmentation). `Bump::reset()` clears the entire arena in O(1) time.

**Pattern**: Each query gets its own arena. At query end, `drop(arena)` → all intermediate memory freed at once.

### Spill-to-Disk — Out-of-Core Execution

When memory exceeds the configured budget, operators spill intermediate data to disk:

```rust
use tempfile::TempFile;
use std::io::{Write, Read, Seek, SeekFrom};

struct SpillFile {
    file: TempFile,
    row_count: usize,
    schema: Arc<Schema>,
}

impl SpillFile {
    fn spill_batches(batches: &[RecordBatch], memory_limit: usize) -> Result<Vec<SpillFile>, SpillError> {
        let current_usage = estimate_memory(batches);
        if current_usage <= memory_limit {
            return Ok(vec![]);
        }

        // Sort batches by size, spill largest first
        let mut sorted: Vec<_> = batches.iter().enumerate().collect();
        sorted.sort_by_key(|(_, b)| std::cmp::Reverse(b.get_array_memory_size()));

        let mut spill_files = Vec::new();
        let mut spill_target = current_usage - memory_limit;

        for (idx, batch) in sorted {
            if spill_target == 0 { break; }

            let mut file = tempfile::tempfile()?;
            let mut writer = arrow::ipc::writer::FileWriter::try_new(
                &mut file,
                &batch.schema(),
            )?;
            writer.write(batch)?;
            writer.finish()?;
            file.seek(SeekFrom::Start(0))?;

            spill_files.push(SpillFile {
                file: TempFile::from_parts(file, false),
                row_count: batch.num_rows(),
                schema: batch.schema(),
            });
            spill_target = spill_target.saturating_sub(batch.get_array_memory_size());
        }

        Ok(spill_files)
    }

    fn read_back(&mut self) -> Result<Vec<RecordBatch>, SpillError> {
        self.file.seek(SeekFrom::Start(0))?;
        let reader = arrow::ipc::reader::FileReader::try_new(&mut self.file, None)?;
        reader.collect::<Result<Vec<_>, _>>().map_err(Into::into)
    }
}
```

**Spill trigger conditions:**

| Operator | Spill trigger | Spill mechanism |
|----------|-------------|-----------------|
| Sort | Input > memory | External merge sort (write sorted runs to disk, k-way merge) |
| HashJoin | Build side > memory | Grace hash join (partition hash table to disk, probe in passes) |
| Aggregate | Groups > memory | Hybrid hash aggregation (spill partitions, merge results) |
| Window | Partition > memory | Sort-spill per window partition |

### Memory Limit Enforcement

A per-query memory budget with enforcement at the allocator level:

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

struct MemoryBudget {
    limit: usize,
    used: AtomicUsize,
}

impl MemoryBudget {
    fn new(limit_bytes: usize) -> Self {
        MemoryBudget {
            limit: limit_bytes,
            used: AtomicUsize::new(0),
        }
    }

    fn try_reserve(&self, bytes: usize) -> Result<MemoryReservation, MemoryError> {
        let current = self.used.fetch_add(bytes, Ordering::SeqCst);
        if current + bytes > self.limit {
            self.used.fetch_sub(bytes, Ordering::SeqCst);
            return Err(MemoryError::BudgetExceeded {
                limit: self.limit,
                requested: bytes,
                used: current,
            });
        }
        Ok(MemoryReservation {
            budget: self,
            bytes,
        })
    }
}

struct MemoryReservation<'a> {
    budget: &'a MemoryBudget,
    bytes: usize,
}

impl Drop for MemoryReservation<'_> {
    fn drop(&mut self) {
        self.budget.used.fetch_sub(self.bytes, Ordering::SeqCst);
    }
}

// Usage:
let budget = MemoryBudget::new(2 * 1024 * 1024 * 1024); // 2GB
let reservation = budget.try_reserve(512 * 1024 * 1024)?; // reserve 512MB
// ... use memory ...
drop(reservation); // release back
```

### String Interning — Eliminate Duplicate Strings

Analytics datasets have many repeated strings (dimension values: cities, statuses, categories). Interning stores one canonical copy per distinct value:

```rust
use lasso::{Rodeo, Spur};

struct StringInterner {
    rodeo: Rodeo<Spur>,
}

impl StringInterner {
    fn new() -> Self {
        StringInterner { rodeo: Rodeo::new() }
    }

    fn intern(&mut self, s: &str) -> Spur {
        self.rodeo.get_or_intern(s)
    }

    fn resolve(&self, spur: Spur) -> &str {
        self.rodeo.resolve(&spur)
    }
}

fn intern_dimension_column(column: &StringArray) -> (Vec<Spur>, StringInterner) {
    let mut interner = StringInterner::new();
    let spurs: Vec<Spur> = (0..column.len())
        .map(|i| interner.intern(column.value(i)))
        .collect();
    (spurs, interner)
}
```

**Memory comparison** — 10M rows, 1000 distinct city names:

| Approach | Memory |
|----------|--------|
| `Vec<String>` (each allocated) | ~300 MB |
| `Vec<&str>` (references to centralized store) | ~80 MB |
| `lasso::Spur` (u32 interned) | ~40 MB |
| DictionaryArray (i16 keys) | ~20 MB |

### Allocator Selection

The global allocator matters at GB-scale. `snmalloc` and `jemalloc` reduce fragmentation:

```rust
// In main.rs or lib.rs:
#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static ALLOC: snmalloc_rs::SnMalloc = snmalloc_rs::SnMalloc;

// snmalloc provides:
//  - Message-passing based allocator (good for multi-threaded analytics)
//  - Low fragmentation for varied allocation sizes (8B to 1GB)
//  - Better cache locality than default system allocator
```

### Memory Profiling Pattern

```rust
use std::alloc::{GlobalAlloc, Layout, System};

struct TrackingAllocator;

unsafe impl GlobalAlloc for TrackingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = System.alloc(layout);
        if !ptr.is_null() {
            log_alloc(layout.size()); // track in metrics
        }
        ptr
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        log_dealloc(layout.size());
        System.dealloc(ptr, layout);
    }
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Unbounded memory per query — must configure `memory_limit` and enforce spill** | Single query can OOM the entire process; cascading failure |
| 2 | **Use arena (bumpalo) for query-scoped intermediates, not `Box::new`** | Global allocator thrashing; 10× slower allocation |
| 3 | **String columns with ≤100K distinct values must use interning or dictionary encoding** | 10-50× memory bloat for dimension columns |
| 4 | **Spill-to-disk must use `O_DIRECT` or `tempfile` to avoid OS page cache pollution** | Double memory: once in app, once in OS page cache |
| 5 | **Never `clone()` intermediate RecordBatches in hot loops** | Memory doubles each step; 8-hop pipeline = 8× memory |

---

## References

- [bumpalo crate](https://docs.rs/bumpalo/latest/bumpalo/)
- [lasso crate — string interning](https://docs.rs/lasso/latest/lasso/)
- [string_cache crate](https://docs.rs/string_cache/latest/string_cache/)
- [snmalloc-rs](https://docs.rs/snmalloc-rs/latest/snmalloc_rs/)
- [tempfile crate](https://docs.rs/tempfile/latest/tempfile/)
- [DataFusion MemoryManager](https://docs.rs/datafusion/latest/datafusion/execution/memory_manager/index.html)