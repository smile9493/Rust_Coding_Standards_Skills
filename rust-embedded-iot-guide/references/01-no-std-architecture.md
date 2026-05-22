# 01 — no_std Architecture & Crate Layering

**Status**: Mandatory | **Parent**: rust-architecture-guide P0 Safety

## Five-Layer Model

Embedded Rust follows a strict abstraction hierarchy. Each layer must only depend on the layer below it. This is not a suggestion — it is enforced by the compiler (PAC crates are `unsafe` at the register level; HAL wraps that in safe APIs).

```
Application Logic
    │
    ▼
BSP (Board Support Package)    — Board-specific pinout + clock tree
    │
    ▼
HAL (Hardware Abstraction)     — Implements embedded-hal traits
    │
    ▼
PAC (Peripheral Access Crate)  — Auto-generated from SVD, unsafe MMIO
    │
    ▼
Micro-Architecture Crate       — Core peripherals (NVIC, SysTick, MPU)
    │
    ▼
Cortex-M / RISC-V Core         — Hardware
```

## Layer Responsibilities

### Micro-Architecture Crate
- **cortex-m**: `Peripherals::take()`, `NVIC`, `SysTick`, interrupt management. Common to ALL Cortex-M chips regardless of manufacturer (ST, NXP, TI, Nordic).
- **riscv**: `riscv::register::mstatus`, `mie`, `mtime`. RISC-V equivalent.
- This layer is vendor-agnostic. If you're writing code that uses STM32-specific register names at this level, you've crossed the boundary.

### PAC (Peripheral Access Crate)
- Auto-generated from CMSIS-SVD or ATDF files by `svd2rust`.
- Thin wrapper over memory-mapped registers: `pac::GPIOA::odr().write(|w| w.odr0().set_bit())`.
- All PAC code is `unsafe`. No ownership tracking. No exclusivity. You can configure GPIOA as input and output simultaneously — the PAC won't stop you.
- **Examples**: `stm32f4xx`, `nrf52840-pac`, `esp32c3-pac`.

### HAL (Hardware Abstraction Layer)
- Implements `embedded-hal` traits: `InputPin`, `OutputPin`, `SpiBus`, `I2c`, `SerialWrite`, `DelayNs`.
- Consumes PAC peripherals via `constrain()` or `split()`. Returns typed state machines.
- After constraint, the PAC peripheral is consumed — you cannot accidentally double-claim a pin.
- **Examples**: `stm32f4xx-hal`, `nrf52840-hal`, `embassy-stm32`, `esp-hal`.

```rust
// HAL pattern: consume PAC, return typed pins
let dp = pac::Peripherals::take().unwrap();
let rcc = dp.RCC.constrain();
let gpioa = dp.GPIOA.split();
// gpioa.pa0 is now Output<PushPull> — typed, safe, exclusive
```

### BSP (Board Support Package)
- Configures the chip for a specific board: pin assignments, oscillator frequencies, peripheral routing.
- Applications that use a BSP never touch PAC/HAL directly.
- **Examples**: `stm32f3-discovery`, `nucleo-f446re`, `raspberry-pi-pico`.

### Application
- Calls BSP or HAL APIs. Must never import PAC.
- **Red Line Violation**: `use stm32f4xx::Peripherals` in application code.

## `#![no_std]` Pragmatics

### What you lose
| std module | no_std replacement |
|------------|-------------------|
| `std::vec::Vec` | `heapless::Vec<N>` (fixed capacity) |
| `std::string::String` | `heapless::String<N>` |
| `std::collections::HashMap` | `heapless::FnvIndexMap<K, V, N>` |
| `std::fs` | Not applicable (no filesystem) |
| `std::net` | Embassy-net / smoltcp |
| `std::thread` | RTIC tasks / Embassy async |
| `std::sync::Mutex` | `cortex_m::interrupt::Mutex` / RTIC resource locks |

### What you keep
- `core::*` — `Option`, `Result`, `Iterator`, `core::fmt::Write`
- `core::sync::atomic` — `AtomicBool`, `AtomicU32`
- `core::cell::RefCell`, `core::cell::UnsafeCell`

### Optional: `extern crate alloc`
If you have sufficient RAM (> 32KB) and need dynamic allocation:
```rust
extern crate alloc;
use alloc::vec::Vec; // Now available, but requires a global allocator
```
Required: custom `#[global_allocator]` backed by a memory pool in a `.bss` static buffer.

## Red Lines
1. Application code MUST NOT import PAC crates directly
2. No `use std::*` — instant compile error with `#![no_std]`
3. Unsynchronized shared mutable `static mut` is UB — use `core::cell::UnsafeCell` with `critical_section`, RTIC `#[shared]` resource locks, `portable-atomic`, or `static_cell`/`once_cell` patterns.
4. Every embedded crate must specify `#![no_std]` at crate root

## References
- [The Embedded Rust Book — Memory Mapped Registers](https://docs.rust-embedded.org/book/start/registers.html)
- [The Embedonomicon — Creating no_std applications](https://docs.rust-embedded.org/embedonomicon/)
- [embedded-hal crate](https://crates.io/crates/embedded-hal) — trait definitions
- [heapless crate](https://crates.io/crates/heapless) — static-capacity data structures