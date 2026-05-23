# Rust 编码标准技能

欢迎来到 Rust 编码标准技能文档 —— 一套全面、有主见的工程指南，涵盖十个专业领域中构建生产级 Rust 软件的最佳实践。

## 概述

本项目提供 **10 个技能指南**，每个指南专为 Cursor AI 代理设计，用于在编写 Rust 代码时强制执行编码标准、架构模式和领域最佳实践。

### 宪法

**[架构指南](rust-architecture-guide/SKILL.md)** 是通用宪法 —— 它定义了优先级金字塔（P0 安全 → P1 可维护性 → P2 编译时间 → P3 性能）、执行模式、错误处理策略、并发模式、FFI 安全边界以及决策摘要契约。所有垂直指南在跨领域问题上均服从架构指南。

### 垂直领域指南

每个垂直指南为其特定领域提供领域规则、参考架构和禁止模式：

| 指南 | 领域 |
|------|------|
| [系统与云基础设施](rust-systems-cloud-infra-guide/SKILL.md) | 后端服务、I/O 模型、无锁并发、CI/CD |
| [Wasm 前端基础设施](rust-wasm-frontend-infra-guide/SKILL.md) | WebAssembly 前端、零拷贝 FFI、帧分配 |
| [嵌入式与物联网](rust-embedded-iot-guide/SKILL.md) | `no_std` 裸金属、RTIC/Embassy、外设驱动 |
| [数据工程](rust-data-engineering-guide/SKILL.md) | Arrow 列式内存、流式 ETL、查询优化 |
| [网络协议](rust-networking-protocols-guide/SKILL.md) | QUIC/HTTP3、TLS、零拷贝解析、拥塞控制 |
| [CLI 与 DevOps](rust-cli-devops-guide/SKILL.md) | 基于 clap 的 CLI、分发、K8s Operator、信号处理 |
| [游戏开发](rust-gamedev-guide/SKILL.md) | Bevy ECS、帧预算、wgpu 渲染、资源管线 |
| [区块链](rust-blockchain-guide/SKILL.md) | Solana/NEAR 程序、Anchor 框架、Stacks 清晰合约 |
| [AI/ML 推理](rust-ai-ml-inference-guide/SKILL.md) | 模型加载、量化、KV-cache、推理服务 API |

## 核心概念

### 优先级金字塔

所有设计决策均通过四级优先级层级来解决：

1. **P0 —— 安全与正确性**：内存安全、类型可靠性、热路径无 panic
2. **P1 —— 可维护性与清晰度**：可读性、惯用模式、意图明确
3. **P2 —— 编译时间**：构建速度、单态化膨胀、增量编译
4. **P3 —— 性能**：微优化、SIMD、缓存局部性、减少分配

当两条规则冲突时，优先级更高的规则胜出。永远不要为了性能牺牲安全。

### 执行模式

每条规则都标有执行模式，决定其执行级别：

- **`rapid`（快速）**：原型开发 —— 规则仅为建议
- **`standard`（标准）**：默认模式 —— 大多数规则强制执行，允许合理例外
- **`strict`（严格）**：生产模式 —— 所有规则强制执行，偏离需正式文档记录

请参阅[架构指南的模式文档](rust-architecture-guide/references/00-mode-guide.md)，了解详细的模式定义和转换检查清单。

## 入门指南

1. **浏览[架构指南](rust-architecture-guide/SKILL.md)**，了解基础规则
2. **从导航侧边栏选择您的领域指南**
3. **每个 SKILL.md** 提供概述、红线规则和指向详细文档的参考表
4. **参考文件**（`references/NN-topic.md`）包含代码示例、决策树和实现模式

## 项目结构

```
Rust_Coding_Standards_Skills/
├── rust-architecture-guide/          # 通用宪法（33 个参考文档）
│   ├── SKILL.md
│   └── references/
├── rust-systems-cloud-infra-guide/   # 后端基础设施（14 个参考文档）
├── rust-wasm-frontend-infra-guide/   # Wasm 前端（13 个参考文档）
├── rust-embedded-iot-guide/          # 嵌入式系统（8 个参考文档）
├── rust-data-engineering-guide/      # 数据工程（8 个参考文档）
├── rust-networking-protocols-guide/  # 网络协议（8 个参考文档）
├── rust-cli-devops-guide/            # CLI 与 DevOps（8 个参考文档）
├── rust-gamedev-guide/               # 游戏开发（8 个参考文档）
├── rust-blockchain-guide/            # 区块链（8 个参考文档）
└── rust-ai-ml-inference-guide/       # AI/ML 推理（8 个参考文档）
```

**总计：10 个指南，116 个参考文档**
