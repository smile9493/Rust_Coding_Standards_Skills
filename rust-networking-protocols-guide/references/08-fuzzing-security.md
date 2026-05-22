# 08 — Fuzzing & Protocol Security

**Status**: Core  
**Prerequisites**: `cargo-fuzz`, `proptest` ≥ 1.4, `arbitrary` ≥ 1.3

---

## Overview

Network protocols are the largest attack surface in any networked application. Every byte from the wire is hostile. Fuzzing feeds malformed, random, or structure-aware inputs to protocol parsers to discover crashes, hangs, and memory safety violations. Property-based testing verifies that protocol invariants (round-trip, idempotency) hold for all valid inputs. This document covers `cargo-fuzz`/`libfuzzer`, `proptest`, and AFL++ integration.

---

## 1. cargo-fuzz / libfuzzer — Structure-Aware Fuzz Target

Every protocol parser must have a `cargo-fuzz` target. The fuzz harness takes arbitrary bytes, feeds them to the parser, and asserts that it never panics, OOMs, or produces logically invalid output.

```bash
cargo fuzz init
```

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if data.len() < 9 {
        return;
    }

    let payload_len = (data[0] as usize) << 16 | (data[1] as usize) << 8 | data[2] as usize;
    let frame_type = data[3];
    let flags = data[4];

    let total_len = 9usize.saturating_add(payload_len);
    if data.len() < total_len {
        return;
    }

    match frame_type {
        0x0 => {
            let _payload = &data[9..9 + payload_len];
        }
        0x1 => {
            let header_block = &data[9..9 + payload_len];
            let _ = std::str::from_utf8(header_block);
        }
        0x4 => {
            let params = &data[9..9 + payload_len];
            for chunk in params.chunks(6) {
                if chunk.len() == 6 {
                    let id = u16::from_be_bytes([chunk[0], chunk[1]]);
                    let val = u32::from_be_bytes([chunk[2], chunk[3], chunk[4], chunk[5]]);
                    let _ = (id, val);
                }
            }
        }
        0x7 => {
            if payload_len >= 8 {
                let last_stream_id = u32::from_be_bytes([
                    data[9], data[10], data[11], data[12],
                ]);
                let error_code = u32::from_be_bytes([
                    data[13], data[14], data[15], data[16],
                ]);
                let _ = (last_stream_id, error_code);
            }
        }
        _ => {}
    }

    let _ = flags;
});
```

---

## 2. Structure-Aware Fuzzing with `arbitrary`

Instead of raw bytes, use `Arbitrary` to generate structurally valid protocol messages. This reaches deeper code paths.

```rust
use arbitrary::{Arbitrary, Unstructured};

#[derive(Arbitrary, Debug)]
struct FuzzFrame {
    #[arbitrary(value = 0..=7u8)]
    frame_type: u8,
    flags: u8,
    stream_id: u32,
    payload: Vec<u8>,
}

fuzz_target!(|frame: FuzzFrame| {
    let mut buf = Vec::new();
    let payload_len = frame.payload.len();

    buf.extend_from_slice(&[
        (payload_len >> 16) as u8,
        (payload_len >> 8) as u8,
        payload_len as u8,
    ]);
    buf.push(frame.frame_type);
    buf.push(frame.flags);
    buf.extend_from_slice(&frame.stream_id.to_be_bytes());
    buf.extend_from_slice(&frame.payload);

    match frame.frame_type {
        0x0 => {
            assert!(buf.len() >= 9 + payload_len);
        }
        0x4 if payload_len % 6 == 0 => {
            for i in (9..9 + payload_len).step_by(6) {
                if i + 6 <= buf.len() {
                    let id = u16::from_be_bytes([buf[i], buf[i + 1]]);
                    let val = u32::from_be_bytes([
                        buf[i + 2], buf[i + 3], buf[i + 4], buf[i + 5],
                    ]);
                    let _ = (id, val);
                }
            }
        }
        _ => {}
    }
});
```

---

## 3. proptest — Property-Based Testing for Protocol Invariants

`proptest` generates valid inputs and checks that invariants hold. Common protocol invariants include round-trip encoding/decoding and idempotency of frame serialization.

```rust
use proptest::prelude::*;

fn encode_frame(frame_type: u8, flags: u8, stream_id: u32, payload: &[u8]) -> Vec<u8> {
    let mut buf = Vec::with_capacity(9 + payload.len());
    buf.extend_from_slice(&[
        (payload.len() >> 16) as u8,
        (payload.len() >> 8) as u8,
        payload.len() as u8,
    ]);
    buf.push(frame_type);
    buf.push(flags);
    buf.extend_from_slice(&stream_id.to_be_bytes());
    buf.extend_from_slice(payload);
    buf
}

fn decode_frame(buf: &[u8]) -> Option<(u8, u8, u32, Vec<u8>)> {
    if buf.len() < 9 {
        return None;
    }
    let payload_len = (buf[0] as usize) << 16 | (buf[1] as usize) << 8 | buf[2] as usize;
    if buf.len() < 9 + payload_len {
        return None;
    }
    let frame_type = buf[3];
    let flags = buf[4];
    let stream_id = u32::from_be_bytes([buf[5], buf[6], buf[7], buf[8]]);
    let payload = buf[9..9 + payload_len].to_vec();
    Some((frame_type, flags, stream_id, payload))
}

proptest! {
    #[test]
    fn round_trip_encoding(
        frame_type in 0u8..8,
        flags in 0u8..=255,
        stream_id in 0u32..=0x7FFF_FFFF,
        payload in prop::collection::vec(0u8..=255, 0..4096),
    ) {
        let encoded = encode_frame(frame_type, flags, stream_id, &payload);
        let decoded = decode_frame(&encoded).expect("failed to decode frame");

        assert_eq!(frame_type, decoded.0);
        assert_eq!(flags, decoded.1);
        assert_eq!(stream_id, decoded.2);
        assert_eq!(payload, decoded.3);
    }

    #[test]
    fn idempotent_re_encoding(
        frame_type in 0u8..8,
        flags in 0u8..=255,
        stream_id in 0u32..=0x7FFF_FFFF,
        payload in prop::collection::vec(0u8..=255, 0..1024),
    ) {
        let encoded = encode_frame(frame_type, flags, stream_id, &payload);
        let decoded = decode_frame(&encoded).expect("first decode failed");
        let re_encoded = encode_frame(decoded.0, decoded.1, decoded.2, &decoded.3);
        assert_eq!(encoded, re_encoded, "re-encoding produced different bytes");
    }

    #[test]
    fn never_panics_on_malformed_input(
        buf in prop::collection::vec(0u8..=255, 0..65536),
    ) {
        let _ = decode_frame(&buf);
    }

    #[test]
    fn settings_payload_multiple_of_six(
        params in prop::collection::vec((0u16.., 0u32..), 0..32),
    ) {
        let mut payload = Vec::new();
        for (id, val) in &params {
            payload.extend_from_slice(&id.to_be_bytes());
            payload.extend_from_slice(&val.to_be_bytes());
        }
        assert_eq!(payload.len() % 6, 0);

        let encoded = encode_frame(0x4, 0, 0, &payload);
        let decoded = decode_frame(&encoded).expect("settings decode failed");
        let decoded_payload = decoded.3;

        for (i, (id, val)) in params.iter().enumerate() {
            let offset = i * 6;
            let d_id = u16::from_be_bytes([decoded_payload[offset], decoded_payload[offset + 1]]);
            let d_val = u32::from_be_bytes([
                decoded_payload[offset + 2],
                decoded_payload[offset + 3],
                decoded_payload[offset + 4],
                decoded_payload[offset + 5],
            ]);
            assert_eq!(*id, d_id);
            assert_eq!(*val, d_val);
        }
    }
}
```

---

## 4. AFL++ Integration

AFL++ provides coverage-guided fuzzing for complex protocol parsers. Use `afl` crate for Rust integration.

```rust
fn main() {
    afl::fuzz!(|data: &[u8]| {
        if data.len() < 14 {
            return;
        }

        let ethertype = u16::from_be_bytes([data[12], data[13]]);
        if ethertype != 0x0800 {
            return;
        }

        if data.len() < 34 {
            return;
        }

        let total_length = u16::from_be_bytes([data[16], data[17]]) as usize;
        if total_length < 20 || total_length > 65535 {
            return;
        }

        let actual_len = data.len().min(total_length).min(1500);
        let _src_ip = &data[26..30];
        let _dst_ip = &data[30..34];

        if actual_len > 34 {
            let tcp_payload = &data[34..actual_len];
            let _ = tcp_payload.len();
        }
    });
}
```

---

## 5. Fuzz Target Organization

Every protocol parser module must include a `fuzz/` directory or inline fuzz targets. Standard project layout:

```
src/
  protocol/
    http2/
      mod.rs
      frame.rs
      fuzz/
        mod.rs           -- re-exports fuzz targets
        frame_fuzz.rs    -- libfuzzer target for frame parsing
        hpack_fuzz.rs    -- libfuzzer target for HPACK
tests/
  protocol/
    http2/
      proptest.rs        -- proptest property tests
fuzz/
  fuzz_targets/
    http2_frame.rs       -- cargo-fuzz entry point
    http2_hpack.rs
    ipv4_parser.rs
```

---

## 6. Continuous Fuzzing Strategy

1. **CI Integration**: Run `cargo fuzz` for 60 seconds per target on every PR. Block merge on crash.
2. **Nightly Deep Fuzz**: Run fuzz targets for 8+ hours overnight. Store/restore corpora in CI artifacts.
3. **Corpus Minimization**: Run `cargo fuzz cmin` periodically to shrink corpora. Smaller corpora = faster fuzzing.
4. **Coverage Tracking**: Use `cargo fuzz coverage` and upload to coverage dashboard. Aim for 80%+ branch coverage in parsers.

```bash
cargo fuzz run http2_frame -- -max_total_time=60

cargo fuzz run http2_frame -- -max_total_time=28800

cargo fuzz cmin http2_frame
```

---

## Red Lines

1. Every protocol parser must have a fuzz target. No exceptions. Crashing on malformed input is unacceptable.
2. Fuzz targets must never panic, call `unwrap()`, or allocate unbounded memory. Use defensive checks and early returns.
3. Fuzzing must be part of CI. Run `cargo fuzz` for at least 60 seconds per target on every pull request.
4. Property-based tests must cover round-trip encoding/decoding for every frame type. If `encode(decode(x)) != x`, the codec is broken.
5. Never deploy a protocol parser that has not been fuzzed for at least 8 hours. New parsers must complete an overnight fuzz run before merge.

---

## References

- [cargo-fuzz Book](https://rust-fuzz.github.io/book/cargo-fuzz.html)
- [proptest docs](https://docs.rs/proptest/latest/proptest/)
- [arbitrary docs](https://docs.rs/arbitrary/latest/arbitrary/)
- [AFL++ docs](https://aflplus.plus/)
- [libfuzzer](https://llvm.org/docs/LibFuzzer.html)
- [Rust Fuzz Book](https://rust-fuzz.github.io/book/)