# Rust Architecture Guide

**Universal Rust Engineering Decision Constitution** — Covering architecture design, idiomatic coding style, metaprogramming, FFI interop, performance tuning, and advanced quality assurance for all Rust projects.

English | [简体中文](README.zh-CN.md)

## Overview

This guide is the **constitutional foundation** for AI coding assistants, providing:

- **Four-level priority system** (P0 Safety → P1 Maintainability → P2 Compile Time → P3 Performance)
- **Three execution modes** (rapid / standard / strict)
- **Type-driven architecture** (state machines, Newtype, zero-cost abstraction boundaries)
- **Ownership layering strategy** (business layer Owned, hotpath layer zero-copy)
- **Error handling layering** (library-level `thiserror`, application-level `anyhow`)
- **Jeet Kune Do coding style** (Intercepting Boilerplate, Economy of Motion, Hardware Sympathy)
- **Agent self-check list** + Decision Summary output contract

## Core Philosophy

> **Pursue excellence at system boundaries and hot paths; release mental load for internal flows and cold paths.**

| Priority | Focus | Rule |
|----------|-------|------|
| **P0** | Safety & Correctness | Memory safety, data consistency — non-negotiable |
| **P1** | Maintainability | Readability, local complexity control — default pursuit |
| **P2** | Compile Time | Build speed, CI/CD efficiency — measure then decide |
| **P3** | Runtime Performance | Only for proven bottlenecks — requires Profiler evidence |

## Document Index

### Execution & Strategy (7)

| File | Topic |
|------|-------|
| `reference/00-mode-guide.md` | Execution modes — rapid / standard / strict definitions and transitions |
| `reference/01-priority-pyramid.md` | Four-level priority pyramid |
| `reference/02-conflict-resolution.md` | Typical conflicts and resolutions |
| `reference/03-progressive-architecture.md` | MVP → Production progressive architecture |
| `reference/04-trade-offs.md` | Trade-off decision analysis framework |
| `reference/05-glossary.md` | Centralized terminology glossary |
| `reference/06-deviation-process.md` | Deviation process (`// DEVIATION:` annotation) |

### Architecture Patterns (11)

| File | Topic |
|------|-------|
| `reference/07-state-machine.md` | Type-driven state machine design |
| `reference/08-newtype.md` | Newtype pattern and type-safe IDs |
| `reference/09-data-architecture.md` | Ownership, cloning, memory layout |
| `reference/10-error-handling.md` | Library-level `thiserror` vs application-level `anyhow` |
| `reference/11-concurrency.md` | Concurrency: channels, locks, RwLock, parking_lot, deadlock prevention |
| `reference/12-async-internals.md` | Async internals: Future, Pin/Unpin, select!/join!, cancellation safety |
| `reference/13-api-design.md` | Public API: `#[non_exhaustive]`, Sealed Trait, `#[deprecated]` |
| `reference/14-metaprogramming.md` | Intercepting Boilerplate: declarative macros, procedural macros, const fn, const generics |
| `reference/15-ffi-interop.md` | The Defense Wall: three-layer isolation, opaque pointers, panic containment, repr(C) |
| `reference/16-observability.md` | Tracing, Metrics, Panic Hook, Coredump |
| `reference/17-toolchain.md` | CI, Clippy, rustfmt, Workspace, Feature Flags, cargo deny |

### Idiomatic Style (7)

| File | Topic |
|------|-------|
| `reference/18-control-flow.md` | `let else`, `matches!`, intercepting deep nesting |
| `reference/19-iterators.md` | Iterator chains, `filter_map`, flowing force |
| `reference/20-traits.md` | `From` vs `Into`, `Default`, Hardware Sympathy |
| `reference/21-errors.md` | `unwrap_or_else`, `map_err`, `and_then` |
| `reference/22-data-struct.md` | Field shorthand, type stuttering |
| `reference/23-borrowing.md` | `AsRef`, `Cow`, memory economy |
| `reference/24-refactor.md` | Agent Self-Check List, Reduction Directive |

### Performance & QA (4)

| File | Topic |
|------|-------|
| `reference/25-performance-tuning.md` | Mechanical Sympathy: memory, cache, lock-free, SIMD, BCE, prefetching |
| `reference/26-advanced-testing.md` | Machine vs Machine: proptest, fuzzing, loom, Miri, turmoil, defense report |
| `reference/27-review.md` | Comprehensive review checklist |
| `reference/28-usage-examples.md` | Real-world usage examples |

## Relationship

- **Standalone use**: Applicable to all Rust projects (web services, CLI tools, libraries, etc.)
- **Combined use**: `rust-systems-cloud-infra-guide` provides vertical deepening for cloud infrastructure scenarios

## License

MIT
