# 05 — Configuration Management

## Status
**Approved.** Use `figment` or `config` crate for layered configuration. Always respect the XDG Base Directory spec via the `dirs` crate.

## Prerequisites
- `cargo add figment --features env,toml,json,yaml`
- `cargo add dirs`
- `cargo add serde --features derive`

---

## 1. Configuration Cascade — CLI > Env > File > Default

```
Priority (highest first):
 1. CLI arguments       (--port 9090)
 2. Environment variables (DEVKIT_PORT=9090)
 3. Config file           ($XDG_CONFIG_HOME/devkit/config.toml)
 4. Default config        (embedded in binary)
```

---

## 2. Config Struct Definition

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,

    #[serde(default = "default_port")]
    pub port: u16,

    #[serde(default)]
    pub tls: Option<TlsConfig>,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct TlsConfig {
    pub cert_path: String,
    pub key_path: String,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct DatabaseConfig {
    pub url: String,
    #[serde(default = "default_pool_size")]
    pub max_connections: u32,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct LoggingConfig {
    #[serde(default = "default_log_level")]
    pub level: String,

    #[serde(default)]
    pub json: bool,
}

fn default_host() -> String {
    "127.0.0.1".into()
}

fn default_port() -> u16 {
    3000
}

fn default_pool_size() -> u32 {
    10
}

fn default_log_level() -> String {
    "info".into()
}
```

---

## 3. Layered Configuration with `figment`

```rust
use figment::{
    Figment,
    providers::{Env, Format, Toml, Serialized},
};

fn load_config(config_path: Option<&std::path::Path>) -> Result<AppConfig, figment::Error> {
    let default_config = AppConfig {
        server: ServerConfig {
            host: default_host(),
            port: default_port(),
            tls: None,
        },
        database: DatabaseConfig {
            url: "postgres://localhost:5432/devkit".into(),
            max_connections: default_pool_size(),
        },
        logging: LoggingConfig {
            level: default_log_level(),
            json: false,
        },
    };

    let mut figment = Figment::new()
        .merge(Serialized::defaults(default_config));

    // Layer 2: User config file
    if let Some(path) = config_path {
        if path.exists() {
            figment = figment.merge(Toml::file(path));
        }
    } else {
        let xdg_path = xdg_config_path();
        if xdg_path.exists() {
            figment = figment.merge(Toml::file(&xdg_path));
        }
    }

    // Layer 3: Environment variables with DEVKIT_ prefix
    figment = figment.merge(
        Env::prefixed("DEVKIT_")
            .split("__")
            .map(|key| key.as_str().to_lowercase().replace('_', ".")),
    );

    figment.extract()
}
```

---

## 4. XDG Path Resolution with `dirs`

```rust
use std::path::PathBuf;

fn xdg_config_path() -> PathBuf {
    dirs::config_dir()
        .unwrap_or_else(|| PathBuf::from("~/.config"))
        .join("devkit")
        .join("config.toml")
}

fn xdg_data_path() -> PathBuf {
    dirs::data_dir()
        .unwrap_or_else(|| PathBuf::from("~/.local/share"))
        .join("devkit")
}

fn xdg_cache_path() -> PathBuf {
    dirs::cache_dir()
        .unwrap_or_else(|| PathBuf::from("~/.cache"))
        .join("devkit")
}

fn ensure_dirs() -> std::io::Result<()> {
    std::fs::create_dir_all(xdg_config_path().parent().unwrap())?;
    std::fs::create_dir_all(&xdg_data_path())?;
    std::fs::create_dir_all(&xdg_cache_path())?;
    Ok(())
}
```

---

## 5. Config + CLI Merge Pattern

```rust
use clap::Parser;

#[derive(Parser)]
pub struct Cli {
    #[arg(short, long)]
    pub port: Option<u16>,

    #[arg(short = 'H', long)]
    pub host: Option<String>,

    #[arg(short, long)]
    pub config: Option<PathBuf>,

    #[arg(long)]
    pub database_url: Option<String>,
}

fn build_config(cli: &Cli) -> Result<AppConfig, Box<dyn std::error::Error>> {
    let mut config = load_config(cli.config.as_deref())?;

    // CLI args override file/env config
    if let Some(port) = cli.port {
        config.server.port = port;
    }
    if let Some(ref host) = cli.host {
        config.server.host = host.clone();
    }
    if let Some(ref url) = cli.database_url {
        config.database.url = url.clone();
    }

    Ok(config)
}
```

---

## 6. Configuration Validation

```rust
impl AppConfig {
    pub fn validate(&self) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();

        if self.server.port == 0 {
            errors.push("server.port must not be 0".into());
        }

        if self.database.url.is_empty() {
            errors.push("database.url must not be empty".into());
        }

        if let Some(ref tls) = self.server.tls {
            if !std::path::Path::new(&tls.cert_path).exists() {
                errors.push(format!("TLS cert not found: {}", tls.cert_path));
            }
            if !std::path::Path::new(&tls.key_path).exists() {
                errors.push(format!("TLS key not found: {}", tls.key_path));
            }
        }

        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
}
```

---

## 7. Cross-Platform Path Convention

```rust
fn config_paths_per_platform() {
    #[cfg(target_os = "linux")]
    let config_dir = dirs::config_dir()
        .unwrap_or_else(|| PathBuf::from("~/.config"));

    #[cfg(target_os = "macos")]
    let config_dir = dirs::home_dir()
        .unwrap()
        .join("Library")
        .join("Application Support");

    #[cfg(target_os = "windows")]
    let config_dir = dirs::config_dir()
        .unwrap_or_else(|| PathBuf::from(r"C:\Users\Default\AppData\Roaming"));
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: use `dirs` crate for paths | Never hardcode `~/.config` or `~/Library`. `dirs` handles Windows (`%APPDATA%`), macOS (`~/Library/Application Support`), and Linux (`$XDG_CONFIG_HOME`) correctly. |
| **Mandatory**: CLI overrides all other sources | User intent at invocation time is the highest authority. Never let a config file silently override a CLI flag. |
| **Forbid** writing config outside XDG directories | Never write to `/etc`, `/tmp`, or the current directory. Everything goes under `$XDG_CONFIG_HOME/$APP/`. |
| **Forbid** panicking on missing config file | Default config must be embedded. Missing config file is normal (first run). Never crash on it. |
| **Mandatory**: `Env::prefixed()` with namespace | Environment variables must be namespaced (`DEVKIT_PORT`, not `PORT`). Avoid collisions with system vars. |

---

## References
- [figment crate](https://docs.rs/figment/latest/figment/)
- [config crate](https://docs.rs/config/latest/config/)
- [dirs crate](https://docs.rs/dirs/latest/dirs/)
- [XDG Base Directory specification](https://specifications.freedesktop.org/basedir-spec/latest/)