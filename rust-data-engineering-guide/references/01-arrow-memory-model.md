# 01 — Arrow Columnar Memory Model

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Understanding of Rust ownership (`Box`, `Arc`), `Vec<T>` layout |
| Aligned Crates | `arrow-rs` (Apache Arrow Rust), `arrow2` (legacy), `polars-arrow` |

---

## Core Architecture

Apache Arrow defines a columnar memory layout where data is organized as contiguous arrays of homogeneous types rather than row-oriented structs. This layout is the physical foundation for all zero-copy operations in the query engine stack.

### Three-Layer Data Model

```
Schema  →  logical column names + types (metadata only)
  │
RecordBatch → a table slice: one Schema + N Arrays of equal length
  │
Array → typed data buffer + optional null bitmap + optional offsets
```

A `RecordBatch` is the unit of work in vectorized execution. Each column within a batch is an `Array` subclass: `Int32Array`, `StringArray`, `ListArray`, `StructArray`, etc.

### Physical Buffer Layout

Every Arrow array is backed by one or more contiguous `Buffer` objects — byte slices with reference counting:

```
PrimitiveArray<i64>:
  ┌──────────────────┐
  │ null_bitmap      │  Buffer (ceil(n/8) bytes, 1 = valid)
  │ values           │  Buffer (n × 8 bytes, i64 aligned)
  └──────────────────┘

StringArray:
  ┌──────────────────┐
  │ null_bitmap      │  Buffer
  │ offsets          │  Buffer ((n+1) × 4/8 bytes, offset into data)
  │ data             │  Buffer (all UTF-8 payloads concatenated)
  └──────────────────┘
```

This is fundamentally different from `Vec<Option<String>>` — there is no pointer indirection per element, and cache lines are filled with homogeneous data.

### ArrayData & Shared Ownership

The internal representation is `ArrayData`, wrapped in `Arc` for zero-copy sharing:

```rust
use arrow::array::{Int32Array, ArrayData};
use arrow::buffer::Buffer;
use arrow::datatypes::DataType;
use std::sync::Arc;

fn build_primitive_array() -> Int32Array {
    let raw_data = Buffer::from(&[1i32, 2, 3, 4, 5][..]);
    let null_bitmap = Buffer::from(&[0b00011111u8]);

    let array_data = ArrayData::builder(DataType::Int32)
        .len(5)
        .add_buffer(raw_data)
        .null_bit_buffer(Some(null_bitmap))
        .build()
        .unwrap();

    Int32Array::from(array_data)
}
```

**Critical pattern**: Multiple operators share `Arc<ArrayData>` without copying. A filter operation produces a new `ArrayData` that references the original buffers with a different null bitmap — the payload buffers are never copied.

### FixedSizeList & Nested Arrays

`FixedSizeListArray` stores fixed-width sub-arrays contiguously — commonly used for embedding vectors and geo-coordinates:

```rust
use arrow::array::{FixedSizeListArray, Float64Array};
use arrow::datatypes::{DataType, Field};

fn build_embedding_array() -> FixedSizeListArray {
    let values = Float64Array::from(vec![0.1, 0.2, 0.3, 0.4, 0.5, 0.6]);
    let list_data_type = DataType::FixedSizeList(
        Arc::new(Field::new("item", DataType::Float64, true)),
        3,
    );

    FixedSizeListArray::try_new(list_data_type, Arc::new(values), None).unwrap()
}
```

The inner `values` buffer is flat — 6 f64 values — and the list boundaries are implicit from the fixed size. No per-element offset indirection.

### DictionaryArray — Factorized Encoding

For columns with low cardinality (e.g., `status`, `city`, `country`), `DictionaryArray` stores integer keys pointing into a shared value dictionary:

```rust
use arrow::array::{DictionaryArray, Int16Array, StringArray};
use arrow::datatypes::DataType;

fn build_dictionary() -> DictionaryArray<Int16Type> {
    let keys = Int16Array::from(vec![0i16, 1, 2, 0, 1, 0]);
    let values = Arc::new(StringArray::from(vec!["active", "paused", "deleted"]));

    // keys: [0,1,2,0,1,0] → values: ["active","paused","deleted","active","paused","active"]
    DictionaryArray::new(keys, values)
}
```

**Rationale**: With 1M rows and 100 distinct city names, dictionary encoding reduces the string column from ~20MB (full UTF-8) to ~2MB (i16 keys) + a tiny dictionary, fitting in L2 cache.

### Validate Data Integrity

`validate()` is the gate for data quality enforcement across record batches:

```rust
use arrow::array::{Array, Int64Array};

fn validate_example(arr: &Int64Array) {
    assert!(arr.null_count() <= arr.len() / 2,
        "null ratio > 50%, check upstream data source");

    for i in 0..arr.len() {
        if arr.is_valid(i) {
            let val = arr.value(i);
            assert!(val >= 0, "negative value at index {}", i);
        }
    }
}
```

**Production pattern**: validate() runs at the boundary (after deserialization, before compute) to catch corrupted Parquet files or broken upstream pipelines. Failing early prevents downstream corruption.

### Compute Kernels — Zero-Copy by Default

Arrow compute kernels operate on arrays and return new arrays. They leverage bitmap propagation and SIMD internally:

```rust
use arrow::compute::{add, filter_record_batch, lt_scalar};
use arrow::array::{Int64Array, BooleanArray};
use arrow::record_batch::RecordBatch;

fn filter_and_compute(batch: &RecordBatch) -> RecordBatch {
    let col_a = batch.column(0).as_any().downcast_ref::<Int64Array>().unwrap();

    let predicate: BooleanArray = lt_scalar(col_a, 100).unwrap();
    let filtered = filter_record_batch(batch, &predicate).unwrap();

    let col_a_filtered = filtered.column(0).as_any().downcast_ref::<Int64Array>().unwrap();
    let col_b = filtered.column(1).as_any().downcast_ref::<Int64Array>().unwrap();
    let sum = add(col_a_filtered, col_b).unwrap(); // SIMD-accelerated on filtered data

    // build a new RecordBatch — buffers from original are NOT copied
    RecordBatch::try_from_iter(vec![
        ("col_a", Arc::new(col_a_filtered.clone())),
        ("sum", Arc::new(sum)),
    ])
    .unwrap()
}
```

Key point: `add(col_a, col_b)` internally uses `arrow::compute::kernels::arithmetic` which dispatches to SIMD-accelerated implementations. No element-level loop in user code.

### RecordBatch — The Execution Currency

A `RecordBatch` is a Schema + a Vec of Arrays, all equal length. It is the fundamental unit passed between operators:

```rust
use arrow::record_batch::RecordBatch;
use arrow::datatypes::{Schema, Field, DataType};
use std::sync::Arc;

fn make_batch() -> RecordBatch {
    let schema = Schema::new(vec![
        Field::new("id", DataType::Int64, false),
        Field::new("name", DataType::Utf8, true),
        Field::new("score", DataType::Float64, true),
    ]);

    RecordBatch::try_new(
        Arc::new(schema),
        vec![
            Arc::new(Int64Array::from(vec![1, 2, 3, 4])),
            Arc::new(StringArray::from(vec!["a", "b", "c", "d"])),
            Arc::new(Float64Array::from(vec![0.1, 0.2, 0.3, 0.4])),
        ],
    )
    .unwrap()
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Avoid deep buffer copies in hot paths** — pass `Arc<ArrayData>` or `Arc<dyn Array>` between operators. Explicit copies are permitted at shuffle/spill/encode boundaries. | Memory doubles under GB-scale data if copying on every operator |
| 2 | **Never iterate element-by-element to compute aggregates on hot paths** — use compute kernels | 100x+ slower than vectorized; branch predictor destroyed |
| 3 | **Always validate() at ingestion boundary** | Corrupted data silently propagates into downstream aggregations |
| 4 | **Use `DictionaryArray<Int16Type>` for columns with ≤ 2^15 distinct values** | String columns waste L2/L3 cache and increase deserialization cost |
| 5 | **RecordBatch length must be consistent across all columns** | `len()` mismatch breaks all downstream kernels silently |

---

## References

- [Apache Arrow Memory Layout Specification](https://arrow.apache.org/docs/format/Columnar.html)
- [arrow-rs crate documentation](https://docs.rs/arrow/latest/arrow/)
- [Arrow Columnar Format — Physical Layout](https://arrow.apache.org/docs/format/Columnar.html#physical-memory-layout)
- DataFusion Physical Plan: `physical_plan/src/expressions/` for kernel usage examples