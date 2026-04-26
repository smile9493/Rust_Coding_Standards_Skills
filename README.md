# Rust Coding Standards Skills

A comprehensive Rust engineering decision guide implemented as a Trae IDE Skill, covering architecture decisions, coding style, and production best practices.

## Overview

This Skill serves as the **constitutional foundation** for Rust engineering decisions, structured for deterministic, reproducible decision-making suitable for AI-assisted development.

### Core Coverage

| Domain | Topics |
|--------|--------|
| **Architecture** | Priority pyramid (P0-P3), ownership & memory, error handling layering, concurrency & async |
| **Coding Style** | Control flow patterns, iterators, idiomatic traits, borrowing, data structures |
| **Performance** | Allocator strategy, cache locality, lock-free concurrency, compile-time magic, unsafe interventions |
| **Quality Assurance** | Property-based testing, fuzzing, concurrency model checking (Loom), UB detection (Miri) |
| **Interoperability** | FFI boundaries, C interop, panic containment, cbindgen/bindgen |
| **Observability** | Structured tracing, zero-cost metrics, panic hooks, core dumps |
| **Build Engineering** | Cargo Workspace, feature flags, API evolution (`#[non_exhaustive]`, sealed traits), `cargo deny` |
| **Metaprogramming** | Declarative macros, procedural macros, const generics, `pin-project` |

## Skill Structure

```
rust-architecture-guide/
  SKILL.md                          # Skill entry point (core definitions + full guide)
  reference/                        # 26 reference documents
    priority-pyramid.md             # The four-level hierarchy
    conflict-resolution.md          # Typical conflicts and resolutions
    progressive-architecture.md     # MVP to production migration
    state-machine.md                # Type-driven state machines
    newtype.md                      # Type-safe IDs and credentials
    data-architecture.md            # Ownership, cloning, memory layout
    error-handling.md               # Library vs application error strategies
    concurrency.md                  # Message passing, channels, locking
    async-internals.md              # Async internals, Pin/Unpin, custom executors
    api-design.md                   # Public API boundaries, #[non_exhaustive], sealed traits
    observability.md                # Tracing, metrics, panic hooks, coredumps
    toolchain.md                    # CI, Clippy, unsafe, workspace, feature flags, cargo deny
    performance-tuning.md           # Complete performance tuning guide
    advanced-testing.md             # Property tests, fuzzing, Loom, Miri
    metaprogramming.md              # Declarative/procedural macros, const generics
    ffi-interop.md                  # FFI boundaries, C interop, panic containment
    ...                             # And more
```

## Quick Start

Invoke the Skill in Trae IDE:

```
/rust-architecture-guide [command] [target]
```

### Common Commands

| Command | Description |
|---------|-------------|
| `priority` | Apply priority matrix to conflicts |
| `conflict` | Resolve typical conflicts |
| `state-machine` | Design state machine for entities |
| `error-handling` | Set up error strategy |
| `concurrency` | Choose concurrency model |
| `api-design` | Design public API boundaries |
| `review` | Comprehensive code review |
| `performance` | Performance tuning guide |

## Core Philosophy

**Pragmatism over Dogma** — Pursue excellence at system boundaries and hot paths; release mental load for internal flows and cold paths.

### Priority Pyramid

```
P0: Safety & Correctness (Memory Safety, Data Consistency)
                  ↓
P1: Maintainability (Readability, Local Complexity Control)
                  ↓
P2: Compile Time (Build Speed, CI/CD Velocity)
                  ↓
P3: Runtime Performance (Only in Proven Bottlenecks)
```

## License

MIT
