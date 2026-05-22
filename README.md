# Rust Coding Standards Skills

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Architecture Guide](https://img.shields.io/badge/Architecture%20Guide-v9.1.0-brightgreen.svg)]()
[![Cloud Infra Guide](https://img.shields.io/badge/Cloud%20Infra%20Guide-v6.1.0-orange.svg)]()
[![Wasm Infra Guide](https://img.shields.io/badge/Wasm%20Infra%20Guide-v4.1.0-purple.svg)]()
[![Skills](https://img.shields.io/badge/Skills-10-blue.svg)]()

**Rust Engineering Decision Wiki — 10 constitutional guides for AI coding assistants, following the [Cursor Agent Skills](https://cursor.com/cn/docs/skills) format, covering universal engineering, cloud infrastructure, Wasm frontend, embedded, data engineering, networking, CLI/DevOps, gamedev, blockchain, and AI/ML inference.**

English | [简体中文](README.zh-CN.md)

---

## 📖 About the Naming Style in This Guide

This specification uses metaphors from **Eastern philosophy and martial arts** to describe Rust engineering principles. This is **not advocating violence or any specific ideology**, but borrowing the concept of **"intercepting attacks at their weakest point with minimal force"** to describe compile-time error prevention mechanisms.

| Metaphor | Engineering Equivalent | Core Concept |
|----------|----------------------|--------------|
| **"Jeet Kune Do" (截拳道)** | Compile-time Defense | Intercept bugs before runtime through type systems |
| **"Dialectical Materialism"** | Hardware Sympathy | Acknowledge physical laws, Safe/Unsafe unity of opposites |
| **"One-Inch Punch" (寸劲)** | Batch Deallocation | Maximum effect with minimal cost (Arena, zero-copy) |
| **"Economy of Motion"** | Zero-Cost Abstraction | Every line of code points directly to intent |
| **"Intercepting Fist"** | Parse, Don't Validate | Stop errors at compilation, not runtime |

> 💡 **For Global Readers**: If you encounter unfamiliar philosophical terms, please refer to [`glossary.md`](rust-architecture-guide/references/05-glossary.md) for detailed definitions with "Engineering Translation" explanations.

> 🤖 **For AI Agents**: These metaphors describe compile-time defense mechanisms, not violence. Map them to: type safety, borrow checking, zero-cost abstractions, and mechanical sympathy.

---

## Project Overview

Rust Coding Standards Skills is a **Rust engineering decision guide skill collection** for AI coding assistants, containing a universal engineering constitution, cloud infrastructure-specific specifications, and WebAssembly frontend infrastructure specifications, covering the complete chain from architecture decisions, coding style, to production best practices.

This skill collection serves as the Agent's **constitutional foundation**, ensuring every code generation, review, and refactoring follows a unified priority judgment and conflict resolution framework.

### Problems Solved

| Without Skill | With Skill |
|--------------|------------|
| AI randomly chooses `unwrap` vs `expect` | Auto-judged by P0/P1 priority |
| Generic monomorphization explosion unchecked | Three-level annealing strategy auto-downgrade |
| Error handling library/app level mixed | `thiserror` / `anyhow` layered isolation |
| Concurrency model chosen by intuition | MPSC bounded + minimum lock scope + prohibit Mutex across await |
| API evolution breaks compatibility | `#[non_exhaustive]` + Sealed Trait defense |
| I/O model selection without basis | Tokio epoll vs io_uring selection decision tree |
| Backpressure mechanism missing | Bounded channels, Semaphore, 503 propagation |
| Consensus algorithm implementation non-standard | Deterministic state machines (Raft/Paxos Apply), prohibit time/randomness dependencies |
| WASM binary size bloated | `opt-level="z"` + `wasm-opt -Oz` + allocator replacement |
| JS-WASM FFI per-element overhead | `WasmSlice` zero-copy batch pattern |
| Wasm linear memory leak | Arena per-frame lifecycle + explicit `.free()` |

---

## Core Features

### rust-architecture-guide — Universal Engineering Constitution

Applicable to **all Rust projects**, providing priority decision framework, architecture patterns, and coding style.

- **Four-level priority pyramid** — P0 Safety → P1 Maintainability → P2 Compile Time → P3 Performance
- **Three execution modes** — `rapid` (prototype) / `standard` (default) / `strict` (production)
- **Type-driven architecture** — State machines, Newtype, zero-cost abstraction boundaries
- **Ownership layering strategy** — Business layer Owned + `.clone()` decoupling, hotpath `Cow`/`Bytes` zero-copy
- **Error handling layering** — Library-level `thiserror` structured, application-level `anyhow` + lazy context
- **Concurrency & async specifications** — Bounded channel backpressure, RwLock/parking_lot strategy, `Pin`/`Unpin` semantics
- **API evolution defense** — `#[non_exhaustive]`, Sealed Trait, `#[deprecated]` migration
- **Advanced QA** — proptest + cargo fuzz + Loom + Miri
- **Performance deep tuning** — jemalloc/mimalloc, SoA layout, SmallVec, PGO, SIMD, LTO
- **FFI safety boundary** — `-sys` crate separation, `cxx` safe C++ interop
- **Engineering build specifications** — Cargo Workspace division, Feature Flags isolation, `cargo deny` audit
- **Memory layout transparency** — struct padding audit, `#[repr(C)]` mandate, cache-friendly design, false sharing prevention
- **Breakwater architecture** — Facade/Core layered design, boundary interception protocol, type contraction (de-oxygenation)
- **Physical feasibility audit** — I/O budget, memory ceiling, concurrency true cost with red-line thresholds
- **Jeet Kune Do coding style** — Intercepting Boilerplate, Economy of Motion, Hardware Sympathy
- **Agent self-check directives** — Decision Summary output contract

Full document index: [rust-architecture-guide/README.md](rust-architecture-guide/README.md)

---

### rust-systems-cloud-infra-guide — Cloud Infrastructure Specific

Applicable to **database kernels, distributed storage, high-performance gateways, container runtimes, eBPF control planes, OS components** and other long-running systems. Vertical deepening of the universal constitution.

**Environment assumptions**: Long-running nodes (uptime > 1 year), 10GbE+ networks, multi-NUMA architectures.

- **I/O model decision** — Tokio epoll vs io_uring vs monoio selection decision tree
- **Zero-copy pipeline** — `splice`/`sendfile`/`copy_file_range`, `bytes::Bytes` O(1) clone
- **Backpressure mechanism** — Bounded channels, Semaphore, 503 propagation, absolute prohibition of unbounded channels
- **Cancellation safety** — Non-idempotent writes must use `spawn` + `oneshot`
- **Graceful shutdown** — `SIGTERM`/`SIGINT` capture, CancellationToken, fsync WAL
- **Consensus algorithm specification** — Deterministic state machines, prohibit `Instant::now()`, `rand`, `HashMap` ordering
- **Syscall wrappers** — `rustix` wrapper + eBPF integration
- **Advanced memory architecture** — Arena (`bumpalo`), Slab pre-allocation (`mmap` + `mlock`), NUMA/PMEM
- **Lock-free concurrency** — RCU (`arc-swap`), Epoch reclamation (`crossbeam-epoch`), memory ordering precision
- **Vectorized execution** — SIMD instructions (`std::simd`/AVX-512), Bitmask branch elimination, SoA columnar layout
- **Memory exhaustion backpressure** — `Result<T, AllocError>` return, 503 rejection, prohibit `panic!`
- **Mandatory CI Lints** — 11 strict checks (`await_holding_lock`, `unwrap_used`, etc.)

Full document index: [rust-systems-cloud-infra-guide/README.md](rust-systems-cloud-infra-guide/README.md)

---

### rust-wasm-frontend-infra-guide — WebAssembly Frontend Infrastructure Specific

Applicable to **all Rust projects targeting `wasm32-unknown-unknown`**, providing compilation & boundary layer hard constraints. Vertical deepening of the universal constitution.

**Environment assumptions**: Wasm linear memory only grows, JS ↔ Wasm boundary is expensive RPC, browser main thread must not be blocked, zero-copy views are unsafe operations.

- **[IRON-01] Binary size is paramount** — `opt-level="z"`, `lto=true`, `codegen-units=1`, `panic="abort"`, `strip=true`
- **[IRON-02] Zero-copy at boundary** — `WasmSlice` safe encapsulation, prohibit serialization on hot paths
- **[IRON-03] Memory partitioning** — Global state static residency, per-frame Arena lifecycle (`bumpalo` + `reset()`)
- **[IRON-04] Cross-origin isolation documented** — `SharedArrayBuffer` requires documented COOP/COEP configuration
- **Build control** — `wasm-opt -Oz` mandatory, allocator replacement (`talc`/`MiniAlloc`, prohibit `wee_alloc`)
- **FFI boundary** — Explicit scalar or `WasmSlice` parameters, prohibit `JsValue` pass-through
- **Concurrency** — `wasm-bindgen-futures` only, prohibit blocking APIs, Worker isolation with COOP/COEP
- **Error handling** — `Result<T, JsValue>` + `thiserror`, `panic="abort"` means use `Result` not `panic!`
- **Logging** — `console_error_panic_hook` + `tracing_wasm` mandatory initialization
- **7 hard prohibitions** + **10-item compliance checklist**

Full document index: [rust-wasm-frontend-infra-guide/README.md](rust-wasm-frontend-infra-guide/README.md)

---

### rust-embedded-iot-guide — Embedded & IoT Guide

Applicable to **bare-metal firmware, RTOS-based systems (ARM Cortex-M, RISC-V, ESP32)**, where there is no OS, KB of RAM, and real-time deadlines. Vertical deepening of the universal constitution.

- **no_std architecture** — PAC→HAL→BSP→App five-layer abstraction, `svd2rust` peripheral generation
- **Memory layout control** — `memory.x` linker script, `.bss/.data` init, `flip-link` stack overflow protection
- **Interrupt-driven concurrency** — RTIC priority ceiling protocol, Embassy async executor on bare-metal
- **Peripheral drivers** — SPI/I2C/UART/GPIO/ADC typed state machines via `embedded-hal` traits
- **Power management** — Sleep/deep-sleep state machine, clock gating, < 10µA deep sleep target
- **Hardware debugging** — probe-rs, defmt, RTT channels, ITM/SWO trace, logic analyzer
- **HIL testing** — defmt-test harness, hardware-in-the-loop CI on real target hardware

Full document index: [rust-embedded-iot-guide/SKILL.md](rust-embedded-iot-guide/SKILL.md)

---

### rust-data-engineering-guide — Data Engineering Guide

Applicable to **query engines, ETL pipelines, OLAP databases, stream processors** handling TB-PB scale data. Vertical deepening of universal + cloud infra constitutions.

- **Arrow columnar memory model** — `PrimitiveArray`/`RecordBatch`/`ChunkedArray` zero-copy layering
- **Vectorized expressions** — SIMD bulk evaluation, bitmap filtering, branch-free null propagation
- **Query optimizer** — RBO (predicate pushdown, projection pruning) + CBO (statistics, join reordering)
- **Parquet storage** — Row group statistics, predicate pushdown to storage layer, dictionary/RLE encoding
- **Streaming ETL** — `Stream` trait backpressure, tumbling/sliding windows, watermark, exactly-once checkpoint
- **Partitioning** — Hash/range/broadcast shuffle, data skew detection and salt mitigation
- **Cross-language interchange** — PyO3 Arrow C Data Interface, napi-rs zero-copy, Flight/Flight SQL protocol

Full document index: [rust-data-engineering-guide/SKILL.md](rust-data-engineering-guide/SKILL.md)

---

### rust-networking-protocols-guide — Network Protocols Guide

Applicable to **QUIC/HTTP3 servers, gRPC proxies, custom protocol implementations, DNS resolvers**. Vertical deepening of universal + cloud infra constitutions.

- **Protocol state machines** — Typed states, `tokio-util` Codec, layered stacks (L2→L7)
- **Zero-copy parsing** — `nom`/`winnow` combinator parsing on `&[u8]`, streaming parsers for incomplete input
- **QUIC & HTTP/3** — Stream multiplexing, 0-RTT with replay protection, connection migration
- **TLS integration** — rustls with mTLS, ALPN negotiation, Let's Encrypt automation via rustls-acme
- **Congestion control** — BBR/CUBIC, pacing, ECN, QUIC flow control credits
- **Connection pooling** — deadpool/bb8, HPACK dynamic tables, dead connection detection
- **Fuzzing** — cargo-fuzz structure-aware targets per parser, proptest for round-trip invariants

Full document index: [rust-networking-protocols-guide/SKILL.md](rust-networking-protocols-guide/SKILL.md)

---

### rust-cli-devops-guide — CLI & DevOps Guide

Applicable to **command-line tools, developer toolchains, Kubernetes operators**. Vertical deepening of the universal constitution.

- **clap derive** — Subcommand enums, value validation, shell completion generation
- **Cross-platform distribution** — cargo-dist, Homebrew formulas, `curl | sh` installers, matrix CI
- **Signal handling** — SIGINT/CTRL_C graceful shutdown, SIGPIPE suppression, Windows console handler
- **Progress UX** — indicatif MultiProgress with ETA, ratatui TUI widgets, tracing structured logging
- **Configuration cascade** — CLI args > Env vars > Config file > Defaults, XDG-compliant paths
- **Kubernetes operator** — kube-rs reconciler loop, CRD generation, finalizers, idempotency guarantee
- **CLI testing** — assert_cmd integration tests, trycmd snapshot testing, miette rich error messages

Full document index: [rust-cli-devops-guide/SKILL.md](rust-cli-devops-guide/SKILL.md)

---

### rust-gamedev-guide — Game Development Guide

Applicable to **game engines, rendering pipelines, real-time interactive applications** with 16ms frame budgets. Vertical deepening of the universal constitution.

- **ECS architecture** — Bevy Entities/Components/Systems/Resources, flat data-oriented design
- **Frame budget** — Fixed timestep physics, interpolation rendering, frame pacing, budget profiling
- **GPU resource lifecycle** — wgpu buffers/textures/bind groups, staging belts, async upload
- **Rendering pipeline** — Render graph, frustum culling, LOD instancing, < 1000 draw calls/frame
- **Shaders** — WGSL, naga cross-compilation (GLSL/SPIR-V/MSL), compute shaders, hot-reload
- **Asset pipeline** — AssetServer async loading, hot-reloading, texture compression, atlas packing
- **Physics** — rapier rigid bodies/colliders, spatial partitioning, fixed timestep enforcement

Full document index: [rust-gamedev-guide/SKILL.md](rust-gamedev-guide/SKILL.md)

---

### rust-blockchain-guide — Blockchain & Web3 Guide

Applicable to **Solana programs, Substrate/Polkadot pallets, smart contract development**. Vertical deepening of universal + embedded constitutions.

- **Account model** — Solana stateless programs + account state, Substrate FRAME StorageMap
- **Program Derived Addresses (PDA)** — Deterministic derivation, bump seeds, escrow/vault patterns
- **BPF constraints** — No `std`, no floating point, no dynamic dispatch, 200KB limit, CU budget
- **Anchor framework** — `#[program]`, `#[derive(Accounts)]` validation, `#[account]` data, init space
- **Cross-Program Invocation (CPI)** — SPL Token CPI, `invoke_signed` for PDA authority
- **Security** — Checks-Effects-Interactions, checked math, access control, trident fuzzing

Full document index: [rust-blockchain-guide/SKILL.md](rust-blockchain-guide/SKILL.md)

---

### rust-ai-ml-inference-guide — AI/ML Inference Guide

Applicable to **LLM serving, embedding vector search, ONNX model inference**. Vertical deepening of universal + cloud infra constitutions.

- **Model loading** — GGUF (llama.cpp), SafeTensors, ONNX runtime, checksum verification
- **Quantization** — Q4_0/Q4_K_M/Q8_0 quantization, perplexity evaluation before deployment
- **GPU memory** — KV-cache pre-allocation, layer offloading, OOM backpressure
- **Tokenizer** — HuggingFace tokenizers (BPE/WordPiece), batched encoding, special token alignment
- **Continuous batching** — vLLM-style dynamic batching, prefill/decode splitting, padding elimination
- **Embedding & vector search** — BERT embeddings, L2 normalization, HNSW approximate nearest neighbor
- **Serving API** — SSE token streaming, gRPC bidirectional, TTFT/throughput evaluation

Full document index: [rust-ai-ml-inference-guide/SKILL.md](rust-ai-ml-inference-guide/SKILL.md)

---

## Quick Start

### Installation

This project follows the [Cursor Agent Skills](https://cursor.com/cn/docs/skills) format, an open standard compatible with any AI agent platform supporting Skills.

**Cursor IDE**:

1. Open **Cursor Settings → Rules**
2. Click **Add Rule** → **Remote Rule (Github)**
3. Enter: `https://github.com/smile9493/Rust_Coding_Standards_Skills`

Or clone manually to your skills directory:

```bash
git clone https://github.com/smile9493/Rust_Coding_Standards_Skills.git

# Copy to Cursor skills directory (user-level, globally available)
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.cursor/skills/
cp -r Rust_Coding_Standards_Skills/rust-systems-cloud-infra-guide ~/.cursor/skills/
cp -r Rust_Coding_Standards_Skills/rust-wasm-frontend-infra-guide ~/.cursor/skills/

# Or copy to project-level (project-scoped only)
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide your-project/.cursor/skills/
cp -r Rust_Coding_Standards_Skills/rust-systems-cloud-infra-guide your-project/.cursor/skills/
cp -r Rust_Coding_Standards_Skills/rust-wasm-frontend-infra-guide your-project/.cursor/skills/

# Also compatible with Claude Code / Codex
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.claude/skills/
```

### Usage

**Natural Language Trigger** — Agent automatically applies skills based on context:

```
Help me refactor the Order entity with type-driven state machine
Should this module use thiserror or anyhow for error handling?
Review this concurrent code for Mutex across await issues
Optimize this storage engine's I/O path with io_uring
What memory ordering should I use for this atomic counter?
Write a procedural macro with proper Span-level error reporting
Configure my WASM project's Cargo.toml for minimal binary size
Design a zero-copy JS-WASM boundary for image processing
```

**Slash Command Invocation** — Explicitly invoke skills:

```
/rust-architecture-guide
/rust-systems-cloud-infra-guide
/rust-wasm-frontend-infra-guide
```

### Skill Format

Each skill follows the [Cursor Agent Skills](https://cursor.com/cn/docs/skills) format:

```
skill-name/
├── SKILL.md              # Entry point (YAML frontmatter + Agent instructions)
├── references/            # Deep-dive reference documents (loaded on-demand)
├── scripts/               # Executable code the agent can run (optional)
└── assets/                # Static resources: templates, images, data files (optional)
```

**SKILL.md frontmatter fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✅ | Skill identifier (hyphen-case, must match directory name) |
| `description` | ✅ | What the skill does and when to use it (used by agent for relevance) |
| `paths` | ❌ | Glob pattern(s) to scope the skill to matching files only |
| `disable-model-invocation` | ❌ | When `true`, skill is only invoked via explicit `/skill-name`, not auto |
| `metadata` | ❌ | Arbitrary key-value map for extended metadata |

---

## Skills Index

### rust-architecture-guide — Universal Engineering Constitution

| Layer | Coverage |
|-------|----------|
| **Execution Mode** | `rapid` (prototype) → `standard` (default) → `strict` (production) |
| **P0 Safety** | Memory safety, data consistency, unsafe specifications, FFI boundaries, Miri verification |
| **P1 Maintainability** | Semantic naming, ownership transfer, trait decoupling, Owned > complex lifetimes |
| **P2 Compile Time** | `Box<dyn Trait>` downgrade, workspace division, reject >2x compile growth |
| **P3 Performance** | SoA layout, SIMD, profiling-driven optimization (proven bottlenecks only) |
| **Zero-cost Abstraction** | Marker Traits interception, PhantomData cautious use, monomorphization annealing |
| **Coding Style** | `let else`, iterator chaining, `From`/`Into`, interior mutability |
| **Architecture Patterns** | State machines, Newtype, API evolution, error layering |
| **Async Depth** | `Pin`/`Unpin`, `select!`/`join!`, cancellation safety, custom executors |
| **QA** | proptest property testing, cargo fuzz, Loom concurrency model, Miri UB detection |
| **Metaprogramming** | Declarative/procedural macros, const generics, `const fn` |
| **FFI Interop** | Three-layer isolation, opaque pointers, panic containment, repr(C) |
| **Memory Layout** | Struct padding audit, `#[repr(C)]` mandate, cache-friendly design (≤64 bytes), false sharing prevention |
| **Breakwater Pattern** | Facade/Core layered architecture, boundary interception protocol, type contraction (de-oxygenation), O(1) conversion mandate |
| **Physical Audit** | I/O budget (>30% → batching), memory ceiling (<20% margin → backpressure), concurrency true cost (>20% contention → lock-free) |
| **Toolchain** | CI, Clippy, rustfmt, cargo deny, workspace, feature flags |

Entry: [SKILL.md](rust-architecture-guide/SKILL.md) · Document Index: [README.md](rust-architecture-guide/README.md)

---

### rust-systems-cloud-infra-guide — Cloud Infrastructure Specific

| Domain | Coverage |
|--------|----------|
| **I/O Model** | Tokio epoll vs io_uring vs monoio selection, mixed runtime red line |
| **Zero-Copy** | `splice`, `sendfile`, `bytes::Bytes`, Direct I/O (`O_DIRECT` + alignment) |
| **Backpressure** | Bounded channels, Semaphore, 503 propagation, prohibit unbounded channels |
| **Cancellation Safety** | Non-idempotent writes `spawn` + `oneshot`, `select!` safety |
| **Syscalls** | `rustix` wrappers, eBPF integration (`aya`/`libbpf-rs`), error code mapping |
| **Consensus** | Deterministic state machines, prohibit `Instant::now()`/`rand`/`HashMap` ordering |
| **Resilience** | Graceful shutdown (`SIGTERM` → CancellationToken → fsync WAL), circuit breaker, Lock Poisoning |
| **Observability** | Tracing + Metrics + Panic Hook, `turmoil` network fault simulation |
| **Advanced Memory** | Arena (`bumpalo`), Slab pre-allocation (`mmap` + `mlock`), NUMA/PMEM, `allocator_api2` |
| **Lock-Free** | RCU (`arc-swap`), Epoch (`crossbeam-epoch`), memory ordering (Release+Acquire/Relaxed) |
| **Vectorized** | SIMD (`std::simd`/AVX-512), Bitmask branch elimination, SoA columnar |
| **Breakwater Pattern** | Facade/Core layered architecture, boundary interception protocol, de-oxygenation |
| **Physical Audit** | Container memory limits, network latency budgets, NUMA topology audit |
| **FFI Safety** | `catch_unwind`, error code return, trampoline pattern |
| **Memory Exhaustion** | `Result<T, AllocError>` + backpressure, prohibit `panic!` |
| **CI Lints** | 11 strict checks (`await_holding_lock`, `unwrap_used`, etc.) |

Entry: [SKILL.md](rust-systems-cloud-infra-guide/SKILL.md) · Document Index: [README.md](rust-systems-cloud-infra-guide/README.md)

---

### rust-wasm-frontend-infra-guide — WebAssembly Frontend Infrastructure Specific

| Domain | Coverage |
|--------|----------|
| **Iron Rules** | IRON-01~04: Binary size paramount, zero-copy at boundary, memory partitioning, cross-origin isolation documented |
| **Build Control** | `Cargo.toml` release profile (MUST), `wasm-opt -Oz` (MUST), allocator replacement `talc`/`MiniAlloc` (SHOULD) |
| **FFI Boundary** | `WasmSlice` zero-copy encapsulation (MUST), boundary type explicit contracts, prohibit `JsValue` pass-through |
| **Memory Lifecycle** | Global residency vs per-frame Arena (`bumpalo` + `reset()`), memory leak physical defense |
| **Concurrency** | `wasm-bindgen-futures` (MUST), prohibit blocking APIs (MUST NOT), Worker isolation with COOP/COEP |
| **Wasm Adaptation** | `Result<T, JsValue>` + `thiserror`, `console_error_panic_hook` + `tracing_wasm` |
| **Compliance** | 7 hard prohibitions [F-01]~[F-07], 10-item compliance checklist |

Entry: [SKILL.md](rust-wasm-frontend-infra-guide/SKILL.md) · Document Index: [README.md](rust-wasm-frontend-infra-guide/README.md)

---

## Relationship

```
rust-architecture-guide (Universal Constitution)
          │
          ├── Four-level priority framework (P0 → P3)
          ├── Three execution modes (rapid / standard / strict)
          ├── Jeet Kune Do coding style
          ├── Agent self-check list + Decision Summary
          │
          ├──► rust-systems-cloud-infra-guide (Vertical Deepening)
          │           │
          │           ├── Core Philosophy
          │           │   ├── Mechanical Sympathy — Software aligned with hardware
          │           │   ├── Determinism — Eliminate non-determinism
          │           │   ├── Resilience — Graceful degradation over crash
          │           │   └── Jeet Kune Do — One-strike memory lifecycle
          │           │
          │           ├── I/O Model (epoll vs io_uring vs monoio)
          │           ├── Zero-copy pipeline (splice, sendfile, bytes::Bytes)
          │           ├── Bounded resource backpressure + cancellation safety
          │           ├── Deterministic state machines (consensus algorithms)
          │           ├── Graceful shutdown (CancellationToken flow)
          │           ├── Advanced memory architecture (Arena / Slab / NUMA / PMEM / Allocator API)
          │           ├── Lock-free concurrency (RCU + Epoch + memory ordering)
          │           ├── Vectorized execution (SIMD + SoA)
          │           ├── Breakwater pattern (Facade/Core layered architecture)
          │           ├── Physical feasibility audit (pre-design mandatory)
          │           └── Mandatory CI Lints (11 strict checks)
          │
          └──► rust-wasm-frontend-infra-guide (Vertical Deepening)
                      │
                      ├── Iron Rules
                      │   ├── IRON-01 — Binary size is paramount
                      │   ├── IRON-02 — Zero-copy at boundary
                      │   ├── IRON-03 — Memory partitioning
                      │   └── IRON-04 — Cross-origin isolation documented
                      │
                      ├── Build Control (Cargo.toml + wasm-opt + allocator)
                      ├── FFI Boundary (WasmSlice + explicit contracts)
                      ├── Memory Lifecycle (Arena per-frame + leak defense)
                      ├── Concurrency (wasm-bindgen-futures + Worker isolation)
                      ├── Wasm Adaptation (Result<T, JsValue> + logging)
                      └── Compliance (7 prohibitions + 10-item checklist)
```

- **`rust-architecture-guide`** (v9.1.0): Constitutional foundation for all Rust engineering — adds memory layout transparency, breakwater pattern, physical feasibility audit, and Rust 2024 Edition alignment
- **`rust-systems-cloud-infra-guide`** (v6.1.0): Cloud-native scenario **amendment**, adds system-level red lines, Facade/Core architecture, deployment physical audit, and modern resilience patterns on top of P0 safety
- **`rust-wasm-frontend-infra-guide`** (v4.1.0): `wasm32-unknown-unknown` scenario **amendment**, adds compilation & boundary layer hard constraints, binary size economics, and ecosystem convergence on top of P0 safety
- Complementary use: The universal constitution provides the priority framework; vertical deepening guides add domain-specific red lines

---

## Design Philosophy

### 1. Dialectical Materialism — Contradiction Drives Evolution

Engineering decisions are not static truths but dynamic processes resolved through contradiction. Every `unsafe` block is a contradiction between safety and performance; every `.clone()` is a contradiction between simplicity and efficiency. The priority pyramid is the tool for resolving these contradictions — higher levels veto lower, and the resolution itself becomes the architecture.

**Core dialectics**:
- **Unity of Opposites**: `unsafe` and safe are not enemies — `unsafe` is the material foundation of safe abstractions. The `-sys` crate is unsafe so the wrapper can be safe.
- **Quantitative to Qualitative Change**: An MVP's `Option<bool>` flags accumulate until the business model stabilizes, then must undergo qualitative change into an Enum state machine. The compiler assists by identifying all call sites.
- **Negation of Negation**: Errors are not endpoints but starting points — panic → catch → graceful degradation. Each negation reaches a higher level of resilience.

### 2. Jeet Kune Do — No Rule Is Absolute

> "Using no way as way; having no limitation as limitation." — Bruce Lee

The priority pyramid provides "the way," but the DEVIATION protocol provides "no way" — a formal channel for when breaking a rule is the correct decision. This creates a **self-referential philosophical closure**: the rules contain the means to transcend the rules.

**Applied principles**:
- **Intercepting Boilerplate**: If logic can be expressed in 1 line of pattern matching, never use 5 lines of nesting. `let else` over nested `if let`; `filter_map` over `filter` + `map`.
- **Economy of Motion**: Every line of code should point directly to intent. Eliminate redundant intermediate variables and implicit copies. Arena allocates once, reclaims in bulk — one strike, no wasted motion.
- **Hardware Sympathy**: Leverage iterators and zero-copy types, align with the compiler's inline optimization. SoA layout resonates with CPU cache lines; `const fn` shifts computation from runtime to compile time.

### 3. Pragmatism Over Dogmatism

> **Pursue excellence at system boundaries and hot paths; release mental load for internal flows and cold paths.**

Not every line of code deserves the same level of scrutiny. The execution mode system (`rapid` / `standard` / `strict`) ensures that the cost of rigor is proportional to the cost of failure:

| Mode | Enforce | Trade-off |
|------|---------|-----------|
| `rapid` | P0 only | Unlimited `.clone()`, `anyhow` in libraries, no doc-tests |
| `standard` | P0 + P1 | Default for most projects |
| `strict` | P0–P3 | All deviations require formal `// DEVIATION:` annotation |

### 4. Priority Pyramid — The Constitutional Framework

```
P0: Safety & Correctness (memory safety, data consistency) — Non-negotiable
                  ↓
P1: Maintainability (readability, local complexity control) — Default pursuit
                  ↓
P2: Compile Time (build speed, CI/CD efficiency) — Measure then decide
                  ↓
P3: Runtime Performance (proven bottlenecks only) — Requires Profiler data
```

**Conflict Resolution Rules**:
- **Higher priority vetoes lower**: P0 safety requirements veto P3 performance optimization
- **Same level: choose simpler**: Two P1 solutions conflict, pick the simpler one
- **P3 requires evidence**: Any performance optimization must include Profiler data

### 5. Mechanical Sympathy — Align with Physics

Ultimate performance comes not from clever tricks, but from deep resonance between code logic and underlying physical hardware. Software runs not on abstract machines, but on:

```
L1 Cache (32KB, 4 cycles) → L2 (256KB, 12 cycles) → L3 (shared, 40 cycles) → DRAM (200+ cycles)
NUMA Node 0 ← QPI/UPI → NUMA Node 1
NIC Ring Buffer → Kernel TCP Stack → User Space
```

Performance is not optimized — it is **aligned**. When you understand cache lines are 64 bytes, false sharing destroys concurrency, and `mmap` page faults cost microseconds, you stop "optimizing" and start **designing** structures that resonate with hardware.

### 6. Machine vs Machine — Deterministic Quality

Human-written example-based tests only cover known territory. Against the extreme complexity of concurrency and edge cases, we must unleash machine computation:

- **Property-based Testing** (`proptest`): From "concrete examples" to "universal physical laws" — verify symmetry, idempotency, monotonicity
- **Fuzzing** (`cargo-fuzz`): Genetic mutation and evolutionary selection, searching for crash points in chaos
- **Concurrency Model Checking** (`loom`): Eliminate the randomness of thread scheduling, achieve exhaustive deterministic exploration
- **UB Detection** (`Miri`): Before LLVM optimizes your code, use an interpreter to scrutinize every memory contract

### Agent Decision Log

Each conversation generates a **Decision Summary**, recording:
1. Current execution mode (rapid/standard/strict)
2. Conflicts faced and priority judgments
3. Chosen solution and rationale
4. Rejected solutions and reasons
5. Deviation records (if any)

---

## Project Structure

```
├── rust-architecture-guide/           # P0: Universal Constitution
│   ├── SKILL.md
│   └── references/                    # 33 reference documents
│
├── rust-systems-cloud-infra-guide/    # Cloud Infrastructure Vertical
│   ├── SKILL.md
│   └── references/                    # 14 reference documents
│
├── rust-wasm-frontend-infra-guide/    # Wasm Frontend Infrastructure Vertical
│   ├── SKILL.md
│   └── references/                    # 13 reference documents
│
├── rust-embedded-iot-guide/           # Embedded & IoT Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-data-engineering-guide/       # Data Engineering Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-networking-protocols-guide/   # Network Protocols Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-cli-devops-guide/             # CLI & DevOps Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-gamedev-guide/                # Game Development Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-blockchain-guide/             # Blockchain & Web3 Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── rust-ai-ml-inference-guide/        # AI/ML Inference Vertical
│   ├── SKILL.md
│   └── references/                    # 8 reference targets
│
├── README.md                          # This file — Wiki index (overview)
└── README.zh-CN.md                    # Chinese version
```

---

## License

[MIT](LICENSE)

---

## Acknowledgments

This skill collection's specification system synthesizes engineering practices from the following sources:

### Rust Ecosystem

- [Rust Official Documentation](https://doc.rust-lang.org/) — Language specification and standard library API
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) — Public API design checklist
- [The Rust Programming Language](https://doc.rust-lang.org/book/) — Official book
- [Rust Reference](https://doc.rust-lang.org/references/) — Language reference
- [Rustonomicon](https://doc.rust-lang.org/nomicon/) — Unsafe Rust dark arts
- [Too Many Lists](https://rust-unofficial.github.io/too-many-lists/) — Unsafe and pointer safety tutorial
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial) — Async runtime best practices
- [Cargo Book](https://doc.rust-lang.org/cargo/) — Build system and package manager

### Async & Concurrency

- [Tokio](https://tokio.rs/) — Async runtime ecosystem
- [async-book](https://rust-lang.github.io/async-book/) — Asynchronous programming in Rust
- [crossbeam](https://github.com/crossbeam-rs/crossbeam) — Concurrent programming tools
- [loom](https://github.com/tokio-rs/loom) — Concurrency model checking

### Performance & Systems

- [Martin Thompson — Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/) — Hardware-aligned software design philosophy
- [rustix](https://github.com/bytecodealliance/rustix) — Safe Rust wrappers for POSIX/Win32 syscalls
- [bumpalo](https://github.com/fitzgen/bumpalo) — Fast arena allocator
- [arc-swap](https://github.com/vorner/arc-swap) — RCU-style atomic reference swapping
- [DashMap](https://github.com/xacrimon/dashmap) — Sharded concurrent HashMap

### Quality Assurance

- [proptest](https://github.com/proptest-rs/proptest) — Property-based testing framework
- [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz) — Fuzzing infrastructure
- [Miri](https://github.com/rust-lang/miri) — Undefined behavior detection interpreter
- [trybuild](https://github.com/dtolnay/trybuild) — Compile-time macro testing
- [Kani](https://github.com/model-checking/kani) — Formal verification tool

### FFI & Interop

- [cxx](https://github.com/dtolnay/cxx) — Safe C++ interop
- [bindgen](https://github.com/rust-lang/rust-bindgen) — Automatic FFI binding generation
- [wasmtime](https://github.com/bytecodealliance/wasmtime) — WebAssembly runtime

### Agent Skills Specification

- [Agent Skills Spec v1.0](https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx) — Universal AI agent skills format specification

### Community

- Architecture experience summaries from numerous production-grade Rust projects in the community
- Rust community forum, Reddit r/rust, and Zulip discussions that shaped these best practices
