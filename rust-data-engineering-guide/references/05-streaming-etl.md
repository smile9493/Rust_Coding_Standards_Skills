# 05 — Streaming ETL & Windowing

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | Rust async/await, `Stream` trait, `tokio` runtime, basic windowing concepts |
| Aligned Crates | `tokio-stream`, `datafusion`, `arrow-rs`, `futures` |

---

## Core Architecture

Streaming ETL pipelines process infinite or unbounded data sources using async iteration with backpressure. The `Stream` trait is the fundamental abstraction — it emits items as they become available and pauses when the consumer is slow.

### Stream Trait & Backpressure

```rust
use futures::stream::{Stream, StreamExt};
use std::pin::Pin;
use std::task::{Context, Poll};

struct KafkaSource {
    consumer: KafkaConsumer,
    pending: VecDeque<RecordBatch>,
}

impl Stream for KafkaSource {
    type Item = Result<RecordBatch, KafkaError>;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        // Drain pending buffer first
        if let Some(batch) = self.pending.pop_front() {
            return Poll::Ready(Some(Ok(batch)));
        }

        // Poll Kafka — returns Poll::Pending if no data, waking cx later
        match self.consumer.poll_next(cx) {
            Poll::Ready(Some(Ok(messages))) => {
                let batch = convert_messages_to_batch(&messages);
                Poll::Ready(Some(Ok(batch)))
            }
            Poll::Ready(None) => Poll::Ready(None),
            Poll::Ready(Some(Err(e))) => Poll::Ready(Some(Err(e))),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

**Backpressure semantics**: When the consumer calls `poll_next()` slowly, the stream naturally pauses — the upstream `KafkaConsumer` stops fetching new messages. This is implicit backpressure without explicit buffer management.

### Controlled Concurrency with `buffered(n)`

```rust
use futures::stream::StreamExt;

async fn parallel_transform(
    source: impl Stream<Item = RecordBatch>,
) -> impl Stream<Item = RecordBatch> {
    source
        .map(|batch| {
            tokio::spawn(async move {
                // expensive CPU-bound transformation
                transform_batch(batch)
            })
        })
        .buffered(4) // at most 4 concurrent transforms
        .map(|result| result.unwrap())
}
```

`buffered(n)` limits concurrent in-flight tasks to `n`. When all `n` slots are occupied, the stream stops pulling from upstream — IO-level backpressure.

### Tumbling Windows

Fixed-size, non-overlapping time windows: every batch belongs to exactly one window.

```rust
use std::collections::HashMap;

struct TumblingWindow {
    window_size_ms: u64,
    current_window_start: u64,
    accumulator: HashMap<String, f64>, // group_key → aggregate
}

impl TumblingWindow {
    fn process_batch(&mut self, batch: &RecordBatch, timestamp_ms: u64) -> Option<HashMap<String, f64>> {
        let window_start = (timestamp_ms / self.window_size_ms) * self.window_size_ms;

        if window_start > self.current_window_start {
            // window complete — emit the accumulated result
            let result = std::mem::take(&mut self.accumulator);
            self.current_window_start = window_start;
            self.accumulator = HashMap::new();

            // accumulate current batch into new window
            self.accumulate(batch);
            Some(result)
        } else {
            self.accumulate(batch);
            None // window not yet closed
        }
    }

    fn accumulate(&mut self, batch: &RecordBatch) {
        let keys = batch.column(0).as_any().downcast_ref::<StringArray>().unwrap();
        let values = batch.column(1).as_any().downcast_ref::<Float64Array>().unwrap();

        for i in 0..batch.num_rows() {
            let key = keys.value(i);
            let val = values.value(i);
            *self.accumulator.entry(key.to_string()).or_insert(0.0) += val;
        }
    }
}
```

### Sliding Windows

Overlapping windows: each event belongs to multiple windows. Uses a ring buffer of partial aggregates.

```rust
struct SlidingWindow {
    window_size_ms: u64,
    slide_interval_ms: u64,
    buckets: VecDeque<HashMap<String, f64>>,
}

impl SlidingWindow {
    fn new(window_size_ms: u64, slide_interval_ms: u64) -> Self {
        let num_buckets = (window_size_ms / slide_interval_ms) as usize;
        SlidingWindow {
            window_size_ms,
            slide_interval_ms,
            buckets: VecDeque::from(vec![HashMap::new(); num_buckets]),
        }
    }

    fn add_event(&mut self, key: &str, value: f64) {
        for bucket in self.buckets.iter_mut() {
            *bucket.entry(key.to_string()).or_insert(0.0) += value;
        }
    }

    fn advance(&mut self) -> HashMap<String, f64> {
        // pop oldest bucket (completed window), push new empty
        let result = self.buckets.pop_front().unwrap();
        self.buckets.push_back(HashMap::new());
        result
    }
}
```

### Session Windows

Dynamic windows based on activity gaps — no fixed boundaries:

```rust
struct SessionWindow {
    gap_threshold_ms: u64,
    sessions: HashMap<String, SessionState>,
}

struct SessionState {
    start_ms: u64,
    last_event_ms: u64,
    aggregate: f64,
}

impl SessionWindow {
    fn process_event(&mut self, key: &str, value: f64, timestamp_ms: u64) {
        let session = self.sessions.entry(key.to_string()).or_insert_with(|| SessionState {
            start_ms: timestamp_ms,
            last_event_ms: timestamp_ms,
            aggregate: 0.0,
        });

        if timestamp_ms - session.last_event_ms > self.gap_threshold_ms {
            // gap exceeded — emit previous session, start new one
            let old = std::mem::replace(session, SessionState {
                start_ms: timestamp_ms,
                last_event_ms: timestamp_ms,
                aggregate: value,
            });
            // emit old.aggregate ...
        } else {
            session.last_event_ms = timestamp_ms;
            session.aggregate += value;
        }
    }
}
```

### Watermark — Late Data Handling

Watermarks declare a threshold: "I will not see events older than T." Late data (timestamp < watermark) can be discarded or routed to a side output.

```rust
struct WatermarkTracker {
    max_seen_timestamp: u64,
    allowed_lateness_ms: u64,
}

impl WatermarkTracker {
    fn watermark(&self) -> u64 {
        self.max_seen_timestamp.saturating_sub(self.allowed_lateness_ms)
    }

    fn update(&mut self, timestamp_ms: u64) {
        self.max_seen_timestamp = self.max_seen_timestamp.max(timestamp_ms);
    }

    fn is_late(&self, event_timestamp_ms: u64) -> bool {
        event_timestamp_ms < self.watermark()
    }
}

fn process_with_watermark(
    event_timestamp_ms: u64,
    watermark: &mut WatermarkTracker,
    main_output: &mut Vec<RecordBatch>,
    late_output: &mut Vec<RecordBatch>,
    batch: RecordBatch,
) {
    watermark.update(event_timestamp_ms);
    if watermark.is_late(event_timestamp_ms) {
        late_output.push(batch);
    } else {
        main_output.push(batch);
    }
}
```

### Exactly-Once with Checkpoint

Exactly-once semantics require coordinated checkpointing of source offset + operator state:

```rust
struct Checkpoint {
    kafka_offsets: HashMap<i32, i64>, // partition → offset
    window_state: Vec<u8>,            // serialized window accumulator
    epoch: u64,
}

trait CheckpointableOperator {
    fn snapshot_state(&self) -> Vec<u8>;
    fn restore_state(&mut self, state: &[u8]);
}

async fn exactly_once_pipeline(
    mut source: KafkaSource,
    mut window: TumblingWindow,
    checkpoint_store: &dyn CheckpointStore,
) {
    let mut last_checkpoint = tokio::time::Instant::now();
    let checkpoint_interval = tokio::time::Duration::from_secs(10);

    while let Some(batch) = source.next().await {
        let result = window.process_batch(&batch, batch.timestamp_ms);

        if last_checkpoint.elapsed() >= checkpoint_interval {
            let checkpoint = Checkpoint {
                kafka_offsets: source.current_offsets(),
                window_state: window.snapshot_state(),
                epoch: source.current_epoch(),
            };
            checkpoint_store.save(&checkpoint).await.unwrap();
            last_checkpoint = tokio::time::Instant::now();
        }

        if let Some(aggregated) = result {
            // emit downstream — at-least-once within checkpoint boundary
            emit_result(aggregated).await;
        }
    }
}
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Unbounded state → OOM. Must enforce eviction policy** (TTL, LRU, watermark-based) | Memory usage grows linearly with event time → crash |
| 2 | **Always use `buffered(n)` for concurrent IO**, never `for_each_concurrent` without limit | Unbounded concurrency exhausts file descriptors and memory |
| 3 | **Watermark must be configured with allowed lateness** (typical: 1-5 min) | Windows never close; state accumulates indefinitely |
| 4 | **Exactly-once requires coordinated checkpoint of source offset + all operator states** | Restart from mid-point produces duplicates or data loss |
| 5 | **Window state must be serializable** (for checkpoint/restore) | Cannot recover after failure; data loss on restart |

---

## References

- [Rust `Stream` trait](https://docs.rs/futures/latest/futures/stream/trait.Stream.html)
- [tokio-stream crate](https://docs.rs/tokio-stream/latest/tokio_stream/)
- [Apache Flink Windowing](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/operators/windows/) (design reference)
- [DataFusion StreamingExecutionPlan](https://docs.rs/datafusion/latest/datafusion/physical_plan/trait.ExecutionPlan.html)