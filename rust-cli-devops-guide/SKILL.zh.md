---
name: rust-cli-devops-guide
description: 在构建 Rust CLI 工具、开发者工具链或 Kubernetes Operator/控制器时使用。涵盖 clap 参数解析、跨平台分发、信号处理、进度 UX 和 kube-rs operator 模式。作为 rust-architecture-guide 对 CLI 和 DevOps 工具的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Simplicity, Composability, Cross-Platform, Jeet Kune Do"
  domain: "CLI tools & DevOps"
  relationship: "vertical-deepening-of:rust-architecture-guide"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["clap derive API", "cargo-dist", "kube-rs operator pattern", "indicatif UX", "ratatui TUI"]
---

# Rust CLI 与 DevOps 指南 V1.0.0

作为 `rust-architecture-guide` 对命令行工具、开发者工具链和 Kubernetes 原生 operator 的垂直深化。CLI 是通用接口——每个后端工程师最终都会构建一个。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **简洁** | CLI 应做好一件事。通过管道而非单体进行组合。 |
| **可组合性** | 输出结构化数据（JSON/CSV）。接受管道输入。做一个好的 Unix 公民。 |
| **跨平台** | Windows、macOS、Linux。路径分隔符、换行符、信号——全部透明处理。 |
| **截拳道** | 一击安装（`curl \| sh`）。亚毫秒级启动。无需预热。 |
