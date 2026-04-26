<div align="center">

# Rust Coding Standards Skills

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Skill Version](https://img.shields.io/badge/Skill-3.0.0-brightgreen.svg)]()
[![Reference Docs](https://img.shields.io/badge/Reference-26%20Docs-orange.svg)]()
[![Platform](https://img.shields.io/badge/Platform-Trae%20%7C%20OpenClaw-9cf.svg)]()

**Rust 工程决策宪法 — 让 AI 编码助手输出确定性的、可复现的架构决策**

[快速开始](#-快速开始) · [核心特性](#-核心特性) · [命令参考](#-命令参考) · [设计哲学](#-设计哲学) · [参考文档](#-参考文档)

</div>

---

## 📖 项目简介

Rust Coding Standards Skills 是一个面向 AI 编码助手的 **Rust 工程决策指南 Skill**，覆盖从架构决策、编码风格到生产最佳实践的完整链路。

本 Skill 作为 Agent 的**宪法性基础**，确保每次代码生成、审查、重构都遵循统一的优先级判定和冲突解决框架，避免"看心情写代码"。

### 🎯 解决的问题

| 没有 Skill | 有本 Skill |
|-----------|-----------|
| AI 随意选择 `unwrap` 还是 `expect` | 按 P0/P1 优先级自动判定 |
| 泛型单态化爆炸无人把关 | 三级退火策略自动降级 |
| 错误处理库级/应用级混用 | `thiserror` / `anyhow` 分层隔离 |
| 并发模型凭直觉选 | MPSC bounded + 最小锁区 + 禁止 Mutex 跨 await |
| API 演进破坏兼容性 | `#[non_exhaustive]` + Sealed Trait 防御 |

---

## ✨ 核心特性

- 🏛️ **四级优先级金字塔** — P0 安全 → P1 可维护 → P2 编译时 → P3 运行时性能，冲突时高优先级否决低优先级
- 🧬 **类型驱动架构** — 状态机、Newtype、零成本抽象边界（Marker Traits / PhantomData / 单态化退火）
- 📦 **所有权分层策略** — 业务层 Owned + `.clone()` 解耦，热点层 `Cow`/`Bytes` 零拷贝
- ⚠️ **错误处理分层** — 库级 `thiserror` 结构化，应用级 `anyhow` + 惰性上下文 + 重试/退避/熔断
- 🔄 **并发与异步规范** — 有界通道背压、RwLock/parking_lot 策略、死锁预防、`Pin`/`Unpin` 语义、`select!`/`join!` 组合、取消安全
- 🛡️ **API 演进防御** — `#[non_exhaustive]`、Sealed Trait、`#[deprecated]` 迁移、Trait Object Safety、Builder 必填字段
- 🔍 **可观测性三件套** — `tracing::instrument` + 零开销 Metrics + Panic Hook → abort
- 🧪 **高阶质量保证** — proptest 属性测试 + cargo fuzz + Loom 并发模型 + Miri UB 检测
- ⚡ **性能深度调优** — jemalloc/mimalloc、SoA 布局、SmallVec 栈分配、PGO、False Sharing 防御、SIMD、LTO
- 🔗 **FFI 安全边界** — `-sys` crate 分离、`cxx` 安全 C++ 互操作、回调 trampoline 模式、`catch_unwind`
- 🏗️ **工程构建规范** — Cargo Workspace 分治、Feature Flags 隔离、`cargo deny` 供应链审计、rustfmt 强制、CI 流水线
- 🤖 **Agent 自检指令** — Trade-off 分析 + 所有权/并发/错误/安全四类审查清单（66 项）

---

## 🚀 快速开始

### 安装

```bash
# 方式一：克隆到工作空间 skills 目录（推荐）
git clone https://github.com/smile9493/Rust_Coding_Standards_Skills.git
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.trae/skills/

# 方式二：直接在项目中使用
# 将 rust-architecture-guide/ 目录复制到项目的 .trae/skills/ 下
```

### 使用

在 Trae IDE 中直接调用：

```
/rust-architecture-guide priority my_conflict
/rust-architecture-guide state-machine Order
/rust-architecture-guide review src/
```

或自然语言触发：

```
帮我用类型驱动状态机重构 Order 实体
这个模块的错误处理应该用 thiserror 还是 anyhow？
审查这段并发代码有没有 Mutex 跨 await 的问题
```

---

## 📋 命令参考

### Strategy — 战略决策

| 命令 | 说明 | 参考文档 |
|------|------|---------|
| `priority` | 应用优先级矩阵判定冲突 | `priority-pyramid.md` |
| `conflict` | 解决典型冲突场景 | `conflict-resolution.md` |
| `progressive` | MVP → Production 渐进式架构 | `progressive-architecture.md` |

### Design — 架构设计

| 命令 | 说明 | 参考文档 |
|------|------|---------|
| `state-machine` | 类型驱动状态机设计 | `state-machine.md` |
| `newtype` | Newtype 模式与类型安全 ID | `newtype.md` |
| `data-arch` | 数据架构与所有权策略 | `data-architecture.md` |
| `error-handling` | 错误处理分层规范 | `error-handling.md` |
| `concurrency` | 并发模型选择与规范 | `concurrency.md` |
| `async` | 异步运行时与规范 | `async-internals.md` |
| `select` | `select!`/`join!` 组合与取消安全 | `async-internals.md` |
| `cancellation` | 异步取消语义与安全模式 | `async-internals.md` |
| `api-design` | 公共 API 边界设计 | `api-design.md` |
| `non-exhaustive` | `#[non_exhaustive]` 向前兼容 | `api-design.md` |
| `sealed-trait` | Sealed Trait 模式 | `api-design.md` |
| `deprecated` | `#[deprecated]` 迁移策略 | `api-design.md` |
| `object-safety` | Trait Object Safety 规则 | `api-design.md` |

### Refactor — 重构路径

| 命令 | 说明 | 参考文档 |
|------|------|---------|
| `control-flow` | 控制流模式重构 | `control-flow.md` |
| `iterators` | 迭代器链式重构 | `iterators.md` |
| `traits` | 惯用 Trait 模式 | `traits.md` |
| `borrowing` | 借用与可变性优化 | `borrowing.md` |
| `interior-mutability` | 内部可变性模式（Cell/RefCell/OnceCell） | `borrowing.md` |

### Evaluate — 质量评估

| 命令 | 说明 | 参考文档 |
|------|------|---------|
| `review` | 综合代码审查 | `review.md` |
| `property-test` | 属性测试策略 | `advanced-testing.md` |
| `fuzz` | Fuzz 测试策略 | `advanced-testing.md` |
| `miri` | Miri UB 检测 | `advanced-testing.md` |
| `performance` | 性能调优指南 | `performance-tuning.md` |
| `smallvec` | 栈分配小集合策略 | `performance-tuning.md` |
| `pgo` | Profile-Guided Optimization | `performance-tuning.md` |

### Configure — 工程配置

| 命令 | 说明 | 参考文档 |
|------|------|---------|
| `toolchain` | 工具链与 CI 配置 | `toolchain.md` |
| `workspace` | Cargo Workspace 分治 | `toolchain.md` |
| `feature-flags` | Feature Flags 隔离 | `toolchain.md` |
| `cargo-deny` | 供应链安全审计 | `toolchain.md` |
| `rustfmt` | 代码格式化配置 | `toolchain.md` |
| `ci` | CI 流水线设计 | `toolchain.md` |
| `observability` | 可观测性配置 | `observability.md` |
| `ffi` / `interop` | FFI 与跨语言互操作 | `ffi-interop.md` |
| `cxx` | 安全 C++ 互操作 | `ffi-interop.md` |
| `metaprogramming` | 宏与元编程规范 | `metaprogramming.md` |

---

## 🧠 设计哲学

### 务实主义优于教条主义

> **系统边界与热点路径追求极致，内部流程与冷路径释放心智负担。**

### 优先级金字塔

```
P0: 安全与正确性（内存安全、数据一致性）— 不可妥协
                  ↓
P1: 可维护性（可读性、局部复杂度控制）— 默认追求
                  ↓
P2: 编译时（构建速度、CI/CD 效率）— 测量后决策
                  ↓
P3: 运行时性能（仅限已证明的瓶颈）— 需 Profiler 数据
```

### 冲突解决规则

- **高优先级否决低优先级**：P0 安全要求否决 P3 性能优化
- **同级取更简方案**：两个 P1 方案冲突，选更简单的
- **P3 需要证据**：任何性能优化必须附带 Profiler 数据

### Agent 决策日志

每次对话生成 Decision Log，记录：
1. 面临的冲突与优先级判定
2. 选择的方案与理由
3. 被否决的方案与原因

---

## 📚 参考文档

`reference/` 目录包含 26 份深度参考文档，按领域分类：

| 领域 | 文档 | 覆盖范围 |
|------|------|---------|
| **战略框架** | `priority-pyramid.md` `conflict-resolution.md` `trade-offs.md` `progressive-architecture.md` | 优先级矩阵、冲突解决、权衡分析、渐进式架构 |
| **架构设计** | `state-machine.md` `newtype.md` `data-architecture.md` `error-handling.md` `concurrency.md` `api-design.md` | 状态机、Newtype、数据架构、错误处理（重试/退避/分类）、并发（RwLock/parking_lot/死锁）、API 边界（deprecated/object safety） |
| **异步与互操作** | `async-internals.md` `ffi-interop.md` | Future 状态机、select!/join!、取消安全、Pin/Unpin、cxx 安全 C++ 互操作、回调 trampoline |
| **编码风格** | `control-flow.md` `iterators.md` `traits.md` `errors.md` `data-struct.md` `borrowing.md` | 控制流、迭代器（自定义/高级组合子）、Trait 惯用法、错误组合子、数据结构（repr/enum 布局/derive）、借用（内部可变性/分割借用） |
| **性能与质量** | `performance-tuning.md` `advanced-testing.md` `observability.md` | 内存/缓存/锁自由/编译时/unsafe 调优、SmallVec/PGO、属性/模糊/并发/UB 测试、Tracing/Metrics/Panic |
| **元编程与工具链** | `metaprogramming.md` `toolchain.md` | 声明/过程宏、const 泛型、Clippy/rustfmt/Workspace/Feature Flags/cargo deny/CI 流水线 |
| **审查与示例** | `review.md` `refactor.md` `usage-examples.md` | 综合审查清单、重构路径、使用示例 |

---

## 📁 项目结构

```
rust-architecture-guide/
  SKILL.md                          # Skill 入口（宪法性定义 + 完整指南）
  reference/                        # 26 份参考文档
    priority-pyramid.md             # 四级优先级体系
    conflict-resolution.md          # 典型冲突与解决方案
    progressive-architecture.md     # MVP → Production 渐进迁移
    state-machine.md                # 类型驱动状态机
    newtype.md                      # 类型安全 ID 与凭证
    data-architecture.md            # 所有权、克隆、内存布局
    error-handling.md               # 库级 vs 应用级错误策略
    concurrency.md                  # 消息传递、通道、锁、RwLock、parking_lot、死锁预防
    async-internals.md              # 异步内幕、select!/join!、取消安全、Pin/Unpin、自定义执行器
    api-design.md                   # 公共 API 边界、#[non_exhaustive]、Sealed Trait、#[deprecated]、Object Safety
    observability.md                # Tracing、Metrics、Panic Hook、Coredump
    toolchain.md                    # CI、Clippy、rustfmt、unsafe、Workspace、Feature Flags、cargo deny
    performance-tuning.md           # 完整性能调优指南（SmallVec、PGO）
    advanced-testing.md             # 属性测试、Fuzzing、Loom、Miri
    metaprogramming.md              # 声明/过程宏、const 泛型
    ffi-interop.md                  # FFI 边界、cxx 安全 C++ 互操作、回调 trampoline
    control-flow.md                 # 控制流模式
    iterators.md                    # 迭代器最佳实践
    traits.md                       # Trait 惯用法 + 零成本抽象边界
    errors.md                       # 错误组合子
    data-struct.md                  # 数据结构模式、#[repr] 注解、enum 布局、derive 规范
    borrowing.md                    # 借用与可变性、内部可变性、分割借用
    trade-offs.md                   # 权衡决策记录
    refactor.md                     # 重构路径
    review.md                       # 综合审查清单
    usage-examples.md               # 使用示例
```

---

## 📄 许可证

[MIT](LICENSE)

---

## 🙏 致谢

本 Skill 的规范体系综合了以下来源的工程实践：

- [Rust 官方文档](https://doc.rust-lang.org/) — 语言规范与标准库 API
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — 公共 API 设计检查清单
- [Too Many Lists](https://rust-unofficial.github.io/too-many-lists/) — unsafe 与指针安全教程
- [Tokio 教程](https://tokio.rs/tokio/tutorial) — 异步运行时最佳实践
- 社区大量生产级 Rust 项目的架构经验总结
