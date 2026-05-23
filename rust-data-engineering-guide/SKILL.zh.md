---
name: rust-data-engineering-guide
description: 在构建 Rust 数据基础设施（查询引擎、ETL 管道、OLAP 数据库、流处理器）时使用。涵盖 Arrow 列式内存模型、向量化表达式求值、查询优化器设计、分区策略、流式管道和跨语言数据交换。作为 rust-architecture-guide 和 rust-systems-cloud-infra-guide 对数据密集型系统的垂直深化。
metadata:
  version: "1.0.0"
  philosophy: "Mechanical Sympathy, Columnar Physics, Parquet-Native, Jeet Kune Do"
  domain: "data engineering & analytics"
  relationship: "vertical-deepening-of:[rust-architecture-guide, rust-systems-cloud-infra-guide]"
  default_edition: "2024"
  supported_editions: ["2021", "2024"]
  aligned_with: ["Apache Arrow Specification", "Parquet Format", "DataFusion Architecture", "Polars Internals", "Apache Iceberg Rust"]
---

# Rust 数据工程指南 V1.0.0

作为 `rust-architecture-guide` 和 `rust-systems-cloud-infra-guide` 对数据密集型系统的垂直深化。假设 TB-PB 级数据、列式执行和 SIMD 加速计算。

## 核心哲学

| 原则 | 描述 |
|-----------|-------------|
| **列式物理** | 数据存在于列而非行中。缓存行填充同构类型，向量化是自然而然的结果。 |
| **Parquet 原生** | 存储格式不是事后考虑的——谓词下推、行组修剪、统计信息是一等公民 |
| **机械同理心** | Arrow 数组与 CPU 缓存线对齐。字符串字典适配 L2。位图过滤器适配 L1。 |
| **截拳道** | 一趟多列投影。延迟物化。不进行不必要的反序列化。 |
