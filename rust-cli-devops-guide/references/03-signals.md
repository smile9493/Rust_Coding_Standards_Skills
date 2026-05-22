# 03 — Signal Handling & Graceful Shutdown

## Status
**Approved.** All long-running CLI tools must handle `SIGINT`/`SIGTERM` gracefully. Use `tokio::signal` for async runtimes.

## Prerequisites
- `cargo add tokio --features signal,process`
- `cargo add ctrlc` (cross-platform alternative)
- `cargo add signal-hook` (sync signal handling on Unix)

---

## 1. Graceful Shutdown with `tokio::signal::ctrl_c`

```rust
use tokio::signal;
use tokio::sync::watch;
use tokio::time::{sleep, Duration};

async fn run_server(shutdown_rx: watch::Receiver<bool>) {
    tokio::select! {
        _ = do_work() => {
            tracing::info!("work completed normally");
        }
        _ = wait_for_shutdown(shutdown_rx) => {
            tracing::warn!("received shutdown signal, draining connections...");
            drain_connections().await;
            tracing::info!("graceful shutdown complete");
        }
    }
}

async fn wait_for_shutdown(mut rx: watch::Receiver<bool>) {
    rx.changed().await.ok();
}

async fn do_work() {
    sleep(Duration::from_secs(3600)).await;
}

async fn drain_connections() {
    sleep(Duration::from_secs(2)).await;
}

#[tokio::main]
async fn main() {
    let (shutdown_tx, shutdown_rx) = watch::channel(false);
    let shutdown_tx2 = shutdown_tx.clone();

    let signal_handle = tokio::spawn(async move {
        signal::ctrl_c().await.expect("failed to listen for ctrl_c");
        tracing::info!("ctrl_c received, initiating shutdown");
        shutdown_tx.send(true).ok();
    });

    let sigterm_handle = tokio::spawn(async move {
        let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to register SIGTERM handler");
        sigterm.recv().await;
        tracing::info!("SIGTERM received, initiating shutdown");
        shutdown_tx2.send(true).ok();
    });

    run_server(shutdown_rx).await;

    signal_handle.abort();
    sigterm_handle.abort();
}
```

---

## 2. SIGPIPE Suppression (Unix)

```rust
fn suppress_sigpipe() {
    #[cfg(unix)]
    unsafe {
        libc::signal(libc::SIGPIPE, libc::SIG_DFL);
    }
}

fn main() {
    suppress_sigpipe();
    // ... rest of your CLI logic
}
```

---

## 3. Cross-Platform Signal Handling with `ctrlc` Crate

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

fn main() {
    let running = Arc::new(AtomicBool::new(true));
    let r = running.clone();

    ctrlc::set_handler(move || {
        tracing::warn!("received interrupt, shutting down...");

        if r.load(Ordering::SeqCst) {
            r.store(false, Ordering::SeqCst);
        } else {
            tracing::error!("forced exit — second interrupt received");
            std::process::exit(130);
        }
    })
    .expect("failed to set ctrl-c handler");

    while running.load(Ordering::SeqCst) {
        do_iteration();
    }

    tracing::info!("cleanup complete");
}

fn do_iteration() {
    std::thread::sleep(std::time::Duration::from_millis(100));
}
```

---

## 4. `signal_hook` — Low-Level Unix Signal Handler

```rust
use signal_hook::{consts::signal::*, iterator::Signals};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;
use std::thread;

fn setup_signal_handler(running: Arc<AtomicBool>) {
    thread::spawn(move || {
        let mut signals = Signals::new(&[SIGINT, SIGTERM, SIGQUIT, SIGHUP])
            .expect("failed to register signal handlers");

        for sig in signals.forever() {
            match sig {
                SIGINT | SIGTERM | SIGQUIT => {
                    tracing::info!("received signal {sig}, shutting down");
                    running.store(false, Ordering::SeqCst);
                    break;
                }
                SIGHUP => {
                    tracing::info!("received SIGHUP, reloading configuration");
                }
                _ => {}
            }
        }
    });
}

fn main() {
    let running = Arc::new(AtomicBool::new(true));
    setup_signal_handler(running.clone());

    while running.load(Ordering::SeqCst) {
        std::thread::sleep(std::time::Duration::from_secs(1));
    }

    tracing::info!("exiting cleanly");
}
```

---

## 5. Async Cancellation Token Pattern

```rust
use tokio::sync::watch;
use tokio::signal;
use tokio_util::sync::CancellationToken;

#[tokio::main]
async fn main() {
    let cancel_token = CancellationToken::new();

    let worker_ct = cancel_token.clone();
    let worker = tokio::spawn(async move {
        tokio::select! {
            _ = worker_ct.cancelled() => {
                tracing::info!("worker: cancellation received, flushing buffers");
                flush_buffers().await;
            }
            _ = long_running_task() => {
                tracing::info!("worker: task completed");
            }
        }
    });

    tokio::select! {
        _ = signal::ctrl_c() => {
            tracing::info!("main: ctrl_c received");
            cancel_token.cancel();
        }
        _ = tokio::signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("SIGTERM handler")
            .recv() => {
            tracing::info!("main: SIGTERM received");
            cancel_token.cancel();
        }
    }

    worker.await.ok();
    tracing::info!("shutdown complete");
}

async fn flush_buffers() {
    tokio::time::sleep(std::time::Duration::from_millis(500)).await;
}

async fn long_running_task() {
    tokio::time::sleep(std::time::Duration::from_secs(3600)).await;
}
```

---

## 6. Exit Codes for Signal Termination

```rust
fn exit_code_for_signal() -> i32 {
    // Convention: 128 + signal number
    const SIGINT_EXIT: i32 = 128 + 2;   // 130
    const SIGTERM_EXIT: i32 = 128 + 15; // 143
    SIGINT_EXIT
}

fn main() {
    let interrupted = true;

    if interrupted {
        std::process::exit(exit_code_for_signal());
    }
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: handle both `SIGINT` and `SIGTERM` | `docker stop` sends SIGTERM. Kubernetes sends SIGTERM. Missing this = data loss on container shutdown. |
| **Mandatory**: second interrupt = force exit | If the user presses Ctrl+C twice, they mean it. Don't block shutdown. |
| **Mandatory**: `#[cfg(unix)]` suppress SIGPIPE | Writing to a closed pipe is normal behavior (e.g., `cmd | head`). Must not panic. |
| **Forbid** `process::exit()` in library code | Only binary crates exit. Libraries return `Result`. |
| **Forbid** blocking signal handlers in async context | Use `tokio::signal::ctrl_c()`. Never call `thread::sleep` in async signal handlers. |

---

## References
- [tokio signal module](https://docs.rs/tokio/latest/tokio/signal/index.html)
- [ctrlc crate](https://docs.rs/ctrlc/latest/ctrlc/)
- [signal-hook crate](https://docs.rs/signal-hook/latest/signal_hook/)
- [CancellationToken pattern (tokio-util)](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html)