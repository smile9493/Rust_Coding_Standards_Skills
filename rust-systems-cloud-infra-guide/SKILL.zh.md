---
name: rust-systems-cloud-infra-guide
description: 在构建 Rust 云原生基础设施（数据库内核、分布式存储、高性能网关、容器运行时、eBPF 控制平面）时使用。涵盖 I/O 模型选型、零拷贝管道、背压机制、确定性共识、无锁并发、SIMD 向量化、高级内存架构、防波堤模式和物理可行性审计。作为 rust-architecture-guide 对长时间运行系统的垂直深化。
metadata:
  version: "6.1.0"
  philosophy: "Mechanical Sympathy, Determinism, Resilience, Jeet Kune Do, Unity of False and Real"
  domain: "cloud-native infrastructure"
  relationship: "vertical-deepening-of:rust-architecture-guide"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Tokio Graceful Shutdown", "io_uring Best Practices", "Safety-Critical Rust Guidelines", "Raft/Paxos Determinism Patterns"]
---

# Rust 系统与云基础设施指南 V6.1.0

作为 `rust-architecture-guide` 对世界级云原生基础设施的垂直深化。假设长时间运行节点（正常运行时间 > 1 年）、10GbE+ 网络、多 NUMA 架构。

V6.0.0 新增内容：
1. **防波堤架构**（ref/12）：云服务的 Facade/Core 分层架构 — 高容错外部 API，零过载内部核心
2. **物理可行性审计**（ref/13）：容器内存限制、网络延迟预算、NUMA 拓扑感知

V6.1.0 — **现代韧性与可观测性**：
- 集成 `tokio-graceful-shutdown` 和 `async-shutdown` crate 模式，实现生产级优雅关闭
- 新增 `tokio::task::JoinSet` 结构化并发，用于任务生命周期管理
- 融入 Tokio 官方优雅关闭最佳实践和 `CancellationToken` 模式
- 新增 `ntex-neon-uring` 基准测试感知，用于量化 io_uring 与 epoll 的权衡

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **机械同理心** | 使软件与硬件物理特性对齐（CPU 缓存、NUMA、PMEM、内核 I/O 栈） |
| **确定性** | 消除非确定性（时间、随机数、HashMap 顺序），实现可复现的状态机 |
| **韧性** | 优雅降级优于崩溃；背压优于 OOM；结构化并发优于泄漏 |
| **截拳道** | 一击内存生命周期（Arena）；如水般流向硬件通道（Allocator API） |
