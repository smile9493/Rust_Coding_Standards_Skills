---
name: rust-blockchain-guide
description: 在构建 Rust 区块链程序（Solana、Substrate/Polkadot、NEAR、Cosmos SDK、Stacks）时使用。涵盖账户模型架构、程序派生地址（PDA）、BPF 字节码约束、no_std + Solana 混合开发、Substrate FRAME pallet、跨程序调用（CPI）和确定性交易执行。作为 rust-architecture-guide 对区块链和 Web3 的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Determinism, Account Model, Verifiable Compute, Jeet Kune Do"
  domain: "blockchain & Web3"
  relationship: "vertical-deepening-of:[rust-architecture-guide, rust-embedded-iot-guide]"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Solana BPF constraints", "Substrate FRAME", "Anchor framework", "SPL Token standard"]
  note: "Scope covers Solana programs and Substrate pallets. Cosmos SDK and Stacks are mentioned in description but not covered in depth."
---

# Rust 区块链与 Web3 指南 V1.0.0

作为 `rust-architecture-guide` 和 `rust-embedded-iot-guide` 对区块链程序（智能合约）的垂直深化。假设确定性执行、可验证计算和严格的资源约束（BPF 字节码、Gas 计量）。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **确定性** | 每笔交易必须在每个验证器上产生相同的状态。相同输入 → 相同输出。 |
| **账户模型** | 状态被分区到账户中。程序是无状态的。数据存在于账户中。 |
| **可验证计算** | 每个程序执行必须是可重放的。Gas 计量限制计算量。 |
| **截拳道** | 一趟交易处理。Anchor 约束将验证 + 执行折叠到一个 derive 中。 |
