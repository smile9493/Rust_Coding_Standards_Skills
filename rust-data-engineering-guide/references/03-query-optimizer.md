# 03 — Query Optimizer Design

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | SQL logical operators, relational algebra, Rust trait system (`enum`, `Box<dyn>`), `Arc` ownership |
| Aligned Crates | `datafusion`, `polars-lazy`, `sqlparser-rs` |

---

## Core Architecture

The optimizer transforms a naive logical plan into an efficient physical plan in two stages:

```
SQL Text
  │  sqlparser-rs
  ▼
AST (Raw parse tree)
  │  planner
  ▼
Logical Plan (unoptimized)
  │  Rule-Based Optimizer (RBO)
  ▼
Optimized Logical Plan
  │  Cost-Based Optimizer (CBO)
  ▼
Physical Plan
  │  execution engine
  ▼
RecordBatch stream
```

### Logical Plan Representation

A plan node is an enum. Each variant carries child plans and metadata:

```rust
use std::sync::Arc;

enum LogicalPlan {
    Scan {
        source: String,
        projection: Option<Vec<usize>>,
        filters: Vec<Expression>,
    },
    Filter {
        predicate: Expression,
        input: Arc<LogicalPlan>,
    },
    Projection {
        exprs: Vec<Expression>,
        input: Arc<LogicalPlan>,
    },
    Aggregate {
        group_keys: Vec<Expression>,
        aggs: Vec<Expression>,
        input: Arc<LogicalPlan>,
    },
    Join {
        left: Arc<LogicalPlan>,
        right: Arc<LogicalPlan>,
        join_type: JoinType,
        condition: Expression,
    },
    Sort {
        order: Vec<SortExpr>,
        input: Arc<LogicalPlan>,
    },
}
```

### Rule-Based Optimization (RBO)

Each rule is a function: `LogicalPlan → Transformed<LogicalPlan>`. Rules are applied iteratively until a fixpoint.

**Predicate Pushdown** — moves filters as close to Scan as possible:

```rust
use std::sync::Arc;

fn pushdown_predicate(plan: &LogicalPlan) -> LogicalPlan {
    match plan {
        LogicalPlan::Filter { predicate, input } => {
            match input.as_ref() {
                LogicalPlan::Scan { source, projection, filters } => {
                    let mut new_filters = filters.clone();
                    new_filters.push(predicate.clone());
                    LogicalPlan::Scan {
                        source: source.clone(),
                        projection: projection.clone(),
                        filters: new_filters,
                    }
                }
                LogicalPlan::Projection { exprs, input: proj_input } => {
                    LogicalPlan::Projection {
                        exprs: exprs.clone(),
                        input: Arc::new(pushdown_predicate(&LogicalPlan::Filter {
                            predicate: predicate.clone(),
                            input: proj_input.clone(),
                        })),
                    }
                }
                _ => LogicalPlan::Filter {
                    predicate: predicate.clone(),
                    input: Arc::new(pushdown_predicate(input)),
                },
            }
        }
        LogicalPlan::Projection { exprs, input } => {
            LogicalPlan::Projection {
                exprs: exprs.clone(),
                input: Arc::new(pushdown_predicate(input)),
            }
        }
        other => other.clone(),
    }
}
```

**Projection Pruning** — removes unused columns at the Scan level:

```rust
fn prune_projection(plan: &LogicalPlan, required_cols: &[usize]) -> LogicalPlan {
    match plan {
        LogicalPlan::Scan { source, filters, .. } => {
            LogicalPlan::Scan {
                source: source.clone(),
                projection: Some(required_cols.to_vec()),
                filters: filters.clone(),
            }
        }
        LogicalPlan::Projection { exprs, input } => {
            let ref_cols: Vec<usize> = extract_referenced_columns(exprs);
            let pruned_input = prune_projection(input, &ref_cols);
            LogicalPlan::Projection {
                exprs: exprs.clone(),
                input: Arc::new(pruned_input),
            }
        }
        other => other.clone(),
    }
}
```

**Constant Folding**: `price * (1.0 - 0.2)` → `price * 0.8`

### Cost-Based Optimization (CBO)

CBO requires statistics on every plan node. Statistics drive join ordering and physical operator selection.

```rust
struct ColumnStatistics {
    null_count: usize,
    distinct_count: usize,
    min_value: Option<ScalarValue>,
    max_value: Option<ScalarValue>,
}

struct PlanStatistics {
    row_count: usize,
    column_stats: HashMap<String, ColumnStatistics>,
}

impl PlanStatistics {
    fn filter_selectivity(&self, predicate: &Expression) -> f64 {
        // Estimate selectivity: 0.0 (no rows) .. 1.0 (all rows)
        // For `col > 100` on col with min=0, max=1000:
        //   selectivity ≈ (max - 100) / (max - min) = 0.9
        match predicate {
            Expression::Gt { col, value } => {
                let stats = &self.column_stats[col];
                if let (Some(max), Some(min)) = (&stats.max_value, &stats.min_value) {
                    // simplified: assumes uniform distribution
                    let range = max.sub(min).as_f64().unwrap_or(1.0);
                    let threshold = value.as_f64().unwrap_or(0.0);
                    ((range - threshold) / range).clamp(0.0, 1.0)
                } else {
                    0.33 // default guess: 1/3 selectivity
                }
            }
            _ => 0.33,
        }
    }
}
```

**Join Reordering**: For a 3-table join, order by ascending row count:

```
A ⋈ B ⋈ C  with |A|=1M, |B|=10K, |C|=100

  Optimal: (B ⋈ C) → intermediate → (result ⋈ A)
  Worst:   (A ⋈ B) → 100M intermediate → (result ⋈ C)
```

```rust
fn reorder_joins(tables: &[(String, usize)]) -> Vec<String> {
    let mut sorted: Vec<_> = tables.iter().collect();
    sorted.sort_by_key(|(_, row_count)| *row_count);
    sorted.into_iter().map(|(name, _)| name.clone()).collect()
}
```

### Physical Plan Conversion

Physical operators are concrete implementations with fixed execution strategies:

```rust
enum PhysicalPlan {
    ParquetScan {
        path: String,
        projection: Vec<usize>,
        row_selection: Option<RowSelection>,
        statistics: Option<ParquetStatistics>,
    },
    HashJoin {
        left: Arc<PhysicalPlan>,
        right: Arc<PhysicalPlan>,
        join_keys: (Vec<String>, Vec<String>),
        join_type: JoinType,
        partition_count: usize,
    },
    SortMergeJoin {
        left: Arc<PhysicalPlan>,
        right: Arc<PhysicalPlan>,
        join_keys: (Vec<String>, Vec<String>),
    },
    ShuffleExchange {
        input: Arc<PhysicalPlan>,
        partitioning: Partitioning,
    },
}

enum Partitioning {
    Hash(Vec<String>, usize),    // hash(join_key) % partitions
    RoundRobin(usize),
    Broadcast,                   // full table replicated
}
```

**Join strategy selection** — CBO decides HashJoin vs SortMergeJoin vs Broadcast:

```rust
fn choose_join_strategy(
    left_stats: &PlanStatistics,
    right_stats: &PlanStatistics,
) -> JoinStrategy {
    if right_stats.row_count < 10_000 {
        JoinStrategy::Broadcast // small table fits in memory
    } else if right_stats.row_count > left_stats.row_count * 5 {
        JoinStrategy::HashJoin // build side is the smaller one
    } else {
        JoinStrategy::SortMergeJoin // both large, no hash table needed
    }
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Predicate pushdown must reach the storage layer** (Parquet row group statistics) | Full table scans on every query; IO cost = O(file size) |
| 2 | **Projection pruning must execute BEFORE predicate pushdown** | Unused columns still deserialized through the entire pipeline |
| 3 | **Join order must be cost-based, not syntactic** (smallest first) | Intermediate cardinality explodes; O(n³) effective cost |
| 4 | **Always collect column statistics at Scan time** (min, max, null_count, ndv) | CBO degenerates to random guess; plan quality becomes luck |
| 5 | **Never convert physical plan too early** — keep optimization opportunities in logical form | Rules cannot cross physical operator boundaries |

---

## References

- [DataFusion Optimizer Rules](https://docs.rs/datafusion/latest/datafusion/optimizer/index.html)
- [Polars Lazy API & Query Optimizer](https://docs.pola.rs/user-guide/lazy/optimizations/)
- [Apache Calcite Query Optimization](https://calcite.apache.org/docs/algebra.html) (design reference)
- [How Query Engines Work — Andy Pavlo CMU](https://db.cs.cmu.edu/papers/2022/)