# Rust Coding Standards Skills

Welcome to the Rust Coding Standards Skills documentation — a comprehensive, opinionated set of engineering guidelines for building production-grade Rust software across ten specialized domains.

## Overview

This project provides **10 skill guides**, each designed for Cursor AI agents to enforce coding standards, architectural patterns, and domain-specific best practices when writing Rust code.

### The Constitution

The **[Architecture Guide](rust-architecture-guide/SKILL.md)** is the universal constitution — it defines the Priority Pyramid (P0 Safety → P1 Maintainability → P2 Compile Time → P3 Performance), execution modes, error handling strategies, concurrency patterns, FFI safety boundaries, and the Decision Summary contract. All vertical guides defer to the architecture guide for cross-cutting concerns.

### Vertical Domain Guides

Each vertical guide provides domain-specific rules, reference architectures, and prohibited patterns for its niche:

| Guide | Domain |
|-------|--------|
| [Systems & Cloud Infra](rust-systems-cloud-infra-guide/SKILL.md) | Backend services, I/O models, lock-free concurrency, CI/CD |
| [Wasm Frontend Infra](rust-wasm-frontend-infra-guide/SKILL.md) | WebAssembly frontends, zero-copy FFI, frame allocation |
| [Embedded & IoT](rust-embedded-iot-guide/SKILL.md) | `no_std` bare-metal, RTIC/Embassy, peripheral drivers |
| [Data Engineering](rust-data-engineering-guide/SKILL.md) | Arrow columnar memory, streaming ETL, query optimization |
| [Networking Protocols](rust-networking-protocols-guide/SKILL.md) | QUIC/HTTP3, TLS, zero-copy parsing, congestion control |
| [CLI & DevOps](rust-cli-devops-guide/SKILL.md) | clap-based CLIs, distribution, K8s operators, signals |
| [Gamedev](rust-gamedev-guide/SKILL.md) | Bevy ECS, frame budgets, wgpu rendering, asset pipelines |
| [Blockchain](rust-blockchain-guide/SKILL.md) | Solana/NEAR programs, Anchor framework, Stacks clarity |
| [AI/ML Inference](rust-ai-ml-inference-guide/SKILL.md) | Model loading, quantization, KV-cache, serving APIs |

## Key Concepts

### Priority Pyramid

All design decisions are resolved by the four-level priority hierarchy:

1. **P0 — Safety & Correctness**: Memory safety, type soundness, panic-free hot paths
2. **P1 — Maintainability & Clarity**: Readability, idiomatic patterns, explicit intent
3. **P2 — Compile Time**: Build speed, monomorphization bloat, incremental compilation
4. **P3 — Performance**: Micro-optimizations, SIMD, cache locality, allocation reduction

When two rules conflict, the higher-priority rule wins. Never sacrifice safety for performance.

### Execution Modes

Each rule is tagged with an execution mode that determines its enforcement level:

- **`rapid`**: Prototyping — rules are suggestions
- **`standard`**: Default — most rules enforced, with reasonable exceptions
- **`strict`**: Production — all rules enforced, deviations require formal documentation

See the [Architecture Guide's mode documentation](rust-architecture-guide/references/00-mode-guide.md) for detailed mode definitions and transition checklists.

## Getting Started

1. **Browse the [Architecture Guide](rust-architecture-guide/SKILL.md)** to understand the foundational rules
2. **Select your domain guide** from the navigation sidebar
3. **Each SKILL.md** provides an overview, red lines, and a reference table linking to detailed documents
4. **Reference files** (`references/NN-topic.md`) contain code examples, decision trees, and implementation patterns

## Project Structure

```
Rust_Coding_Standards_Skills/
├── rust-architecture-guide/          # Universal constitution (33 reference docs)
│   ├── SKILL.md
│   └── references/
├── rust-systems-cloud-infra-guide/   # Backend infra (14 reference docs)
├── rust-wasm-frontend-infra-guide/   # Wasm frontends (13 reference docs)
├── rust-embedded-iot-guide/          # Embedded systems (8 reference docs)
├── rust-data-engineering-guide/      # Data engineering (8 reference docs)
├── rust-networking-protocols-guide/  # Networking (8 reference docs)
├── rust-cli-devops-guide/            # CLI & DevOps (8 reference docs)
├── rust-gamedev-guide/               # Gamedev (8 reference docs)
├── rust-blockchain-guide/            # Blockchain (8 reference docs)
└── rust-ai-ml-inference-guide/       # AI/ML inference (8 reference docs)
```

**Total: 10 guides, 116 reference documents**
