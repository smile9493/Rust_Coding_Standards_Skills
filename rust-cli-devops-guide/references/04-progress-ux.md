# 04 — Progress UX & Terminal Output

## Status
**Approved.** `indicatif` for progress bars, `tracing` for structured logging, `ratatui` for full-screen TUI apps.

## Prerequisites
- `cargo add indicatif --features improved_unicode,rayon`
- `cargo add tracing tracing-subscriber --features env-filter,json`
- `cargo add console`
- `cargo add ratatui crossterm` (for TUI mode)

---

## 1. indicatif — Single ProgressBar with ETA

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

fn process_items() {
    let count = 120;
    let pb = ProgressBar::new(count);

    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} ({eta})"
            )
            .unwrap()
            .progress_chars("#>-"),
    );

    for i in 0..count {
        pb.set_message(format!("processing item {i}"));
        std::thread::sleep(Duration::from_millis(30));
        pb.inc(1);
    }

    pb.finish_with_message("done");
}
```

---

## 2. indicatif — MultiProgress for Parallel Tasks

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};

fn parallel_download(urls: Vec<String>) {
    let mp = MultiProgress::new();

    let sty = ProgressStyle::default_bar()
        .template("{prefix:.bold.dim} [{bar:30.green}] {bytes}/{total_bytes} ({eta})")
        .unwrap()
        .progress_chars("=> ");

    let handles: Vec<_> = urls
        .into_iter()
        .map(|url| {
            let pb = mp.add(ProgressBar::new(1024 * 1024));
            pb.set_style(sty.clone());
            pb.set_prefix(url.clone());

            std::thread::spawn(move || {
                for _ in 0..1024 {
                    std::thread::sleep(Duration::from_micros(100));
                    pb.inc(1024);
                }
                pb.finish_with_message("✓");
            })
        })
        .collect();

    for h in handles {
        h.join().unwrap();
    }

    mp.clear().unwrap();
}
```

---

## 3. TTY Guard — Disable Progress When Piped

```rust
use std::io::IsTerminal;

fn create_progress_bar(total: u64) -> ProgressBar {
    if std::io::stdout().is_terminal() {
        let pb = ProgressBar::new(total);
        pb.set_style(
            ProgressStyle::default_bar()
                .template("{spinner} [{bar:30}] {pos}/{len} {msg}")
                .unwrap(),
        );
        pb
    } else {
        ProgressBar::hidden()
    }
}
```

---

## 4. tracing-subscriber — EnvFilter + JSON for CI

```rust
use tracing_subscriber::{EnvFilter, fmt, layer::SubscriberExt, util::SubscriberInitExt};

fn init_tracing(verbose: u8, json_output: bool) {
    let env_filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| {
            match verbose {
                0 => EnvFilter::new("warn"),
                1 => EnvFilter::new("info"),
                2 => EnvFilter::new("debug"),
                _ => EnvFilter::new("trace"),
            }
        });

    if json_output || std::env::var("CI").is_ok() {
        tracing_subscriber::registry()
            .with(env_filter)
            .with(fmt::layer().json().flatten_event(true).with_current_span(false))
            .init();
    } else {
        tracing_subscriber::registry()
            .with(env_filter)
            .with(
                fmt::layer()
                    .with_target(false)
                    .with_thread_ids(false)
                    .with_ansi(std::io::stdout().is_terminal()),
            )
            .init();
    }
}
```

---

## 5. console — Terminal Colors and User Input

```rust
use console::{style, Term};

fn styled_output() {
    println!("{}", style("SUCCESS").green().bold());
    println!("{}", style("WARNING").yellow().bold());
    println!("{}", style("ERROR").red().bold().underlined());
}

fn interactive_prompt() -> std::io::Result<()> {
    let term = Term::stdout();

    term.write_line("What is your project name?")?;
    let name = term.read_line()?;

    term.write_line(&format!(
        "Creating project {}...",
        style(&name).cyan().bold()
    ))?;
    Ok(())
}
```

---

## 6. ratatui — TUI with Widgets

```rust
use ratatui::{
    backend::CrosstermBackend,
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Style},
    text::{Line, Span, Text},
    widgets::{Block, Borders, Gauge, List, ListItem, Paragraph, Tabs},
    Terminal,
};
use crossterm::{
    event::{self, DisableMouseCapture, EnableMouseCapture, Event, KeyCode},
    execute,
    terminal::{disable_raw_mode, enable_raw_mode, EnterAlternateScreen, LeaveAlternateScreen},
};
use std::io;

#[derive(Clone)]
struct AppState {
    selected_tab: usize,
    progress: f64,
}

fn run_tui() -> io::Result<()> {
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen, EnableMouseCapture)?;

    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;

    let mut app = AppState {
        selected_tab: 0,
        progress: 42.0,
    };

    loop {
        terminal.draw(|f| ui(f, &app))?;

        if event::poll(std::time::Duration::from_millis(200))? {
            if let Event::Key(key) = event::read()? {
                match key.code {
                    KeyCode::Char('q') | KeyCode::Esc => break,
                    KeyCode::Tab => {
                        app.selected_tab = (app.selected_tab + 1) % 4;
                    }
                    _ => {}
                }
            }
        }
    }

    disable_raw_mode()?;
    execute!(
        terminal.backend_mut(),
        LeaveAlternateScreen,
        DisableMouseCapture
    )?;
    terminal.show_cursor()?;
    Ok(())
}

fn ui(f: &mut ratatui::Frame, app: &AppState) {
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .margin(1)
        .constraints([Constraint::Length(3), Constraint::Min(0)].as_ref())
        .split(f.area());

    let titles = ["Build", "Test", "Deploy", "Logs"];
    let tabs = Tabs::new(
        titles
            .iter()
            .map(|t| Line::from(*t))
            .collect::<Vec<_>>(),
    )
    .select(app.selected_tab)
    .block(Block::default().borders(Borders::ALL).title("Pipeline"))
    .highlight_style(Style::default().fg(Color::Yellow));
    f.render_widget(tabs, chunks[0]);

    let gauge = Gauge::default()
        .block(Block::default().borders(Borders::ALL).title("Progress"))
        .gauge_style(Style::default().fg(Color::Green))
        .percent(app.progress as u16);
    f.render_widget(
        gauge,
        Rect {
            x: chunks[1].x,
            y: chunks[1].y,
            width: chunks[1].width,
            height: 3,
        },
    );

    let items: Vec<ListItem> = vec![
        ListItem::new("✓ Compile src/main.rs"),
        ListItem::new("✓ Resolve dependencies"),
        ListItem::new("⏳ Running tests..."),
        ListItem::new("○ Deploy to staging"),
    ];
    let list = List::new(items)
        .block(Block::default().borders(Borders::ALL).title("Steps"))
        .style(Style::default().fg(Color::White));
    f.render_widget(
        list,
        Rect {
            x: chunks[1].x,
            y: chunks[1].y + 4,
            width: chunks[1].width,
            height: chunks[1].height.saturating_sub(4),
        },
    );
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: check `stdout.is_terminal()` before creating `ProgressBar` | Piped output with progress bar garbage is a terrible UX. Use `ProgressBar::hidden()` or log-only mode. |
| **Mandatory**: JSON logging in CI (`$CI` or `--json`) | Structured logs are machine-parseable. Plain text is for humans only. |
| **Forbid** ANSI escape codes in non-TTY output | Redirected output must be plain text. Use `console::colors_enabled()` or `supports-color` detection. |
| **Forbid** blocking IO on `Term::read_key()` in async context | ratatui runs in its own thread. Async IO uses crossterm's event polling. Never mix. |

---

## References
- [indicatif documentation](https://docs.rs/indicatif/latest/indicatif/)
- [tracing-subscriber EnvFilter](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/filter/struct.EnvFilter.html)
- [console crate](https://docs.rs/console/latest/console/)
- [ratatui book](https://ratatui.rs/)