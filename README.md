# Rust Coding Standards Skills

[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Architecture Guide](https://img.shields.io/badge/Architecture%20Guide-v8.0.0-brightgreen.svg)]()
[![Cloud Infra Guide](https://img.shields.io/badge/Cloud%20Infra%20Guide-v5.0.0-orange.svg)]()
[![Reference Docs](https://img.shields.io/badge/Reference-40%20Docs-orange.svg)]()

**Rust Engineering Decision Wiki — Constitutional guides for AI coding assistants, covering universal engineering decisions and cloud infrastructure-specific specifications.**

English | [简体中文](README.zh-CN.md)

---

## Project Overview

Rust Coding Standards Skills is a **Rust engineering decision guide skill collection** for AI coding assistants, containing a universal engineering constitution and cloud infrastructure-specific specifications, covering the complete chain from architecture decisions, coding style, to production best practices.

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

## Quick Start

### Installation

This project follows the [Agent Skills Spec v1.0](https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx) format and is compatible with any AI agent platform that supports the Skills specification.

```bash
# Clone the repository
git clone https://github.com/smile9493/Rust_Coding_Standards_Skills.git

# Install to your agent's skills directory
# Trae IDE
cp -r Rust_Coding_Standards_Skills/rust-architecture-guide ~/.trae/skills/
cp -r Rust_Coding_Standards_Skills/rust-systems-cloud-infra-guide ~/.trae/skills/

# Claude Code / other Agent Skills compatible platforms
# Copy both skill directories to your agent's skills configuration path
```

### Usage

**Skill Invocation** (platform-specific syntax):

```
# Trae IDE
/rust-architecture-guide priority my_conflict
/rust-architecture-guide state-machine Order
/rust-systems-cloud-infra-guide io-model
/rust-systems-cloud-infra-guide backpressure

# Claude Code / other platforms — reference skill name in prompt
# "According to rust-architecture-guide, what priority should I assign?"
# "Follow rust-systems-cloud-infra-guide to select the I/O model for this gateway"
```

**Natural Language Trigger** — The Agent automatically matches skills based on context:

```
Help me refactor the Order entity with type-driven state machine
Should this module use thiserror or anyhow for error handling?
Review this concurrent code for Mutex across await issues
Optimize this storage engine's I/O path with io_uring
What memory ordering should I use for this atomic counter?
Write a procedural macro with proper Span-level error reporting
```

### Skill Format

Each skill follows the Agent Skills Spec v1.0 structure:

```
skill-name/
├── SKILL.md              # Entry point (YAML frontmatter + Agent instructions)
└── reference/            # Deep-dive reference documents
    ├── 01-topic.md
    ├── 02-topic.md
    └── ...
```

**SKILL.md frontmatter fields**:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✅ | Skill identifier (hyphen-case, matches directory name) |
| `description` | ✅ | When to invoke this skill (max 1024 chars) |
| `license` | ❌ | License identifier |
| `metadata` | ❌ | Extended metadata (version, philosophy, domain, etc.) |
| `allowed-tools` | ❌ | Tools the skill is permitted to use |

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
| **Advanced Memory** | Arena (`bumpalo`), Slab (`mmap`+`mlock`), NUMA/PMEM, `allocator_api2` |
| **Lock-Free** | RCU (`arc-swap`), Epoch (`crossbeam-epoch`), memory ordering (Release+Acquire/Relaxed) |
| **Vectorized** | SIMD (`std::simd`/AVX-512), Bitmask branch elimination, SoA columnar |
| **FFI Safety** | `catch_unwind`, error code return, trampoline pattern |
| **Memory Exhaustion** | `Result<T, AllocError>` + backpressure, prohibit `panic!` |
| **CI Lints** | 11 strict checks (`await_holding_lock`, `unwrap_used`, etc.) |

Entry: [SKILL.md](rust-systems-cloud-infra-guide/SKILL.md) · Document Index: [README.md](rust-systems-cloud-infra-guide/README.md)

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
          └──► rust-systems-cloud-infra-guide (Vertical Deepening)
                      │
                      ├── Core Philosophy
                      │   ├── Mechanical Sympathy — Software aligned with hardware
                      │   ├── Determinism — Eliminate non-determinism
                      │   ├── Resilience — Graceful degradation over crash
                      │   └── Jeet Kune Do — One-strike memory lifecycle
                      │
                      ├── I/O Model (epoll vs io_uring vs monoio)
                      ├── Zero-copy pipeline (splice, sendfile, bytes::Bytes)
                      ├── Bounded resource backpressure + cancellation safety
                      ├── Deterministic state machines (consensus algorithms)
                      ├── Graceful shutdown (CancellationToken flow)
                      ├── Advanced memory architecture (Arena / Slab / NUMA / PMEM / Allocator API)
                      ├── Lock-free concurrency (RCU + Epoch + memory ordering)
                      ├── Vectorized execution (SIMD + SoA)
                      └── Mandatory CI Lints (11 strict checks)
```

- **`rust-architecture-guide`** (v8.0.0): Constitutional foundation for all Rust engineering
- **`rust-systems-cloud-infra-guide`** (v5.0.0): Cloud-native scenario **amendment**, adding system-level red lines and hardware alignment constraints on top of P0 safety
- Complementary use: The universal constitution provides the priority framework; the cloud infrastructure guide vertically deepens on top of it

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
├── rust-architecture-guide/
│   ├── SKILL.md                          # Skill entry
│   ├── README.md                         # Document index (detailed)
│   └── reference/                        # 29 reference documents
│
├── rust-systems-cloud-infra-guide/
│   ├── SKILL.md                          # Skill entry
│   ├── README.md                         # Document index (detailed)
│   └── reference/                        # 11 reference documents
│
└── README.md                              # This file — Wiki index (overview)
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
- [Rust Reference](https://doc.rust-lang.org/reference/) — Language reference
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
