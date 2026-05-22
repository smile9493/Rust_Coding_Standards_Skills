# 06 — Partitioning & Shuffling

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Hash functions, Rust `HashMap`, `tokio` async I/O, distributed computing concepts |
| Aligned Crates | `datafusion`, `polars-core`, `ahash`, `tokio` |

---

## Core Architecture

Partitioning determines how data is distributed across parallel execution units (threads, tasks, or machines). The goal is data-parallel execution: each partition processes independently, then results are merged.

### Hash Partitioning — Deterministic Routing

Hash partitioning is the workhorse of join and aggregate distribution:

```rust
use std::hash::{Hash, Hasher};
use ahash::AHasher;
use arrow::array::StringArray;

struct HashPartitioner {
    num_partitions: usize,
}

impl HashPartitioner {
    fn partition_for_key(&self, key: &str) -> usize {
        let mut hasher = AHasher::default();
        key.hash(&mut hasher);
        (hasher.finish() as usize) % self.num_partitions
    }

    fn partition_batch(&self, batch: &RecordBatch, key_col: usize) -> Vec<Vec<usize>> {
        let keys = batch.column(key_col).as_any().downcast_ref::<StringArray>().unwrap();
        let mut partitions: Vec<Vec<usize>> = vec![Vec::new(); self.num_partitions];

        for i in 0..batch.num_rows() {
            let part = self.partition_for_key(keys.value(i));
            partitions[part].push(i);
        }
        partitions
    }
}
```

**Critical property**: `hash(key) % N` ensures the same key always lands in the same partition — essential for correctness in joins and group-by aggregations.

### Range Partitioning — Sort-Based Distribution

Range partitioning divides data by value boundaries, useful for sort-merge joins and ordered output:

```rust
fn range_partition(values: &[i64], num_partitions: usize) -> Vec<Vec<i64>> {
    let mut sorted: Vec<(i64, usize)> = values.iter().enumerate()
        .map(|(i, &v)| (v, i))
        .collect();
    sorted.sort_by_key(|(v, _)| *v);

    let rows_per_partition = values.len() / num_partitions;
    let mut partitions = vec![Vec::new(); num_partitions];

    for (part_idx, chunk) in sorted.chunks(rows_per_partition).enumerate() {
        for &(val, _orig_idx) in chunk {
            partitions[part_idx].push(val);
        }
    }
    partitions
}
```

**Cost**: Requires a full sort of the column — O(n log n). Use only when sort order matters (e.g., `ORDER BY`, `SortMergeJoin`).

### Broadcast Join — Small Table Replication

When one side of a join is small (fits in memory), replicate it to all partitions:

```rust
use std::sync::Arc;

struct BroadcastJoin {
    build_side: Arc<HashMap<String, Vec<usize>>>, // key → row indices
    probe_side: Vec<RecordBatch>,
}

impl BroadcastJoin {
    fn new(build_batches: &[RecordBatch], key_col: usize) -> Self {
        let mut index: HashMap<String, Vec<usize>> = HashMap::new();
        let mut row_offset = 0usize;

        for batch in build_batches {
            let keys = batch.column(key_col).as_any().downcast_ref::<StringArray>().unwrap();
            for i in 0..batch.num_rows() {
                index.entry(keys.value(i).to_string())
                    .or_default()
                    .push(row_offset + i);
            }
            row_offset += batch.num_rows();
        }

        BroadcastJoin {
            build_side: Arc::new(index),
            probe_side: vec![],
        }
    }

    fn probe(&self, batch: &RecordBatch, key_col: usize) -> RecordBatch {
        let keys = batch.column(key_col).as_any().downcast_ref::<StringArray>().unwrap();
        let build_index = &self.build_side;

        for i in 0..batch.num_rows() {
            if let Some(matches) = build_index.get(keys.value(i)) {
                // emit joined rows: one per match
            }
        }
        // ... construct result batch ...
        todo!()
    }
}
```

**Threshold**: Broadcast when build side row_count ≤ 10,000 and total size < 100MB. Above that, use hash-partitioned join.

### Data Skew Detection

Skew is measured as the ratio of largest-partition-size to average-partition-size:

```rust
struct SkewDetector {
    partition_counts: Vec<usize>,
    threshold_multiplier: f64, // default: 10.0
}

impl SkewDetector {
    fn new(num_partitions: usize) -> Self {
        SkewDetector {
            partition_counts: vec![0; num_partitions],
            threshold_multiplier: 10.0,
        }
    }

    fn record_row(&mut self, partition: usize) {
        self.partition_counts[partition] += 1;
    }

    fn detect_skewed_partitions(&self) -> Vec<(usize, usize)> {
        let total: usize = self.partition_counts.iter().sum();
        let avg = total as f64 / self.partition_counts.len() as f64;
        let threshold = (avg * self.threshold_multiplier) as usize;

        self.partition_counts.iter()
            .enumerate()
            .filter(|(_, &count)| count > threshold)
            .map(|(idx, &count)| (idx, count))
            .collect()
    }
}
```

**Example**: 8 partitions, total 1M rows, avg 125K. Partition 3 has 1.5M rows → 12× average → flagged as skewed.

### Salt Mitigation

For skewed hash partitions, add a random salt to break hot keys across artificial sub-partitions:

```rust
struct SaltedPartitioner {
    base_partitions: usize,
    salt_range: usize, // split hot keys across this many sub-partitions
}

impl SaltedPartitioner {
    fn partition_for_key(&self, key: &str, is_hot: bool) -> usize {
        let base_hash = {
            let mut h = AHasher::default();
            key.hash(&mut h);
            h.finish() as usize
        };

        if is_hot {
            let salt = rand::thread_rng().gen_range(0..self.salt_range);
            (base_hash ^ salt) % self.base_partitions
        } else {
            base_hash % self.base_partitions
        }
    }
}

fn mitigate_skew(
    key: &str,
    hot_keys: &HashSet<String>,
    partitioner: &SaltedPartitioner,
) -> (usize, Option<usize>) {
    if hot_keys.contains(key) {
        let salt = rand::thread_rng().gen_range(0..partitioner.salt_range);
        (partitioner.partition_for_key(key, true), Some(salt))
    } else {
        (partitioner.partition_for_key(key, false), None)
    }
}
```

**The salt adds a random suffix to the hash, scattering the same logical key across multiple physical partitions.** The downstream aggregator must then perform a second-level merge per salted key.

### Shuffle — Network Transfer

When partitions span multiple nodes, shuffle moves data via async network I/O:

```rust
use tokio::sync::mpsc;

struct ShuffleSender {
    channels: Vec<mpsc::Sender<RecordBatch>>,
}

impl ShuffleSender {
    async fn shuffle_batch(&self, batch: RecordBatch, partitioner: &HashPartitioner, key_col: usize) {
        let keys = batch.column(key_col).as_any().downcast_ref::<StringArray>().unwrap();

        // Pre-group rows by destination partition
        let mut partition_buffers: Vec<Vec<usize>> = vec![Vec::new(); self.channels.len()];
        for i in 0..batch.num_rows() {
            let dest = partitioner.partition_for_key(keys.value(i));
            partition_buffers[dest].push(i);
        }

        // Send each partition's slice to the corresponding channel
        for (dest, rows) in partition_buffers.into_iter().enumerate() {
            if rows.is_empty() { continue; }
            let indices = UInt64Array::from(rows.into_iter().map(|r| r as u64).collect::<Vec<_>>());
            let sub_batch = take_record_batch(&batch, &indices).unwrap();
            self.channels[dest].send(sub_batch).await.unwrap();
        }
    }
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Data skew > 10× must be detected and mitigated** (salt, split, or broadcast replan) | Single partition becomes straggler; 99% of wall time on 1 thread |
| 2 | **Hash function must be deterministic and uniformly distributed** | `hash(key) % N` with poor hash → partition imbalance |
| 3 | **Broadcast only tables < 10K rows / < 100MB** | O(n²) memory on each executor; guaranteed OOM |
| 4 | **Shuffle channels must use bounded mpsc channels** (backpressure) | Unbounded queue causes unbounded memory during spike |
| 5 | **Skew detection must run on sampled data before full execution** | Full execution of skewed plan wastes resources; detection too late |

---

## References

- [DataFusion Partitioning](https://docs.rs/datafusion/latest/datafusion/physical_plan/enum.Partitioning.html)
- [Apache Spark Partitioning & Skew Handling](https://spark.apache.org/docs/latest/sql-performance-tuning.html) (design reference)
- [ahash crate](https://docs.rs/ahash/latest/ahash/)
- [Tokio mpsc channels](https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html)