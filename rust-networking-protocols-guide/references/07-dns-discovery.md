# 07 — DNS & Service Discovery

**Status**: Core  
**Prerequisites**: `hickory-resolver` ≥ 0.24, `hickory-proto` ≥ 0.24

---

## Overview

DNS resolution is the first step in every outbound connection. hickory-resolver (formerly trust-dns) provides an async, caching DNS resolver with built-in DNSSEC validation, DoH (DNS-over-HTTPS), and DoT (DNS-over-TLS) support. For service discovery, SRV records encode priority and weight for load distribution. Happy Eyeballs races IPv6 and IPv4 to minimize connection latency.

---

## 1. hickory-resolver — Async DNS with Caching

`hickory-resolver` builds on `hickory-proto` to provide a high-level async resolver. Configuration covers upstream servers, cache size, and DNS protocol selection.

```rust
use hickory_resolver::config::{ResolverConfig, ResolverOpts};
use hickory_resolver::TokioAsyncResolver;
use std::time::Duration;

async fn build_resolver() -> Result<TokioAsyncResolver, Box<dyn std::error::Error>> {
    let mut opts = ResolverOpts::default();
    opts.timeout = Duration::from_secs(5);
    opts.cache_size = 1024;
    opts.positive_min_ttl = Some(Duration::from_secs(60));
    opts.positive_max_ttl = Some(Duration::from_secs(86400));
    opts.negative_min_ttl = Some(Duration::from_secs(10));
    opts.negative_max_ttl = Some(Duration::from_secs(3600));
    opts.num_concurrent_reqs = 4;
    opts.use_hosts_file = true;
    opts.validate = true;

    let resolver =
        TokioAsyncResolver::tokio(ResolverConfig::google(), opts);

    Ok(resolver)
}

async fn resolve_host(
    resolver: &TokioAsyncResolver,
    hostname: &str,
) -> Result<Vec<std::net::IpAddr>, Box<dyn std::error::Error>> {
    let response = resolver.lookup_ip(hostname).await?;
    let addresses: Vec<std::net::IpAddr> = response.iter().collect();

    for addr in &addresses {
        println!("resolved {hostname} -> {addr} (TTL: {:?})", response.record_iter().next().map(|r| r.ttl()));
    }

    Ok(addresses)
}
```

---

## 2. DoH / DoT — Encrypted DNS

DoH sends DNS queries over HTTPS; DoT uses a dedicated TLS connection on port 853. hickory-resolver supports both natively.

```rust
use hickory_resolver::config::{
    NameServerConfig, NameServerConfigGroup, Protocol, ResolverConfig, ResolverOpts,
};
use hickory_resolver::TokioAsyncResolver;
use std::net::SocketAddr;

fn build_doh_resolver() -> TokioAsyncResolver {
    let mut name_servers = NameServerConfigGroup::new();

    name_servers.push(NameServerConfig {
        socket_addr: SocketAddr::new("8.8.8.8".parse().unwrap(), 443),
        protocol: Protocol::Https,
        tls_dns_name: Some("dns.google".to_string()),
        trust_negative_responses: true,
        bind_addr: None,
    });

    name_servers.push(NameServerConfig {
        socket_addr: SocketAddr::new("1.1.1.1".parse().unwrap(), 443),
        protocol: Protocol::Https,
        tls_dns_name: Some("cloudflare-dns.com".to_string()),
        trust_negative_responses: true,
        bind_addr: None,
    });

    let config = ResolverConfig::from_parts(None, vec![], name_servers);
    let mut opts = ResolverOpts::default();
    opts.validate = true;

    TokioAsyncResolver::tokio(config, opts)
}

fn build_dot_resolver() -> TokioAsyncResolver {
    let mut name_servers = NameServerConfigGroup::new();

    name_servers.push(NameServerConfig {
        socket_addr: SocketAddr::new("1.1.1.1".parse().unwrap(), 853),
        protocol: Protocol::Tls,
        tls_dns_name: Some("cloudflare-dns.com".to_string()),
        trust_negative_responses: true,
        bind_addr: None,
    });

    let config = ResolverConfig::from_parts(None, vec![], name_servers);
    TokioAsyncResolver::tokio(config, ResolverOpts::default())
}
```

---

## 3. SRV Records — Service Discovery with Priority/Weight

SRV records map a service name to a set of host:port pairs, each with a priority and weight. Lower priority values are preferred. Weight distributes traffic proportionally among servers with equal priority.

```rust
use hickory_resolver::TokioAsyncResolver;
use rand::Rng;

#[derive(Debug, Clone)]
struct SrvTarget {
    host: String,
    port: u16,
    priority: u16,
    weight: u16,
}

async fn resolve_srv(
    resolver: &TokioAsyncResolver,
    service: &str,
) -> Result<Vec<SrvTarget>, Box<dyn std::error::Error>> {
    let response = resolver.srv_lookup(service).await?;

    let mut targets: Vec<SrvTarget> = response
        .iter()
        .map(|srv| SrvTarget {
            host: srv.target().to_string(),
            port: srv.port(),
            priority: srv.priority(),
            weight: srv.weight(),
        })
        .collect();

    targets.sort_by_key(|t| t.priority);
    Ok(targets)
}

fn weighted_pick(targets: &[SrvTarget]) -> Option<&SrvTarget> {
    if targets.is_empty() {
        return None;
    }

    let min_priority = targets[0].priority;
    let same_prio: Vec<&SrvTarget> = targets
        .iter()
        .take_while(|t| t.priority == min_priority)
        .collect();

    let total_weight: u32 = same_prio.iter().map(|t| t.weight as u32).sum();

    if total_weight == 0 {
        let idx = rand::thread_rng().gen_range(0..same_prio.len());
        return Some(same_prio[idx]);
    }

    let mut rng = rand::thread_rng();
    let mut pick = rng.gen_range(0..total_weight);
    for target in same_prio {
        if pick < target.weight as u32 {
            return Some(target);
        }
        pick -= target.weight as u32;
    }

    same_prio.last().copied()
}

async fn discover_and_connect(
    resolver: &TokioAsyncResolver,
    service: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    let targets = resolve_srv(resolver, service).await?;
    println!("discovered {} targets for {service}", targets.len());

    if let Some(target) = weighted_pick(&targets) {
        println!(
            "connecting to {}:{} (priority={}, weight={})",
            target.host, target.port, target.priority, target.weight
        );
    }

    Ok(())
}
```

---

## 4. Happy Eyeballs — IPv6/IPv4 Racing

Happy Eyeballs (RFC 8305) races IPv6 and IPv4 connections, using whichever succeeds first. This mitigates broken IPv6 configurations without adding latency.

```rust
use hickory_resolver::TokioAsyncResolver;
use std::net::SocketAddr;
use tokio::net::TcpStream;

async fn happy_eyeballs_connect(
    resolver: &TokioAsyncResolver,
    host: &str,
    port: u16,
    connect_timeout: std::time::Duration,
) -> Result<TcpStream, Box<dyn std::error::Error>> {
    let ips = resolver.lookup_ip(host).await?;

    let mut v6_addrs: Vec<SocketAddr> = Vec::new();
    let mut v4_addrs: Vec<SocketAddr> = Vec::new();

    for ip in ips {
        let addr = SocketAddr::new(ip, port);
        match ip {
            std::net::IpAddr::V6(_) => v6_addrs.push(addr),
            std::net::IpAddr::V4(_) => v4_addrs.push(addr),
        }
    }

    if v6_addrs.is_empty() {
        return Ok(tokio::time::timeout(
            connect_timeout,
            TcpStream::connect(&v4_addrs[..][0]),
        )
        .await??);
    }
    if v4_addrs.is_empty() {
        return Ok(tokio::time::timeout(
            connect_timeout,
            TcpStream::connect(&v6_addrs[..][0]),
        )
        .await??);
    }

    let ipv6_v4_offset = Duration::from_millis(50);

    let v6_future = tokio::time::timeout(
        connect_timeout,
        TcpStream::connect(v6_addrs[0]),
    );
    let v4_future = async {
        tokio::time::sleep(ipv6_v4_offset).await;
        tokio::time::timeout(
            connect_timeout.saturating_sub(ipv6_v4_offset),
            TcpStream::connect(v4_addrs[0]),
        )
        .await
    };

    tokio::select! {
        result = v6_future => {
            match result {
                Ok(Ok(stream)) => {
                    println!("IPv6 connected first");
                    Ok(stream)
                }
                _ => {
                    let result = v4_future.await;
                    match result {
                        Ok(Ok(stream)) => {
                            println!("IPv4 fallback connected");
                            Ok(stream)
                        }
                        _ => Err("all connection attempts failed".into()),
                    }
                }
            }
        }
        result = v4_future => {
            match result {
                Ok(Ok(stream)) => {
                    println!("IPv4 connected first");
                    Ok(stream)
                }
                _ => {
                    let result = v6_future.await;
                    match result {
                        Ok(Ok(stream)) => {
                            println!("IPv6 fallback connected");
                            Ok(stream)
                        }
                        _ => Err("all connection attempts failed".into()),
                    }
                }
            }
        }
    }
}

use std::time::Duration;

async fn connect_example() -> Result<(), Box<dyn std::error::Error>> {
    let resolver = build_resolver().await?;
    let stream = happy_eyeballs_connect(
        &resolver,
        "example.com",
        443,
        Duration::from_secs(10),
    )
    .await?;

    println!("connected to {}", stream.peer_addr()?);
    Ok(())
}
```

---

## 5. DNSSEC Validation

hickory-resolver supports DNSSEC validation out of the box. Set `opts.validate = true` and the resolver will verify RRSIG records.

```rust
use hickory_resolver::config::ResolverOpts;

fn dnssec_config() -> ResolverOpts {
    let mut opts = ResolverOpts::default();
    opts.validate = true;
    opts.cache_size = 2048;
    opts
}
```

---

## Red Lines

1. DNS resolution must have a timeout. The default is 5 seconds. Infinite DNS wait = infinite connection wait.
2. DoH or DoT must be used in production. Plaintext DNS on port 53 leaks query information and is vulnerable to spoofing.
3. Happy Eyeballs must race IPv6 and IPv4 with a 50ms head start for IPv6. This is the RFC 8305 recommendation.
4. SRV record weight distribution must be proportional. Each server should receive traffic in proportion to `weight / sum_of_weights`.
5. DNSSEC validation must be enabled in production. Set `validate = true` in `ResolverOpts`.

---

## References

- [hickory-resolver docs](https://docs.rs/hickory-resolver/latest/hickory_resolver/)
- [hickory-proto docs](https://docs.rs/hickory-proto/latest/hickory_proto/)
- [RFC 1035 — DNS](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 2782 — SRV RR](https://datatracker.ietf.org/doc/html/rfc2782)
- [RFC 8305 — Happy Eyeballs v2](https://datatracker.ietf.org/doc/html/rfc8305)
- [RFC 8484 — DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484)
- [RFC 7858 — DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858)