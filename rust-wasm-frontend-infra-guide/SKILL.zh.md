---
name: rust-wasm-frontend-infra-guide
description: >
  针对 wasm32-unknown-unknown 目标的 Rust 代码的硬约束，涵盖编译配置、跨语言边界、
  线性内存管理、并发模型和通用代码适配。本 Skill 是所有 Rust+Wasm 前端架构
  （包括无 DOM 渲染引擎）的编译与边界层基础，必须被继承和遵守。
metadata:
  version: "4.1.0"
  philosophy: "Dialectical Materialism & Jeet Kune Do — Wasm Vertical Base"
  domain: "wasm32-unknown-unknown compilation & boundary layer"
  relationship: "vertical-deepening-of:rust-architecture-guide"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Leptos Binary Size Guide", "twiggy diagnostics", "wasm-opt -Oz", "talc allocator", "Wasm Component Model proposals"]
---

# Rust to Wasm 垂直编译与边界规范 V4.1.0

本规范继承 Rust 架构指南和 Rust 项目生命周期指南的核心哲学，针对 `wasm32-unknown-unknown` 目标的特殊性（线性内存、单线程事件循环、跨语言边界）进行垂直深化。

V4.1.0 — **二进制体积经济学与生态收敛**：
- 对齐 Leptos 官方二进制体积优化指南：流式 WASM 编译原理、`twiggy` 顶层函数分析
- 推荐 `talc` 分配器（替代已废弃的 `wee_alloc`），附带量化基准线（约 10KB → 约 1KB）
- 引用 Wasm Component Model 提案，用于未来规范 ABI 对齐
- 在工具链中新增 `wasm-tools` 剥离和 `wasm-objdump` 体积审计

## 架构哲学：唯物辩证法与截拳道，虚实合一

本规范是"唯物辩证法与截拳道"架构哲学在编译和边界层的工程实现。与 `rust-architecture-guide` 冲突时，架构指南作为**宪法基础**（参见 [constitution conflict resolution](../../rust-architecture-guide/references/06-deviation-process.md)）。本 Wasm 规范为 `wasm32-unknown-unknown` 目标提供**垂直默认指导**，而非对高级规则的绝对否决权。

### 0.1 唯物主义基础：敬畏物理边界

- **线性内存只增不减**，必须在内部回收。
- **FFI 边界是昂贵的 RPC**，任何隐式转换都是性能负债。
- **浏览器是沙箱化的操作系统**，事件循环、多线程限制和安全策略是必须遵守的物理法则。

### 0.2 辩证法核心：拥抱并转化矛盾

- **`unsafe` 是安全抽象的物质基础** —— Wasm 边界本质上是 `unsafe` 的，但这恰恰是零拷贝的物理前提。
- **二元对立引导实践** —— 双缓冲区分 Front/Back Buffer。Front Buffer（实）由 Wasm 消耗，Back Buffer（虚）由 JS 写入。`AcqRel` 交换使虚变实。
- **量变到质变** —— Arena 帧级分配积累，通过 `reset()` 实现批量回收，而非逐个释放。
