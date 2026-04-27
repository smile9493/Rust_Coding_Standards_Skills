---
name: rust-architecture-guide
description: Comprehensive Rust engineering guide covering priority pyramid, architecture decisions, and coding style. Invoke when starting Rust projects, making trade-offs, or writing idiomatic code.
version: 4.0.0
user-invocable: true
argument-hint: "[mode|priority|conflict|state-machine|newtype|data-arch|error-handling|concurrency|async|select|cancellation|api-design|non-exhaustive|sealed-trait|deprecated|object-safety|performance|allocator|cache-layout|lock-free|smallvec|pgo|property-test|fuzz|loom|miri|control-flow|iterators|traits|borrowing|interior-mutability|review|metaprogramming|macro|const-fn|ffi|interop|cxx|cbindgen|bindgen|poll|pin|executor|tracing|metrics|observability|panic|coredump|workspace|feature-flags|cargo-deny|rustfmt|ci|progressive|glossary|deviation] [target]"
license: MIT
---

# Rust Architecture & Engineering Decision Guide

This document serves as the **constitutional foundation** for Rust engineering decisions. It is structured for deterministic, reproducible decision-making suitable for AI-assisted development.

### Execution Mode

Before applying any rule, determine the execution mode. Rules are enforced relative to the project's lifecycle stage:

- **`rapid`** (prototype): Enforce P0 only; `anyhow` in libraries, unlimited `.clone()`, no doc-tests
- **`standard`** (default): Enforce P0+P1; warn on P2 violations
- **`strict`** (production): Enforce P0-P3; all deviations require formal annotation

See [reference/00-mode-guide.md](reference/00-mode-guide.md) for full mode definitions, selection decision tree, and transition checklists.

### Mandatory Output Contract

**Every code generation or review output MUST end with a Decision Summary block:**

```markdown
## Decision Summary
- **Mode**: [rapid|standard|strict]
- **Rules Applied**: [list specific rules from P0-P3]
- **Conflicts Resolved**: [PX > PY with justification, or "None"]
- **Deviations**: [list with `// DEVIATION: reason` references, or "None"]
- **Trade-offs**: [key trade-off decisions made]
```

For **review outputs**, additionally include a checklist table:

```markdown
## Review Checklist
| Category | Items Checked | Issues Found | Status |
|----------|--------------|-------------|--------|
| Safety (P0) | 5 | 0 | ✅ |
| Maintainability (P1) | 8 | 2 | ⚠️ |
| Compile Time (P2) | 3 | 0 | ✅ |
| Performance (P3) | 4 | 1 | ⚠️ |
```

---

## 1. Priority Decision Matrix

When conflicts arise, apply decisions top-down. Upper levels have absolute veto power.

### 1.1 Decision Framework

| Level | Core Principle | Accept Conditions | Reject Conditions |
|:------|:--------------|:------------------|:------------------|
| **P0** | **Safety & Correctness** | Compile-time borrow checking, explicit thread synchronization, Miri-verified unsafe | Any UB-causing code, race conditions, unproven unsafe |
| **P1** | **Maintainability** | Semantic naming, ownership transfer (Move), trait-based decoupling, `.clone()` (non-hotpaths) | Deeply nested lifetimes (`<'a, 'b, 'c>`), overly complex generics, magic macros |
| **P2** | **Engineering Efficiency** | `Box<dyn Trait>`, workspace division, feature isolation | Binary bloat from monomorphization, proc-macro causing >2x compile time increase |
| **P3** | **Runtime Performance** | SoA layout, lock-free structures, SIMD, profiling-proven optimizations | Intuition-based premature optimization, optimization adding dev burden without measurable gains |

### 1.2 Priority Pyramid (Visual)

```
        P0: Safety & Correctness (Memory Safety, Data Consistency)
                          ↓
        P1: Maintainability (Readability, Local Complexity Control)
                          ↓
        P2: Compile Time (Build Speed, CI/CD Velocity)
                          ↓
        P3: Runtime Performance (Only in Proven Bottlenecks)
```

### 1.3 Priority-Specific Rules

#### P0: Absolute Red Line — Safety & Correctness
- **Non-negotiable**: Never use `unsafe` to bypass lifetime checks for convenience
- **Enforcement**: All unsafe code requires SAFETY comments explaining why memory safety is preserved
- **Verification**: Unsafe code must pass Miri UB detection before production

#### P1: Core Requirement — Maintainability
- **Rule**: Code is for team and future self
- **Trade-off**: When "zero-cost abstractions" create cognitive overload, prefer clarity
- **Preference**: Owned types over complex lifetime annotations

#### P1.1: Zero-Cost Abstraction Boundaries
- **Marker Traits**: Use marker traits to constrain generic behavior and intercept illegal operations at compile time
- **PhantomData**: Use cautiously — avoid creating deeply nested or cryptic `PhantomData` chains ("type magic")
- **Monomorphization Retreat**: When generic monomorphization severely impacts compile speed or binary size, allow retreat to runtime error checking or `Box<dyn Trait>` (P2 > P0 for non-safety-critical internal code)

```rust
// ✅ Marker trait: compile-time constraint
trait Verified {}
fn publish(doc: impl Verified) { /* only verified docs */ }

// ⚠️ PhantomData: simple use only
struct PhantomSlice<'a, T> { _marker: PhantomData<&'a T> }

// ✅ Monomorphization retreat: runtime dispatch when compile cost is too high
fn process(data: Box<dyn Processor>) -> Result<Output> { /* ... */ }
```

#### P2: Engineering Efficiency — Compile Time
- **Rule**: Rust compilation is inherently slow
- **Trade-off**: When complex generics kill CI/CD, compromise to runtime with `Box<dyn Trait>`
- **Threshold**: Reject solutions causing >2x compile time increase

#### P3: Specific Goal — Runtime Performance
- **Rule**: Never micro-optimize based on intuition
- **Prerequisite**: Profiler (Flamegraph) must identify hotspots first
- **Authorization**: Only then allowed to break P1/P2 principles

---

## 2. Ownership & Memory Architecture Strategy

### 2.1 Business Logic Layer

- **Ownership Model**: Prefer owned types (`String`, `Vec<T>`)
- **Decoupling**: Allow `.clone()` to eliminate lifetime interference on non-hotpaths
- **Reference Constraints**: Prohibit storing non-`'static` references in structs, unless struct is transient view (e.g., parser)

```rust
// ✅ Business layer: Prefer owned types
fn process_user_input(input: String) -> Result<User> {
    let user = User::new(input)?;  // Clone is acceptable
    Ok(user)
}
```

### 2.2 Performance Hotpaths Layer

- **Zero-Copy Design**: Use `Cow<'a, T>` or `bytes::Bytes` for cross-layer sharing
- **Memory Layout**: For bulk data processing, enforce AoS → SoA (Struct of Arrays) conversion
- **Alignment Optimization**: High-frequency concurrent access structs require `#[repr(align(64))]` to eliminate false sharing

```rust
// ✅ Hotpath: SoA layout for cache efficiency
#[repr(C)]
struct ParticleSystem {
    positions: Vec<f32>,      // Contiguous for SIMD
    velocities: Vec<f32>,
    accelerations: Vec<f32>,
}
```

---

## 3. Error Handling Layering Specification

### 3.1 Library Crate (Library-level)

- **Required**: Use `thiserror` or manually implement `Display` + `Error` trait
- **Prohibited**: Never return `anyhow::Error` from public API
- **Visibility**: Error variants must contain sufficient context (e.g., `InvalidConfig { key: String, reason: String }`)

```rust
// ✅ Library: Rich error context
#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("Invalid key '{key}': {reason}")]
    InvalidKey { key: String, reason: String },
}
```

### 3.2 Application Binary (Application-level)

- **Processing**: Use `anyhow` with `?` operator for error propagation
- **Context**: Attach `.with_context(|| ...)` at error propagation points for business context
- **Panic Handling**: Must register global Panic Hook at `main` entry, writing Backtrace to synchronous log stream

```rust
// ✅ Application: Contextual error propagation
fn load_config(path: &Path) -> anyhow::Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config from {}", path.display()))?;
    Config::parse(&content)
        .with_context(|| "Failed to parse config file")?;
}
```

---

## 4. Agent Execution & Self-Review Directives

**MUST** execute the following workflow before outputting any Rust solution:

### 4.1 Trade-off Analysis

Before output block, document decision rationale:
```rust
// Trade-off Analysis:
// - P1 (simplicity): Using String.clone() adds ~2KB runtime allocation
// - P3 (performance): Acceptable for this cold path (config loading)
// - Decision: Clone for clarity; revisit if profiling shows bottleneck
```

### 4.2 Ownership Review Checklist

- [ ] Unnecessary lifetime annotations removed?
- [ ] Closures capturing only necessary ownership?
- [ ] Large objects boxed before `.await` boundaries?

### 4.3 Concurrency Review Checklist

- [ ] `std::sync::Mutex` held during `await`? If yes, **mandatory** replacement with `tokio::sync::Mutex` or refactor to release lock before await
- [ ] Blocking operations in async context? If yes, dispatch to `spawn_blocking`

### 4.4 Error Robustness Review Checklist

- [ ] All `unwrap()` have corresponding `expect("REASON")` with sufficient justification?
- [ ] External IO handles timeout and retry logic?
- [ ] Error propagation includes business context via `.with_context()`?

### 4.5 Safety Review Checklist

- [ ] Unsafe blocks have SAFETY comments explaining invariants?
- [ ] Raw pointers converted to references at boundary?
- [ ] FFI boundaries use opaque handles?
- [ ] Unsafe code passes Miri UB detection (`cargo miri test`)?
- [ ] `cargo deny check` passes (no vulnerabilities, license compliance)?
- [ ] All `extern "C"` wrapped in `catch_unwind`?

---

## 5. Typical Refactoring Paths

### 5.1 From Fast to Robust

**Path**: `Option/bool` flags → `Enum` state machine

```rust
// ❌ Before: Boolean flags
struct Order { is_paid: bool, is_shipped: bool }

// ✅ After: Type-driven state machine
enum OrderState {
    Created,
    Paid(PaymentInfo),
    Shipped(TrackingInfo),
}
```

### 5.2 From Simple to Efficient

**Path**: `String` storage → `Cow<'a, str>` or `Arc<str>` sharing

```rust
// ❌ Before: Unnecessary owned allocation
fn display_name(name: String) -> String { name }

// ✅ After: Zero-copy with Cow
fn display_name(name: &str) -> &str { name }
```

### 5.3 From Verbose to Concise

**Path**: Manual `for` loops → `Iterator` chaining

```rust
// ❌ Before: Verbose imperative
let mut results = Vec::new();
for item in items {
    if let Some(transformed) = transform(item) {
        results.push(transformed);
    }
}

// ✅ After: Concise functional
let results: Vec<_> = items.iter().filter_map(transform).collect();
```

### 5.4 From MVP to Production (Progressive Architecture)

**Path**: Single struct + runtime validation → Type-driven state machine + compile-time guarantees

```rust
// ✅ MVP: Pragmatic — single struct with enum + Option fields
struct Order {
    status: OrderStatus,
    paid_at: Option<DateTime>,
    shipped_at: Option<DateTime>,
}

// ✅ Production: Type-driven — impossible states unrepresentable
enum OrderState {
    Created(OrderInfo),
    Paid(PaymentInfo),
    Shipped(TrackingInfo),
}
```

**Decision Framework**: See [reference/03-progressive-architecture.md](reference/03-progressive-architecture.md)

| Signal | Action |
|--------|--------|
| Business model unproven | Start with MVP patterns |
| Core domain logic stable | Refactor to production patterns |
| Compiler errors are helpful | Embrace type-driven refactoring |

---

## 6. Production Observability Specification

### 6.1 Tracing Requirements

- **Mandatory**: All `pub async fn` must be annotated with `#[tracing::instrument]`
- **Exclusion**: Use `skip(...)` to exclude large objects (db pools, configs)
- **Context**: All internal `.await` calls automatically inherit parent Span

```rust
// ✅ Compliant tracing
#[tracing::instrument(name = "order_processing", skip(db_pool, cache), err)]
pub async fn process_order(
    order_id: u64,
    db_pool: &DbPool,
    cache: &RedisCache,
) -> Result<Order> {
    // All logs within automatically carry order_id=xxx
}
```

### 6.2 Metrics Requirements

- **Hotpath Rule**: Prohibit string formatting for metric labels; use static label groups
- **Collection**: Hot paths must use atomic counters (`AtomicU64`) with `Ordering::Relaxed`
- **Model**: Prefer Prometheus Pull model (bypass port) over Push model

```rust
// ✅ Compliant: Static labels, atomic counter
metrics::counter!("http_requests_total", "method" => "GET", "path" => "/api").increment(1);
```

### 6.3 Panic Hook Requirements

- **Mandatory**: Install custom hook at `main` entry
- **Synchronous**: Write critical errors to stderr/filesystem, bypass async queue
- **Termination**: Use `std::process::abort()` instead of unwinding to prevent zombie processes holding corrupted global resources

```rust
pub fn setup_panic_hook() {
    std::panic::set_hook(Box::new(|panic_info| {
        let backtrace = std::backtrace::Backtrace::force_capture();
        eprintln!("CRITICAL: {}\nStack:\n{}", panic_info, backtrace);
        std::process::abort();
    }));
}
```

---

## 7. Commands Reference

### Strategic Decisions

| Command | Category | Description | Reference |
|---|---|---|---|
| `mode [level]` | Strategy | Set execution mode (rapid/standard/strict) | [reference/00-mode-guide.md](reference/00-mode-guide.md) |
| `priority [context]` | Strategy | Apply priority matrix to conflicts | [reference/01-priority-pyramid.md](reference/01-priority-pyramid.md) |
| `conflict [scenario]` | Strategy | Resolve typical conflicts | [reference/02-conflict-resolution.md](reference/02-conflict-resolution.md) |
| `progressive [stage]` | Strategy | MVP vs production decisions | [reference/03-progressive-architecture.md](reference/03-progressive-architecture.md) |
| `glossary [term]` | Reference | Look up terminology definitions | [reference/05-glossary.md](reference/05-glossary.md) |
| `deviation [rule-id]` | Process | Document or review rule deviations | [reference/06-deviation-process.md](reference/06-deviation-process.md) |

### Architecture Patterns

| Command | Category | Description | Reference |
|---|---|---|---|
| `state-machine [entity]` | Design | Design state machine for entities | [reference/07-state-machine.md](reference/07-state-machine.md) |
| `newtype [type]` | Design | Apply newtype pattern | [reference/08-newtype.md](reference/08-newtype.md) |
| `data-arch [structure]` | Design | Design ownership and memory layout | [reference/09-data-architecture.md](reference/09-data-architecture.md) |
| `error-handling [layer]` | Design | Set up error strategy | [reference/10-error-handling.md](reference/10-error-handling.md) |
| `concurrency [pattern]` | Design | Choose concurrency model | [reference/11-concurrency.md](reference/11-concurrency.md) |
| `async [runtime]` | Design | Async internals, custom executors | [reference/12-async-internals.md](reference/12-async-internals.md) |
| `api-design [interface]` | Design | Design public API boundaries | [reference/13-api-design.md](reference/13-api-design.md) |
| `metaprogramming [type]` | Design | Macros, const generics | [reference/14-metaprogramming.md](reference/14-metaprogramming.md) |
| `ffi [language]` | Design | FFI boundaries, C interoperability | [reference/15-ffi-interop.md](reference/15-ffi-interop.md) |
| `observability [type]` | Design | Tracing, metrics, panic hooks | [reference/16-observability.md](reference/16-observability.md) |
| `toolchain [config]` | Configure | Set up CI, Clippy, unsafe guidelines | [reference/17-toolchain.md](reference/17-toolchain.md) |

### Coding Style

| Command | Category | Description | Reference |
|---|---|---|---|
| `control-flow [code]` | Refactor | Apply `let else`, `matches!` | [reference/18-control-flow.md](reference/18-control-flow.md) |
| `iterators [code]` | Refactor | Convert to iterator chains | [reference/19-iterators.md](reference/19-iterators.md) |
| `traits [impl]` | Refactor | Idiomatic trait patterns | [reference/20-traits.md](reference/20-traits.md) |
| `errors [handling]` | Refactor | Error combinators | [reference/21-errors.md](reference/21-errors.md) |
| `data-struct [definition]` | Refactor | Simplify structures | [reference/22-data-struct.md](reference/22-data-struct.md) |
| `borrowing [code]` | Refactor | Optimize mutability | [reference/23-borrowing.md](reference/23-borrowing.md) |

### Evaluation

| Command | Category | Description | Reference |
|---|---|---|---|
| `trade-off [decision]` | Evaluate | Analyze trade-offs | [reference/04-trade-offs.md](reference/04-trade-offs.md) |
| `review [code]` | Evaluate | Comprehensive review | [reference/27-review.md](reference/27-review.md) |
| `refactor [code]` | Refactor | Style refactoring | [reference/24-refactor.md](reference/24-refactor.md) |

---

## 8. Quick Decision Checklist (Automated)

### Strategic (Priority Pyramid)

1. **P0 Check**: Does this violate memory safety? → **Reject immediately**
2. **P1 Check**: Is this business logic? → Prefer clarity over "clever" abstractions
3. **P2 Check**: Will this slow compilation? → Consider `Box<dyn Trait>`
4. **P3 Check**: Is this a proven bottleneck? → Profile first, optimize second

### Architecture

5. **Type Design**: Does state need compile-time guarantees? → Use Enum, else bool
6. **Data Architecture**: Global or local? → Global uses indexing, local can use smart pointers
7. **Error Handling**: Library or application? → Library uses thiserror, application uses anyhow
8. **Concurrency**: Message passing or shared state? → Prefer channels, use locks cautiously
9. **Marker Traits**: Need compile-time constraint enforcement? → Use marker traits to intercept illegal operations
10. **PhantomData**: Carrying type info without runtime data? → Use only for well-understood patterns, avoid cryptic chains
11. **Monomorphization**: Generic explosion hurting compile time? → Retreat to `Box<dyn Trait>` or runtime validation
12. **API Evolution**: Public types may evolve? → Mark `#[non_exhaustive]`, use sealed traits
13. **Workspace**: Slow incremental compilation? → Split into independent crates with Cargo Workspace
14. **Feature Flags**: Heavy optional dependencies? → Use `[features]` to isolate serde/tokio etc.
15. **Supply Chain**: Dependency vulnerabilities or license issues? → Run `cargo deny check`

### Style

16. **Control Flow**: Can `let else` flatten nesting? → Use it
17. **Iterators**: Manual `for` + `push`? → Convert to iterator chains
18. **Traits**: Implemented `From` not `Into`? → Always implement `From`
19. **Borrowing**: Can `mut` scope be reduced? → Use shadowing
20. **Interior Mutability**: Need mutation behind `&self`? → `Cell` for Copy, `RefCell` for single-threaded, `Mutex` for multi-threaded
21. **Split Borrowing**: Borrowing disjoint fields? → Prefer field-level borrows over struct-level

### API & Evolution

22. **API Evolution**: Public types may evolve? → Mark `#[non_exhaustive]`, use sealed traits
23. **Deprecation**: Removing public API? → `#[deprecated]` with migration note, remove only in major version
24. **Object Safety**: Need `dyn Trait`? → Design trait for object safety (no `Self` return, no generics)
25. **Workspace**: Slow incremental compilation? → Split into independent crates with Cargo Workspace
26. **Feature Flags**: Heavy optional dependencies? → Use `[features]` to isolate serde/tokio etc.
27. **Supply Chain**: Dependency vulnerabilities or license issues? → Run `cargo deny check`

### Concurrency (Extended)

28. **RwLock vs Mutex**: Read-heavy access? → `RwLock`; balanced or write-heavy? → `Mutex`
29. **parking_lot**: Production system? → Prefer `parking_lot` (smaller, faster, no poisoning)
30. **Deadlock**: Multiple locks needed? → Prefer channels; if locks, always same acquisition order
31. **Synchronization**: Need one-time init? → `OnceLock`; need barrier? → `Barrier`

### Performance (P3 - Only After Profiling)

32. **Memory**: Using appropriate allocator? Pre-allocated collections?
33. **Cache**: Data layout cache-friendly (SoA)? Avoided pointer chasing?
34. **Concurrency**: Lock-free where needed? Sharded locks for hot paths?
35. **Unsafe**: Profiling-justified? SAFETY comments present?
36. **Small Collections**: Hot path with usually-small Vec? → `SmallVec` for stack allocation
37. **PGO**: Latency-critical service? → Profile-Guided Optimization for 5-15% gain

### Testing & QA (P0 Safety Verification)

38. **Property Tests**: Core algorithms have invariant tests?
39. **Fuzzing**: External parsers have fuzz targets?
40. **Loom**: Concurrent code model-checked?
41. **Miri**: Unsafe code passes UB detection?

### Metaprogramming & Const

42. **Macro Decision**: Can generics/traits/functions solve this? Use macros only as last resort
43. **Macro Hygiene**: All identifiers explicitly passed as parameters?
44. **Procedural Macro**: Isolated in `proc-macro = true` crate? Using `syn::Error::new_spanned`?
45. **Const Functions**: Pure functions with `const fn`? Using `[T; N]` over `Vec<T>`?

### FFI & Interoperability (P0 Safety at Boundaries)

46. **Crate Separation**: Two-crate design (`-sys` + wrapper)? No business logic in `-sys`?
47. **Pointer Safety**: Raw pointers converted to `&'a [u8]` at boundary?
48. **Memory Ownership**: Symmetric allocation/deallocation? Paired `create_xxx`/`destroy_xxx`?
49. **Panic Containment**: All `extern "C"` wrapped in `catch_unwind`? Error codes instead of unwinds?
50. **C++ Interop**: Need C++ bindings? → Use `cxx` for type-safe interop over raw FFI
51. **Callbacks**: C callback with Rust closure? → Trampoline + user_data pattern, never cast closures

### Async Internals & Runtime (Deep Performance)

52. **Future Size**: No large variables across `.await`? Box large objects before await?
53. **Async Recursion**: Recursive `async fn` wrapped in `Box::pin()`?
54. **Blocking Prevention**: No blocking operations in async context? Using `spawn_blocking`?
55. **Custom Poll**: Following polling rules (non-blocking → register waker → return Pending)?
56. **no_std Async**: Using Embassy or custom executor? ISR only sets wake flag?
57. **C10M Performance**: Thread-per-core (monoio/glommio) for extreme concurrency?
58. **select!/join!**: Using `select!`? → Handle `else` branch, check cancellation safety, use `biased;` for priority
59. **Cancellation**: Future dropped on cancel? → Ensure I/O handles partial writes, check cancellation safety

### Observability & Production Diagnostics (Ops)

60. **Tracing**: All `pub async fn` annotated with `#[tracing::instrument]`? Large fields skipped?
61. **Metrics**: Hot paths use atomic counters? Prometheus pull on bypass port?
62. **Distributed Context**: W3C Trace Context propagation via Headers?
63. **Panic Hook**: Global panic hook installed? Backtrace captured? Synchronous write?
64. **Core Dump**: `ulimit -c unlimited` set? `core_pattern` configured?

### Toolchain & CI

65. **rustfmt**: `rustfmt.toml` committed? `cargo fmt -- --check` in CI?
66. **CI Speed**: CI under 10 minutes? Heavy checks (miri, bench) on merge only?

---

## 9. Typical Conflict Scenarios

### Conflict 1: Performance vs Elegance

**Scenario**: String processing — complex lifetimes `&'a str` vs simple `String`?

**Resolution**: 80/20 rule based on data flow

| Path Type | Priority | Strategy |
|-----------|---------|----------|
| Cold path (config, validation) | P1 > P3 | Use `String`, clone freely |
| Hot path (protocol parsing) | P3 > P1 | Use lifetimes or `bytes::Bytes` |

### Conflict 2: Macro Magic vs Compile Time

**Scenario**: Powerful procedural macro reduces code but build time: 1min → 5min.

**Resolution**: P2 > P1 (superficial cleanliness)

**Verdict**: Code reduction at 5x compile time is a **failed abstraction**.

### Conflict 3: FFI vs Type Safety

**Scenario**: Must call C library with raw pointers.

**Resolution**: Establish "Safety Isolation Zone" with P0 protection

**Rules**:
- **Never** expose C structs with raw pointers to business layer
- Build **thin Safe Wrapper** between C and Rust
- Internal `unsafe`, but public API must be 100% safe

### Conflict 4: Strong Typing vs Development Speed

**Scenario**: State machine requires 10 structs, but MVP ships tomorrow.

**Resolution**: Progressive Architecture

```rust
// ✅ MVP: Single struct with enum + Options
struct Order {
    status: OrderStatus,
    paid_at: Option<DateTime>,
    shipped_at: Option<DateTime>,
}

// ✅ Production: Type-driven state machine
enum OrderState {
    Created(OrderInfo),
    Paid(PaymentInfo),
    Shipped(TrackingInfo),
}
```

---

## 10. Reference Files

All detailed guidelines are in the [reference/](reference/) directory, organized by decision flow order:

### Execution & Strategy (7)
- [00-mode-guide.md](reference/00-mode-guide.md) — Execution modes (rapid/standard/strict) and transition checklists
- [01-priority-pyramid.md](reference/01-priority-pyramid.md) — The four-level hierarchy
- [02-conflict-resolution.md](reference/02-conflict-resolution.md) — Typical conflicts and resolutions
- [03-progressive-architecture.md](reference/03-progressive-architecture.md) — MVP to production migration
- [04-trade-offs.md](reference/04-trade-offs.md) — Decision analysis framework
- [05-glossary.md](reference/05-glossary.md) — Centralized terminology definitions
- [06-deviation-process.md](reference/06-deviation-process.md) — Formal rule exception handling (DEVIATION annotation)

### Architecture Patterns (11)
- [07-state-machine.md](reference/07-state-machine.md) — When and how to use state machines
- [08-newtype.md](reference/08-newtype.md) — Type-safe IDs and credentials
- [09-data-architecture.md](reference/09-data-architecture.md) — Ownership, cloning, memory layout
- [10-error-handling.md](reference/10-error-handling.md) — Library vs application error strategies
- [11-concurrency.md](reference/11-concurrency.md) — Message passing, channels, locking
- [12-async-internals.md](reference/12-async-internals.md) — Async internals, Pin/Unpin, custom executors
- [13-api-design.md](reference/13-api-design.md) — Public API boundaries, `#[non_exhaustive]`, sealed traits
- [14-metaprogramming.md](reference/14-metaprogramming.md) — Declarative/procedural macros, const generics
- [15-ffi-interop.md](reference/15-ffi-interop.md) — FFI boundaries, C interoperability, panic containment
- [16-observability.md](reference/16-observability.md) — Tracing, metrics, panic hooks, coredumps
- [17-toolchain.md](reference/17-toolchain.md) — CI, Clippy, unsafe guidelines, workspace, feature flags, cargo deny

### Coding Style (7)
- [18-control-flow.md](reference/18-control-flow.md) — `let else`, `matches!`, pattern matching
- [19-iterators.md](reference/19-iterators.md) — Iterator chains, `filter_map`, functional style
- [20-traits.md](reference/20-traits.md) — `From` vs `Into`, `Default`, idiomatic traits, zero-cost abstraction boundaries (marker traits, PhantomData, monomorphization)
- [21-errors.md](reference/21-errors.md) — `unwrap_or_else`, `map_err`, combinators
- [22-data-struct.md](reference/22-data-struct.md) — Field shorthand, avoiding stuttering
- [23-borrowing.md](reference/23-borrowing.md) — Mutability scoping, slices
- [24-refactor.md](reference/24-refactor.md) — Comprehensive refactoring workflow

### Performance Deep Dive (1)
- [25-performance-tuning.md](reference/25-performance-tuning.md) — Complete performance tuning guide
  - Memory & Allocator Strategy (jemalloc, mimalloc, Arena)
  - Cache Locality & Data Layout (SoA, alignment, false sharing)
  - Lock-Free Concurrency (sharded locks, atomics, TLS)
  - Compile-Time Magic (const generics, LTO, codegen-units)
  - Unsafe Interventions (bounds check elimination, SIMD, MaybeUninit)

### Testing & Quality Assurance (1)
- [26-advanced-testing.md](reference/26-advanced-testing.md) — Advanced testing & QA strategies
  - Property-based Testing (proptest, invariants, shrinking)
  - Fuzzing (cargo fuzz, coverage-guided, crash detection)
  - Concurrency Model Checking (loom, state explosion control)
  - Undefined Behavior Detection (Miri, UB catches, FFI mocking)

### Review & Examples (2)
- [27-review.md](reference/27-review.md) — Comprehensive review checklist
- [28-usage-examples.md](reference/28-usage-examples.md) — Real-world scenarios

---

## 11. Constitutional Summary

**No rule is absolute.** The ultimate measure of code quality is whether it matches:
- Its lifecycle stage (MVP vs production) — governed by [execution mode](reference/00-mode-guide.md)
- Business constraints (deadline vs long-term maintenance)
- Team context (solo vs large organization)

**The Pragmatic Principle**: 
> Pursue excellence at system boundaries and hot paths; release mental load for internal flows and cold paths.

**Deviation Protocol**: When breaking a rule is the correct decision, annotate with `// DEVIATION: reason` and record in the Decision Summary. See [deviation-process.md](reference/06-deviation-process.md) for the full protocol.

**Terminology**: All specialized terms used in this guide are defined in [glossary.md](reference/05-glossary.md). Consult it before inferring meaning.

**Remember**: This guide is the Agent's **constitutional foundation**. Before each response, generate a **Decision Summary** (see Mandatory Output Contract) recording rationale for choosing specific implementation paths, ensuring consistency and professional depth.
