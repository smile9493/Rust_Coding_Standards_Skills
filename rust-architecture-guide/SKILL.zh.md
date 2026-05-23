---
name: rust-architecture-guide
description: 全面的 Rust 工程指南，涵盖优先级金字塔、架构决策、惯用编码风格、内存布局透明化、防波堤架构和物理可行性审计。在启动 Rust 项目、进行权衡决策、编写惯用代码或设计系统架构时调用。基于截拳道哲学 — 截击样板代码、顺应硬件、聚合逻辑至最大能量密度、虚实合一。
metadata:
  version: "9.1.0"
  philosophy: "Dialectical Materialism & Jeet Kune Do — Unity of False and Real"
  domain: "general Rust engineering"
  author: "rust-architect"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Rust API Guidelines", "Rust 2024 Edition", "Tokio Best Practices", "Clippy pedantic/nursery", "Safety-Critical Rust Coding Guidelines"]
---

# Rust 架构与工程决策指南 V9.1.0 — Rust 2024 Edition

本文档作为 Rust 工程决策的**宪法性基础**，专为适合 AI 辅助开发的确定性、可复现决策而设计。

V9.0.0 引入三个范式转变：
1. **内存布局透明化** (ref/30)：从"意识形态安全"到"唯物主义物理" — 结构体空隙审计、`#[repr(C)]` 强制、缓存行友好设计
2. **防波堤架构** (ref/31)：Facade/Core 分层模式 — 符合人体工学的门面吸收混乱，零开销核心保持确定性
3. **物理可行性审计** (ref/32)：启动与设计之间的强制审计 — I/O 预算、内存天花板、并发真实代价

V9.1.0 — **Rust 2024 Edition 对齐**：
- 采用 Rust 2024 Edition 惯用法：`unsafe extern` 块、RPIT 生命周期捕获、返回位置的 `impl Trait` 语法糖
- 对齐 [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) 检查清单（C-CASE、C-CONV、C-MACRO-VIS、C-EXAMPLE）
- 集成 [Safety-Critical Rust Coding Guidelines](https://github.com/rustfoundation/safety-critical-rust-coding-guidelines) 用于 unsafe 代码治理
- 将 `cargo-mutants` 变异测试、`kani` 形式化验证、`turmoil` 网络混沌测试加入高级测试武器库

### 执行模式

在应用任何规则之前，先确定执行模式：

- **`rapid`** (原型开发)：仅执行 P0；库中允许 `anyhow`、无限 `.clone()`、无需文档测试
- **`standard`**（默认）：执行 P0+P1；对 P2 违规发出警告
- **`strict`**（生产）：执行 P0-P3；所有偏离需正式注解

完整模式定义请参阅 [references/00-mode-guide.md](references/00-mode-guide.md)。

### 决策摘要契约

**架构决策、安全敏感变更和多规则冲突必须以防务摘要块结尾。对于琐碎编辑（注释修复、纯格式变更、单一规则的简单判例），摘要是可选的。**

```markdown
## Decision Summary
- **Mode**: [rapid|standard|strict]
- **Edition**: [2021|2024] — as configured in project Cargo.toml
- **Rules Applied**: [list specific rules from P0-P3]
- **Conflicts Resolved**: [PX > PY with justification, or "None"]
- **Deviations**: [list with `// DEVIATION: reason` references, or "None"]
- **Trade-offs**: [key trade-off decisions made]
```
