# 06 — Connection Pooling & Multiplexing

**Status**: Core  
**Prerequisites**: `deadpool` ≥ 0.12, `bb8` ≥ 0.8, `tokio` ≥ 1.35

---

## Overview

Opening a new connection for every request is expensive: TCP handshake, TLS negotiation, and HPACK dictionary warmup all add latency. Connection pooling reuses established connections across requests. For HTTP/2 and QUIC, multiplexing further allows multiple concurrent streams over a single connection.

---

## 1. deadpool — Managed Connection Pool

`deadpool` provides a generic async pool. Define a `Manager` that creates, recycles, and validates connections.

```rust
use deadpool::managed;
use std::time::Duration;
use tokio::net::TcpStream;

struct TcpConnection {
    stream: TcpStream,
    created_at: std::time::Instant,
}

struct TcpManager {
    addr: String,
}

impl managed::Manager for TcpManager {
    type Type = TcpConnection;
    type Error = std::io::Error;

    async fn create(&self) -> Result<TcpConnection, std::io::Error> {
        let stream = TcpStream::connect(&self.addr).await?;
        stream.set_nodelay(true)?;
        Ok(TcpConnection {
            stream,
            created_at: std::time::Instant::now(),
        })
    }

    async fn recycle(
        &self,
        conn: &mut TcpConnection,
        _metrics: &managed::Metrics,
    ) -> managed::RecycleResult<std::io::Error> {
        let elapsed = conn.created_at.elapsed();
        if elapsed > Duration::from_secs(300) {
            return Err(managed::RecycleError::message(
                "connection expired",
            ));
        }

        let mut buf = [0u8; 1];
        match conn.stream.try_read(&mut buf) {
            Ok(0) => Err(managed::RecycleError::message("connection closed")),
            Err(ref e) if e.kind() == std::io::ErrorKind::WouldBlock => Ok(()),
            Err(e) => Err(managed::RecycleError::from(e)),
            Ok(_) => Err(managed::RecycleError::message("unexpected data")),
        }
    }
}

fn build_pool(addr: &str) -> managed::Pool<TcpManager> {
    let manager = TcpManager {
        addr: addr.to_string(),
    };

    managed::Pool::builder(manager)
        .max_size(32)
        .min_idle(Some(4))
        .max_lifetime(Some(Duration::from_secs(600)))
        .idle_timeout(Some(Duration::from_secs(60)))
        .build()
        .unwrap()
}

async fn use_pool(pool: &managed::Pool<TcpManager>) -> Result<(), Box<dyn std::error::Error>> {
    let conn = pool.get().await?;
    conn.stream.writable().await?;
    conn.stream.try_write(b"PING\r\n")?;

    let mut response = vec![0u8; 1024];
    conn.stream.readable().await?;
    let n = conn.stream.try_read(&mut response)?;
    println!("received {} bytes", n);

    Ok(())
}
```

---

## 2. bb8 — Diesel-style Connection Pool

`bb8` is an alternative pool with a slightly different API. The pattern is the same: `ManageConnection` trait, `Pool::builder()`.

```rust
use bb8::{ManageConnection, Pool};
use tokio::net::TcpStream;

struct Bb8TcpManager {
    addr: String,
}

impl ManageConnection for Bb8TcpManager {
    type Connection = TcpStream;
    type Error = std::io::Error;

    async fn connect(&self) -> Result<Self::Connection, Self::Error> {
        TcpStream::connect(&self.addr).await
    }

    async fn is_valid(&self, conn: &mut Self::Connection) -> Result<(), Self::Error> {
        conn.writable().await?;
        conn.try_write(b"\0")?;
        Ok(())
    }

    fn has_broken(&self, conn: &mut Self::Connection) -> bool {
        match conn.try_read(&mut [0u8; 1]) {
            Ok(0) => true,
            Err(ref e) if e.kind() == std::io::ErrorKind::WouldBlock => false,
            _ => true,
        }
    }
}

async fn bb8_example() -> Result<(), Box<dyn std::error::Error>> {
    let manager = Bb8TcpManager {
        addr: "127.0.0.1:8080".to_string(),
    };

    let pool = Pool::builder()
        .max_size(16)
        .min_idle(Some(2))
        .build(manager)
        .await?;

    let conn = pool.get().await?;
    conn.writable().await?;
    println!("connection acquired from bb8 pool");

    Ok(())
}
```

---

## 3. HTTP/2 HPACK Dynamic Table

HPACK compresses HTTP headers using a dynamic table shared between encoder and decoder. The table grows with each header set and is bounded by `SETTINGS_HEADER_TABLE_SIZE`.

```rust
use std::collections::VecDeque;

#[derive(Debug, Clone)]
struct HpackHeaderField {
    name: Vec<u8>,
    value: Vec<u8>,
}

struct HpackDynamicTable {
    entries: VecDeque<HpackHeaderField>,
    max_size: usize,
    current_size: usize,
}

impl HpackDynamicTable {
    fn new(max_size: usize) -> Self {
        HpackDynamicTable {
            entries: VecDeque::new(),
            max_size,
            current_size: 0,
        }
    }

    fn add(&mut self, name: Vec<u8>, value: Vec<u8>) {
        let entry_size = name.len() + value.len() + 32;

        while self.current_size + entry_size > self.max_size && !self.entries.is_empty() {
            let evicted = self.entries.pop_back().unwrap();
            self.current_size -= evicted.name.len() + evicted.value.len() + 32;
        }

        if entry_size <= self.max_size {
            self.entries.push_front(HpackHeaderField { name, value });
            self.current_size += entry_size;
        }
    }

    fn get(&self, index: usize) -> Option<&HpackHeaderField> {
        self.entries.get(index)
    }

    fn resize(&mut self, new_max_size: usize) {
        self.max_size = new_max_size;
        while self.current_size > self.max_size && !self.entries.is_empty() {
            let evicted = self.entries.pop_back().unwrap();
            self.current_size -= evicted.name.len() + evicted.value.len() + 32;
        }
    }
}

const DEFAULT_HEADER_TABLE_SIZE: usize = 4096;
```

---

## 4. Multiplexing Strategy — Stream Prioritization

HTTP/2 and QUIC both support stream prioritization. Weighted fair queuing ensures that high-priority streams get more bandwidth share.

```rust
use std::collections::BinaryHeap;
use std::cmp::Ordering;

#[derive(Debug)]
struct StreamPriority {
    stream_id: u32,
    weight: u16,
    depends_on: u32,
    virtual_time: f64,
}

impl PartialEq for StreamPriority {
    fn eq(&self, other: &Self) -> bool {
        self.stream_id == other.stream_id
    }
}

impl Eq for StreamPriority {}

impl Ord for StreamPriority {
    fn cmp(&self, other: &Self) -> Ordering {
        other
            .virtual_time
            .partial_cmp(&self.virtual_time)
            .unwrap_or(Ordering::Equal)
    }
}

impl PartialOrd for StreamPriority {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

struct MultiplexedScheduler {
    streams: BinaryHeap<StreamPriority>,
    total_weight: f64,
}

impl MultiplexedScheduler {
    fn new() -> Self {
        MultiplexedScheduler {
            streams: BinaryHeap::new(),
            total_weight: 0.0,
        }
    }

    fn add_stream(&mut self, stream_id: u32, weight: u16) {
        self.total_weight += weight as f64;
        self.streams.push(StreamPriority {
            stream_id,
            weight,
            depends_on: 0,
            virtual_time: 0.0,
        });
    }

    fn next_stream(&mut self) -> Option<u32> {
        let mut entry = self.streams.pop()?;
        let stream_id = entry.stream_id;
        entry.virtual_time += 1.0 / entry.weight as f64;
        self.streams.push(entry);
        Some(stream_id)
    }

    fn update_weight(&mut self, stream_id: u32, new_weight: u16) {
        let mut found: Option<StreamPriority> = None;
        let mut remaining = BinaryHeap::new();

        while let Some(mut s) = self.streams.pop() {
            if s.stream_id == stream_id {
                self.total_weight -= s.weight as f64;
                s.weight = new_weight;
                self.total_weight += new_weight as f64;
                found = Some(s);
                break;
            }
            remaining.push(s);
        }

        self.streams.append(&mut remaining);
        if let Some(s) = found {
            self.streams.push(s);
        }
    }
}
```

---

## 5. Dead Connection Detection & Eviction

Connections can silently fail: TCP RST not delivered, NAT timeouts, or server crashes. The pool must detect dead connections before handing them to callers.

```rust
use std::time::{Duration, Instant};
use tokio::sync::RwLock;

struct ConnectionWithHealth {
    created: Instant,
    last_used: Instant,
    idle_timeout: Duration,
    max_lifetime: Duration,
}

impl ConnectionWithHealth {
    fn new(idle_timeout: Duration, max_lifetime: Duration) -> Self {
        let now = Instant::now();
        ConnectionWithHealth {
            created: now,
            last_used: now,
            idle_timeout,
            max_lifetime,
        }
    }

    fn is_expired(&self) -> bool {
        self.created.elapsed() > self.max_lifetime
    }

    fn is_idle_expired(&self) -> bool {
        self.last_used.elapsed() > self.idle_timeout
    }

    fn touch(&mut self) {
        self.last_used = Instant::now();
    }

    async fn is_alive(stream: &tokio::net::TcpStream) -> bool {
        match stream.try_read(&mut [0u8; 1]) {
            Ok(0) => false,
            Err(ref e) if e.kind() == std::io::ErrorKind::WouldBlock => true,
            _ => false,
        }
    }
}

struct EvictingPool {
    connections: Vec<ConnectionWithHealth>,
    check_interval: Duration,
}

impl EvictingPool {
    async fn eviction_loop(&mut self) {
        let mut interval = tokio::time::interval(self.check_interval);
        loop {
            interval.tick().await;
            self.connections.retain(|conn| {
                if conn.is_expired() {
                    println!("evicting expired connection");
                    return false;
                }
                if conn.is_idle_expired() {
                    println!("evicting idle connection");
                    return false;
                }
                true
            });
        }
    }
}
```

---

## Red Lines

1. Connection pools must detect dead connections (TCP RST, idle timeout, max lifetime) and evict them before checkout.
2. Health checks must use non-blocking I/O (`try_read` / `try_write`). Never block the pool's checkout path on network I/O.
3. HPACK dynamic table size must be bounded. `SETTINGS_HEADER_TABLE_SIZE` from the peer is authoritative; exceeding it is a protocol violation.
4. Stream multiplexing must limit concurrent streams per connection. The default is 100; exceeding this without `SETTINGS_MAX_CONCURRENT_STREAMS` negotiation is a protocol error.
5. Pool sizes must be tuned. Test with realistic load. Oversized pools waste file descriptors; undersized pools cause checkout timeouts.

---

## References

- [deadpool crate](https://docs.rs/deadpool/latest/deadpool/)
- [bb8 crate](https://docs.rs/bb8/latest/bb8/)
- [RFC 7541 — HPACK Header Compression](https://datatracker.ietf.org/doc/html/rfc7541)
- [RFC 9113 §5.3 — HTTP/2 Stream Prioritization](https://datatracker.ietf.org/doc/html/rfc9113#section-5.3)
- [RFC 9000 §4.1 — QUIC Stream Limits](https://datatracker.ietf.org/doc/html/rfc9000#section-4.1)