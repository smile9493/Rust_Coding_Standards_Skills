---
name: rust-gamedev-guide
description: 在构建 Rust 游戏引擎、渲染管线、游戏逻辑系统或实时交互式应用时使用。涵盖 ECS 架构（Bevy）、帧预算管理、GPU 资源生命周期（wgpu）、资产管线、物理及输入处理。作为 rust-architecture-guide 对实时游戏开发的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Frame Budget, Data-Oriented, GPU-Friendly, Jeet Kune Do"
  domain: "game development & real-time rendering"
  relationship: "vertical-deepening-of:rust-architecture-guide"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Bevy ECS", "wgpu API", "naga shader compiler", "kira audio", "rapier physics"]
---

# Rust 游戏开发指南 V1.0.0

作为 `rust-architecture-guide` 对实时游戏引擎、渲染和交互式应用的垂直深化。假设 16ms 帧预算、GPU 驱动渲染和数据导向设计。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **帧预算** | 每个系统在 < 16ms（60 FPS）或 < 8ms（120 FPS）内运行。预算是法律。 |
| **数据导向** | 实体是数据，系统是函数。ECS 将数据与行为分离。SoA 布局有利于缓存。 |
| **GPU 友好** | 最小化 CPU-GPU 同步。批量绘制调用。使用 Staging Belt 上传。 |
| **截拳道** | 一趟渲染。在调度前进行视锥体裁剪。在片段着色器前进行 Early-Z。 |
