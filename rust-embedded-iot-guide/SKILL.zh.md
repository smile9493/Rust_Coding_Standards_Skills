---
name: rust-embedded-iot-guide
description: 在构建 Rust 嵌入式系统、物联网设备或裸机固件（ARM Cortex-M、RISC-V、ESP32）时使用。涵盖 no_std 架构、PAC/HAL 分层、RTIC/Embassy 异步、链接脚本控制、电源管理和硬件调试。作为 rust-architecture-guide 对微控制器目标的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Mechanical Sympathy, Determinism, Minimalism, Jeet Kune Do"
  domain: "embedded systems & IoT"
  relationship: "vertical-deepening-of:rust-architecture-guide"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["embedded-hal traits", "RTIC framework", "Embassy async", "Cortex-M RTFM"]
  note: "ESP-IDF Rust (std + FreeRTOS) 需要单独的垂直指南；本指南仅涵盖裸机 no_std 目标。"
---

# Rust 嵌入式与物联网指南 V1.0.0

作为 `rust-architecture-guide` 对裸机和基于 RTOS 的嵌入式系统的垂直深化。假设资源受限的硬件（KB 级 RAM、MHz 级时钟）、无操作系统、实时截止期限。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **机械同理心** | 代码直接映射到 MMIO 寄存器 — 无内核、无系统调用、无页表 |
| **确定性** | 每个指令周期都可被追踪；中断延迟有界且可测量 |
| **极简主义** | 默认 `no_std`；`alloc` 是奢侈决策；`.bss` 的每个字节都经过审计 |
| **截拳道** | 一击电源周期；闪存友好的数据结构；DMA 作为终极零拷贝 |
