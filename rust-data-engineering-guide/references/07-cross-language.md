# 07 — Cross-Language Data Interchange

## Status

| Field | Value |
|-------|-------|
| Status | Stable |
| Prerequisites | PyO3 FFI, Node.js N-API, gRPC basics, Arrow IPC format |
| Aligned Crates | `pyo3`, `napi-rs`, `arrow-flight`, `arrow-rs` |

---

## Core Architecture

Cross-language data interchange in analytics is dominated by the **Arrow C Data Interface** — a stable C ABI that allows passing Arrow arrays between runtime environments without serialization or copying. The same memory is referenced by Rust, Python, and C++ simultaneously.

### The Arrow C Data Interface (C Data Interface)

Two core structures bridge language boundaries:

```c
// From the Arrow C Data Interface specification:
struct ArrowSchema {
    const char* format;        // "i" for int32, "U" for UTF-8, "+l" for list<int64>
    const char* name;
    int64_t flags;             // nullable, dictionary-ordered, etc.
    // ... metadata ...
};

struct ArrowArray {
    int64_t length;
    int64_t null_count;
    const void** buffers;      // [null_bitmap, values, offsets...]
    int64_t n_buffers;
    struct ArrowArray** children;   // nested: list elements, struct fields
    struct ArrowArray* dictionary;  // for dictionary-encoded columns
};
```

These structs contain **pointers to existing Arrow buffers** — no data duplication. The consumer reads the pointers, the producer retains ownership.

### PyO3 + Arrow — Zero-Copy Rust ↔ Python

The bridge between `arrow-rs` and `pyarrow` uses `ArrowArrayStream`:

```rust
use pyo3::prelude::*;
use arrow::array::Int64Array;
use arrow::pyarrow::IntoPyArrow;

#[pyfunction]
fn compute_in_rust(py: Python<'_>, data: PyObject) -> PyResult<PyObject> {
    // 1. Convert Python PyArrow array → Rust Arrow array (zero-copy)
    let rust_array: Int64Array = data.extract(py)?;

    // 2. Compute in Rust
    let result = arrow::compute::kernels::arithmetic::add_scalar(
        &rust_array,
        10i64,
    ).unwrap();

    // 3. Convert back to Python PyArrow (zero-copy)
    result.into_pyarrow(py)
}
```

**Key mechanism**: `IntoPyArrow` and `FromPyArrow` traits handle the C Data Interface under the hood. The Rust code never copies the underlying buffers — it passes `Arc<ArrayData>` which PyArrow references via the C struct.

**Production example — batch conversion:**

```rust
use arrow::pyarrow::{FromPyArrow, ToPyArrow};
use arrow::record_batch::RecordBatch;
use pyo3::prelude::*;

#[pyfunction]
fn filter_batch_py(py: Python<'_>, batch: PyObject, threshold: i64) -> PyResult<PyObject> {
    let rust_batch: RecordBatch = FromPyArrow::from_pyarrow(&batch)?;

    let col = rust_batch.column(0)
        .as_any().downcast_ref::<Int64Array>().unwrap();

    let mask = arrow::compute::gt_scalar(col, threshold)?;
    let filtered = arrow::compute::filter_record_batch(&rust_batch, &mask)?;

    filtered.to_pyarrow(py)
}
```

**Throughput**: A 1GB `RecordBatch` passes from Python to Rust in ~10µs (just pointer swaps), versus ~2-5 seconds for `serde_json` serialization.

### napi-rs — Zero-Copy Node.js ↔ Rust

Node.js `Buffer` maps directly to Arrow `Buffer` via N-API external buffers:

```rust
use napi::bindgen_prelude::*;
use napi_derive::napi;

#[napi]
pub fn process_arrow_buffer(external_buffer: Buffer) -> Buffer {
    // external_buffer is a Node.js Buffer — points to Arrow data
    let slice: &[u8] = external_buffer.as_ref();

    // Interpret as Arrow IPC buffer (zero-copy)
    let arrow_buffer = arrow::buffer::Buffer::from(slice);

    // ... compute using arrow-rs ...
    let result = arrow::compute::kernels::cast::cast(
        &Int64Array::from(vec![1, 2, 3]),
        &DataType::Float64,
    ).unwrap();

    // Return as Node.js Buffer (shares memory)
    let result_data = result.to_data();
    let raw_bytes: &[u8] = result_data.buffers()[0].as_slice();

    Buffer::from(raw_bytes.to_vec()) // ideally would be external reference
}
```

**Important**: For true zero-copy return, use `napi::External` to pass an `Arc<Buffer>` that Node.js's GC can hold. The above `to_vec()` is simplified for clarity.

### Arrow Flight — gRPC Streaming Protocol

Flight wraps the C Data Interface in a gRPC stream for network transfer:

```rust
use arrow_flight::flight_service_client::FlightServiceClient;
use arrow_flight::{FlightDescriptor, Ticket};
use tonic::transport::Channel;

async fn flight_client_example(addr: &str) -> Result<(), Box<dyn std::error::Error>> {
    let mut client = FlightServiceClient::connect(format!("http://{}", addr)).await?;

    let ticket = Ticket {
        ticket: "SELECT * FROM orders WHERE status = 'active'".into(),
    };

    let mut stream = client.do_get(ticket).await?.into_inner();

    while let Some(flight_data) = stream.message().await? {
        let batch = arrow::ipc::reader::read_record_batch(
            &flight_data.data_header,
            flight_data.data_body,
            Arc::new(Schema::empty()),
            &Default::default(),
        )?;
        // batch is now a zero-copy RecordBatch
        println!("Received batch with {} rows", batch.num_rows());
    }

    Ok(())
}
```

**Flight server example:**

```rust
use arrow_flight::{
    flight_service_server::{FlightService, FlightServiceServer},
    Action, ActionType, Criteria, Empty, FlightData, FlightDescriptor, FlightInfo,
    HandshakeRequest, HandshakeResponse, PutResult, Result as FlightResult, SchemaResult, Ticket,
};
use tonic::{Request, Response, Status, Streaming};

#[derive(Default)]
struct MyFlightService;

#[tonic::async_trait]
impl FlightService for MyFlightService {
    type DoGetStream = tokio_stream::wrappers::ReceiverStream<FlightResult<FlightData>>;
    type DoPutStream = ...;
    type DoExchangeStream = ...;

    async fn do_get(
        &self,
        request: Request<Ticket>,
    ) -> FlightResult<Response<Self::DoGetStream>> {
        let (tx, rx) = tokio::sync::mpsc::channel(16);

        tokio::spawn(async move {
            let batches = execute_query(&request.into_inner().ticket).await;
            for batch in batches {
                let (flight_descriptor, flight_data) =
                    arrow_flight::utils::flight_data_from_arrow_batch(&batch, &Default::default());
                tx.send(Ok(flight_data)).await.unwrap();
            }
        });

        Ok(Response::new(tokio_stream::wrappers::ReceiverStream::new(rx)))
    }

    async fn do_put(
        &self,
        request: Request<Streaming<FlightData>>,
    ) -> FlightResult<Response<Self::DoPutStream>> { todo!() }

    async fn handshake(
        &self,
        request: Request<Streaming<HandshakeRequest>>,
    ) -> FlightResult<Response<Self::DoHandshakeStream>> { todo!() }

    // ... other required methods ...
}
```

### Flight SQL — JDBC/ODBC Replacement

Flight SQL adds SQL semantics on top of Flight. A Flight SQL client can query a Rust engine from any language that has a Flight SQL driver (Python, Java, C++, Go):

```rust
// Flight SQL extends Flight with:
//  - GetTables, GetSchemas, GetCatalogs (metadata)
//  - Execute (run SQL, returns FlightInfo)
//  - Prepared statements (parameterized queries)
// Clients use standard arrow-flight-sql-client libraries
```

---

## Red Lines

| # | Rule | Consequence if violated |
|---|------|------------------------|
| 1 | **Prohibit `serde_json` for bulk data handoff** — use Arrow C Data Interface / Flight | 10-100× slower; serialization dominates wall time |
| 2 | **Never copy Arrow buffers at language boundaries** — pass `Arc<ArrayData>` via C Data Interface | Memory doubles, throughput halved |
| 3 | **Flight connections must use TLS in production** (tonic + rustls) | Raw gRPC over plaintext leaks query data |
| 4 | **Always validate schema compatibility at boundary** — mismatched types cause silent corruption | Int32 ↔ Int64 mismatch truncates data |
| 5 | **Prefer Flight streams over single-response APIs for >10K rows** | Single-response blocks sender until all data computed; OOM risk |

---

## References

- [Arrow C Data Interface Specification](https://arrow.apache.org/docs/format/CDataInterface.html)
- [PyO3 + Arrow Integration (arro3)](https://docs.rs/arro3/latest/arro3/)
- [napi-rs crate](https://docs.rs/napi/latest/napi/)
- [Arrow Flight gRPC Protocol](https://arrow.apache.org/docs/format/Flight.html)
- [Flight SQL Specification](https://arrow.apache.org/docs/format/FlightSql.html)