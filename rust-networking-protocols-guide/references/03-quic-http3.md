# 03 — QUIC & HTTP/3 Implementation

**Status**: Core  
**Prerequisites**: `quinn` ≥ 0.11, `s2n-quic` ≥ 1.11, `quiche` ≥ 0.22

---

## Overview

QUIC (RFC 9000) reimagines TCP+TLS+HTTP/2 on UDP with 0-RTT handshakes, stream multiplexing without head-of-line blocking, and connection migration. Choosing the right Rust implementation depends on platform constraints, TLS backend requirements, and feature completeness.

---

## 1. Quinn / s2n-quic / quiche — Selection Guide

| Crate | TLS Backend | Strengths | When to Use |
|-------|------------|-----------|-------------|
| **quinn** | rustls (pure Rust) | Async-native, Tokio-integrated, pure Rust stack | Standard Rust services, no C deps |
| **s2n-quic** | s2n-tls (C) | AWS-backed, fuzzed heavily, low latency | Cloud infrastructure, AWS environments |
| **quiche** | BoringSSL (C) | Cloudflare-maintained, HTTP/3 server | CDN/edge, BoringSSL required |

```toml
[dependencies]
quinn = "0.11"
rustls = { version = "0.23", features = ["ring"] }
tokio = { version = "1", features = ["full"] }
rcgen = "0.13"
```

---

## 2. Quinn Server — Endpoint, Configuration, Incoming

A Quinn endpoint binds to a UDP socket. Each incoming connection is handled in its own task. Streams within a connection are multiplexed independently.

```rust
use quinn::{Endpoint, ServerConfig};
use rustls::pki_types::{CertificateDer, PrivateKeyDer, PrivatePkcs8KeyDer};
use std::{net::SocketAddr, sync::Arc};

fn configure_server() -> Result<(ServerConfig, CertificateDer<'static>), Box<dyn std::error::Error>> {
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".into()])?;
    let cert_der = CertificateDer::from(cert.cert);
    let priv_key = PrivateKeyDer::Pkcs8(PrivatePkcs8KeyDer::from(cert.key_pair.serialize_der()));

    let mut server_crypto = rustls::ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(vec![cert_der.clone()], priv_key)?;

    server_crypto.alpn_protocols = vec![b"h3".to_vec(), b"hq-interop".to_vec()];

    let transport = quinn::TransportConfig::default();
    let mut server_config = ServerConfig::with_crypto(Arc::new(server_crypto));
    server_config.transport = Arc::new(transport);

    Ok((server_config, cert_der))
}

async fn run_quinn_server(addr: SocketAddr) -> Result<(), Box<dyn std::error::Error>> {
    let (server_config, _cert) = configure_server()?;
    let endpoint = Endpoint::server(server_config, addr)?;

    println!("QUIC server listening on {addr}");

    while let Some(conn) = endpoint.accept().await {
        tokio::spawn(handle_connection(conn));
    }

    Ok(())
}

async fn handle_connection(conn: quinn::Incoming) {
    match conn.await {
        Ok(connection) => {
            println!("connection established: {}", connection.remote_address());
            while let Ok(stream) = connection.accept_bi().await {
                tokio::spawn(handle_bidirectional_stream(stream));
            }
        }
        Err(e) => eprintln!("connection failed: {e}"),
    }
}

async fn handle_bidirectional_stream(
    stream: (quinn::SendStream, quinn::RecvStream),
) {
    let (mut send, mut recv) = stream;
    let mut buf = vec![0u8; 4096];

    match recv.read(&mut buf).await {
        Ok(Some(n)) => {
            let response = b"HTTP/3 200 OK\r\n";
            send.write_all(response).await.ok();
            send.finish().ok();
        }
        Ok(None) => {}
        Err(e) => eprintln!("stream read error: {e}"),
    }
}
```

---

## 3. Stream Multiplexing

QUIC supports unlimited independent streams per connection. Each stream has its own flow control and can be read/written concurrently. There is no head-of-line blocking between streams.

```rust
use quinn::{Connection, SendStream, RecvStream};

async fn multiplex_request(
    connection: &Connection,
    requests: Vec<Vec<u8>>,
) -> Vec<Result<Vec<u8>, quinn::WriteError>> {
    let mut handles = Vec::new();

    for req in requests {
        let conn = connection.clone();
        handles.push(tokio::spawn(async move {
            let (mut send, mut recv) = conn.open_bi().await?;
            send.write_all(&req).await?;
            send.finish()?;

            let mut response = Vec::new();
            recv.read_to_end(&mut response).await.map_err(|e| {
                quinn::WriteError::ConnectionLost(quinn::ConnectionError::ApplicationClosed(
                    quinn::ApplicationClose::default(),
                ))
            })?;
            Ok::<_, quinn::WriteError>(response)
        }));
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}
```

---

## 4. s2n-quic — Low-Latency Alternative

`s2n-quic` is designed for AWS workloads. It provides a `Server` type with a `Provider` pattern.

```rust
async fn run_s2n_quic_server(addr: std::net::SocketAddr) -> Result<(), Box<dyn std::error::Error>> {
    let tls = s2n_quic::provider::tls::default::Server::builder()
        .with_certificate(
            include_bytes!("cert.pem"),
            include_bytes!("key.pem"),
        )?
        .with_application_protocols(&[b"h3"])?
        .build()?;

    let mut server = s2n_quic::Server::builder()
        .with_io(addr)?
        .with_tls(tls)?
        .start()
        .unwrap();

    while let Some(mut conn) = server.accept().await {
        tokio::spawn(async move {
            while let Ok(Some(mut stream)) = conn.accept_bidirectional_stream().await {
                tokio::spawn(async move {
                    let mut data = vec![0u8; 65535];
                    match stream.receive(&mut data).await {
                        Ok(Some(len)) => {
                            stream.send(&data[..len]).await.ok();
                        }
                        _ => {}
                    }
                });
            }
        });
    }

    Ok(())
}
```

---

## 5. 0-RTT — Security Considerations

0-RTT (Early Data) allows the client to send data before the TLS handshake completes. This data is replayable — an attacker can capture and resend it. Application protocols must mark 0-RTT-safe endpoints as idempotent only.

```rust
use quinn::{Connection, ZeroRttAccepted};

async fn handle_0rtt(connection: Connection) -> Result<(), Box<dyn std::error::Error>> {
    let zerortt = match connection.accept_bi().await {
        Ok(stream) => stream,
        Err(quinn::ConnectionError::ZeroRttRejected) => {
            eprintln!("0-RTT rejected by server, retry without early data");
            return Ok(());
        }
        Err(e) => return Err(e.into()),
    };

    let (mut send, mut recv) = zerortt;
    let mut early_data = vec![0u8; 1024];

    let n = match recv.read(&mut early_data).await? {
        Some(n) => n,
        None => return Ok(()),
    };

    let request = std::str::from_utf8(&early_data[..n])?;

    if !is_idempotent_request(request) {
        eprintln!("non-idempotent request in 0-RTT, rejecting");
        send.write_all(b"425 Too Early\r\n").await?;
        return Ok(());
    }

    send.write_all(b"HTTP/3 200 OK\r\nresponse-body\r\n").await?;
    send.finish()?;
    Ok(())
}

fn is_idempotent_request(request: &str) -> bool {
    request.starts_with("GET") || request.starts_with("HEAD") || request.starts_with("OPTIONS")
}
```

---

## 6. Connection Migration

QUIC identifies connections by a Connection ID, not by IP/port tuple. When a client's IP changes (Wi-Fi to cellular), the connection survives seamlessly.

```rust
use quinn::TransportConfig;

fn migration_config() -> TransportConfig {
    let mut transport = TransportConfig::default();

    transport.max_idle_timeout(Some(
        quinn::IdleTimeout::from(quinn::VarInt::from_u32(30_000)),
    ));

    transport.max_concurrent_bidi_streams(quinn::VarInt::from_u32(100));
    transport.max_concurrent_uni_streams(quinn::VarInt::from_u32(100));

    transport
}
```

---

## Red Lines

1. 0-RTT data must be replay-safe and idempotent. Never process `POST`, `PUT`, `DELETE`, or `PATCH` in 0-RTT.
2. Always set `alpn_protocols` on both client and server. ALPN is mandatory for HTTP/3 (`h3`).
3. Connection IDs must be cryptographically random. Never use sequential or predictable CIDs — they enable connection tracking attacks.
4. Stream limits (`max_concurrent_bidi_streams`) must be enforced. Unbounded streams are a DoS vector.
5. `quiche` requires BoringSSL. Do not use `quiche` if your deployment cannot ship a C TLS library.

---

## References

- [Quinn docs](https://docs.rs/quinn/latest/quinn/)
- [s2n-quic docs](https://docs.rs/s2n-quic/latest/s2n_quic/)
- [quiche docs](https://docs.rs/quiche/latest/quiche/)
- [RFC 9000 — QUIC Transport](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 9001 — QUIC TLS](https://datatracker.ietf.org/doc/html/rfc9001)
- [RFC 9114 — HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)