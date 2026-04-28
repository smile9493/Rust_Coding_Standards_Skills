# Rust 编码规范 Skills

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Architecture Guide](https://img.shields.io/badge/Architecture%20Guide-v9.0.0-brightgreen.svg)]()
[![Cloud Infra Guide](https://img.shields.io/badge/Cloud%20Infra%20Guide-v6.0.0-orange.svg)]()
[![Wasm Infra Guide](https://img.shields.io/badge/Wasm%20Infra%20Guide-v4.0.0-purple.svg)]()
[![Reference Docs](https://img.shields.io/badge/Reference-53%20Docs-orange.svg)]()

**Rust 工程决策 Wiki — AI 编码助手的宪法性指南，覆盖通用工程决策、云基础设施专用规范与 WebAssembly 前端基建专用规范。**

[English](README.md) | 简体中文

***

## 项目简介

Rust Coding Standards Skills 是一个面向 AI 编码助手的 **Rust 工程决策指南 Skill 集合**，包含通用工程宪法、云基础设施专用规范与 WebAssembly 前端基建专用规范，覆盖从架构决策、编码风格到生产最佳实践的完整链路。

本 Skill 集合作为 Agent 的**宪法性基础**，确保每次代码生成、审查、重构都遵循统一的优先级判定和冲突解决框架，避免"看心情写代码"。

### 解决的问题

| 没有 Skill                     | 有本 Skill                               |
| ---------------------------- | -------------------------------------- |
| AI 随意选择 `unwrap` 还是 `expect` | 按 P0/P1 优先级自动判定                        |
| 泛型单态化爆炸无人把关                  | 三级退火策略自动降级                             |
| 错误处理库级/应用级混用                 | `thiserror` / `anyhow` 分层隔离            |
| 并发模型凭直觉选                     | MPSC bounded + 最小锁区 + 禁止 Mutex 跨 await |
| API 演进破坏兼容性                  | `#[non_exhaustive]` + Sealed Trait 防御  |
| I/O 模型选择无依据                  | Tokio epoll vs io\_uring 选型决策树         |
| 背压机制缺失                       | 有界通道、Semaphore、503 传播                  |
| 共识算法实现不规范                    | 确定性状态机（Raft/Paxos Apply）、禁止时间/随机依赖     |
| WASM 二进制体积膨胀                  | `opt-level="z"` + `wasm-opt -Oz` + 分配器替换 |
| JS-WASM FFI 逐元素开销               | `WasmSlice` 零拷贝批量模式 |
| Wasm 线性内存泄漏                    | Arena 帧级生命周期 + 显式 `.free()` |

***

## 核心特性

### rust-architecture-guide — 通用工程宪法

适用于**所有 Rust 项目**，提供优先级决策框架、架构模式与编码风格。

- 🏛️ **四级优先级金字塔** — P0 安全 → P1 可维护 → P2 编译时 → P3 运行时性能
- 🔧 **三级执行模式** — `rapid`（原型）/ `standard`（默认）/ `strict`（发布）
- 🧬 **类型驱动架构** — 状态机、Newtype、零成本抽象边界
- 📦 **所有权分层策略** — 业务层 Owned + `.clone()` 解耦，热点层 `Cow`/`Bytes` 零拷贝
- ⚠️ **错误处理分层** — 库级 `thiserror` 结构化，应用级 `anyhow` + 惰性上下文
- 🔄 **并发与异步规范** — 有界通道背压、RwLock/parking\_lot 策略、`Pin`/`Unpin` 语义
- 🛡️ **API 演进防御** — `#[non_exhaustive]`、Sealed Trait、`#[deprecated]` 迁移
- 🧪 **高阶质量保证** — proptest + cargo fuzz + Loom + Miri
- ⚡ **性能深度调优** — jemalloc/mimalloc、SoA 布局、SmallVec、PGO、SIMD、LTO
- 🔗 **FFI 安全边界** — `-sys` crate 分离、三层隔离架构、`catch_unwind` 防波堤
- 🧙 **元编程与宏魔法** — 截击样板代码、过程宏 Span 级错误、`const fn` 零开销
- 🧊 **内存布局透明化** — 结构体空隙审计、`#[repr(C)]` 强制、缓存行友好设计（≤64 字节）、伪共享防御
- 🌊 **防波堤架构** — Facade/Core 分层设计、边界截击协议、类型收缩（去氧）、O(1) 转换强制
- 📏 **物理可行性审计** — I/O 预算（>30% → 批量处理）、内存天花板（<20% 余量 → 背压）、并发真实代价（>20% 争用率 → 无锁重构）
- 🥋 **截拳道编码风格** — 截击样板、经济法则、硬件同理心
- 🤖 **Agent 自检指令** — Decision Summary 输出契约

📖 **完整文档索引**：[rust-architecture-guide/README.zh-CN.md](rust-architecture-guide/README.zh-CN.md)

***

### rust-systems-cloud-infra-guide — 云基础设施专用

适用于**数据库内核、分布式存储、高性能网关、容器运行时、eBPF 控制平面、OS 组件**等长时间运行系统，是通用宪法的垂直深化。

**环境假设**：长时间运行节点（uptime > 1 年）、10GbE+ 网络、多 NUMA 架构。

- 🌐 **I/O 模型决策** — Tokio epoll vs io\_uring vs monoio 选型决策树
- 📡 **零拷贝管道** — `splice`/`sendfile`/`copy_file_range`、`bytes::Bytes` O(1) clone
- 📊 **背压机制** — 有界通道、Semaphore、503 传播，绝对禁止无界通道
- 🔄 **取消安全** — 非幂等写必须 `spawn` + `oneshot`
- 🛡️ **优雅关闭** — `SIGTERM`/`SIGINT` 捕获、CancellationToken、fsync WAL
- 🗳️ **共识算法规范** — 确定性状态机，禁止 `Instant::now()`、`rand`、`HashMap` 顺序
- 🔧 **系统调用封装** — 使用 `rustix` 封装 + eBPF 集成
- 🧠 **高级内存架构** — Arena（`bumpalo`）、Slab 预分配（`mmap` + `mlock`）、NUMA/PMEM
- 🔓 **无锁并发** — RCU（`arc-swap`）、Epoch 回收（`crossbeam-epoch`）、内存序精确控制
- 🚀 **向量化执行** — SIMD 指令（`std::simd`/AVX-512）、Bitmask 消除分支、SoA 列式布局
- 🌊 **防波堤架构** — Facade/Core 分层、边界截击协议、去氧转换
- 📏 **物理可行性审计** — 容器内存限制、网络延迟预算、NUMA 拓扑审计
- 🚫 **内存耗尽背压** — `Result<T, AllocError>` 返回、503 拒绝、禁止 `panic!`
- ✅ **强制 CI Lints** — 11 项严格检查（`await_holding_lock`、`unwrap_used` 等）

📖 **完整文档索引**：[rust-systems-cloud-infra-guide/README.zh-CN.md](rust-systems-cloud-infra-guide/README.zh-CN.md)

***

### rust-wasm-frontend-infra-guide — WebAssembly 前端基建专用

适用于**所有编译为 `wasm32-unknown-unknown` 目标的 Rust 项目**，提供编译与边界层硬约束规范。通用宪法的垂直深化。

**环境假设**：Wasm 线性内存只增不减、JS ↔ Wasm 边界是昂贵的 RPC、浏览器主线程不容阻塞、零拷贝视图是 unsafe 操作。

- 🛡️ **[IRON-01] 体积即王道** — `opt-level="z"`, `lto=true`, `codegen-units=1`, `panic="abort"`, `strip=true`
- 🛡️ **[IRON-02] 边界零拷贝** — `WasmSlice` 安全封装，高频路径禁止序列化
- 🛡️ **[IRON-03] 内存分治** — 全局状态静态驻留，帧级 Arena 生命周期（`bumpalo` + `reset()`）
- 🛡️ **[IRON-04] 跨源隔离可见即所得** — `SharedArrayBuffer` 必须文档化 COOP/COEP 配置
- 🔧 **编译控制** — `wasm-opt -Oz` 强制，分配器替换（`talc`/`MiniAlloc`，禁用 `wee_alloc`）
- 🔗 **FFI 边界** — 显式标量或 `WasmSlice` 参数，禁止 `JsValue` 透传
- 🔄 **并发** — 仅 `wasm-bindgen-futures`，禁止阻塞 API，Worker 隔离 + COOP/COEP
- ⚠️ **错误处理** — `Result<T, JsValue>` + `thiserror`，`panic="abort"` 下必须用 `Result` 不用 `panic!`
- 📝 **日志** — `console_error_panic_hook` + `tracing_wasm` 强制初始化
- ✅ **7 条硬禁令** + **10 项合规自检清单**

📖 **完整文档索引**：[rust-wasm-frontend-infra-guide/README.zh-CN.md](rust-wasm-frontend-infra-guide/README.zh-CN.md)

***

## 快速开始

### 安装

本项目遵循 [Agent Skills Spec v1.0](https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx) 格式，兼容任何支持 Skills 规范的 AI Agent 平台。

```bash
# 克隆仓库
git clone https://github.com/smile9493/Rust_Coding_Standards_Skills.git

# 安装到你的 Agent skills 目录
# Trae IDE
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.trae/skills/
cp -r Rust_Coding_Standards_Skills/rust-systems-cloud-infra-guide ~/.trae/skills/
cp -r Rust_Coding_Standards_Skills/rust-wasm-frontend-infra-guide ~/.trae/skills/

# Claude Code / 其他 Agent Skills 兼容平台
# 将两个 skill 目录复制到你 Agent 的 skills 配置路径下
```

### 使用

**Skill 调用**（语法因平台而异）：

```
# Trae IDE
/rust-architecture-guide priority my_conflict
/rust-architecture-guide state-machine Order
/rust-systems-cloud-infra-guide io-model
/rust-systems-cloud-infra-guide backpressure
/rust-wasm-frontend-infra-guide build-control
/rust-wasm-frontend-infra-guide ffi-boundary

# Claude Code / 其他平台 — 在提示中引用 skill 名称
# "According to rust-architecture-guide, what priority should I assign?"
# "Follow rust-systems-cloud-infra-guide to select the I/O model for this gateway"
```

**自然语言触发** — Agent 根据上下文自动匹配 Skill：

```
帮我用类型驱动状态机重构 Order 实体
这个模块的错误处理应该用 thiserror 还是 anyhow？
审查这段并发代码有没有 Mutex 跨 await 的问题
用 io_uring 优化这个存储引擎的 I/O 路径
这个原子计数器应该用什么内存序？
写一个过程宏，要求 Span 级别的错误报告
配置我的 WASM 项目 Cargo.toml 以获得最小二进制体积
设计一个零拷贝的 JS-WASM 边界用于图像处理
```

### Skill 格式

每个 Skill 遵循 Agent Skills Spec v1.0 结构：

```
skill-name/
├── SKILL.md              # 入口文件（YAML 前置信息 + Agent 指令）
└── references/            # 深度参考文档
    ├── 01-topic.md
    ├── 02-topic.md
    └── ...
```

**SKILL.md 前置信息字段**：

| 字段 | 必需 | 描述 |
|------|------|------|
| `name` | ✅ | Skill 标识符（hyphen-case，与目录名匹配） |
| `description` | ✅ | 何时调用此 Skill（最大 1024 字符） |
| `license` | ❌ | 许可证标识 |
| `metadata` | ❌ | 扩展元数据（版本、哲学、领域等） |
| `allowed-tools` | ❌ | Skill 允许使用的工具 |

***

## 两者关系

```
rust-architecture-guide (通用宪法)
          │
          ├── 四级优先级框架 (P0 → P3)
          ├── 三级执行模式 (rapid / standard / strict)
          ├── 截拳道编码风格
          ├── Agent 自检清单 + Decision Summary
          │
          ├──► rust-systems-cloud-infra-guide (垂直深化)
          │           │
          │           ├── 核心哲学
          │           │   ├── Mechanical Sympathy — 软件对齐硬件物理特性
          │           │   ├── Determinism — 消除非确定性
          │           │   ├── Resilience — 优雅降级 > 崩溃
          │           │   └── Jeet Kune Do — 一击内存生命周期
          │           │
          │           ├── I/O 模型（epoll vs io_uring vs monoio）
          │           ├── 零拷贝管道（splice, sendfile, bytes::Bytes）
          │           ├── 有界资源背压 + 取消安全
          │           ├── 确定性状态机（共识算法）
          │           ├── 优雅关闭（CancellationToken 流程）
          │           ├── 高级内存架构（Arena / Slab / NUMA / PMEM / Allocator API）
          │           ├── 无锁并发（RCU + Epoch + 内存序）
          │           ├── 向量化执行（SIMD + SoA）
          │           ├── 防波堤架构（Facade/Core 分层）
          │           ├── 物理可行性审计（设计前强制）
          │           └── 强制 CI Lints（11 项严格检查）
          │
          └──► rust-wasm-frontend-infra-guide (垂直深化)
                      │
                      ├── 铁律
                      │   ├── IRON-01 — 体积即王道
                      │   ├── IRON-02 — 边界零拷贝
                      │   ├── IRON-03 — 内存分治
                      │   └── IRON-04 — 跨源隔离可见即所得
                      │
                      ├── 编译控制（Cargo.toml + wasm-opt + 分配器）
                      ├── FFI 边界（WasmSlice + 显式契约）
                      ├── 内存生命周期（Arena 帧级 + 泄漏防御）
                      ├── 并发（wasm-bindgen-futures + Worker 隔离）
                      ├── Wasm 适配（Result<T, JsValue> + 日志）
                      └── 合规（7 条禁令 + 10 项自检清单）
```

- **`rust-architecture-guide`**（v9.0.0）：所有 Rust 工程的宪法基础 — 新增内存布局透明化、防波堤架构、物理可行性审计三大范式
- **`rust-systems-cloud-infra-guide`**（v6.0.0）：云原生场景的**附加条款**，在 P0 安全之上增加系统级红线、Facade/Core 架构与部署物理审计
- **`rust-wasm-frontend-infra-guide`**（v4.0.0）：`wasm32-unknown-unknown` 场景的**附加条款**，在 P0 安全之上增加编译与边界层硬约束
- 两者互补使用：通用宪法提供优先级框架，垂直深化指南增加领域专用红线

***

## 设计哲学

### 1. 唯物辩证法 — 矛盾驱动进化

工程决策不是静态真理，而是通过矛盾解决的动态过程。每个 `unsafe` 块都是安全与性能的矛盾；每个 `.clone()` 都是简洁与效率的矛盾。优先级金字塔是解决这些矛盾的工具——高层否决低层，解决本身成为架构。

**核心辩证法**：

- **对立统一**：`unsafe` 与 safe 不是敌人——`unsafe` 是安全抽象的物质基础。`-sys` crate 是 unsafe 的，所以 wrapper 才能是 safe 的。
- **量变到质变**：MVP 的 `Option<bool>` 标志积累到业务模型稳定后，必须质变为 Enum 状态机。编译器通过识别所有调用点来协助这一过程。
- 否定之否定：错误不是终点而是起点——panic → catch → 优雅降级。每一次否定都走向更高层的韧性。

### 2. 截拳道 — 以无法为有法

> "以无法为有法，以无限为有限。" — 李小龙

优先级金字塔提供了"法"，但 DEVIATION 协议提供了"无法"——当打破规则是正确决策时的正式通道。这形成了一个**自指的哲学闭环**：规则本身包含了对规则的超越。

**应用原则**：

- **截击样板**：如果一个逻辑可以用 1 行模式匹配表达，绝不使用 5 行嵌套。`let else` 优于嵌套 `if let`；`filter_map` 优于 `filter` + `map`。
- **经济法则**：每一行代码都应直接指向意图。消除多余的中间变量与隐式拷贝。Arena 一次分配、批量回收——一击必杀，不拖泥带水。
- **硬件同理心**：利用迭代器和零拷贝类型，顺应编译器的内联优化。SoA 布局与 CPU 缓存行共振；`const fn` 将计算从运行时前置到编译期。

### 3. 务实主义优于教条主义

> **系统边界与热点路径追求极致，内部流程与冷路径释放心智负担。**

并非每一行代码都值得同等的审视。执行模式系统（`rapid` / `standard` / `strict`）确保严谨的成本与失败的成本成正比：

| 模式         | 强制执行    | 权衡                               |
| ---------- | ------- | -------------------------------- |
| `rapid`    | 仅 P0    | 无限 `.clone()`、库中用 `anyhow`、无文档测试 |
| `standard` | P0 + P1 | 大多数项目的默认选择                       |
| `strict`   | P0–P3   | 所有偏差需要正式的 `// DEVIATION:` 注解     |

### 4. 优先级金字塔 — 宪法性框架

```
P0: 安全与正确性（内存安全、数据一致性）— 不可妥协
                  ↓
P1: 可维护性（可读性、局部复杂度控制）— 默认追求
                  ↓
P2: 编译时（构建速度、CI/CD 效率）— 测量后决策
                  ↓
P3: 运行时性能（仅限已证明的瓶颈）— 需 Profiler 数据
```

**冲突解决规则**：

- **高优先级否决低优先级**：P0 安全要求否决 P3 性能优化
- **同级取更简方案**：两个 P1 方案冲突，选更简单的
- **P3 需要证据**：任何性能优化必须附带 Profiler 数据

### 5. 硬件同理心 — 顺应物理

极致的性能不是来自"奇技淫巧"，而是来自代码逻辑与底层物理硬件的深度共鸣。软件不是运行在抽象机上，而是运行在：

```
L1 Cache (32KB, 4 cycles) → L2 (256KB, 12 cycles) → L3 (共享, 40 cycles) → DRAM (200+ cycles)
NUMA Node 0 ← QPI/UPI → NUMA Node 1
NIC Ring Buffer → Kernel TCP Stack → User Space
```

性能不是优化出来的，而是**对齐**出来的。当你理解了 CPU cache line 是 64 字节、false sharing 会摧毁并发、mmap 的 page fault 代价是微秒级，你就不会再"优化"——你会**设计**出与硬件物理共振的结构。

### 6. 机器对抗机器 — 确定性质量

人类编写的单元测试只能覆盖已知范围。面对极其复杂的并发与边界，必须释放机器的算力：

- **属性测试**（`proptest`）：从"具体的例证"上升为"普遍的物理定律"——验证对称性、幂等性、单调性
- **模糊测试**（`cargo-fuzz`）：引入基因变异与进化论，在混沌中寻找崩溃点
- **并发模型检验**（`loom`）：消除线程调度的随机性，实现并发状态的穷举确定
- **UB 检测**（`Miri`）：在 LLVM 优化前，用解释器审视每一行内存契约

***

## 项目结构

```
├── rust-architecture-guide/
│   ├── SKILL.md                          # Skill 入口
│   ├── README.md                         # 文档索引（英文）
│   ├── README.zh-CN.md                   # 文档索引（中文）
│   └── references/                        # 29 份参考文档
│
├── rust-systems-cloud-infra-guide/
│   ├── SKILL.md                          # Skill 入口
│   ├── README.md                         # 文档索引（英文）
│   ├── README.zh-CN.md                   # 文档索引（中文）
│   └── references/                        # 13 份参考文档
│
├── rust-wasm-frontend-infra-guide/
│   ├── SKILL.md                          # Skill 入口
│   ├── README.md                         # 文档索引（英文）
│   ├── README.zh-CN.md                   # 文档索引（中文）
│   └── references/                        # 7 份参考文档
│
├── README.md                              # Wiki 索引（英文）
└── README.zh-CN.md                        # Wiki 索引（中文）
```

***

## 许可证

[MIT](LICENSE)

***

## 致谢

本 Skill 集合的规范体系综合了以下来源的工程实践：

### Rust 生态

- [Rust 官方文档](https://doc.rust-lang.org/) — 语言规范与标准库 API
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — 公共 API 设计检查清单
- [The Rust Programming Language](https://doc.rust-lang.org/book/) — 官方书籍
- [Rust Reference](https://doc.rust-lang.org/references/) — 语言参考
- [Rustonomicon](https://doc.rust-lang.org/nomicon/) — Unsafe Rust 黑魔法
- [Too Many Lists](https://rust-unofficial.github.io/too-many-lists/) — Unsafe 与指针安全教程
- [Tokio 教程](https://tokio.rs/tokio/tutorial) — 异步运行时最佳实践
- [Cargo Book](https://doc.rust-lang.org/cargo/) — 构建系统与包管理器

### 异步与并发

- [Tokio](https://tokio.rs/) — 异步运行时生态
- [async-book](https://rust-lang.github.io/async-book/) — Rust 异步编程
- [crossbeam](https://github.com/crossbeam-rs/crossbeam) — 并发编程工具
- [loom](https://github.com/tokio-rs/loom) — 并发模型检验

### 性能与系统

- [Martin Thompson — Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/) — 硬件对齐的软件设计哲学
- [rustix](https://github.com/bytecodealliance/rustix) — 安全的 POSIX/Win32 系统调用封装
- [bumpalo](https://github.com/fitzgen/bumpalo) — 快速 Arena 分配器
- [arc-swap](https://github.com/vorner/arc-swap) — RCU 风格原子引用交换
- [DashMap](https://github.com/xacrimon/dashmap) — 分片并发 HashMap

### 质量保证

- [proptest](https://github.com/proptest-rs/proptest) — 属性测试框架
- [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz) — 模糊测试基础设施
- [Miri](https://github.com/rust-lang/miri) — 未定义行为检测解释器
- [trybuild](https://github.com/dtolnay/trybuild) — 编译时宏测试
- [Kani](https://github.com/model-checking/kani) — 形式化验证工具

### FFI 与互操作

- [cxx](https://github.com/dtolnay/cxx) — 安全的 C++ 互操作
- [bindgen](https://github.com/rust-lang/rust-bindgen) — 自动 FFI 绑定生成
- [wasmtime](https://github.com/bytecodealliance/wasmtime) — WebAssembly 运行时

### Agent Skills 规范

- [Agent Skills Spec v1.0](https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx) — 通用 AI Agent Skills 格式规范

### 社区

- 社区大量生产级 Rust 项目的架构经验总结
- Rust 社区论坛、Reddit r/rust、Zulip 讨论中塑造的这些最佳实践

