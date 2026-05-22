# 02 — Vectorized Expression Evaluation

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Arrow columnar memory model, trait objects (`dyn Array`), Rust `std::simd` (nightly) |
| Aligned Crates | `arrow-rs`, `datafusion`, `polars-core`, `std::simd` |

---

## Core Architecture

Vectorized expression evaluation replaces row-by-row processing with bulk columnar operations. The expression tree is compiled into a pipeline of compute kernels that operate on entire `Array` instances, utilizing SIMD instructions and bitmap-level parallelism.

### Expression Tree Model

```
SELECT price * quantity * (1.0 - discount) FROM orders
```

translates to:

```
            Multiply
           /        \
      Multiply    Subtract
      /     \     /      \
   price  quantity 1.0  discount
```

Each node is a `PhysicalExpression` that produces a `ColumnarValue` — either an `Array` or a scalar broadcast to all rows:

```rust
use arrow::array::{Array, Float64Array};
use arrow::datatypes::DataType;
use std::sync::Arc;

enum ColumnarValue {
    Array(Arc<dyn Array>),
    Scalar(f64), // broadcast to N rows at evaluation time
}

trait PhysicalExpression {
    fn evaluate(&self, batch: &RecordBatch) -> ColumnarValue;
    fn data_type(&self, schema: &Schema) -> DataType;
}
```

### SIMD Batch Evaluation

For arithmetic expressions, SIMD processes lanes of data simultaneously. Example: multiplying two float64 columns using `std::simd`:

```rust
#![feature(portable_simd)]
use std::simd::f64x8;

fn simd_multiply(a: &[f64], b: &[f64], out: &mut [f64], len: usize) {
    let chunks = len / 8;
    for i in 0..chunks {
        let idx = i * 8;
        let va = f64x8::from_slice(&a[idx..]);
        let vb = f64x8::from_slice(&b[idx..]);
        let vc = va * vb;
        vc.copy_to_slice(&mut out[idx..]);
    }
    // scalar remainder for last len % 8 elements
    for i in (chunks * 8)..len {
        out[i] = a[i] * b[i];
    }
}
```

**In practice**, `arrow::compute::kernels::arithmetic::multiply` handles SIMD dispatch via auto-vectorization or explicit SIMD depending on target features. The user writes `multiply(col_a, col_b)?` and the kernel does the rest.

### Bitmap Filtering — Branch-Free Selection

Filtering uses a `BooleanArray` as a selection mask. `arrow::compute::filter()` processes the bitmap in bulk:

```rust
use arrow::compute::filter_record_batch;
use arrow::array::BooleanArray;

fn filter_batch(batch: &RecordBatch, keep_mask: &BooleanArray) -> RecordBatch {
    // Internally: evaluates bitmap word-by-word,
    // no per-row if-branch in hot loop
    filter_record_batch(batch, keep_mask).unwrap()
}
```

The kernel uses **population-count** (popcount) on the bitmap to compute output indices. Reading the bitmap is branchless — each word is processed with bitwise operations.

**Custom bitmap filter with pre-computed validity:**

```rust
use arrow::array::{Array, Int64Array, BooleanArray};
use arrow::buffer::{BooleanBuffer, MutableBuffer};

fn build_predicate_array(array: &Int64Array, threshold: i64) -> BooleanArray {
    let mut bits = MutableBuffer::new(array.len().div_ceil(8));
    let raw = array.values();

    for (i, &val) in raw.iter().enumerate() {
        if val > threshold {
            let byte_idx = i / 8;
            let bit_idx = i % 8;
            bits[byte_idx] |= 1 << bit_idx;
        }
    }
    BooleanArray::new(BooleanBuffer::new(bits.into(), 0, array.len()), None)
}
```

### Null Bitmap Propagation

Every Arrow array carries a null bitmap. Expressions must propagate nulls without branching:

```
Null Propagation Rule:
  if any operand is null → result is null
```

The kernel computes `result_null_bitmap = operand1_null & operand2_null` using **bitwise AND** on bitmap words — no per-element branch:

```rust
use arrow::buffer::BooleanBuffer;

fn combine_nulls(a_valid: &BooleanBuffer, b_valid: &BooleanBuffer) -> BooleanBuffer {
    let mut combined = BooleanBuffer::new_unset(a_valid.len());
    // word-level bitwise AND — one instruction per 64 elements
    for (dst, (a_word, b_word)) in combined.inner().iter_mut()
        .zip(a_valid.inner().iter().zip(b_valid.inner().iter()))
    {
        *dst = a_word & b_word;
    }
    combined
}
```

### Branch Avoidance in Conditional Expressions

CASE WHEN is the classic branching challenge. The vectorized approach evaluates all branches and merges:

```rust
use arrow::compute::{zip, is_not_null};

fn case_when(
    condition: &BooleanArray,
    then_col: &Float64Array,
    else_col: &Float64Array,
) -> Float64Array {
    // zip selects from then_col where condition=true, else_col where false.
    // Computationally: no if/else per row — uses bulk merge
    let valid_mask = is_not_null(condition).unwrap();
    zip(&valid_mask, then_col, else_col).unwrap()
}
```

### Chained Expression Pipeline

A real-world expression like `col_a + col_b * col_c` should minimize intermediate allocations:

```rust
use arrow::compute::kernels::arithmetic::{add, multiply};
use arrow::array::{Int64Array, Array};
use std::sync::Arc;

fn evaluate_batch(col_a: &Int64Array, col_b: &Int64Array, col_c: &Int64Array) -> Int64Array {
    // Step 1: b * c — intermediate allocation
    let product = multiply(col_b, col_c).unwrap();

    // Step 2: a + product — second allocation
    let result = add(col_a, &product).unwrap();

    // Limitation: two intermediate arrays allocated.
    // Production engines fuse these via expression JIT or multi-output kernels.
    result
}
```

**Fusion pattern** (simplified): For `col_a + col_b * col_c`, the optimizer can combine into a single pass:

```rust
fn fused_add_mul(a: &[i64], b: &[i64], c: &[i64], out: &mut [i64], len: usize) {
    for i in 0..len {
        out[i] = a[i] + b[i] * c[i]; // single pass, single write
    }
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **No `for row in df.iter()` on hot paths** | Scalar evaluation is 50-200x slower; cache misses explode |
| 2 | **No per-row `if null {}` in expression evaluation** — must propagate via bitmap word operations | Branch misprediction destroys pipeline throughput |
| 3 | **Always pre-allocate output buffers** — compute output length from bitmap popcount | Repeated `Vec::push` triggers reallocation chains |
| 4 | **Fuse multi-step expressions into single-pass kernels when memory-bound** | Intermediate arrays double memory pressure |
| 5 | **Use `BooleanArray` + `filter()` for selection, never iterate and collect** | Collect-based filtering is O(n²) due to repeated reallocations |

---

## References

- [DataFusion Physical Expressions](https://docs.rs/datafusion/latest/datafusion/physical_expr/index.html)
- [Polars Expression Architecture](https://docs.pola.rs/user-guide/expressions/)
- [Rust SIMD Documentation](https://doc.rust-lang.org/std/simd/index.html)
- [Arrow Compute Kernels](https://docs.rs/arrow/latest/arrow/compute/index.html)