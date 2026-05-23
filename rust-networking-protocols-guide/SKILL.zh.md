---
name: rust-networking-protocols-guide
description: 在构建 Rust 网络协议实现（QUIC、HTTP/3、gRPC、自定义 TCP/UDP 协议、WebSocket、DNS、TLS 终结代理）时使用。涵盖协议状态机建模、零拷贝解析、TLS 集成、拥塞控制、连接池和 QUIC 多路复用。作为 rust-architecture-guide 对网络协议工程的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Protocol First, Mechanical Sympathy, Defense in Depth, Jeet Kune Do"
  domain: "network protocol engineering"
  relationship: "vertical-deepening-of:[rust-architecture-guide, rust-systems-cloud-infra-guide]"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["QUIC RFC 9000", "HTTP/3 RFC 9114", "rustls architecture", "tokio codec", "nom/winnow parsing"]
---

# Rust 网络协议指南 V1.0.0

作为 `rust-architecture-guide` 和 `rust-systems-cloud-infra-guide` 对网络协议工程的垂直深化。涵盖从线格式解析到应用层协议状态机的完整栈。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **协议优先** | 协议规范是真理之源。代码是 RFC 的可执行版本。 |
| **机械同理心** | 线格式解析与 CPU 分支预测器对齐。每一层边界都是零拷贝。 |
| **纵深防御** | 来自网络的每个字节都是敌对的。在入口处验证。在集成处模糊测试。 |
| **截拳道** | 一趟解析。状态机将所有边界情况折叠为一组封闭转换。 |
