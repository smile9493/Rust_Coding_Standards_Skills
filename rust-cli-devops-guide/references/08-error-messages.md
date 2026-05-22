# 08 — Error Messages & Debugging UX

## Status
**Approved.** `miette` for rich error reporting with source code snippets. `color-eyre` as a lighter alternative. Unique exit codes for each failure mode.

## Prerequisites
- `cargo add miette --features fancy`
- `cargo add thiserror`
- Alternative: `cargo add color-eyre`

---

## 1. miette — Rich Error Types with Source Snippets

```rust
use miette::{Diagnostic, NamedSource, Result, SourceSpan};
use thiserror::Error;

#[derive(Debug, Error, Diagnostic)]
pub enum DevkitError {
    #[error("Configuration file not found")]
    #[diagnostic(
        code(devkit::config::not_found),
        help("Run `devkit init` to create a default configuration file"),
        url("https://docs.devkit.io/configuration")
    )]
    ConfigNotFound {
        #[source_code]
        src: NamedSource<String>,

        #[label("file referenced here")]
        span: SourceSpan,
    },

    #[error("Invalid port number: {port}")]
    #[diagnostic(
        code(devkit::server::invalid_port),
        help("Port must be between 1024 and 49151")
    )]
    InvalidPort {
        port: u16,

        #[source_code]
        src: NamedSource<String>,

        #[label("this port is out of range")]
        span: SourceSpan,
    },

    #[error("Failed to connect to database at {url}")]
    #[diagnostic(
        code(devkit::database::connection_failed),
        help("Verify that the database is running and the URL is correct.")
    )]
    DatabaseConnection {
        url: String,

        #[related]
        related: Vec<DevkitError>,
    },

    #[error("Template '{template}' not found")]
    #[diagnostic(
        code(devkit::template::not_found),
        help("Available templates: rust, python, node. Run `devkit init --list-templates`.")
    )]
    TemplateNotFound {
        template: String,

        #[label("unknown template")]
        span: SourceSpan,

        #[source_code]
        src: NamedSource<String>,
    },
}
```

---

## 2. Error Reporting with Source File Reference

```rust
use miette::{IntoDiagnostic, WrapErr};

fn parse_config_file(path: &std::path::Path) -> Result<AppConfig> {
    let content = std::fs::read_to_string(path)
        .into_diagnostic()
        .wrap_err_with(|| format!("Failed to read config from {}", path.display()))?;

    let src = NamedSource::new(path.display().to_string(), content.clone());

    toml::from_str::<AppConfig>(&content)
        .map_err(|e| {
            let span = e.span().unwrap_or(0..0);
            DevkitError::ConfigError {
                src,
                span: (span.start, span.end - span.start).into(),
                message: e.message().to_string(),
            }
        })
        .into_diagnostic()
}
```

---

## 3. Error Handler Setup in `main()`

```rust
use miette::{set_panic_hook, Report};

fn main() -> Result<()> {
    set_panic_hook();

    let cli = Cli::parse();

    match run(cli) {
        Ok(()) => Ok(()),
        Err(err) => {
            let report: Report = err.into();
            if cli.verbose >= 2 {
                eprintln!("{:?}", report);
            } else {
                eprintln!("{:?}", report);
            }
            std::process::exit(exit_code_for_error(&report));
        }
    }
}

fn exit_code_for_error(report: &Report) -> i32 {
    if let Some(code) = report.code() {
        match code.to_string().as_str() {
            "devkit::config::not_found" => 10,
            "devkit::server::invalid_port" => 20,
            "devkit::database::connection_failed" => 30,
            "devkit::template::not_found" => 40,
            _ => 1,
        }
    } else {
        1
    }
}
```

---

## 4. Verbosity Levels — `--verbose` / `--quiet`

```rust
use clap::Parser;

#[derive(Parser)]
pub struct Cli {
    #[arg(
        short = 'v',
        long,
        action = clap::ArgAction::Count,
        global = true,
        help = "Increase verbosity (-v, -vv, -vvv)"
    )]
    pub verbose: u8,

    #[arg(
        short = 'q',
        long,
        action = clap::ArgAction::Count,
        conflicts_with = "verbose",
        global = true,
        help = "Decrease verbosity (-q, -qq)"
    )]
    pub quiet: u8,
}

fn configure_logging(cli: &Cli) {
    let filter = if cli.quiet >= 2 {
        "error"
    } else if cli.quiet >= 1 {
        "warn"
    } else if cli.verbose >= 3 {
        "trace"
    } else if cli.verbose >= 2 {
        "debug"
    } else if cli.verbose >= 1 {
        "info"
    } else {
        "warn"
    };

    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .with_target(cli.verbose >= 3)
        .init();
}
```

---

## 5. color-eyre — Lightweight Alternative to miette

```rust
use color_eyre::{
    eyre::{eyre, WrapErr},
    Result,
};

fn main() -> Result<()> {
    color_eyre::install()?;

    let config = load_config().wrap_err("Failed to load application configuration")?;

    deploy(&config).map_err(|e| {
        eyre!(
            "Deployment failed for {}: {}",
            config.server.host,
            e
        )
    })?;

    Ok(())
}

fn load_config() -> Result<AppConfig> {
    let content = std::fs::read_to_string("config.toml")
        .wrap_err("Could not read config.toml. Does the file exist?")?;

    toml::from_str(&content).wrap_err("Failed to parse config.toml. Is it valid TOML?")
}

fn deploy(config: &AppConfig) -> Result<()> {
    if config.server.port == 0 {
        return Err(eyre!("Server port must not be 0"));
    }
    Ok(())
}
```

---

## 6. Structured Error Output for CI

```rust
use serde::Serialize;

#[derive(Serialize)]
struct ErrorOutput {
    code: String,
    message: String,
    help: Option<String>,
    location: Option<ErrorLocation>,
}

#[derive(Serialize)]
struct ErrorLocation {
    file: String,
    line: usize,
    column: usize,
}

fn format_error_for_ci(err: &dyn std::error::Error, verbose: bool) -> String {
    if !verbose {
        let output = ErrorOutput {
            code: "devkit::error".to_string(),
            message: err.to_string(),
            help: None,
            location: None,
        };
        serde_json::to_string(&output).unwrap_or_else(|_| err.to_string())
    } else {
        format!("{err:#?}")
    }
}
```

---

## 7. Custom Exit Codes Table

```rust
pub mod exit_code {
    pub const SUCCESS: i32 = 0;
    pub const GENERAL_ERROR: i32 = 1;
    pub const INVALID_ARGS: i32 = 2;
    pub const CONFIG_NOT_FOUND: i32 = 10;
    pub const CONFIG_PARSE_ERROR: i32 = 11;
    pub const INVALID_PORT: i32 = 20;
    pub const DATABASE_CONNECTION_FAILED: i32 = 30;
    pub const TEMPLATE_NOT_FOUND: i32 = 40;
    pub const NETWORK_TIMEOUT: i32 = 50;
    pub const PERMISSION_DENIED: i32 = 60;
    pub const INTERRUPTED: i32 = 130;
    pub const SIGTERM: i32 = 143;
}

// Usage:
// std::process::exit(exit_code::INTERRUPTED);
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: unique exit codes for each failure category | Scripts and CI pipelines must branch on exit codes. A universal exit code `1` is indistinguishable from any other failure. |
| **Forbid** raw backtraces in user-facing output | Users don't read backtraces. Use `miette::Report` with `#[diagnostic(help(...))]` for actionable messages. |
| **Forbid** `unwrap()` / `expect()` in CLI handler code | Every unwrap is a potential panic with zero context. Use `?` with contextual errors via `wrap_err` or `Diagnostic`. |
| **Forbid** printing errors to stdout | Errors go to stderr. Stdout is for structured output (JSON, CSV) that may be piped to `jq` or another tool. |
| **Mandatory**: `--verbose` / `--quiet` flags are global | Users must control output detail from any subcommand. Verbosity toggles backtrace visibility and log levels. |

---

## References
- [miette crate](https://docs.rs/miette/latest/miette/)
- [color-eyre crate](https://docs.rs/color-eyre/latest/color_eyre/)
- [thiserror crate](https://docs.rs/thiserror/latest/thiserror/)
- [tracing subscriber configuration](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/)