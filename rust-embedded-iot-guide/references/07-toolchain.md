# 07 — Build System & Toolchain

**Status**: Mandatory | **Parent**: rust-architecture-guide P1 Maintainability

## Reproducible Builds Start with `rust-toolchain.toml`

A `rust-toolchain.toml` at the project root pins the exact Rust version and target. Without it, the project builds differently on every developer's machine — and may silently fail to compile on the wrong nightly.

```toml
# rust-toolchain.toml
[toolchain]
channel = "stable"  # Embassy, RTIC 2, probe-rs all support stable Rust
# channel = "nightly-2026-01-15"  # Use only when a documented nightly feature is required
components = [
    "rust-src",      # Required for building core + compiler-builtins
    "rustfmt",
    "clippy",
]
targets = [
    "thumbv7em-none-eabihf",  # Cortex-M4F with FPU
    "thumbv6m-none-eabi",     # Cortex-M0+ (no FPU, baseline Thumb)
]
profile = "minimal"
```

Common target triples:

| Target | Architecture | FPU | Typical Chips |
|--------|-------------|-----|---------------|
| `thumbv6m-none-eabi` | Cortex-M0/M0+/M1 | None | STM32F0, RP2040, nRF51 |
| `thumbv7m-none-eabi` | Cortex-M3 | None | STM32F103, LPC1768 |
| `thumbv7em-none-eabi` | Cortex-M4/M7 | Soft | Kinetis K, EFM32 |
| `thumbv7em-none-eabihf` | Cortex-M4F/M7 | Hard | STM32F4, nRF52, iMX.RT |
| `thumbv8m.main-none-eabihf` | Cortex-M33/M35P | Hard | STM32L5, nRF5340 |
| `riscv32imac-unknown-none-elf` | RISC-V RV32IMAC | None | ESP32-C3, GD32VF103 |
| `riscv32imc-unknown-none-elf` | RISC-V RV32IMC | None | CH32V003 |

**Red Line**: Every embedded project must have `rust-toolchain.toml`. Default to stable Rust (Embassy, RTIC 2, probe-rs, and most HAL crates support stable). Pin to a specific date-stamped nightly only when a documented nightly-only feature is required (e.g., certain test harnesses or unstable compiler intrinsics). Never use a floating `nightly` channel.

## `cargo-generate`: Project Templates

Scaffold a new embedded project in one command:

```bash
cargo install cargo-generate

# Cortex-M quickstart (official template)
cargo generate \
    --git https://github.com/rust-embedded/cortex-m-quickstart \
    --name my-sensor-firmware

# Embassy template
cargo generate \
    --git https://github.com/embassy-rs/embassy \
    --name my-ble-device \
    examples/stm32f4
```

Generated projects include:
- Correct `memory.x` for the target chip
- `.cargo/config.toml` with runner and target configuration
- Pre-configured `Cargo.toml` with appropriate dependencies
- `build.rs` with linker script generation

## `.cargo/config.toml`: Target-Specific Build Settings

```toml
# .cargo/config.toml
[build]
target = "thumbv7em-none-eabihf"

[target.thumbv7em-none-eabihf]
runner = "arm-none-eabi-gdb"  # Optional: start GDB after build
rustflags = [
    "-C", "link-arg=-Tlink.x",       # Use cortex-m-rt linker script
    "-C", "link-arg=-Tdefmt.x",      # defmt linker section (if using defmt)
    "-C", "linker=flip-link",        # Stack overflow protection
    "-C", "link-arg=-Tflip-link.ld",
]
```

## `cargo-flash` and `cargo-embed`

Consistent flashing is critical — every team member must flash the same binary the same way:

```bash
# Flash only — quick iteration cycle
cargo flash --chip STM32F411CEUx --release

# Flash + RTT console
cargo embed --release
```

Both tools read `Embed.toml` for chip-specific configuration. The `Embed.toml` file should be checked into version control:

```toml
# Embed.toml
[default.general]
chip = "STM32F411CEUx"
connect_under_reset = false

[default.flashing]
enabled = true
restore_unwritten_bytes = false

[default.reset]
enabled = true
halt_afterwards = false

[default.rtt]
enabled = true
channels = [
    { up = 0, format = "defmt" },
]
```

## `objcopy` for Binary Size

After compilation, the ELF file contains debug information, symbol tables, and metadata. `objcopy` strips it to a raw binary that the bootloader writes to flash:

```bash
# Strip to raw binary — measure actual flash usage
cargo objcopy --release -- -O binary firmware.bin
ls -lh firmware.bin  # This is your actual flash footprint

# Generate Intel HEX — for bootloaders that use HEX format
cargo objcopy --release -- -O ihex firmware.hex
```

## `cargo-binutils`: Size, nm, objdump

`cargo-binutils` wraps LLVM binutils with Rust-awareness:

```bash
cargo install cargo-binutils

# Section sizes — essential before every release
cargo size --release
# Output:
#    text    data     bss     dec     hex filename
#   45124    1024   16384   62532    f444 firmware

# List symbols sorted by size — find bloat
cargo nm --release -- --size-sort
# Output:
# 00000d20 T main
# 00000480 T embassy_executor::run::...
# 00000240 T my_crate::process_sensor_data  ← suspicious

# Disassemble a specific function
cargo objdump --release -- --disassemble=my_crate::process_sensor_data
```

**Red Line**: `cargo size --release` must be run and its output committed as a comment in the release PR. Every unexpected size increase must be explained.

## `link_section`: Custom Memory Placement

When you need specific data at specific addresses (boot magic numbers, vector table offsets, calibration data), use `link_section`:

```rust
#[link_section = ".boot_magic"]
#[used]
static BOOT_MAGIC: [u8; 8] = *b"BOOTLOAD";

#[link_section = ".factory_cal"]
#[no_mangle]
static FACTORY_CAL_DATA: [u16; 4] = [0; 4];

#[link_section = ".vector_table.config"]
#[used]
static VTABLE_CONFIG: [u32; 1] = [0xB007_10AD]; // Bootloader magic
```

Corresponding linker script additions:

```ld
.boot_magic 0x08007FF8 (NOLOAD) : {
    KEEP(*(.boot_magic))
} > FLASH

.factory_cal 0x0800FC00 (NOLOAD) : {
    KEEP(*(.factory_cal))
} > FLASH
```

`#[used]` tells the linker to keep the symbol even though Rust can't see a direct reference to it. Without this, LTO (Link-Time Optimization) strips it.

## Full Project `Cargo.toml` Profile Configuration

```toml
[profile.release]
opt-level = "z"          # Optimize for size (not "s" — "z" is more aggressive)
lto = true               # Link-Time Optimization across all crates
codegen-units = 1        # Single codegen unit for maximum optimization
strip = true             # Strip symbols from final binary
debug = false            # No debug info in release binary
panic = "abort"          # No unwinding support — saves flash

[profile.dev]
opt-level = 0            # Fast compile, no optimization
debug = true             # Full debug info for GDB/probe-rs
lto = false              # No LTO — faster iteration
panic = "abort"          # Still no unwinding (not supported on embedded)
```

**Explanation of `opt-level = "z"` vs `"s"`**: `"z"` applies the same code size optimizations as `"s"` but additionally disables loop vectorization — vector loops expand to SIMD instructions that can increase code size on small MCU cores.

## Red Lines

1. `rust-toolchain.toml` is MANDATORY — default to stable; pin to date-stamped nightly only when a documented feature requires it
2. `.cargo/config.toml` must specify `target` and `runner` — "it works on my machine" is not acceptable
3. `cargo size --release` output must be reviewed before every release — unexpected growth is a merge blocker
4. Release profile must set `opt-level = "z"`, `lto = true`, `codegen-units = 1`, `strip = true`
5. Never `cargo install` toolchain components globally — all tool dependencies must be in `rust-toolchain.toml` components
6. `link_section` addresses must be verified against actual flash layout — incorrect addresses cause hard faults
7. `Embed.toml` must be version-controlled — different team members must flash identically

## References

- [rustup — Toolchain Management](https://rust-lang.github.io/rustup/overrides.html)
- [cargo-generate — Template Scaffolding](https://cargo-generate.github.io/cargo-generate/)
- [probe-rs — cargo-flash / cargo-embed](https://probe.rs/docs/tools/cargo-flash/)
- [cargo-binutils — LLVM binutils for Rust](https://github.com/rust-embedded/cargo-binutils)
- [The Embedonomicon — Linker Scripts](https://docs.rust-embedded.org/embedonomicon/memory-layout.html)
- [cortex-m-quickstart — Official Template](https://github.com/rust-embedded/cortex-m-quickstart)