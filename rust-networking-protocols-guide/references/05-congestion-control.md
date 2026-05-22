# 05 — Congestion Control & Flow Control

**Status**: Core  
**Prerequisites**: Understanding of TCP congestion control (RFC 5681), QUIC flow control (RFC 9000 §4)

---

## Overview

Congestion control determines how fast a sender can transmit without overwhelming the network. Flow control prevents the receiver's buffer from overflowing. In QUIC, both operate at two levels: connection-wide and per-stream. This document covers algorithm selection, pacing, ECN response, and credit-based flow control windows.

---

## 1. BBR vs CUBIC — Algorithm Selection

CUBIC is the Linux default (loss-based), BBR is Google's model-based algorithm that measures bandwidth and RTT rather than reacting to loss.

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
enum CongestionAlgorithm {
    Cubic,
    Bbr,
    NewReno,
}

struct CongestionController {
    algorithm: CongestionAlgorithm,
    cwnd: f64,
    ssthresh: f64,
    rtt_min: std::time::Duration,
    rtt_estimate: std::time::Duration,
    in_recovery: bool,
    bbr_state: Option<BbrState>,
}

struct BbrState {
    pacing_rate: f64,
    btlbw: f64,
    rtprop: std::time::Duration,
    cycle_phase: BbrPhase,
    prior_cwnd: f64,
}

#[derive(Debug, Clone, Copy, PartialEq)]
enum BbrPhase {
    Startup,
    Drain,
    ProbeBw,
    ProbeRtt,
}

impl CongestionController {
    fn new(algorithm: CongestionAlgorithm) -> Self {
        CongestionController {
            algorithm,
            cwnd: 10.0 * 1500.0,
            ssthresh: f64::MAX,
            rtt_min: std::time::Duration::MAX,
            rtt_estimate: std::time::Duration::from_millis(100),
            in_recovery: false,
            bbr_state: if algorithm == CongestionAlgorithm::Bbr {
                Some(BbrState {
                    pacing_rate: 0.0,
                    btlbw: 0.0,
                    rtprop: std::time::Duration::MAX,
                    cycle_phase: BbrPhase::Startup,
                    prior_cwnd: 0.0,
                })
            } else {
                None
            },
        }
    }

    fn on_ack(&mut self, acked_bytes: usize, rtt_sample: std::time::Duration) {
        match self.algorithm {
            CongestionAlgorithm::Cubic => self.cubic_on_ack(acked_bytes, rtt_sample),
            CongestionAlgorithm::Bbr => self.bbr_on_ack(acked_bytes, rtt_sample),
            CongestionAlgorithm::NewReno => {}
        }
    }

    fn cubic_on_ack(&mut self, acked_bytes: usize, _rtt_sample: std::time::Duration) {
        if self.cwnd < self.ssthresh {
            self.cwnd += acked_bytes as f64;
        } else {
            self.cwnd += (acked_bytes as f64 * 0.3).max(1.0);
        }
    }

    fn on_loss(&mut self) {
        match self.algorithm {
            CongestionAlgorithm::Cubic => {
                self.ssthresh = (self.cwnd * 0.7).max(2.0 * 1500.0);
                self.cwnd = self.ssthresh;
                self.in_recovery = true;
            }
            CongestionAlgorithm::Bbr => {
                if let Some(ref mut bbr) = self.bbr_state {
                    bbr.prior_cwnd = self.cwnd;
                    self.cwnd = acked_bytes_in_flight();
                }
            }
            CongestionAlgorithm::NewReno => {
                self.ssthresh = (self.cwnd * 0.5).max(2.0 * 1500.0);
                self.cwnd = self.ssthresh;
            }
        }
    }

    fn bbr_on_ack(&mut self, acked_bytes: usize, rtt_sample: std::time::Duration) {
        if rtt_sample < self.rtt_min {
            self.rtt_min = rtt_sample;
        }

        if let Some(ref mut bbr) = self.bbr_state {
            match bbr.cycle_phase {
                BbrPhase::Startup => {
                    self.cwnd += acked_bytes as f64;
                    bbr.pacing_rate = self.cwnd / self.rtt_min.as_secs_f64();
                }
                BbrPhase::Drain => {
                    bbr.pacing_rate *= 0.75;
                    self.cwnd = bbr.pacing_rate * self.rtt_min.as_secs_f64();
                }
                BbrPhase::ProbeBw => {
                    bbr.pacing_rate = self.cwnd / self.rtt_min.as_secs_f64();
                }
                BbrPhase::ProbeRtt => {
                    self.cwnd = 4.0 * 1500.0;
                }
            }
        }
    }
}

fn acked_bytes_in_flight() -> f64 {
    4.0 * 1500.0
}
```

---

## 2. Token Bucket Pacing

Bursty sends cause packet loss at bottleneck routers. A token bucket rate-limits the sender to smooth the transmission rate.

```rust
use std::time::{Duration, Instant};

struct TokenBucket {
    rate: f64,
    capacity: f64,
    tokens: f64,
    last_refill: Instant,
}

impl TokenBucket {
    fn new(rate_bytes_per_sec: f64, max_burst_bytes: f64) -> Self {
        TokenBucket {
            rate: rate_bytes_per_sec,
            capacity: max_burst_bytes,
            tokens: max_burst_bytes,
            last_refill: Instant::now(),
        }
    }

    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.rate).min(self.capacity);
        self.last_refill = now;
    }

    fn consume(&mut self, bytes: usize) -> bool {
        self.refill();
        if self.tokens >= bytes as f64 {
            self.tokens -= bytes as f64;
            true
        } else {
            false
        }
    }

    fn time_until_ready(&self, bytes: usize) -> Duration {
        let needed = bytes as f64 - self.tokens;
        if needed <= 0.0 {
            return Duration::ZERO;
        }
        Duration::from_secs_f64(needed / self.rate)
    }
}

async fn paced_send(
    bucket: &mut TokenBucket,
    data: &[u8],
) -> Result<(), std::io::Error> {
    let chunk_size = 1400;

    for chunk in data.chunks(chunk_size) {
        while !bucket.consume(chunk.len()) {
            let wait = bucket.time_until_ready(chunk.len());
            tokio::time::sleep(wait).await;
        }
        send_packet(chunk).await?;
    }
    Ok(())
}

async fn send_packet(_data: &[u8]) -> Result<(), std::io::Error> {
    Ok(())
}
```

---

## 3. ECN — Explicit Congestion Notification

ECN allows routers to mark packets instead of dropping them. The receiver echoes the congestion mark back to the sender, which reduces its rate.

```rust
#[derive(Debug, PartialEq)]
enum EcnCodepoint {
    NotEct,
    Ect0,
    Ect1,
    Ce,
}

struct EcnState {
    ect_enabled: bool,
    ce_received: bool,
    cwr_sent: bool,
}

impl EcnState {
    fn on_ack_with_ecn(&mut self, ce_count: u64, controller: &mut CongestionController) {
        if ce_count > 0 && self.ect_enabled && !self.ce_received {
            self.ce_received = true;
            self.cwr_sent = true;

            controller.ssthresh = (controller.cwnd * 0.5).max(2.0 * 1500.0);
            controller.cwnd = controller.ssthresh;
        }
    }

    fn mark_ecn_outgoing(&self, codepoint: &mut EcnCodepoint) {
        if self.ect_enabled {
            *codepoint = EcnCodepoint::Ect0;
        }
    }
}
```

---

## 4. QUIC Stream & Connection Flow Control

QUIC uses credit-based flow control: the receiver advertises a `max_data` limit, and the sender must not exceed it. Flow control operates at both the connection level and the per-stream level.

```rust
use std::collections::HashMap;

struct QuicFlowController {
    connection_max_data: u64,
    connection_data_sent: u64,
    stream_max_data: HashMap<u64, u64>,
    stream_data_sent: HashMap<u64, u64>,
}

impl QuicFlowController {
    fn new(initial_conn_max: u64) -> Self {
        QuicFlowController {
            connection_max_data: initial_conn_max,
            connection_data_sent: 0,
            stream_max_data: HashMap::new(),
            stream_data_sent: HashMap::new(),
        }
    }

    fn can_send(&self, stream_id: u64, bytes: usize) -> bool {
        let conn_ok =
            self.connection_data_sent + bytes as u64 <= self.connection_max_data;

        let stream_max = self.stream_max_data.get(&stream_id).copied().unwrap_or(0);
        let stream_sent = self.stream_data_sent.get(&stream_id).copied().unwrap_or(0);
        let stream_ok = stream_sent + bytes as u64 <= stream_max;

        conn_ok && stream_ok
    }

    fn record_sent(&mut self, stream_id: u64, bytes: usize) {
        self.connection_data_sent += bytes as u64;
        *self.stream_data_sent.entry(stream_id).or_insert(0) += bytes as u64;
    }

    fn update_max_data(&mut self, new_max: u64) {
        self.connection_max_data = self.connection_max_data.max(new_max);
    }

    fn update_stream_max_data(&mut self, stream_id: u64, new_max: u64) {
        let entry = self.stream_max_data.entry(stream_id).or_insert(0);
        *entry = (*entry).max(new_max);
    }

    fn send_window_available(&self) -> u64 {
        self.connection_max_data
            .saturating_sub(self.connection_data_sent)
    }
}

const INITIAL_MAX_DATA: u64 = 65536;
const INITIAL_MAX_STREAM_DATA: u64 = 32768;
```

---

## Red Lines

1. Senders must enforce `max_data` and stream-level limits. Unbounded send buffers cause bufferbloat and memory exhaustion.
2. Pacing must be enabled by default. Bursts without pacing cause unnecessary packet loss and retransmissions.
3. ECN support must be negotiated in the handshake. Do not mark packets as ECT unless both endpoints support it.
4. Congestion control state must be per-connection. Sharing CWND across connections (as in HTTP/1.1) negates fairness.
5. Flow control credit updates (`MAX_DATA` / `MAX_STREAM_DATA`) must be sent proactively, not only when the peer exhausts its limit.

---

## References

- [RFC 9002 — QUIC Loss Detection and Congestion Control](https://datatracker.ietf.org/doc/html/rfc9002)
- [RFC 8312 — CUBIC](https://datatracker.ietf.org/doc/html/rfc8312)
- [BBR Congestion Control (ACM Queue)](https://queue.acm.org/detail.cfm?id=3022184)
- [RFC 3168 — ECN for IP](https://datatracker.ietf.org/doc/html/rfc3168)
- [RFC 9000 §4 — QUIC Flow Control](https://datatracker.ietf.org/doc/html/rfc9000#section-4)