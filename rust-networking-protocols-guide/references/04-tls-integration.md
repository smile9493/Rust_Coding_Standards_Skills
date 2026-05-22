# 04 — TLS Integration

**Status**: Core  
**Prerequisites**: `rustls` ≥ 0.23, `rustls-pki-types`, `rustls-acme` ≥ 0.10

---

## Overview

TLS is mandatory for all production network protocols. `rustls` provides a pure-Rust TLS implementation with no OpenSSL dependency. This document covers server/client configuration, mutual TLS (mTLS), ALPN negotiation, automated certificate management via Let's Encrypt, and certificate pinning.

---

## 1. rustls ServerConfig — TLS Server Setup

A `ServerConfig` bundles the certificate chain, private key, and ALPN protocols. It is wrapped in `Arc` so multiple connections can share it.

```rust
use rustls::ServerConfig;
use rustls::pki_types::{CertificateDer, PrivateKeyDer, PrivatePkcs8KeyDer};
use std::sync::Arc;

fn build_server_config(
    cert_chain: Vec<CertificateDer<'static>>,
    key: PrivateKeyDer<'static>,
    alpn_protos: Vec<Vec<u8>>,
) -> Result<ServerConfig, rustls::Error> {
    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(cert_chain, key)?;

    config.alpn_protocols = alpn_protos;
    config.key_log = Arc::new(rustls::KeyLogFile::new());

    config.max_early_data_size = 0;
    Ok(config)
}

fn generate_self_signed() -> Result<(Vec<CertificateDer<'static>>, PrivateKeyDer<'static>), Box<dyn std::error::Error>> {
    let cert = rcgen::generate_simple_self_signed(vec![
        "example.com".into(),
        "*.example.com".into(),
    ])?;

    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivateKeyDer::Pkcs8(PrivatePkcs8KeyDer::from(
        cert.key_pair.serialize_der(),
    ));

    Ok((vec![cert_der], key_der))
}
```

---

## 2. rustls ClientConfig — TLS Client Setup

`ClientConfig` requires root CA certificates for server validation. Use `webpki-roots` for system trust store or pin specific certificates.

```rust
use rustls::ClientConfig;
use rustls::pki_types::ServerName;
use std::sync::Arc;

fn build_client_config(alpn_protos: Vec<Vec<u8>>) -> Result<ClientConfig, rustls::Error> {
    let root_store = rustls::RootCertStore {
        roots: webpki_roots::TLS_SERVER_ROOTS.to_vec(),
    };

    let mut config = ClientConfig::builder()
        .with_root_certificates(root_store)
        .with_no_client_auth();

    config.alpn_protocols = alpn_protos;
    config.key_log = Arc::new(rustls::KeyLogFile::new());

    Ok(config)
}

async fn tls_connect_example(domain: &str) -> Result<(), Box<dyn std::error::Error>> {
    let config = build_client_config(vec![b"h2".to_vec(), b"http/1.1".to_vec()])?;
    let connector = tokio_rustls::TlsConnector::from(Arc::new(config));

    let stream = tokio::net::TcpStream::connect((domain, 443)).await?;
    let server_name = ServerName::try_from(domain)?.to_owned();

    let tls_stream = connector.connect(server_name, stream).await?;
    let (_, negotiated) = tls_stream.get_ref();
    println!("ALPN negotiated: {:?}", negotiated.alpn_protocol());

    Ok(())
}
```

---

## 3. mTLS — Mutual TLS Authentication

Mutual TLS requires the server to verify the client's certificate. Both sides present certificates. The server's `ClientCertVerifier` determines which client certificates to trust.

```rust
use rustls::server::WebPkiClientVerifier;
use rustls::RootCertStore;
use std::sync::Arc;

fn build_mtls_server_config(
    server_certs: Vec<CertificateDer<'static>>,
    server_key: PrivateKeyDer<'static>,
    client_ca_certs: Vec<CertificateDer<'static>>,
) -> Result<ServerConfig, Box<dyn std::error::Error>> {
    let mut client_root_store = RootCertStore::empty();
    for ca in client_ca_certs {
        client_root_store.add(ca)?;
    }

    let client_verifier = WebPkiClientVerifier::builder(Arc::new(client_root_store))
        .build()?;

    let mut config = ServerConfig::builder()
        .with_client_cert_verifier(client_verifier)
        .with_single_cert(server_certs, server_key)?;

    config.alpn_protocols = vec![b"h2".to_vec()];
    Ok(config)
}

fn build_mtls_client_config(
    client_cert_chain: Vec<CertificateDer<'static>>,
    client_key: PrivateKeyDer<'static>,
    server_ca_certs: Vec<CertificateDer<'static>>,
) -> Result<ClientConfig, Box<dyn std::error::Error>> {
    let mut root_store = RootCertStore::empty();
    for ca in server_ca_certs {
        root_store.add(ca)?;
    }

    let config = ClientConfig::builder()
        .with_root_certificates(root_store)
        .with_client_auth_cert(client_cert_chain, client_key)?;

    Ok(config)
}
```

---

## 4. ALPN Negotiation

Application-Layer Protocol Negotiation (ALPN) lets the client and server agree on the next-layer protocol during the TLS handshake. This is how a single port (443) can serve both HTTP/2 and HTTP/1.1.

```rust
fn configure_alpn_for_h2_h1() -> (Vec<Vec<u8>>, Vec<Vec<u8>>) {
    let server_protos = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
    let client_protos = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
    (server_protos, client_protos)
}

async fn route_by_alpn(stream: tokio::net::TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    let config = Arc::new(build_server_config(
        vec![],
        PrivateKeyDer::Pkcs8(PrivatePkcs8KeyDer::from(vec![])),
        vec![b"h2".to_vec(), b"http/1.1".to_vec()],
    )?);

    let acceptor = tokio_rustls::TlsAcceptor::from(config);
    let tls_stream = acceptor.accept(stream).await?;
    let (_, conn) = tls_stream.get_ref();

    match conn.alpn_protocol() {
        Some(b"h2") => {
            println!("routing to HTTP/2 handler");
        }
        Some(b"http/1.1") => {
            println!("routing to HTTP/1.1 handler");
        }
        other => {
            eprintln!("unknown ALPN protocol: {other:?}");
        }
    }

    Ok(())
}
```

---

## 5. rustls-acme — Let's Encrypt Automation

`rustls-acme` integrates with ACME providers (Let's Encrypt) for automatic certificate issuance and renewal.

```rust
use rustls_acme::caches::DirCache;
use rustls_acme::AcmeConfig;
use std::path::PathBuf;

async fn setup_acme(domains: Vec<String>) -> Result<(), Box<dyn std::error::Error>> {
    let mut config = AcmeConfig::new(domains)
        .contact_email("admin@example.com")
        .cache_option(Some(DirCache::new(PathBuf::from("./acme_cache"))))
        .directory_lets_encrypt(true);

    let acceptor = config.axum_acceptor(config.default_rustls_config());

    tokio::spawn(async move {
        loop {
            match acceptor.next().await {
                Ok(Some(_accepted)) => {}
                Ok(None) => break,
                Err(e) => eprintln!("ACME error: {e}"),
            }
        }
    });

    Ok(())
}
```

---

## 6. Certificate Pinning

Pin the server's public key or certificate hash to prevent MITM attacks even if a CA is compromised.

```rust
use ring::digest::{digest, SHA256};
use rustls::client::danger::{HandshakeSignatureValid, ServerCertVerified, ServerCertVerifier};
use rustls::pki_types::{CertificateDer, ServerName, UnixTime};
use rustls::{DigitallySignedStruct, Error, SignatureScheme};

struct PinnedCertVerifier {
    expected_hash: [u8; 32],
}

impl ServerCertVerifier for PinnedCertVerifier {
    fn verify_server_cert(
        &self,
        end_entity: &CertificateDer<'_>,
        _intermediates: &[CertificateDer<'_>],
        _server_name: &ServerName<'_>,
        _ocsp_response: &[u8],
        _now: UnixTime,
    ) -> Result<ServerCertVerified, rustls::Error> {
        let cert_hash = digest(&SHA256, end_entity.as_ref());
        if cert_hash.as_ref() == self.expected_hash {
            Ok(ServerCertVerified::assertion())
        } else {
            Err(rustls::Error::General("certificate hash mismatch".into()))
        }
    }

    fn verify_tls12_signature(
        &self,
        _message: &[u8],
        _cert: &CertificateDer<'_>,
        _dss: &DigitallySignedStruct,
    ) -> Result<HandshakeSignatureValid, Error> {
        Ok(HandshakeSignatureValid::assertion())
    }

    fn verify_tls13_signature(
        &self,
        _message: &[u8],
        _cert: &CertificateDer<'_>,
        _dss: &DigitallySignedStruct,
    ) -> Result<HandshakeSignatureValid, Error> {
        Ok(HandshakeSignatureValid::assertion())
    }

    fn supported_verify_schemes(&self) -> Vec<SignatureScheme> {
        vec![
            SignatureScheme::RSA_PSS_SHA256,
            SignatureScheme::ECDSA_NISTP256_SHA256,
            SignatureScheme::ED25519,
        ]
    }
}

fn build_pinned_client_config(pin_hash: [u8; 32]) -> ClientConfig {
    let verifier = Arc::new(PinnedCertVerifier {
        expected_hash: pin_hash,
    });

    ClientConfig::builder()
        .dangerous()
        .with_custom_certificate_verifier(verifier)
        .with_no_client_auth()
}
```

---

## Red Lines

1. Never disable certificate verification in production. `dangerous_configuration` is for tests and pinning only.
2. Validate hostnames against the certificate's SAN (Subject Alternative Name). `ServerName::try_from` handles this.
3. ALPN must be explicitly set on both client and server. No default protocol guessing.
4. Certificate private keys must never be logged or hardcoded. Use environment variables or secret managers.
5. Let's Encrypt certificates expire after 90 days. `rustls-acme` auto-renews, but verify the renewal path works before deployment.
6. `key_log` must be disabled in production to prevent session key leakage via `SSLKEYLOGFILE`.

---

## References

- [rustls docs](https://docs.rs/rustls/latest/rustls/)
- [tokio-rustls docs](https://docs.rs/tokio-rustls/latest/tokio_rustls/)
- [rustls-acme docs](https://docs.rs/rustls-acme/latest/rustls_acme/)
- [webpki-roots](https://docs.rs/webpki-roots/latest/webpki_roots/)
- [RC 8446 — TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [RFC 7301 — ALPN](https://datatracker.ietf.org/doc/html/rfc7301)