# 01 — Protocol State Machines

**Status**: Core  
**Prerequisites**: `tokio-util` ≥ 0.7, `tokio` ≥ 1.35

---

## Overview

Every network protocol is a finite state machine. Rust's enum system with exhaustive pattern matching provides compile-time enforcement that every state transition is handled. This document covers modeling protocols as typed state enums, framing with `tokio-util` Codec, and composing layered protocol stacks from L2 through L7.

---

## 1. Typed Enum States

Model each protocol phase as a variant. State-specific data lives on the variant, not in `Option` fields on a monolithic struct.

```rust
#[derive(Debug)]
enum TcpState {
    Closed,
    Listen { backlog: usize },
    SynSent { isn: u32, mss: u16 },
    SynReceived { isn: u32, peer_isn: u32 },
    Established { send_nxt: u32, recv_nxt: u32, send_wnd: u16 },
    FinWait1,
    FinWait2,
    CloseWait,
    LastAck,
    TimeWait { entered_at: tokio::time::Instant },
}

struct Connection {
    state: TcpState,
    buf: Vec<u8>,
}

impl Connection {
    fn recv_syn(&mut self, mss: u16, peer_isn: u32) {
        self.state = match self.state {
            TcpState::Closed | TcpState::Listen { .. } | TcpState::SynSent { .. } => {
                TcpState::SynReceived {
                    isn: rand::random(),
                    peer_isn,
                }
            }
            ref s => panic!("recv_syn in unexpected state: {s:?}"),
        };
    }
}
```

---

## 2. Exhaustive match — No Missing Transitions

Every state transition function must match all variants. The compiler rejects partial matches.

```rust
enum Http2StreamState {
    Idle,
    Open { stream_id: u32, send_window: i32 },
    HalfClosedLocal { stream_id: u32 },
    HalfClosedRemote { stream_id: u32, recv_window: i32 },
    Closed,
}

impl Http2StreamState {
    fn on_data_frame(&self, payload_len: usize) -> Result<Self, ProtocolError> {
        match self {
            Http2StreamState::Open {
                stream_id,
                send_window,
            } => {
                if *send_window < payload_len as i32 {
                    return Err(ProtocolError::FlowControlExhausted);
                }
                Ok(Http2StreamState::Open {
                    stream_id: *stream_id,
                    send_window: *send_window - payload_len as i32,
                })
            }
            Http2StreamState::HalfClosedRemote { stream_id, .. } => {
                Ok(Http2StreamState::HalfClosedRemote {
                    stream_id: *stream_id,
                    recv_window: 0,
                })
            }
            Http2StreamState::Idle
            | Http2StreamState::HalfClosedLocal { .. }
            | Http2StreamState::Closed => Err(ProtocolError::InvalidState),
        }
    }
}
```

---

## 3. tokio-util Encoder/Decoder Codec

`tokio-util` provides `Encoder` and `Decoder` traits for frame-level parsing on `TcpStream`. The `Framed` adapter composes with `AsyncRead` + `AsyncWrite`.

```rust
use bytes::{Buf, BufMut, BytesMut};
use tokio_util::codec::{Decoder, Encoder};

struct Http2FrameCodec;

#[derive(Debug, PartialEq)]
enum Frame {
    Settings { ack: bool, params: Vec<(u16, u32)> },
    Headers { stream_id: u32, end_stream: bool, header_block: Vec<u8> },
    Data { stream_id: u32, end_stream: bool, payload: bytes::Bytes },
    Ping { ack: bool, opaque: [u8; 8] },
    GoAway { last_stream_id: u32, error_code: u32 },
}

impl Decoder for Http2FrameCodec {
    type Item = Frame;
    type Error = std::io::Error;

    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
        if src.len() < 9 {
            return Ok(None);
        }

        let payload_len = (src[0] as usize) << 16 | (src[1] as usize) << 8 | src[2] as usize;
        let frame_type = src[3];
        let flags = src[4];
        let stream_id = u32::from_be_bytes([0, src[5], src[6], src[7]]) & 0x7FFF_FFFF;

        if src.len() < 9 + payload_len {
            src.reserve(9 + payload_len - src.len());
            return Ok(None);
        }

        src.advance(9);
        let payload = src.split_to(payload_len);

        Ok(Some(match frame_type {
            0x0 => {
                let mut data = payload;
                Frame::Data {
                    stream_id,
                    end_stream: flags & 0x1 != 0,
                    payload: data.copy_to_bytes(data.remaining()),
                }
            }
            0x1 => {
                let mut header_block = payload;
                Frame::Headers {
                    stream_id,
                    end_stream: flags & 0x1 != 0,
                    header_block: header_block.to_vec(),
                }
            }
            0x4 => {
                let mut params = Vec::new();
                let mut chunk = payload;
                while chunk.remaining() >= 6 {
                    let id = chunk.get_u16();
                    let val = chunk.get_u32();
                    params.push((id, val));
                }
                Frame::Settings { ack: flags & 0x1 != 0, params }
            }
            _ => Frame::Settings { ack: false, params: vec![] },
        }))
    }
}

impl Encoder<Frame> for Http2FrameCodec {
    type Error = std::io::Error;

    fn encode(&mut self, item: Frame, dst: &mut BytesMut) -> Result<(), Self::Error> {
        match item {
            Frame::Settings { ack, params } => {
                let payload_len = if ack { 0 } else { params.len() * 6 };
                dst.reserve(9 + payload_len);
                dst.put_uint(payload_len as u64, 3);
                dst.put_u8(0x4);
                dst.put_u8(if ack { 0x1 } else { 0x0 });
                dst.put_u32(0);
                if !ack {
                    for (id, val) in params {
                        dst.put_u16(id);
                        dst.put_u32(val);
                    }
                }
            }
            Frame::Data { stream_id, end_stream, payload } => {
                dst.reserve(9 + payload.len());
                dst.put_uint(payload.len() as u64, 3);
                dst.put_u8(0x0);
                dst.put_u8(if end_stream { 0x1 } else { 0x0 });
                dst.put_u32(stream_id);
                dst.extend_from_slice(&payload);
            }
            Frame::Headers { stream_id, end_stream, header_block } => {
                dst.reserve(9 + header_block.len());
                dst.put_uint(header_block.len() as u64, 3);
                dst.put_u8(0x1);
                dst.put_u8(if end_stream { 0x5 } else { 0x4 });
                dst.put_u32(stream_id);
                dst.extend_from_slice(&header_block);
            }
            Frame::Ping { ack, opaque } => {
                dst.reserve(17);
                dst.put_uint(8u64, 3);
                dst.put_u8(0x6);
                dst.put_u8(if ack { 0x1 } else { 0x0 });
                dst.put_u32(0);
                dst.extend_from_slice(&opaque);
            }
            Frame::GoAway { last_stream_id, error_code } => {
                dst.reserve(17);
                dst.put_uint(8u64, 3);
                dst.put_u8(0x7);
                dst.put_u8(0x0);
                dst.put_u32(0);
                dst.put_u32(last_stream_id);
                dst.put_u32(error_code);
            }
        }
        Ok(())
    }
}
```

---

## 4. Layered Protocol Stack (L2 → L7)

Compose layers via wrapper types. Each layer encodes/decodes its headers, then delegates the payload to the next layer.

```rust
use std::net::Ipv4Addr;

struct EthernetFrame {
    dst_mac: [u8; 6],
    src_mac: [u8; 6],
    ethertype: u16,
    payload: Vec<u8>,
}

struct Ipv4Packet {
    version_ihl: u8,
    dscp_ecn: u8,
    total_length: u16,
    identification: u16,
    flags_fragment: u16,
    ttl: u8,
    protocol: u8,
    checksum: u16,
    src: Ipv4Addr,
    dst: Ipv4Addr,
    payload: Vec<u8>,
}

struct TcpSegment {
    src_port: u16,
    dst_port: u16,
    seq_num: u32,
    ack_num: u32,
    data_offset_flags: u16,
    window: u16,
    checksum: u16,
    urgent_ptr: u16,
    payload: Vec<u8>,
}

impl EthernetFrame {
    fn parse_from(buf: &[u8]) -> Result<Self, ParseError> {
        if buf.len() < 14 {
            return Err(ParseError::Truncated);
        }
        Ok(EthernetFrame {
            dst_mac: buf[0..6].try_into().unwrap(),
            src_mac: buf[6..12].try_into().unwrap(),
            ethertype: u16::from_be_bytes([buf[12], buf[13]]),
            payload: buf[14..].to_vec(),
        })
    }
}

impl Ipv4Packet {
    fn from_ethernet(frame: &EthernetFrame) -> Result<Self, ParseError> {
        if frame.ethertype != 0x0800 {
            return Err(ParseError::WrongLayer);
        }
        let buf = &frame.payload;
        Ok(Ipv4Packet {
            version_ihl: buf[0],
            dscp_ecn: buf[1],
            total_length: u16::from_be_bytes([buf[2], buf[3]]),
            identification: u16::from_be_bytes([buf[4], buf[5]]),
            flags_fragment: u16::from_be_bytes([buf[6], buf[7]]),
            ttl: buf[8],
            protocol: buf[9],
            checksum: u16::from_be_bytes([buf[10], buf[11]]),
            src: Ipv4Addr::new(buf[12], buf[13], buf[14], buf[15]),
            dst: Ipv4Addr::new(buf[16], buf[17], buf[18], buf[19]),
            payload: buf[20..].to_vec(),
        })
    }
}

impl TcpSegment {
    fn from_ipv4(packet: &Ipv4Packet) -> Result<Self, ParseError> {
        if packet.protocol != 6 {
            return Err(ParseError::WrongLayer);
        }
        let buf = &packet.payload;
        Ok(TcpSegment {
            src_port: u16::from_be_bytes([buf[0], buf[1]]),
            dst_port: u16::from_be_bytes([buf[2], buf[3]]),
            seq_num: u32::from_be_bytes([buf[4], buf[5], buf[6], buf[7]]),
            ack_num: u32::from_be_bytes([buf[8], buf[9], buf[10], buf[11]]),
            data_offset_flags: u16::from_be_bytes([buf[12], buf[13]]),
            window: u16::from_be_bytes([buf[14], buf[15]]),
            checksum: u16::from_be_bytes([buf[16], buf[17]]),
            urgent_ptr: u16::from_be_bytes([buf[18], buf[19]]),
            payload: buf[20..].to_vec(),
        })
    }
}
```

---

## 5. Actor-based Protocol Server

Wrap the state machine in a Tokio task. Incoming frames drive transitions. The `Framed` stream yields decoded frames; the task matches on state + frame to compute the next state.

```rust
use tokio::net::TcpListener;
use tokio_stream::StreamExt;
use tokio_util::codec::Framed;

async fn run_http2_server(addr: &str) -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind(addr).await?;
    loop {
        let (stream, peer_addr) = listener.accept().await?;
        tokio::spawn(async move {
            let mut framed = Framed::new(stream, Http2FrameCodec);
            let mut state = ConnectionState::Preface;

            while let Some(result) = framed.next().await {
                match result {
                    Ok(frame) => {
                        state = match (state, &frame) {
                            (ConnectionState::Preface, Frame::Settings { .. }) => {
                                framed.send(Frame::Settings { ack: true, params: vec![] }).await.ok();
                                ConnectionState::Active { next_stream_id: 1 }
                            }
                            (ConnectionState::Active { next_stream_id }, Frame::Headers { stream_id, .. })
                                if *stream_id & 1 == 1 => {
                                framed.send(Frame::Headers {
                                    stream_id: *stream_id,
                                    end_stream: true,
                                    header_block: b":status 200\r\n\r\n".to_vec(),
                                }).await.ok();
                                ConnectionState::Active {
                                    next_stream_id: next_stream_id + 2,
                                }
                            }
                            (ConnectionState::Active { .. }, Frame::GoAway { .. }) => {
                                ConnectionState::Closing
                            }
                            _ => state,
                        };
                    }
                    Err(e) => {
                        eprintln!("frame decode error: {e}");
                        break;
                    }
                }
            }
            println!("Connection from {peer_addr} closed");
        });
    }
}

enum ConnectionState {
    Preface,
    Active { next_stream_id: u32 },
    Closing,
}

#[derive(Debug)]
enum ParseError {
    Truncated,
    WrongLayer,
}
```

---

## Red Lines

1. Protocol states **must** be an exhaustive `enum`. Never use `Option<X>` fields scattered on a struct to represent state transitions.
2. Every state-transition function **must** match all variants. The compiler enforces this; do not suppress it with `_ => unreachable!()`.
3. The `Decoder` **must** return `Ok(None)` when the buffer is incomplete — this is how backpressure works. Never panic on truncated input.
4. Layer composition **must** validate each layer independently before delegating to the next. Do not pass raw buffers across layer boundaries.

---

## References

- [tokio-util Codec docs](https://docs.rs/tokio-util/latest/tokio_util/codec/index.html)
- [RFC 9293 — TCP State Machine](https://datatracker.ietf.org/doc/html/rfc9293)
- [RFC 9113 — HTTP/2 Stream States](https://datatracker.ietf.org/doc/html/rfc9113#section-5.1)
- [IPv4: RFC 791](https://datatracker.ietf.org/doc/html/rfc791) / [Ethernet: IEEE 802.3](https://standards.ieee.org/ieee/802.3/10422/)