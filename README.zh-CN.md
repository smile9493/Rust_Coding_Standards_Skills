# Rust 编码规范 Skills

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Architecture Guide](https://img.shields.io/badge/Architecture%20Guide-v8.0.0-brightgreen.svg)]()
[![Cloud Infra Guide](https://img.shields.io/badge/Cloud%20Infra%20Guide-v5.0.0-orange.svg)]()
[![Reference Docs](https://img.shields.io/badge/Reference-40%20Docs-orange.svg)]()

**Rust 工程决策 Wiki — AI 编码助手的宪法性指南，覆盖通用工程决策与云基础设施专用规范。**

[English](README.md) | 简体中文

---

## 项目简介

Rust Coding Standards Skills 是一个面向 AI 编码助手的 **Rust 工程决策指南 Skill 集合**，包含通用工程宪法与云基础设施专用规范，覆盖从架构决策、编码风格到生产最佳实践的完整链路。

本 Skill 集合作为 Agent 的**宪法性基础**，确保每次代码生成、审查、重构都遵循统一的优先级判定和冲突解决框架，避免"看心情写代码"。

### 解决的问题

| 没有 Skill | 有本 Skill |
|-----------|-----------|
| AI 随意选择 `unwrap` 还是 `expect` | 按 P0/P1 优先级自动判定 |
| 泛型单态化爆炸无人把关 | 三级退火策略自动降级 |
| 错误处理库级/应用级混用 | `thiserror` / `anyhow` 分层隔离 |
| 并发模型凭直觉选 | MPSC bounded + 最小锁区 + 禁止 Mutex 跨 await |
| API 演进破坏兼容性 | `#[non_exhaustive]` + Sealed Trait 防御 |
| I/O 模型选择无依据 | Tokio epoll vs io_uring 选型决策树 |
| 背压机制缺失 | 有界通道、Semaphore、503 传播 |
| 共识算法实现不规范 | 确定性状态机（Raft/Paxos Apply）、禁止时间/随机依赖 |

---

## 核心特性

### rust-architecture-guide — 通用工程宪法

适用于**所有 Rust 项目**，提供优先级决策框架、架构模式与编码风格。

- 🏛️ **四级优先级金字塔** — P0 安全 → P1 可维护 → P2 编译时 → P3 运行时性能
- 🔧 **三级执行模式** — `rapid`（原型）/ `standard`（默认）/ `strict`（发布）
- 🧬 **类型驱动架构** — 状态机、Newtype、零成本抽象边界
- 📦 **所有权分层策略** — 业务层 Owned + `.clone()` 解耦，热点层 `Cow`/`Bytes` 零拷贝
- ⚠️ **错误处理分层** — 库级 `thiserror` 结构化，应用级 `anyhow` + 惰性上下文
- 🔄 **并发与异步规范** — 有界通道背压、RwLock/parking_lot 策略、`Pin`/`Unpin` 语义
- 🛡️ **API 演进防御** — `#[non_exhaustive]`、Sealed Trait、`#[deprecated]` 迁移
- 🧪 **高阶质量保证** — proptest + cargo fuzz + Loom + Miri
- ⚡ **性能深度调优** — jemalloc/mimalloc、SoA 布局、SmallVec、PGO、SIMD、LTO
- 🔗 **FFI 安全边界** — `-sys` crate 分离、三层隔离架构、`catch_unwind` 防波堤
- 🧙 **元编程与宏魔法** — 截击样板代码、过程宏 Span 级错误、`const fn` 零开销
- 🥋 **截拳道编码风格** — 截击样板、经济法则、硬件同理心
- 🤖 **Agent 自检指令** — Decision Summary 输出契约

📖 **完整文档索引**：[rust-architecture-guide/README.zh-CN.md](rust-architecture-guide/README.zh-CN.md)

---

### rust-systems-cloud-infra-guide — 云基础设施专用

适用于**数据库内核、分布式存储、高性能网关、容器运行时、eBPF 控制平面、OS 组件**等长时间运行系统，是通用宪法的垂直深化。

**环境假设**：长时间运行节点（uptime > 1 年）、10GbE+ 网络、多 NUMA 架构。

- 🌐 **I/O 模型决策** — Tokio epoll vs io_uring vs monoio 选型决策树
- 📡 **零拷贝管道** — `splice`/`sendfile`/`copy_file_range`、`bytes::Bytes` O(1) clone
- 📊 **背压机制** — 有界通道、Semaphore、503 传播，绝对禁止无界通道
- 🔄 **取消安全** — 非幂等写必须 `spawn` + `oneshot`
- 🛡️ **优雅关闭** — `SIGTERM`/`SIGINT` 捕获、CancellationToken、fsync WAL
- 🗳️ **共识算法规范** — 确定性状态机，禁止 `Instant::now()`、`rand`、`HashMap` 顺序
- 🔧 **系统调用封装** — 使用 `rustix` 封装 + eBPF 集成
- 🧠 **高级内存架构** — Arena（`bumpalo`）、Slab 预分配（`mmap` + `mlock`）、NUMA/PMEM
- 🔓 **无锁并发** — RCU（`arc-swap`）、Epoch 回收（`crossbeam-epoch`）、内存序精确控制
- 🚀 **向量化执行** — SIMD 指令（`std::simd`/AVX-512）、Bitmask 消除分支、SoA 列式布局
- 🚫 **内存耗尽背压** — `Result<T, AllocError>` 返回、503 拒绝、禁止 `panic!`
- ✅ **强制 CI Lints** — 11 项严格检查（`await_holding_lock`、`unwrap_used` 等）

📖 **完整文档索引**：[rust-systems-cloud-infra-guide/README.zh-CN.md](rust-systems-cloud-infra-guide/README.zh-CN.md)

---

## 快速开始

### 安装

```bash
# 方式一：克隆到工作空间 skills 目录（推荐）
git clone https://github.com/smile9493/Rust_Coding_Standards_Skills.git
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.trae/skills/
cp -r Rust_Coding_Standards_Skills/rust-systems-cloud-infra-guide ~/.trae/skills/

# 方式二：直接在项目中使用
# 将两个目录复制到项目的 .trae/skills/ 下
```

### 使用

在 Trae IDE 中直接调用：

```
/rust-architecture-guide priority my_conflict
/rust-architecture-guide state-machine Order
/rust-systems-cloud-infra-guide io-model
/rust-systems-cloud-infra-guide backpressure
```

或自然语言触发：

```
帮我用类型驱动状态机重构 Order 实体
这个模块的错误处理应该用 thiserror 还是 anyhow？
审查这段并发代码有没有 Mutex 跨 await 的问题
用 io_uring 优化这个存储引擎的 I/O 路径
```

---

## 两者关系

```
rust-architecture-guide (通用宪法)
          │
          ├── 四级优先级框架 (P0 → P3)
          ├── 三级执行模式 (rapid / standard / strict)
          ├── 截拳道编码风格
          ├── Agent 自检清单 + Decision Summary
          │
          └──► rust-systems-cloud-infra-guide (垂直深化)
                      │
                      ├── 核心哲学
                      │   ├── Mechanical Sympathy — 软件对齐硬件物理特性
                      │   ├── Determinism — 消除非确定性
                      │   ├── Resilience — 优雅降级 > 崩溃
                      │   └── Jeet Kune Do — 一击内存生命周期
                      │
                      ├── I/O 模型（epoll vs io_uring vs monoio）
                      ├── 零拷贝管道（splice, sendfile, bytes::Bytes）
                      ├── 有界资源背压 + 取消安全
                      ├── 确定性状态机（共识算法）
                      ├── 优雅关闭（CancellationToken 流程）
                      ├── 高级内存架构（Arena / Slab / NUMA / PMEM / Allocator API）
                      ├── 无锁并发（RCU + Epoch + 内存序）
                      ├── 向量化执行（SIMD + SoA）
                      └── 强制 CI Lints（11 项严格检查）
```

- **`rust-architecture-guide`**（v8.0.0）：所有 Rust 工程的宪法基础
- **`rust-systems-cloud-infra-guide`**（v5.0.0）：云原生场景的**附加条款**，在 P0 安全之上增加系统级红线和硬件对齐约束
- 两者互补使用：通用宪法提供优先级框架，云基础设施指南在其基础上进行垂直深化

---

## 设计哲学

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

---

## 项目结构

```
├── rust-architecture-guide/
│   ├── SKILL.md                          # Skill 入口
│   ├── README.md                         # 文档索引（英文）
│   ├── README.zh-CN.md                   # 文档索引（中文）
│   └── reference/                        # 29 份参考文档
│
├── rust-systems-cloud-infra-guide/
│   ├── SKILL.md                          # Skill 入口
│   ├── README.md                         # 文档索引（英文）
│   ├── README.zh-CN.md                   # 文档索引（中文）
│   └── reference/                        # 11 份参考文档
│
├── README.md                              # Wiki 索引（英文）
└── README.zh-CN.md                        # Wiki 索引（中文）
```

---

## 许可证

[MIT](LICENSE)

---

## 致谢

本 Skill 集合的规范体系综合了以下来源的工程实践：

- [Rust 官方文档](https://doc.rust-lang.org/) — 语言规范与标准库 API
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — 公共 API 设计检查清单
- [Too Many Lists](https://rust-unofficial.github.io/too-many-lists/) — unsafe 与指针安全教程
- [Tokio 教程](https://tokio.rs/tokio/tutorial) — 异步运行时最佳实践
- 社区大量生产级 Rust 项目的架构经验总结
