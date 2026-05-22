# 01 — Argument Parsing with clap

## Status
**Approved.** clap derive API is the mandatory standard for all CLI argument parsing.

## Prerequisites
- `cargo add clap --features derive,env`
- `cargo add clap_complete` (for shell completions)
- Rust edition 2024

---

## 1. Derive API — The `#[derive(Parser)]` Pattern

```rust
use clap::{Parser, Subcommand, Args};
use std::path::PathBuf;

#[derive(Parser)]
#[command(
    name = "devkit",
    version,
    about = "Developer toolkit for project scaffolding",
    long_about = "A suite of tools for generating and validating project structures.",
    after_help = "Run `devkit help <subcommand>` for detailed usage."
)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Commands,

    #[arg(
        global = true,
        short,
        long,
        env = "DEVKIT_CONFIG",
        default_value = "devkit.toml",
        help = "Path to configuration file"
    )]
    pub config: PathBuf,

    #[arg(
        global = true,
        short = 'v',
        long,
        action = clap::ArgAction::Count,
        help = "Increase verbosity (-v, -vv, -vvv)"
    )]
    pub verbose: u8,
}

#[derive(Subcommand)]
pub enum Commands {
    Init(InitArgs),
    Build(BuildArgs),
    Deploy(DeployArgs),
    #[command(name = "completions")]
    GenerateCompletions(CompletionsArgs),
}
```

---

## 2. Subcommand with Typed Args

```rust
#[derive(Args)]
#[command(about = "Initialize a new project from a template")]
pub struct InitArgs {
    #[arg(help = "Name of the project")]
    pub name: String,

    #[arg(
        short,
        long,
        default_value = "rust",
        help = "Project template to use"
    )]
    pub template: String,

    #[arg(
        short,
        long,
        default_value_t = false,
        help = "Skip interactive prompts"
    )]
    pub yes: bool,

    #[arg(
        short = 'd',
        long,
        help = "Target directory (defaults to project name)"
    )]
    pub directory: Option<PathBuf>,
}
```

---

## 3. Value Parser — Custom Validation

```rust
use clap::builder::TypedValueParser;

fn parse_port_range(s: &str) -> Result<u16, String> {
    let port: u16 = s
        .parse()
        .map_err(|_| format!("`{s}` is not a valid port number"))?;
    if port < 1024 {
        Err(format!("Port {port} is privileged. Use a port >= 1024."))
    } else if port > 49151 {
        Err(format!("Port {port} is ephemeral. Use a port <= 49151."))
    } else {
        Ok(port)
    }
}

#[derive(Args)]
pub struct DeployArgs {
    #[arg(
        short,
        long,
        default_value = "3000",
        value_parser = parse_port_range,
        help = "Port to bind the service to"
    )]
    pub port: u16,

    #[arg(
        short,
        long,
        value_parser = clap::value_parser!(u32).range(1..=100),
        default_value = "10",
        help = "Max concurrent connections"
    )]
    pub max_connections: u32,

    #[arg(
        short = 't',
        long,
        value_enum,
        default_value_t = Environment::Production,
        help = "Target environment"
    )]
    pub env: Environment,
}

#[derive(clap::ValueEnum, Clone, Debug)]
pub enum Environment {
    Development,
    Staging,
    Production,
}
```

---

## 4. Shell Completions via `clap_complete`

```rust
use clap::CommandFactory;
use clap_complete::{Shell, generate};
use std::io;

#[derive(Args)]
pub struct CompletionsArgs {
    #[arg(value_enum, help = "Shell to generate completions for")]
    pub shell: Shell,
}

fn print_completions(shell: Shell) {
    let mut cmd = Cli::command();
    let bin_name = cmd.get_name().to_string();
    generate(shell, &mut cmd, bin_name, &mut io::stdout());
}

// Usage in main:
// Commands::GenerateCompletions(args) => print_completions(args.shell),

// Install generated completions:
//   devkit completions bash | sudo tee /usr/share/bash-completion/completions/devkit
//   devkit completions zsh  > ~/.zfunc/_devkit
//   devkit completions fish > ~/.config/fish/completions/devkit.fish
```

---

## 5. Environment Variable Fallback

```rust
#[derive(Args)]
pub struct BuildArgs {
    #[arg(
        short,
        long,
        env = "DEVKIT_REGISTRY",
        default_value = "ghcr.io",
        help = "Container registry URL"
    )]
    pub registry: String,

    #[arg(
        long,
        env = "DEVKIT_BUILD_ARGS",
        value_delimiter = ',',
        help = "Additional build arguments (comma-separated)"
    )]
    pub build_args: Vec<String>,

    #[arg(
        short = 'f',
        long,
        env = "DEVKIT_DOCKERFILE",
        default_value = "Dockerfile",
        help = "Path to Dockerfile"
    )]
    pub dockerfile: PathBuf,
}
```

---

## 6. Argument Groups and Conflicts

```rust
#[derive(Args)]
#[command(group = clap::ArgGroup::new("source").required(true).args(["file", "stdin"]))]
pub struct ParseArgs {
    #[arg(
        short,
        long,
        help = "Input file to parse"
    )]
    pub file: Option<PathBuf>,

    #[arg(
        short,
        long,
        default_value_t = false,
        help = "Read from stdin"
    )]
    pub stdin: bool,

    #[arg(
        short = 'o',
        long,
        default_value = "-",
        help = "Output file (default: stdout)"
    )]
    pub output: String,
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Forbid** `std::env::args()` manual parsing | No type safety, no `--help`, no env fallback. Always use clap derive. |
| **Forbid** `arg.action(ArgAction::SetTrue)` without `default_value_t` | Default-value-less booleans are ambiguous. Always make boolean defaults explicit. |
| **Forbid** stringly-typed validators in handler code | Parse and validate in `value_parser`. Fail fast, before business logic runs. |
| **Mandatory**: every subcommand has `about` | Users read `--help` before reading source. Every subcommand must explain itself. |
| **Mandatory**: `global = true` for `--config` and `--verbose` | These flags must work at any nesting level. |

---

## References
- [clap derive reference](https://docs.rs/clap/latest/clap/_derive/index.html)
- [clap_complete crate](https://docs.rs/clap_complete/latest/clap_complete/)
- [clap cookbook — argument validation](https://docs.rs/clap/latest/clap/_cookbook/index.html)