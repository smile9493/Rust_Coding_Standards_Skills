# 04 — Parquet & Storage Layer

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Columnar layout concepts, file I/O, encoding fundamentals (RLE, delta, dictionary) |
| Aligned Crates | `parquet` (arrow-rs), `datafusion`, `polars-io` |

---

## Core Architecture

Apache Parquet is a columnar, compressed, and self-describing file format. Its hierarchical structure enables fine-grained data skipping and predicate pushdown directly at the storage layer.

### File Structure Hierarchy

```
File
├── Footer (metadata)
│   ├── Schema
│   ├── Row Group metadata (column stats, offsets)
│   └── Version / created_by
│
├── Row Group 0          (~128MB)
│   ├── Column Chunk "col_a"
│   │   ├── Page 0       (dictionary page)
│   │   ├── Page 1       (data page, RLE-encoded)
│   │   └── Page 2       (data page)
│   ├── Column Chunk "col_b"
│   │   └── ...
│   └── Column Chunk "col_c"
│       └── ...
├── Row Group 1
│   └── ...
└── Row Group N
```

Each row group is an independently-readable unit. A scan of 1000 row groups can skip 990 of them via statistics.

### Reading with RowSelection — Selective Row Group Loading

```rust
use parquet::file::reader::{FileReader, SerializedFileReader};
use parquet::record::Row;
use std::fs::File;

fn read_with_predicate(path: &str, threshold: i64, column: usize) {
    let file = File::open(path).unwrap();
    let reader = SerializedFileReader::new(file).unwrap();
    let metadata = reader.metadata();
    let file_metadata = metadata.file_metadata();

    let mut total_rows = 0usize;
    for rg_idx in 0..file_metadata.num_row_groups() {
        let rg_meta = metadata.row_group(rg_idx);
        let col_meta = rg_meta.column(column);
        let stats = col_meta.statistics();

        // Skip row group if max value < threshold
        if let Some(max_val) = stats.max() {
            let max_i64 = i64::from_le_bytes(max_val.as_bytes()[..8].try_into().unwrap());
            if max_i64 < threshold {
                continue; // entire row group can be skipped
            }
        }

        // Only read this row group
        let row_group = reader.get_row_group(rg_idx).unwrap();
        let col_reader = row_group.get_column_reader(column).unwrap();
        // ... process column pages ...
        total_rows += row_group.num_rows();
    }
}
```

**Production pattern**: The predicate pushdown optimizer produces a `RowSelection` that encodes exactly which row groups and pages to read. The storage layer then seeks directly to those byte ranges.

### Writing Parquet — Row Groups & Column Chunks

```rust
use parquet::file::writer::SerializedFileWriter;
use parquet::data_type::Int64Type;
use parquet::file::properties::WriterProperties;
use parquet::schema::parser::parse_message_type;
use std::sync::Arc;

fn write_parquet() {
    let schema_str = "
        message schema {
            required int64 timestamp;
            required int64 user_id;
            required float amount;
            optional binary event_type (STRING);
        }
    ";
    let schema = Arc::new(parse_message_type(schema_str).unwrap());

    let props = WriterProperties::builder()
        .set_compression(parquet::basic::Compression::ZSTD) // modern default
        .set_max_row_group_size(128 * 1024 * 1024)          // 128MB
        .set_dictionary_enabled(true)
        .build();
}
```

### Column Statistics — The Query Skip Engine

Per column chunk, Parquet stores:

| Statistic | Storage cost | Skip benefit |
|-----------|-------------|--------------|
| `null_count` | 8 bytes | Skip column entirely if all null |
| `min_value` | sizeof(T) | Skip if filter_val > max |
| `max_value` | sizeof(T) | Skip if filter_val < min |
| `distinct_count` | 8 bytes | Cardinality estimation for CBO |
| `total_compressed_size` | 8 bytes | IO cost estimation |

```rust
fn parquet_stats_skip_example() {
    // Column: "timestamp" with min=2024-01-01, max=2024-12-31
    // Query: WHERE timestamp >= '2025-01-01'
    // → max(2024-12-31) < threshold(2025-01-01) → skip entire row group
}
```

### Encoding Selection

Parquet supports multiple encodings per column. Choosing the right one matters:

```rust
use parquet::basic::Encoding;

// Encoding selection heuristic
fn choose_encoding(values: &[i64], distinct_count: usize) -> Encoding {
    if distinct_count < values.len() / 20 {
        Encoding::DELTA_BINARY_PACKED // sorted timestamps / IDs
    } else if distinct_count < 10_000 {
        Encoding::RLE_DICTIONARY // low-cardinality categorical
    } else {
        Encoding::PLAIN // high-cardinality, no pattern
    }
}
```

**Encoding table:**

| Encoding | Best for | Compressed size (vs PLAIN) |
|----------|----------|----------------------------|
| PLAIN | High-cardinality numeric, no pattern | Baseline |
| DICTIONARY + RLE | Low-cardinality strings (≤10K distinct) | 5-20× smaller |
| DELTA_BINARY_PACKED | Sorted / monotonic integers | 2-10× smaller |
| DELTA_LENGTH_BYTE_ARRAY | Strings with similar lengths | 1.5-3× smaller |
| BYTE_STREAM_SPLIT | Float arrays (separate sign/exponent/mantissa) | 1.5-3× smaller |

### Predicate Pushdown to Storage — Full Flow

```rust
use datafusion::prelude::*;
use datafusion::datasource::parquet::ParquetTable;

async fn predicate_pushdown_example() -> datafusion::error::Result<()> {
    let ctx = SessionContext::new();
    let table = ParquetTable::try_new(
        "data/orders.parquet",
        Default::default(),
    ).await?;

    ctx.register_table("orders", Arc::new(table))?;

    let df = ctx.sql(
        "SELECT user_id, SUM(amount) FROM orders \
         WHERE timestamp >= '2025-01-01' \
         GROUP BY user_id"
    ).await?;

    // DataFusion pushes the timestamp filter to the Parquet reader.
    // The reader checks row group min/max statistics and skips
    // row groups that cannot contain matching data.
    df.show().await?;
    Ok(())
}
```

### Parquet Footer — Metadata at the End

The footer is read first (seek to end – 8 bytes for magic number – 4 bytes for footer length). It provides all row group offsets, column statistics, and schema information. This "trailer-first" design means query engines can skip the data payload entirely for plan optimization.

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Never read entire Parquet file when predicate can be evaluated via statistics** | IO cost = full file scan; defeats purpose of columnar storage |
| 2 | **Always set `max_row_group_size` to ~128MB** (balance parallelism vs metadata overhead) | Too small: excessive metadata bloat. Too large: poor parallelism, memory spikes |
| 3 | **Use ZSTD compression by default** (best ratio/speed trade-off for analytics) | GZIP: slow decode. LZ4: poor ratio. Snappy: legacy. SNAPPY ok for latency-critical |
| 4 | **Enable dictionary encoding for string columns** unless distinct count > 100K | Uncompressed strings dominate storage and I/O |
| 5 | **Validate column statistics after writing** — corrupt stats cause silent data skipping | Wrong min/max cause query to miss valid rows |

---

## References

- [Apache Parquet Format Specification](https://parquet.apache.org/docs/file-format/)
- [parquet crate (arrow-rs)](https://docs.rs/parquet/latest/parquet/)
- [Parquet Encoding Specification](https://parquet.apache.org/docs/file-format/data-pages/encodings/)
- [DataFusion Parquet Datasource](https://docs.rs/datafusion/latest/datafusion/datasource/parquet/index.html)