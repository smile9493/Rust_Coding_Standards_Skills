# 03 — Interrupt-Driven Concurrency

**Status**: Mandatory | **Parent**: rust-architecture-guide P0 Safety

## The Embedded Concurrency Model

There are no threads. No `std::sync`. No `Mutex` from the standard library. Embedded concurrency is interrupt-driven: the hardware preempts your code, and you must share data between interrupt handlers and the main execution context without data races. Two Rust frameworks solve this differently:

- **RTIC**: Static priority ceiling protocol. Deadlock-free by construction. Compile-time resource graph.
- **Embassy**: Async/await executor on bare-metal. Cooperative multitasking with DMA-driven wake.

## RTIC: Real-Time Interrupt-driven Concurrency

### Priority Ceiling Protocol

RTIC assigns every task a static priority. Resources shared between tasks are protected by locks that inherit the ceiling priority — the maximum priority of any task that accesses the resource. When a task locks a resource, RTIC temporarily raises the preemption threshold to that ceiling, preventing any other task that could access the resource from preempting.

This eliminates deadlocks: a task at priority 3 never waits on a task at priority 2 because priority 2's lock ceiling is already higher than 3.

```rust
#![no_main]
#![no_std]

use rtic::app;

#[app(device = stm32f4xx_hal::pac, peripherals = true)]
mod app {
    use stm32f4xx_hal::{gpio::*, prelude::*, timer::Timer};
    use systick_monotonic::Systick;

    #[shared]
    struct Shared {
        led: gpioc::PC13<Output<PushPull>>,
        adc_buffer: [u16; 256],
    }

    #[local]
    struct Local {
        sample_index: usize,
    }

    #[monotonic(binds = SysTick, default = true)]
    type MyMono = Systick<1000>;

    #[init]
    fn init(cx: init::Context) -> (Shared, Local, init::Monotonics) {
        let dp = cx.device;
        let rcc = dp.RCC.constrain();
        let gpioc = dp.GPIOC.split();
        let led = gpioc.pc13.into_push_pull_output();

        let mono = Systick::new(cx.core.SYST, 48_000_000);

        (
            Shared {
                led,
                adc_buffer: [0u16; 256],
            },
            Local { sample_index: 0 },
            init::Monotonics(mono),
        )
    }

    #[task(priority = 1, shared = [led])]
    fn blink(mut cx: blink::Context) {
        cx.shared.led.lock(|led| {
            led.toggle();
        });
        blink::spawn_after(1.secs()).ok();
    }

    #[task(priority = 2, shared = [adc_buffer])]
    fn adc_sample(mut cx: adc_sample::Context) {
        cx.shared.adc_buffer.lock(|buf| {
            buf[0] = read_adc_value();
        });
    }

    #[task(priority = 3, shared = [led, adc_buffer])]
    fn emergency_shutdown(mut cx: emergency_shutdown::Context) {
        (cx.shared.led, cx.shared.adc_buffer).lock(|led, buf| {
            led.set_low();
            buf.fill(0);
        });
    }
}
```

Priority assignment rules:
- Priority 1: background, non-time-critical (blink LED)
- Priority 2: periodic sensor sampling, soft real-time
- Priority 3: emergency handlers, hard real-time (over-current, stall detection)

### Resource Lock Semantics

RTIC resource locks are zero-cost when there's no contention — the compiler proves the resource is not shared with any higher-priority task and elides the critical section:

```rust
#[task(priority = 3, shared = [high_speed_buffer])]
fn high_prio(mut cx: high_prio::Context) {
    // No lock needed: priority 3 is the ceiling, and no task
    // above priority 3 accesses this resource.
    cx.shared.high_speed_buffer[0] = sample();
}
```

This is the mechanical sympathy advantage — the compiler knows the task graph statically and generates code as if the resource were exclusive when it effectively is.

## Embassy: Async Executor on Bare-Metal

Embassy provides a Rust-native async runtime for microcontrollers. Tasks are declared with `#[embassy_executor::task]` and yield by awaiting futures — typically DMA completions, timer expirations, or peripheral events.

```rust
#![no_main]
#![no_std]

use embassy_executor::Spawner;
use embassy_stm32::gpio::{Level, Output, Speed};
use embassy_stm32::peripherals::PC13;
use embassy_time::{Duration, Timer};

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    let mut led = Output::new(p.PC13, Level::High, Speed::Low);

    spawner.spawn(blink_task(led)).unwrap();
    spawner.spawn(sensor_read_task()).unwrap();
}

#[embassy_executor::task]
async fn blink_task(mut led: Output<'static, PC13>) {
    loop {
        led.set_high();
        Timer::after(Duration::from_millis(500)).await;
        led.set_low();
        Timer::after(Duration::from_millis(500)).await;
    }
}

#[embassy_executor::task]
async fn sensor_read_task() {
    let mut spi = /* configure SPI with DMA */;
    let mut rx_buf = [0u8; 64];

    loop {
        spi.read(&mut rx_buf).await;
        process_data(&rx_buf);
    }
}
```

`Timer::after().await` puts the CPU into WFI sleep until the timer fires — zero busy-waiting. DMA transfers free the CPU entirely while data moves.

### WFI/WFE: Sleep Until Interrupt

Both RTIC and Embassy use CPU sleep instructions when idle:

```rust
// RTIC idle task
#[idle]
fn idle(_: idle::Context) -> ! {
    loop {
        cortex_m::asm::wfi(); // Wait For Interrupt — CPU halts until IRQ
    }
}
```

`WFI` (Wait For Interrupt) stops the CPU clock. `WFE` (Wait For Event) additionally responds to SEV (Send Event) signals, useful for multi-core wake scenarios. Both are essential for power efficiency.

## Shared Single Stack

Unlike threads in an OS (which each have their own stack), RTIC tasks and Embassy async tasks share a single stack. Rust's ownership model guarantees no aliasing, so the compiler can place locals in non-overlapping stack regions.

**Embassy stack sharing**: When an async task `.await`s, its state is stored in a heap-allocated (or statically-allocated via `TaskStorage`) structure. The stack frame is released for other tasks to use. This is cooperative multitasking — a task that never `.await`s starves all others.

## RTIC vs Embassy vs Traditional RTOS

| Aspect | RTIC | Embassy | FreeRTOS (C) |
|--------|------|---------|-------------|
| Scheduling | Preemptive, priority-based | Cooperative, async | Preemptive, priority-based |
| Stack per task | Shared single stack | Shared + task storage | One stack per task (KB waste) |
| Deadlock safety | Proven by priority ceiling | No locks needed (async) | Manual discipline |
| Latency bound | Known at compile time | Depends on await points | Depends on scheduling quantum |
| Memory overhead | ~200 bytes per task | ~4KB per executor | ~1KB per task + stack |
| Interrupt handling | Hardware ISR → task | Hardware ISR → waker | FromISR → task notify |
| Best for | Hard real-time (motor ctrl) | Complex async (BLE, TCP) | Legacy C integration |

### When to choose what

- **RTIC**: Motor control, power converters, safety-critical IO, anything with hard deadlines < 100µs.
- **Embassy**: BLE stack, HTTP/MQTT, complex protocol handling, USB device, anything I/O-bound.
- **Hybrid**: RTIC for time-critical tasks, Embassy executor in RTIC idle task for I/O-bound work.
- **Neither**: If you need POSIX compatibility or existing C middleware, consider an RTOS like FreeRTOS with Rust bindings — but pay the memory and complexity tax.

## Red Lines

1. `static mut` for shared state is **undefined behavior** — use RTIC `#[shared]` resources or `CriticalSection`
2. Busy-waiting (`loop { if flag {} }`) is prohibited — use WFI/WFE or async `.await`
3. RTIC tasks must not perform blocking operations — the entire priority level is blocked
4. RTIC locks must be held for minimal duration — no DMA operations inside a lock
5. Embassy tasks must regularly `.await` — a task that never yields starves all other tasks
6. Mixed priority inversion: a low-priority RTIC task must never wait on a high-priority Embassy future
7. All interrupt handlers must be registered through RTIC `#[task(binds = ...)]` — never raw `#[interrupt]`

## References

- [RTIC Book — Priority Ceiling Protocol](https://rtic.rs/dev/book/en/by-example/resources.html)
- [Embassy Documentation](https://embassy.dev/book/dev/index.html)
- [cortex-m — WFI/WFE instructions](https://docs.rs/cortex-m/latest/cortex_m/asm/fn.wfi.html)
- [Real-Time Interrupt-driven Concurrency (RTIC)](https://rtic.rs/)
- [Embassy Executor](https://docs.rs/embassy-executor/latest/embassy_executor/)