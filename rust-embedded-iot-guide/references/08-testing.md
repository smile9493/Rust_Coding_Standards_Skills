# 08 — Testing on Hardware

**Status**: Mandatory | **Parent**: rust-architecture-guide P0 Safety

## You Cannot Test Embedded Code on x86

An embedded program reads GPIO pins, configures SPI busses, and handles DMA interrupts. None of these exist on a developer's laptop. Host-based unit tests (`cargo test --target x86_64`) can validate pure algorithm logic, but they cannot verify that your SPI driver correctly reads a temperature sensor. For that, you need a test framework that runs on the target hardware.

## `defmt-test`: Test Harness on Hardware

`defmt-test` provides a `#[test]` attribute that runs on the MCU, outputs results via RTT/defmt, and reports pass/fail. The test runner is the firmware itself:

```rust
#![no_main]
#![no_std]
#![feature(test)] // Required for defmt-test on nightly

use defmt_rtt as _;
use panic_probe as _;
use stm32f4xx_hal as _;

#[defmt_test::tests]
mod tests {
    use stm32f4xx_hal::{pac, prelude::*};

    #[init]
    fn init() -> TestPeripherals {
        let dp = pac::Peripherals::take().unwrap();
        let rcc = dp.RCC.constrain();
        let gpioa = dp.GPIOA.split();

        TestPeripherals { gpioa, rcc }
    }

    struct TestPeripherals {
        gpioa: stm32f4xx_hal::gpio::Parts,
        rcc: stm32f4xx_hal::rcc::Rcc,
    }

    #[test]
    fn gpio_output_toggles(p: &mut TestPeripherals) {
        let mut led = p.gpioa.pa5.into_push_pull_output();

        for _ in 0..10 {
            led.set_high();
            assert!(led.is_set_high().unwrap());
            led.set_low();
            assert!(!led.is_set_high().unwrap());
        }
    }

    #[test]
    fn spi_reads_sensor_id(p: &mut TestPeripherals) {
        let spi = p.gpioa.spi1(/* ... */);
        let mut cs = p.gpioa.pa4.into_push_pull_output();

        cs.set_low();
        let id = read_register(&spi, 0x0F); // WHO_AM_I register
        cs.set_high();

        assert_eq!(id, 0x60); // Expected sensor ID
    }

    #[test]
    #[should_panic]
    fn overcurrent_shutdown_triggers() {
        simulate_overcurrent();
    }
}
```

Run the tests:

```bash
cargo embed --release --features test
```

Output appears on RTT channel 0:

```
INFO  (1/3) gpio_output_toggles ... ok
INFO  (2/3) spi_reads_sensor_id ... ok
INFO  (3/3) overcurrent_shutdown_triggers ... ok
INFO  all tests passed!
```

## Hardware-in-the-Loop (HIL) CI

HIL testing automates the full cycle: flash firmware → run tests → capture RTT output → parse results. This can be integrated into CI with a physical board connected to a CI runner:

```
CI Pipeline (GitHub Actions / GitLab CI):
┌─────────────────────────────────────────────────┐
│  1. cargo build --release --tests               │
│  2. probe-rs run --chip STM32F411 target/...elf  │
│     │                                            │
│     ├── RTT Channel 0 ──► capture output       │
│     │                                            │
│  3. Parse output: search for "all tests passed"  │
│  4. Return exit code 0 (pass) or 1 (fail)       │
└─────────────────────────────────────────────────┘

Physical: USB Hub → J-Link/ST-Link → STM32F4 Board
```

Example CI script (shell):

```bash
#!/bin/bash
set -e

# Build the test firmware
cargo build --release --features test

# Flash and run, capture RTT output with timeout
timeout 30 probe-rs run \
    --chip STM32F411CEUx \
    target/thumbv7em-none-eabihf/release/firmware \
    2>&1 | tee test_output.txt

# Parse results
if grep -q "all tests passed" test_output.txt; then
    echo "HIL tests passed"
    exit 0
else
    echo "HIL tests FAILED"
    exit 1
fi
```

### Required HIL Infrastructure

| Component | Purpose | Example |
|-----------|---------|---------|
| Debug probe | Flash + RTT | J-Link EDU Mini, ST-Link/V3 |
| Target board | DUT (Device Under Test) | STM32F411 Black Pill |
| Test harness board | Stimulate inputs, measure outputs | Custom PCB or breadboard |
| Power supply + measurement | Monitor current draw during tests | USB power meter or profiler |
| CI runner | Automate all of the above | GitHub Actions self-hosted runner |

The test harness board is critical — it must:
- Provide known voltages to ADC inputs (voltage divider with precision resistors)
- Pull GPIOs to known states during input tests
- Loop back UART TX to RX for echo tests
- Measure PWM output frequency with a counter/timer

## `cargo-modules`: Audit for PAC Leakage

`cargo-modules` generates a dependency graph of your crate structure. Use it to verify that PAC crates never leak into application layers:

```bash
cargo install cargo-modules
cargo modules generate tree --lib --with-types
```

Example output:

```
crate my_firmware
├── mod app: pub(crate)
│   ├── use stm32f4xx_hal::gpio        ← OK: HAL is allowed
│   ├── use my_bsp::Board               ← OK: BSP is allowed
│   └── use stm32f4xx_hal::pac         ← VIOLATION! PAC in application
├── mod drivers: pub(crate)
│   └── use embedded_hal::spi::SpiBus   ← OK: trait only
└── mod bsp: pub(crate)
    └── use stm32f4xx_hal::pac         ← OK: BSP can use PAC
```

**Red Line**: `cargo-modules` must run as part of CI. Any path where `pac::*` appears outside the BSP/HAL layer is a merge blocker.

## QEMU Emulation for CI

For quick iteration and CI test coverage of platform-independent logic, QEMU can emulate Cortex-M CPUs:

```bash
# Install QEMU for ARM
sudo apt install qemu-system-arm

# Run firmware in QEMU (no hardware needed)
qemu-system-arm \
    -cpu cortex-m4 \
    -machine stm32vldiscovery \
    -nographic \
    -kernel target/thumbv7em-none-eabihf/release/firmware
```

QEMU limitations:
- Only models specific boards (`stm32vldiscovery`, `lm3s6965evb`, `mps2-an385`)
- GPIO/SPI/I2C/UART are not functional — only CPU and NVIC are emulated
- DMA and peripheral-specific features are absent

QEMU is useful for: testing startup sequences, interrupt vector table layout, and algorithm logic. It is NOT a substitute for HIL tests.

## Integration Tests with Power Measurement

For battery-powered devices, integration tests must include power measurements:

```rust
#[test]
fn deep_sleep_current_below_10ua() {
    enter_deep_sleep();
    // External power profiler measures current during sleep.
    // The test framework reads the profiler via USB/UART and
    // asserts that sleep current is below the threshold.

    // This test cannot assert from firmware (the CPU is asleep),
    // so the assertion is in the CI script that reads the profiler:
    //
    //   $ ppk2 measure --duration 5s --channel 1
    //   Average: 4.2µA
    //   $ [ 4.2 -lt 10 ] && echo "PASS: sleep current OK"
}
```

### Power Budgeting Test Checklist

| Test | Assertion | Tool |
|------|-----------|------|
| Deep sleep current | < 10µA average over 10s | Power Profiler Kit II |
| Wake power spike | < 15mA peak, < 5ms duration | Power Profiler Kit II |
| Tx burst (BLE/WiFi) | < 50mA average during TX | Power Profiler Kit II |
| Idle (CPU on, peripherals off) | < 3mA | USB power meter |
| Full load (all peripherals) | < 30mA | USB power meter |

## Critical Safety Paths

For medical devices, motor controllers, and safety-critical IO:

1. Every control path (motor speed, valve position, heater duty cycle) MUST have a HIL test
2. Every failure mode (overcurrent, watchdog timeout, sensor disconnection) MUST be tested with physical fault injection
3. Watchdog configuration MUST be verified in HIL by deliberately blocking the main loop and confirming the MCU resets
4. Flash integrity: readback after flash to verify bit-for-bit correctness

```rust
#[test]
fn watchdog_resets_on_deadlock() {
    // This test spawns a task that deliberately deadlocks.
    // The external test harness monitors the RST pin with a
    // logic analyzer and asserts that the watchdog fires
    // within the configured timeout window.

    // Firmware side:
    configure_watchdog(500.millis()); // 500ms timeout
    // DO NOT pet the watchdog — it will fire
    loop { cortex_m::asm::nop(); }

    // CI script side (pseudocode):
    // assert(rst_pin_high_to_low_delta < 600ms);
    // assert(rst_pin_high_to_low_delta > 500ms);
}
```

## Red Lines

1. Critical safety paths (motor control, medical, aerospace) MUST pass HIL tests on real hardware — QEMU alone is insufficient
2. `cargo-modules` audit MUST run in CI — PAC leakage into application layer is a merge blocker
3. Every SPI/I2C/UART driver MUST have at least one HIL test that reads a known register from a real device
4. Watchdog configuration MUST be verified by deliberate deadlock + external measurement
5. Power consumption assertions MUST use hardware measurement instruments, never firmware self-estimation
6. `defmt-test` output format must be machine-parseable — CI must not rely on human interpretation of logs
7. If a chip has an FPU, the test firmware MUST include at least one floating-point computation to catch FPU initialization bugs

## References

- [defmt-test — Hardware Test Harness](https://docs.rs/defmt-test/latest/defmt_test/)
- [probe-rs — CLI runner for CI](https://probe.rs/docs/tools/probe-rs/)
- [cargo-modules — Module Graph Analysis](https://crates.io/crates/cargo-modules)
- [QEMU — ARM System Emulator](https://www.qemu.org/docs/master/system/arm/stm32.html)
- [Nordic Power Profiler Kit II](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2)
- [The Embedonomicon — Testing](https://docs.rust-embedded.org/embedonomicon/testing.html)