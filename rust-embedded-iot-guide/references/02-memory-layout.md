# 02 — Memory Layout & Linker Script Control

**Status**: Mandatory | **Parent**: rust-architecture-guide P0 Safety

## Why Linker Scripts Matter

In bare-metal embedded, there is no operating system to load your program or initialize memory. The linker script is the contract between your Rust code and the hardware — it defines where `.text` lives in FLASH, where `.bss` and `.data` live in RAM, and where the stack begins. A single mistake in `memory.x` causes silent corruption that no Rust safety check can prevent.

## `memory.x`: The Hardware Contract

Every Cortex-M project starts with a `memory.x` file that declares the physical memory layout. This is consumed by `cortex-m-rt` at link time.

```ld
/* memory.x — STM32F411CEU6 (Black Pill) */
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 512K
    RAM   : ORIGIN = 0x20000000, LENGTH = 128K
}

_stack_start = ORIGIN(RAM) + LENGTH(RAM);
```

Key fields:
- `FLASH` ORIGIN: Start of on-chip flash. 0x08000000 is the standard STM32 boot address.
- `FLASH` LENGTH: Total flash capacity. Exceeding this causes a linker error — which is far better than a silent runtime failure.
- `RAM` ORIGIN: Start of SRAM. 0x20000000 is the Cortex-M default.
- `RAM` LENGTH: Total SRAM. 128KB is typical for mid-range STM32F4.
- `_stack_start`: Stack grows downward from the top of RAM. `cortex-m-rt` initializes SP (MSP) from this symbol on reset.

**Red Line**: `_stack_start` must be set to `ORIGIN(RAM) + LENGTH(RAM)`, not a hardcoded address. Board variants change RAM sizes; hardcoding silently corrupts adjacent memory.

## `.data` Copy and `.bss` Zero Init

`cortex-m-rt` provides the runtime initialization in assembly. On reset, before `main()` (i.e. your `#[entry]` function):

1. Copy `.data` section from FLASH (load address) to RAM (virtual address).
2. Zero-fill the `.bss` section in RAM.
3. Call `fn main()`.

The linker script defines boundary symbols that `cortex-m-rt` uses:

```ld
/* Defined by cortex-m-rt in its internal linker script, */
/* combined with your memory.x. You do not write these: */
 _sidata = LOADADDR(.data);   /* FLASH address of .data LMA */
 _sdata  = ADDR(.data);       /* RAM address of .data VMA   */
 _edata  = ADDR(.data) + SIZEOF(.data);
 _sbss   = ADDR(.bss);
 _ebss   = ADDR(.bss) + SIZEOF(.bss);
```

Reason this matters: Global `static` variables with initializers go into `.data`. If `.data` is not copied, your `static FOO: u32 = 42` will contain garbage at startup. Global `static mut` without initializers go into `.bss` and must be zeroed.

## Stack Overflow Protection

### flip-link

`flip-link` is a linker wrapper that places the stack *below* all `.bss`/`.data` allocations. On stack overflow, the stack pointer collides with `.bss` — this triggers a HardFault immediately instead of silently corrupting data.

```toml
# .cargo/config.toml
[target.thumbv7em-none-eabihf]
runner = "arm-none-eabi-gdb"
rustflags = [
    "-C", "link-arg=-Tflip-link.ld",
    "-C", "linker=flip-link",
]
```

```bash
cargo install flip-link
```

Without flip-link: stack overflow silently overwrites `.bss`/`.data` — your device behaves erratically for hours before crashing. With flip-link: instant HardFault on overflow — deterministic failure you can debug.

### MPU Stack Guard

On chips with a Memory Protection Unit (Cortex-M3/M4/M7/M33), you can configure a no-access region below the stack:

```rust
use cortex_m::peripheral::MPU;

fn configure_stack_guard(mpu: &mut MPU, stack_bottom: u32, guard_size: u32) {
    const REGION_STACK_GUARD: u32 = 0;
    mpu.rnr.write(REGION_STACK_GUARD);
    mpu.rbar.write(stack_bottom);
    mpu.rasr.write(
        (1 << 0)  // ENABLE
        | (guard_size << 1) // SIZE (log2)
        | (0x00 << 24)      // No access (XN + no read/write)
    );
}
```

When the stack grows into the guard region, the MPU triggers a MemManage fault immediately — before any data corruption occurs. This is the strongest form of stack overflow protection.

## `cargo-call-stack`: Static Stack Audit

`cargo-call-stack` performs static analysis of the entire call graph to compute worst-case stack usage per function:

```bash
cargo install cargo-call-stack
cargo call-stack --bin app > stack-report.txt
```

Example output:

```
main: 256 bytes
├── init_peripherals: 128 bytes
├── run_sensor_loop: 512 bytes     ← potential problem
│   ├── spi_read: 96 bytes
│   ├── process_data: 320 bytes
│   └── transmit_packet: 64 bytes
└── idle_loop: 16 bytes
```

If `run_sensor_loop` alone uses 512 bytes of stack, and you have only 2KB stack budget, you must either reduce the frame size or split the function. Common culprits:
- Large local arrays: `let buf = [0u8; 1024]` — move to `static` or heap.
- Deep recursion — prohibited in embedded; unwrap into iteration.
- Formatting machinery with `ufmt`/`defmt` — prefer `defmt` (zero stack cost).

## `#[pre_init]`: Custom Early Initialization

`cortex-m-rt` supports a `#[pre_init]` function that runs before `.data`/`.bss` initialization. Use it for critical hardware setup that must happen before any Rust code executes:

```rust
use cortex_m_rt::pre_init;

#[pre_init]
unsafe fn before_memory_init() {
    // Enable external voltage regulator on STM32 — must happen
    // before any peripheral access or clock configuration
    const PWR_CR: *mut u32 = 0x4000_7000 as *mut u32;
    core::ptr::write_volatile(PWR_CR, 0x0000_C000);

    // Configure Flash latency for HSI speed before any code
    // in .text runs from Flash at the target frequency
    const FLASH_ACR: *mut u32 = 0x4002_3C00 as *mut u32;
    core::ptr::write_volatile(FLASH_ACR, 0x0000_0702);
}
```

`#[pre_init]` runs in assembly context — no stack setup, no `.bss` zeroing has occurred. You cannot use any static variables here. All operations must be register-level MMIO writes.

## Objdump Memory Usage Audit

Before every production release, dump the section sizes:

```bash
cargo objdump --release -- -h
```

Example output:

```
Sections:
Idx Name          Size      VMA       LMA
  0 .vector_table 00000400  08000000  08000000
  1 .text         0000a3f0  08000400  08000400
  2 .rodata       00001234  0800a7f0  0800a7f0
  3 .data         00000080  20000000  0800ba24
  4 .bss          00002000  20000080  00000000
  5 .uninit       00000000  20002080  00000000
```

Interpretation:
- `.text`: 41.9KB of code — within the 512KB budget, healthy.
- `.rodata`: 4.6KB of constants — fine.
- `.data`: 128 bytes of initialized globals — tiny, good.
- `.bss`: 8KB of zero-initialized data — check why this is large. Look for oversized static buffers.
- `.uninit`: 0 bytes — no uninitialized memory declared, fine.

**Red Line**: Release builds must be audited with `cargo-objdump -h` before deployment. Never ship firmware without understanding every section's size.

## Red Lines

1. `memory.x` MUST use symbolic expressions (`ORIGIN(RAM) + LENGTH(RAM)`), not hardcoded addresses
2. `_stack_start` MUST be at the top of RAM; misconfiguration causes silent corruption
3. `flip-link` or MPU stack guard is MANDATORY for production firmware
4. `cargo-call-stack` MUST be run on every release binary to verify stack budget
5. `.bss` size MUST be audited — large `.bss` usually means oversized static buffers
6. `#[pre_init]` MUST NOT access any Rust static variables — no `.bss`/`.data` is available yet
7. FLASH/RAM budget must be validated: `.text + .rodata + .data(LMA) < FLASH(LENGTH)`, `.data(VMA) + .bss + stack < RAM(LENGTH)`

## References

- [cortex-m-rt — Startup & Linker Script](https://docs.rs/cortex-m-rt/latest/cortex_m_rt/)
- [flip-link — Stack Overflow Protection](https://github.com/knurling-rs/flip-link)
- [cargo-call-stack — Static Stack Analysis](https://github.com/stack-overflow-rs/cargo-call-stack)
- [The Embedonomicon — Memory Layout](https://docs.rust-embedded.org/embedonomicon/memory-layout.html)
- [Cortex-M MPU Programming](https://developer.arm.com/documentation/dui0646/b/Cortex-M7-Peripherals/Optional-Memory-Protection-Unit)
- [cargo-binutils — objdump/nm/size](https://github.com/rust-embedded/cargo-binutils)