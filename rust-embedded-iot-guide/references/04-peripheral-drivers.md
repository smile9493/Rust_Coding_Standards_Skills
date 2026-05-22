# 04 — Peripheral Driver Development

**Status**: Mandatory | **Parent**: rust-architecture-guide P1 Maintainability

## The `embedded-hal` Contract

`embedded-hal` defines standard traits for embedded peripherals. Any HAL crate that implements these traits is swappable at the application level — you can change an STM32 for an nRF52 without rewriting driver logic.

```rust
// Driver function accepts any SPI bus — NOT a specific HAL type
fn read_temperature_sensor<Spi: embedded_hal::spi::SpiBus<u8>>(
    spi: &mut Spi,
    cs: &mut impl OutputPin,
) -> Result<u16, Spi::Error> {
    let mut buf = [0u8; 2];
    cs.set_low();
    spi.read(&mut buf)?;
    cs.set_high();
    Ok(u16::from_be_bytes(buf))
}
```

Key traits in `embedded-hal` v1.0:

| Trait | Purpose | Methods |
|-------|---------|---------|
| `InputPin` | Digital input | `is_high()`, `is_low()` |
| `OutputPin` | Digital output | `set_high()`, `set_low()`, `set_state()` |
| `StatefulOutputPin` | Queryable output | `is_set_high()`, `is_set_low()` |
| `SpiBus<Word>` | SPI bus (no CS) | `read()`, `write()`, `transfer()`, `transfer_in_place()` |
| `SpiDevice<Word>` | SPI device (with CS) | `transaction()` |
| `I2c<A>` | I2C bus | `read()`, `write()`, `write_read()`, `transaction()` |
| `SerialWrite<Word>` | UART TX | `write()` (non-blocking) |
| `DelayNs` | Microsecond delays | `delay_ns()` |

## GPIO: Typed State Machines

GPIO pins are modeled as type-state machines. The HAL consumes the PAC's raw pin and returns a typed pin whose API changes based on its state:

```rust
use stm32f4xx_hal::gpio::*;
use stm32f4xx_hal::pac::Peripherals;

let dp = Peripherals::take().unwrap();
let gpioa = dp.GPIOA.split();

// Type state transitions — each call returns a new type
let pa0: PA0<Input<Floating>> = gpioa.pa0.into_floating_input();
let pa0: PA0<Input<PullUp>>   = pa0.into_pull_up_input();
let pa0: PA0<Output<PushPull>> = pa0.into_push_pull_output();
let pa0: PA0<Alternate<AF7>> = pa0.into_alternate();

// Input APIs available ONLY on input-typed pins
if pa0.is_high() { /* ... */ } // Compiles in Input state
// pa0.set_high(); // COMPILE ERROR: Input pins have no set_high()

// After transition to Output, output APIs become available
let mut pa0 = gpioa.pa0.into_push_pull_output();
pa0.set_high(); // Compiles in Output state
// pa0.is_high(); // COMPILE ERROR: Output pin property via StatefulOutputPin
```

Pin mode variants and their electrical behavior:

| Mode | Pull resistor | Direction | Use case |
|------|--------------|-----------|----------|
| `Input<Floating>` | None | Input only | External pull-up/down present |
| `Input<PullUp>` | Internal pull-up | Input only | Button (active low), I2C SDA/SCL |
| `Input<PullDown>` | Internal pull-down | Input only | Button (active high) |
| `Output<PushPull>` | None (driven) | Output only | LED, logic signals |
| `Output<OpenDrain>` | None (open drain) | Output + Input | I2C, wired-OR, bus sharing |
| `Alternate<AFn>` | Pin-specific | Peripheral | SPI, UART, I2C alternate function |

**Red Line**: Never use raw `write_volatile` to configure GPIO registers from application code. The type-state machine exists to prevent electrical conflicts — bypassing it can physically damage hardware (e.g., push-pull output on an externally-driven line).

## SPI with DMA

For high-throughput SPI (sensor data, display updates), DMA offloads the CPU entirely:

```rust
use stm32f4xx_hal::{
    dma::{Stream0, Stream1, Transfer},
    spi::Spi,
};

fn dma_spi_transfer(
    spi: Spi<SPI1, (Sck<PA5>, Miso<PA6>, Mosi<PA7>)>,
    tx_dma: Stream0<DMA2>,
    rx_dma: Stream1<DMA2>,
    tx_buf: &'static mut [u8; 1024],
    rx_buf: &'static mut [u8; 1024],
) -> Transfer<...> {
    let (tx_transfer, rx_transfer) = spi
        .with_tx_dma(tx_dma)
        .with_rx_dma(rx_dma)
        .transfer(tx_buf, rx_buf);

    // Both TX and RX happen simultaneously via DMA in hardware.
    // The Transfer future completes when the DMA interrupt fires.
    tx_transfer
}
```

DMA rules:
- Buffers must be `'static` — DMA hardware accesses the bus independently of CPU; the buffer must outlive the transfer
- Double-buffering: while one buffer is being transferred, prepare the next buffer
- `mem::forget()` is wrong here — use `Transfer::wait()` or `.await` to reclaim ownership

## I2C Transactions

```rust
use embedded_hal::i2c::I2c;

fn read_register<I: I2c>(i2c: &mut I, device_addr: u8, reg: u8) -> Result<u8, I::Error> {
    let mut data = [0u8; 1];
    i2c.write_read(device_addr, &[reg], &mut data)?;
    Ok(data[0])
}

fn scan_bus<I: I2c>(i2c: &mut I) {
    for addr in 0x03..=0x77 {
        let mut dummy = [0u8; 1];
        if i2c.read(addr, &mut dummy).is_ok() {
            defmt::info!("Device found at 0x{:02X}", addr);
        }
    }
}
```

## Shared Bus: I2C Proxy/Manager

When multiple drivers need the same I2C bus, wrap it with a bus manager:

```rust
use shared_bus::BusManager;

type I2cBus = stm32f4xx_hal::i2c::I2c<I2C1, (Scl<PB6>, Sda<PB7>)>;

static I2C_BUS: once_cell::sync::OnceCell<BusManager<I2cBus>> =
    once_cell::sync::OnceCell::new();

// Initialize once
let i2c = I2c::new(p.I2C1, (scl, sda), 400.kHz(), &clocks);
let manager = BusManager::new(i2c);
I2C_BUS.set(manager).ok();

// In each driver, acquire a proxy:
let mut proxy = I2C_BUS.get().unwrap().acquire_i2c();
read_temperature(&mut proxy)?;
// Proxy is released on drop — other drivers can now use the bus
```

## ADC Calibration

```rust
use stm32f4xx_hal::adc::{Adc, config::AdcConfig, config::SampleTime};

let mut adc = Adc::adc1(p.ADC1, true, AdcConfig::default());

adc.set_sample_time(SampleTime::Cycles_480); // 480 ADC cycles for accuracy
adc.calibrate(); // Factory calibration — always call before first read

let raw: u16 = adc.read(&mut pa0).unwrap();
// Convert to voltage: (raw / 4095) * Vref
// With external 3.3V Vref, 12-bit resolution:
let voltage_mv: u32 = (raw as u32 * 3300) / 4095;
```

**Red Line**: Never skip `adc.calibrate()` on STM32 — uncalibrated ADC can have 50+ LSB offset error.

## Timer/PWM: Capture and Compare

```rust
use stm32f4xx_hal::timer::{Timer, Channel};

let mut pwm = Timer::new(p.TIM2, &clocks);

// PWM output on Channel 1: 1kHz, 50% duty cycle
let mut pwm_ch1 = pwm.pwm_with_frequency(
    p.PA5.into_alternate(),
    1.kHz(),
    &clocks,
);

pwm_ch1.set_duty(50.percent()); // 0-100%
pwm_ch1.enable();

// Input Capture: measure external pulse width
let mut capture = Timer::new(p.TIM3, &clocks);
let input_capture = capture.input_capture(
    p.PB4.into_alternate(),
    Channel::C1,
    &clocks,
);

let ticks = input_capture.wait_for_rising_edge();

// Dead-time insertion for motor control (complementary outputs)
let pwm_advanced = Timer::new(p.TIM1, &clocks);
pwm_advanced.set_dead_time(100.ns()); // 100ns dead-band between high/low side
```

## Red Lines

1. Application code MUST NOT bypass HAL to write raw registers — `write_volatile` in application is a violation
2. GPIO type-state transitions must respect electrical constraints: never set push-pull output on a line driven by external hardware
3. DMA buffers must be `'static` — stack-allocated buffers in DMA cause heap-use-after-free equivalent
4. `adc.calibrate()` is mandatory before any ADC read on STM32
5. SPI/I2C/UART drivers MUST accept `embedded-hal` traits, not concrete HAL types
6. I2C bus sharing MUST use a `BusManager`/proxy pattern — never multiple direct references to the I2C peripheral
7. PWM dead-time insertion is mandatory for half-bridge/H-bridge motor drivers — without it, shoot-through destroys MOSFETs

## References

- [embedded-hal v1.0 — Trait Definitions](https://docs.rs/embedded-hal/latest/embedded_hal/)
- [stm32f4xx-hal — GPIO Typestate](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/gpio/index.html)
- [shared-bus — I2C/SPI Bus Manager](https://crates.io/crates/shared-bus)
- [embedded-dma — DMA Abstractions](https://docs.rs/embedded-dma/latest/embedded_dma/)
- [STM32F4 Reference Manual — ADC Calibration](https://www.st.com/resource/en/reference_manual/dm00031020.pdf)