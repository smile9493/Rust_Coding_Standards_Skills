# 06 — Hardware Debugging & Observability

**Status**: Mandatory | **Parent**: rust-architecture-guide P1 Maintainability

## There Is No `println!` on Bare Metal

Embedded systems have no console, no `stdout`, no operating system to write to. Debugging requires hardware-adjacent tools: debug probes, real-time trace, and binary-efficient logging. The tools described here form a complete debugging toolkit that works without an OS.

## probe-rs: Flash, Debug, and Run

`probe-rs` is the Rust-native debug probe driver. It speaks CMSIS-DAP, J-Link, ST-Link, and other protocols — no vendor tools required.

### cargo-flash

```bash
cargo install cargo-flash
cargo flash --chip STM32F411CEUx --release
```

Uploads firmware to the MCU via the debug probe. Faster than vendor tools because it skips the IDE.

### cargo-embed

`cargo-embed` combines flashing, RTT output, and GDB stub into a single command:

```toml
# Embed.toml — configuration in project root
[default.general]
chip = "STM32F411CEUx"

[default.rtt]
enabled = true

[default.gdb]
enabled = true
```

```bash
cargo embed --release
```

Now you have:
- Firmware flashed and running
- RTT channels streaming log output to your terminal
- GDB stub on `:1337` for interactive debugging

## defmt: Binary-Efficient Logging

`defmt` is the standard logging framework for embedded Rust. Unlike `log` or `println!`, `defmt` transmits *indexed format strings* — the format string itself never leaves the MCU. Only argument data is sent over the wire.

```rust
use defmt::{info, error, warn, debug, trace};

#[defmt::timestamp]
fn timestamp() -> u64 {
    // Timestamp from hardware timer — displayed in terminal
    timer_now()
}

fn sensor_loop() {
    let temp = read_temperature();

    info!("Temperature: {}°C", temp); // Transmits: (format_index, temp_value)
    // STM32: ~14 bytes on the wire vs ~80 bytes for a full string

    if temp > 85 {
        error!("Over-temperature: {}°C at {:?}", temp, timer_now());
    }
}
```

How `defmt` works:

```
[MCU]                           [Host (probe-rs)]
  │                                  │
  │  (1) Format strings interned     │
  │      in ELF .defmt section       │
  │                                  │
  │  (2) RTT: send u8 index + args   │
  │─────────────────────────────────>│
  │                                  │  (3) Host looks up format string
  │                                  │      from ELF, reconstructs message
```

**Red Line**: `defmt` output is only available when the debug probe is connected. For standalone devices, use a UART/Segger RTT fallback — but accept that UART uses more power and more flash.

## RTT: Real-Time Transfer

RTT (Real-Time Transfer) is the transport layer. It uses a ring buffer in RAM that the debug probe reads via SWD/JTAG without stopping the CPU:

```
RAM: 0x2000_0000
┌─────────────────────────────────────┐
│  RTT Control Block                  │
│  ├── Channel 0 (Terminal/Up)        │ ← defmt output goes here
│  ├── Channel 1 (Up)                 │
│  └── Channel 2 (Down — host→MCU)    │ ← commands from host
└─────────────────────────────────────┘
```

```rust
use rtt_target::{rprintln, rtt_init_print};

fn init_debug() {
    rtt_init_print!(); // No need for defmt — simpler, but string-based
    rprintln!("Boot complete: firmware v{}", env!("CARGO_PKG_VERSION"));
}
```

`rtt_target` is simpler than `defmt` but transmits full strings — more bandwidth, but easier to use for quick debugging.

## GDB + OpenOCD

When probe-rs GDB stub isn't enough, the classic approach:

```bash
# Terminal 1: Start OpenOCD
openocd -f interface/stlink-v2.cfg -f target/stm32f4x.cfg

# Terminal 2: Connect GDB
arm-none-eabi-gdb target/thumbv7em-none-eabihf/release/firmware \
    -ex "target extended-remote :3333" \
    -ex "monitor reset halt" \
    -ex "load"
```

GDB commands in embedded context:

```
(gdb) info registers           # View all CPU registers (R0-R12, SP, LR, PC, xPSR)
(gdb) x/16xw 0x20000000        # Dump 16 words of RAM
(gdb) monitor mdw 0x40020000 4 # Read 4 peripheral registers via OpenOCD
(gdb) set $pc = 0x08000400     # Manually jump — dangerous, only for recovery
(gdb) bt                       # Backtrace (requires frame pointer or DWARF unwind info)
```

## ITM/SWO: ARM CoreSight Trace

ITM (Instrumentation Trace Macrocell) and SWO (Serial Wire Output) are ARM CoreSight components for real-time trace without interrupting the CPU. Available on Cortex-M3/M4/M7/M33.

```rust
use cortex_m::peripheral::ITM;

fn enable_itm(itm: &mut ITM) {
    // Enable stimulus port 0 — send data through SWO pin
    itm.stim[0].write(0x00);
}

fn trace_value(itm: &mut ITM, value: u32) {
    // Non-blocking write to ITM — zero CPU stall
    while !itm.stim[0].is_fifo_ready() {}
    itm.stim[0].write_u32(value);
}
```

ITM is faster than RTT (hardware FIFO, no RAM ring buffer overhead) but requires:
- SWO pin connected to debug probe
- probe-rs or SEGGER J-Link that supports SWO capture
- More complex wiring (extra pin + level shifter on some boards)

## Logic Analyzer for Protocol Debug

When SPI/I2C/UART don't work at the protocol level, a logic analyzer is the only way to see what's happening:

```
Saleae Logic 8 / DSLogic / cheap FX2LP clone:

[MCU SCLK] ────┐   ┌───┐   ┌───
               │   │   │   │
[MCU MOSI] ──┬─┘   └─┬─┘   └─┬─
             │        │        │
[MCU MISO] ──┘        └────────┘

Decode SPI → See actual bytes exchanged
```

Common protocol issues caught by logic analyzer:
- SPI: Wrong CPOL/CPHA — data sampled on wrong edge
- I2C: Missing ACK — slave address wrong or device not powered
- UART: Baud rate mismatch — 115200 vs 9600, start bit misalignment

## VS Code probe-rs Extension

The `probe-rs.probe-rs-debugger` VS Code extension provides GUI debugging:

`.vscode/launch.json`:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "probe-rs-debug",
            "request": "launch",
            "name": "Debug STM32F4",
            "chip": "STM32F411CEUx",
            "flashingConfig": {
                "flashingEnabled": true,
                "haltAfterReset": false
            }
        }
    ]
}
```

This gives breakpoints, variable inspection, and RTT console — all within VS Code, no external GDB terminal needed.

## Red Lines

1. Release builds MUST strip debug symbols — `strip = true` in `Cargo.toml` or `cargo objcopy --release -- -O binary`
2. `defmt` or RTT is mandatory for log output — never use blocking UART `println!` equivalents in production
3. ITM/SWO trace data must not be routed to a production-only pin — reconfirm pin mux after debugging
4. GDB `set $pc` is a recovery tool, not a debugging workflow — jumping the PC skips register initialization
5. Logic analyzer verification is mandatory for any new SPI/I2C/UART driver before merging
6. Debug builds must fit in flash — if debug exceeds flash budget, refactor; do not shrink release visibility to fit

## References

- [probe-rs — The Embedded Debugger](https://probe.rs/)
- [defmt — Efficient Logging for Embedded Rust](https://defmt.ferrous-systems.com/)
- [RTT — Real-Time Transfer](https://wiki.segger.com/RTT)
- [ARM CoreSight ITM](https://developer.arm.com/documentation/ddi0403/d/Debug-Architecture/ARMv7-M-Debug/Instrumentation-Trace-Macrocell)
- [probe-rs VS Code Extension](https://marketplace.visualstudio.com/items?itemName=probe-rs.probe-rs-debugger)
- [OpenOCD User Guide](https://openocd.org/doc/html/index.html)