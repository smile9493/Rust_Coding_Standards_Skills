# 05 — Power Management & Battery Life

**Status**: Mandatory | **Parent**: rust-architecture-guide P0 Safety

## Every µA Matters

Battery-powered IoT devices live or die by their sleep current. A device drawing 10mA continuously on a 2000mAh battery lasts 200 hours (8 days). The same device drawing 5µA in deep sleep with 10ms wake per second averages ~50µA — lasting over 4 years. This is not optimization; it's the difference between a viable product and e-waste.

## Sleep State Machine

Embedded MCUs offer tiered sleep states. Each deeper state saves more power but loses more wake sources:

| State | Cortex-M | Wake Sources | Exit time | Typical Current (STM32F4) |
|-------|----------|--------------|-----------|---------------------------|
| Run | Active | N/A | N/A | 10mA @ 48MHz |
| Sleep | `wfi()` | Any interrupt | 0 cycles | 3mA |
| Deep Sleep | SLEEPDEEP + `wfi()` | RTC, ext interrupt, WDT | ~10µs | 50µA |
| Standby | PDDS + `wfi()` | RTC, WKUP pin, reset | ~100µs | 2µA |
| Shutdown | PWR_CR | External reset only | ~1ms | 0.5µA |

```rust
use stm32f4xx_hal::pac::PWR;
use stm32f4xx_hal::pac::SCB;
use cortex_m::asm;

fn enter_deep_sleep(pwr: &PWR) {
    cortex_m::interrupt::free(|_| {
        // 1. Set SLEEPDEEP bit in SCB (System Control Block)
        let scb = unsafe { &*SCB::ptr() };
        scb.scr.modify(|_, w| w.sleepdeep().set_bit());

        // 2. Enable power-down in deep sleep (regulator in low-power mode)
        pwr.cr.modify(|_, w| w.pdds().clear_bit().lpds().set_bit());

        // 3. Execute WFI — CPU halts until an enabled interrupt fires
        asm::wfi();

        // 4. CPU resumes here after wake — clear SLEEPDEEP
        scb.scr.modify(|_, w| w.sleepdeep().clear_bit());
    });
}

fn enter_standby(pwr: &PWR) -> ! {
    unsafe {
        // Clear wakeup flag
        pwr.cr.modify(|_, w| w.cwuf().set_bit());

        // Enter standby mode — only RTC and WKUP pin can wake
        pwr.cr.modify(|_, w| w.pdds().set_bit());

        let scb = &*SCB::ptr();
        scb.scr.modify(|_, w| w.sleepdeep().set_bit());

        asm::wfi();

        // NEVER reaches here — standby is full power cycle
        // Execution restarts from reset vector
    }
    loop { asm::wfi(); }
}
```

## RTC Wakeup

The Real-Time Clock (RTC) is the most common wake source for periodic sensors:

```rust
use stm32f4xx_hal::rtc::Rtc;

fn configure_rtc_wakeup(rtc: &mut Rtc) {
    rtc.set_wakeup_timer(60.secs()); // Wake every 60 seconds
    rtc.enable_wakeup_interrupt();
    // RTC runs on LSE 32.768kHz crystal — ~1µA consumption
}

#[interrupt]
fn RTC_WKUP() {
    // Clear the wakeup flag
    let rtc = unsafe { &*stm32f4xx_hal::pac::RTC::ptr() };
    rtc.isr.modify(|_, w| w.wutf().clear_bit());

    // Signal main loop to read sensor and transmit
    cortex_m::asm::sev(); // Send Event — wakes from WFE
}
```

## External Interrupt Wake

For event-driven wake (button press, sensor threshold, motion detection):

```rust
use stm32f4xx_hal::gpio::ExtiPin;

fn configure_pin_wakeup(button: &mut impl ExtiPin) {
    button.make_interrupt_source();
    button.trigger_on_edge(
        stm32f4xx_hal::exti::Edge::Rising,
    );
    button.enable_interrupt();
    // Now the device can sleep and wake on button press
}
```

## Clock Gating: Disable Unused Peripherals

Every enabled peripheral clock draws current — even if the peripheral is idle. Use the RCC (Reset and Clock Control) to disable unused clocks:

```rust
use stm32f4xx_hal::rcc::{RccExt, Clocks};

let dp = Peripherals::take().unwrap();
let rcc = dp.RCC.constrain();

// Explicitly disable peripherals you don't need
rcc.apb1.disable::<stm32f4xx_hal::pac::TIM5>();
rcc.apb2.disable::<stm32f4xx_hal::pac::TIM9>();
rcc.ahb1.disable::<stm32f4xx_hal::pac::GPIOB>();

// Disable unused GPIO ports (they still draw leakage current)
// gpioB configured as analog input = lowest power state
```

Power impact of clock gating:
- Unused USART: ~200µA
- Unused SPI: ~100µA
- Unused Timer (running): ~50µA
- Unused GPIO port (with floating inputs): ~3-5µA per pin

## Dynamic Frequency Scaling

When the CPU is awake, reduce clock speed for non-critical work:

```rust
let clocks = rcc
    .cfgr
    .use_hse(8.MHz())
    .sysclk(168.MHz()) // Full speed for computation
    .freeze();

// After computation, switch to low-power PLL
// This requires re-initializing system clocks — typically
// done through PWR_CR and RCC_CFGR register writes
```

Many STM32 models support two PLL outputs — switch between them for rapid frequency scaling without clock tree reconfiguration.

## Power Profiler: Measure, Don't Guess

Software estimation of power consumption is always wrong. You must measure:

```
Target Hardware Setup:
[ Battery / Power Supply ] → [ Current Sense Resistor 10Ω ]
      → [ Oscilloscope Probe across Resistor ]
      → [ VCC pin of MCU ]

Measure voltage drop → I = V/R
Sleep:  0.05mV across 10Ω = 5µA   ✓ Under target
Wake:   100mV across 10Ω    = 10mA  ✓ Expected
```

Tools:
- **Nordic Power Profiler Kit II**: 200nA to 1A, 100ksps, $100
- **Joulescope**: 1nA to 10A, 2Msps, more advanced
- **Oscilloscope + shunt resistor**: Simple, works everywhere

## Battery Life Budgeting

算出每一毫安的消耗：

```
Profile: Temperature Sensor, BLE advertisement every 60 seconds
Battery: CR2032 (220mAh, 3V)

Sleep current:  2µA  × 59.99s = 0.033µAh per cycle
Wake current:   8mA  × 10ms   = 0.022µAh per cycle (sensor read + BLE TX)
Total per cycle: 0.055µAh

Cycles per day: 24 × 60 = 1440
Daily consumption: 1440 × 0.055µAh = 79.2µAh

Battery life: 220mAh / 79.2µAh = 2777 days ≈ 7.6 years
(Self-discharge of CR2032 is ~1% per year, actual ~5 years)
```

If the same device used 10mA constant (no sleep): 220mAh / 10mA = 22 hours.

## Red Lines

1. Power consumption MUST be measured with hardware tools, not estimated from datasheets
2. Deep sleep target: < 10µA for battery-powered devices (excluding sensor/peripheral draw)
3. Every enabled peripheral clock costs current — disable all unused peripherals in `init()`
4. Floating inputs on GPIO cause leakage (up to 100µA per pin) — configure all unused pins as analog input
5. WFI must be called in an atomic context — otherwise the interrupt fires between the check and the sleep
6. Standby mode restarts from the reset vector — all RAM content is lost; persist critical state in RTC backup registers
7. Dynamic frequency scaling trumps overengineering: a 1% algorithm improvement is meaningless if the clock is running at 168MHz unnecessarily

## References

- [STM32F4 Power Modes](https://www.st.com/resource/en/reference_manual/dm00031020.pdf) — Section 5: Power Controller
- [cortex-m — WFI/WFE](https://docs.rs/cortex-m/latest/cortex_m/asm/index.html)
- [Nordic Power Profiler Kit II](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2)
- [stm32f4xx-hal RTC](https://docs.rs/stm32f4xx-hal/latest/stm32f4xx_hal/rtc/index.html)
- [The Embedonomicon — Power Management](https://docs.rust-embedded.org/embedonomicon/power-management.html)