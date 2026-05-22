---
title: "Prohibitions & Compliance Self-Check"
description: "7 hard prohibitions and 10-item compliance self-check list"
category: "Compliance"
priority: "P0"
applies_to: ["standard", "strict"]
prerequisites: ["01-iron-rules.md", "02-build-control.md", "03-ffi-boundary.md", "04-memory-lifecycle.md", "05-concurrency-events.md", "06-wasm-adaptation.md"]
dependents: []
---

# Prohibitions & Compliance Self-Check

---

## 6. Prohibition List (Hard Forbidden)

- **[F-01]** Must not use `std::thread::sleep` or hold `Mutex::lock()` across `.await` boundaries under the `wasm32-unknown-unknown` target. Short synchronous `Mutex` guards on the main thread are permitted.
- **[F-02]** Must not call `serde_json` or any serialization library on profiled hot FFI paths. Serialization on cold/init/config paths is permitted per constitution 80/20 rule.
- **[F-03]** Must not trigger new heap allocations (`Box::new` or `Vec::push` on empty/unreserved capacity) within the frame loop unless in a pre-allocated container. Cold-path allocation outside the frame loop is permitted. Pre-sized `Vec::with_capacity` and pool-based reuse are encouraged.
- **[F-04]** Must not use `println!` for production release logging. In dev/rapid mode, `println!` is acceptable for debugging. Production must use `tracing` + `console_error_panic_hook`; `log` + `console_log` is an acceptable lightweight alternative.
- **[F-05]** Must not depend on the Wasm GC proposal (reference types, etc.) until Rust toolchain support is verified and the target browsers in the project's support matrix enable it. GC is available in Chrome 119+, Firefox 120+, Safari 18.4+ (2024–2025); verify toolchain maturity before adopting.
- **[F-06]** Must not select `wee_alloc` as the allocator for new projects (no longer maintained).
- **[F-07]** Must not claim `SharedArrayBuffer` support without documented COOP/COEP configuration requirements.

---

## 7. Compliance Self-Check List

- [ ] `[profile.release]` in `Cargo.toml` includes `opt-level="z"`, `lto=true`, `codegen-units=1`, `panic="abort"`, `strip=true`.
- [ ] CI pipeline integrates `wasm-opt -Oz` step with file size regression check.
- [ ] Allocator replaced from default to `talc` or `MiniAlloc` (`wee_alloc` prohibited), volume benchmark completed.
- [ ] All cross-boundary large data paths use zero-copy view encapsulation with lifetimes explicitly documented.
- [ ] FFI interfaces: hot-path prefer `WasmSlice` or scalar parameters; facade/init/cold-path allow `JsValue` + serde per constitution [`breakwater pattern`](../rust-architecture-guide/references/31-breakwater-pattern.md).
- [ ] First line of frame loop entry is Arena `reset()`.
  - `// DEVIATION: Global frame Arena is a wasm32-specific exception per constitution 09-data-architecture.md §2.1, justified by single-threaded execution. Prefer passing &mut Arena into the frame handler when feasible.`
- [ ] Async I/O uses non-blocking single-thread executor discipline (`wasm-bindgen-futures`, `gloo`, Leptos reactive runtime, or Component Model async). No blocking calls.
- [ ] If `SharedArrayBuffer` is enabled, COOP/COEP header configuration is documented in architecture decision record.
- [ ] Initialization code calls `console_error_panic_hook::set_once()` and sets up structured observability (`tracing_wasm`, `tracing-web`, or `log` + `console_log` as fallback).
- [ ] Error handling uses `Result<T, JsValue>` pattern, no misuse of `panic!` for business logic.
