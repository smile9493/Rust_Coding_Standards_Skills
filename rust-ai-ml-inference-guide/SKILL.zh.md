---
name: rust-ai-ml-inference-guide
description: 在构建 Rust AI/ML 推理服务、模型服务基础设施或嵌入式 ML 管道时使用。涵盖模型加载（GGUF/ONNX/SafeTensors）、量化感知推理、GPU 内存管理（CUDA/Metal）、分词器批处理和嵌入向量搜索。作为 rust-architecture-guide 对 AI 推理系统的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Hardware Synergy, Quantization Physics, Batch Efficiency, Jeet Kune Do"
  domain: "AI/ML inference & model serving"
  relationship: "vertical-deepening-of:[rust-architecture-guide, rust-systems-cloud-infra-guide]"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["candle ML framework", "burn deep learning", "ort ONNX runtime", "llama.cpp GGUF", "tokenizers library"]
---

# Rust AI/ML 推理指南 V1.0.0

作为 `rust-architecture-guide` 和 `rust-systems-cloud-infra-guide` 对模型推理、服务端推理和嵌入式 ML 的垂直深化。假设 GPU 加速、量化模型和高吞吐量批处理。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **硬件协同** | 矩阵乘法映射到张量核心。KV-cache 适配 HBM。注意力模式利用 L2。 |
| **量化物理** | Q4_0/Q8_0/K-quant 将模型权重映射到硬件友好的整数类型。4 位 = 4 倍更少的内存带宽。 |
| **批处理效率** | 连续批处理最大化 GPU 利用率。通过动态批处理最小化填充浪费。 |
| **截拳道** | 一趟 token 生成。KV-cache 复用。投机解码用于延迟隐藏。 |
