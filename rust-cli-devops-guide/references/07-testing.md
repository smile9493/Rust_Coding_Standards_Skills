# 07 — Testing CLI Applications

## Status
**Approved.** `assert_cmd` for subprocess integration tests. `trycmd` / `snapbox` for output snapshot testing. `tempfile` for isolated file-system tests.

## Prerequisites
- `cargo add --dev assert_cmd predicates`
- `cargo add --dev trycmd`
- `cargo add --dev tempfile`

---

## 1. assert_cmd — Subprocess Integration Tests

```rust
#[cfg(test)]
mod integration_tests {
    use assert_cmd::Command;
    use predicates::prelude::*;

    #[test]
    fn test_version_output() {
        let mut cmd = Command::cargo_bin("devkit").unwrap();
        cmd.arg("--version");
        cmd.assert()
            .success()
            .stdout(predicate::str::contains("devkit 1.0.0"));
    }

    #[test]
    fn test_init_command_success() {
        let temp = tempfile::tempdir().unwrap();

        let mut cmd = Command::cargo_bin("devkit").unwrap();
        cmd.arg("init")
            .arg("my-project")
            .arg("--yes")
            .current_dir(temp.path());

        cmd.assert()
            .success()
            .stdout(predicate::str::contains("Initialized project my-project"));

        assert!(temp.path().join("my-project").exists());
    }

    #[test]
    fn test_init_rejects_empty_name() {
        let mut cmd = Command::cargo_bin("devkit").unwrap();
        cmd.arg("init").arg("");

        cmd.assert()
            .failure()
            .stderr(predicate::str::contains("name must not be empty"));
    }

    #[test]
    fn test_build_missing_dockerfile() {
        let temp = tempfile::tempdir().unwrap();

        let mut cmd = Command::cargo_bin("devkit").unwrap();
        cmd.arg("build")
            .arg("-f")
            .arg("nonexistent.Dockerfile")
            .current_dir(temp.path());

        cmd.assert()
            .failure()
            .code(predicate::ne(0))
            .stderr(predicate::str::contains("Dockerfile not found"));
    }
}
```

---

## 2. Testing Exit Codes

```rust
#[test]
fn test_exit_code_for_signal() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("deploy")
        .arg("--port")
        .arg("80")
        .timeout(std::time::Duration::from_secs(2));

    let assert = cmd.assert();
    assert.failure();
    // Privileged port should produce exit code 1
    assert.code(1);
}

#[test]
fn test_exit_code_0_on_help() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("--help");
    cmd.assert().success().code(0);
}

#[test]
fn test_exit_code_2_on_invalid_subcommand() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("nonexistent_subcommand");
    cmd.assert().failure().code(2);
}
```

---

## 3. Testing stdin Passthrough

```rust
#[test]
fn test_parse_stdin() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("parse")
        .arg("--stdin")
        .write_stdin("{\"key\": \"value\"}\n")
        .arg("-o")
        .arg("-");

    cmd.assert()
        .success()
        .stdout(predicate::str::contains("key"));
}

#[test]
fn test_pipe_input() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("parse")
        .pipe_stdin("testdata/sample.json")
        .unwrap();

    cmd.assert().success();
}
```

---

## 4. Testing Environment Variable Behavior

```rust
#[test]
fn test_env_var_override() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("deploy")
        .env("DEVKIT_PORT", "9090")
        .env("DEVKIT_REGISTRY", "docker.io");

    cmd.assert()
        .success()
        .stdout(predicate::str::contains("port: 9090"))
        .stdout(predicate::str::contains("registry: docker.io"));
}

#[test]
fn test_cli_overrides_env() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("deploy")
        .arg("--port")
        .arg("8080")
        .env("DEVKIT_PORT", "9090");

    cmd.assert()
        .success()
        .stdout(predicate::str::contains("port: 8080"));
}
```

---

## 5. trycmd — Snapshot Testing

Create `tests/cmd/*.trycmd` test files:

```
# tests/cmd/help.trycmd
$ devkit --help
A suite of tools for generating and validating project structures.

Usage: devkit [OPTIONS] <COMMAND>

Commands:
  init          Initialize a new project from a template
  build         Build the project container image
  deploy        Deploy the project to a target environment
  completions   Generate shell completions
  help          Print this message or the help of the given subcommand(s)

Options:
  -c, --config <CONFIG>    Path to configuration file [env: DEVKIT_CONFIG=] [default: devkit.toml]
  -v, --verbose...         Increase verbosity (-v, -vv, -vvv)
  -h, --help               Print help
  -V, --version            Print version
```

```rust
// Integration test loader:
#[cfg(test)]
mod trycmd_tests {
    #[test]
    fn cli_tests() {
        let t = trycmd::TestCases::new();
        t.case("tests/cmd/*.trycmd");
    }
}
```

---

## 6. Testing with Temporary Filesystem

```rust
use std::io::Write;

#[test]
fn test_config_file_loading() {
    let dir = tempfile::tempdir().unwrap();
    let config_path = dir.path().join("devkit.toml");

    std::fs::write(
        &config_path,
        r#"
[server]
host = "0.0.0.0"
port = 9999
"#,
    )
    .unwrap();

    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("deploy")
        .arg("-c")
        .arg(&config_path);

    cmd.assert()
        .success()
        .stdout(predicate::str::contains("host: 0.0.0.0"))
        .stdout(predicate::str::contains("port: 9999"));
}

#[test]
fn test_output_file_written() {
    let dir = tempfile::tempdir().unwrap();
    let output_file = dir.path().join("output.txt");

    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("parse")
        .arg("-f")
        .arg("testdata/sample.json")
        .arg("-o")
        .arg(&output_file);

    cmd.assert().success();

    let content = std::fs::read_to_string(&output_file).unwrap();
    assert!(content.contains("key"));
}
```

---

## 7. Testing Timeouts and Signal Handling

```rust
#[test]
fn test_interrupt_handling() {
    let mut cmd = Command::cargo_bin("devkit").unwrap();
    cmd.arg("deploy")
        .arg("--port")
        .arg("3000")
        .timeout(std::time::Duration::from_secs(1));

    let output = cmd.assert();

    let stdout = String::from_utf8_lossy(&output.get_output().stdout);
    let stderr = String::from_utf8_lossy(&output.get_output().stderr);

    assert!(
        stdout.contains("shutting down") || stderr.contains("shutting down"),
        "graceful shutdown message not found"
    );
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: every subcommand has at least one `assert_cmd` integration test | Manual testing doesn't scale. Integration tests catch regressions in argument parsing, exit codes, and output formatting. |
| **Forbid** hardcoded paths in tests | Always use `tempfile::tempdir()`. Never write to the real filesystem during tests. |
| **Forbid** relying on global state or env vars without cleanup | Tests must be isolated. Use `env` method on `Command` (not `std::env::set_var`). Use `tempdir` for files. |
| **Forbid** comparing binary output with string predicates | Use `predicate::eq` on `Vec<u8>` for exact binary matching. Strings may fail on different encodings. |
| **Mandatory**: snapshot tests reviewed in PR | `trycmd` diffs must be reviewed like code. Broken snapshots mean changed behavior — intentional or not. |

---

## References
- [assert_cmd documentation](https://docs.rs/assert_cmd/latest/assert_cmd/)
- [trycmd documentation](https://docs.rs/trycmd/latest/trycmd/)
- [snapbox documentation](https://docs.rs/snapbox/latest/snapbox/)
- [predicates crate](https://docs.rs/predicates/latest/predicates/)