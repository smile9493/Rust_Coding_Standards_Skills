# 02 — Cross-Platform Distribution

## Status
**Approved.** `cargo-dist` is the recommended distribution pipeline. Cross-compilation with GitHub Actions matrix.

## Prerequisites
- `cargo install cargo-dist`
- `cargo dist init` (generates `.github/workflows/release.yml`)
- Rust targets: `x86_64-unknown-linux-gnu`, `x86_64-apple-darwin`, `aarch64-apple-darwin`, `x86_64-pc-windows-msvc`

---

## 1. Cargo.toml — Distribution Metadata

```toml
[package]
name = "devkit"
version = "1.0.0"
edition = "2024"

[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.dist]
inherits = "release"
debug = 0

[package.metadata.dist]
dist-version = "0.22.1"
cargo-dist-version = "0.26.0"
ci = ["github"]
targets = [
    "x86_64-unknown-linux-gnu",
    "x86_64-unknown-linux-musl",
    "x86_64-apple-darwin",
    "aarch64-apple-darwin",
    "x86_64-pc-windows-msvc",
]
installers = ["shell", "powershell", "homebrew", "npm"]

[package.metadata.dist.dependencies.homebrew]
tap = "devkit/homebrew-devkit"
name = "devkit"
```

---

## 2. GitHub Actions — Matrix CI Release Workflow

```yaml
name: Release

on:
  push:
    tags: ["v*"]

permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  build-binaries:
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
          - target: x86_64-apple-darwin
            os: macos-latest
          - target: aarch64-apple-darwin
            os: macos-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest

    runs-on: ${{ matrix.os }}

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Build release binary
        run: |
          cargo build --release --target ${{ matrix.target }}
          # Strip debug symbols (already handled by strip = true in release profile)

      - name: Package artifact
        shell: bash
        run: |
          case "${{ matrix.target }}" in
            *windows*) ext=".exe"; archive_ext=".zip" ;;
            *)         ext="";     archive_ext=".tar.gz" ;;
          esac

          ARCHIVE_NAME="devkit-${{ github.ref_name }}-${{ matrix.target }}${archive_ext}"
          BIN_NAME="devkit${ext}"

          mv "target/${{ matrix.target }}/release/${BIN_NAME}" "${BIN_NAME}"

          if [[ "${archive_ext}" == ".zip" ]]; then
            7z a "${ARCHIVE_NAME}" "${BIN_NAME}"
          else
            tar czf "${ARCHIVE_NAME}" "${BIN_NAME}"
          fi

          echo "ARCHIVE_NAME=${ARCHIVE_NAME}" >> $GITHUB_ENV

      - name: Upload archive
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.target }}
          path: ${{ env.ARCHIVE_NAME }}

      - name: Publish to GitHub Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ env.ARCHIVE_NAME }}
          generate_release_notes: true
```

---

## 3. Shell Installer — `curl | sh` Pattern

```sh
#!/bin/sh
set -eu

REPO="devkit-org/devkit"
BIN="devkit"
INSTALL_DIR="${HOME}/.local/bin"

detect_platform() {
    os=$(uname -s | tr '[:upper:]' '[:lower:]')
    arch=$(uname -m)
    case "$arch" in
        x86_64|amd64)   arch="x86_64" ;;
        aarch64|arm64)  arch="aarch64" ;;
        *) echo "Unsupported architecture: $arch"; exit 1 ;;
    esac
    echo "${arch}-${os}"
}

get_latest_version() {
    curl --silent "https://api.github.com/repos/${REPO}/releases/latest" \
        | grep '"tag_name":' \
        | sed -E 's/.*"v?([^"]+)".*/\1/'
}

install_binary() {
    version=$(get_latest_version)
    platform=$(detect_platform)
    case "$platform" in
        *darwin*) target="${platform}" ;;
        *linux*)  target="${platform}-musl" ;;
        *) target="${platform}-msvc" ;;
    esac

    archive="devkit-v${version}-${target}.tar.gz"
    download_url="https://github.com/${REPO}/releases/download/v${version}/${archive}"

    mkdir -p "${INSTALL_DIR}"
    echo "Downloading devkit v${version} for ${target}..."

    curl -fsSL "${download_url}" | tar xz -C "${INSTALL_DIR}"
    chmod +x "${INSTALL_DIR}/${BIN}"

    echo "Installed ${BIN} to ${INSTALL_DIR}/${BIN}"

    if ! echo "${PATH}" | grep -q "${INSTALL_DIR}"; then
        echo "Add ${INSTALL_DIR} to your PATH:"
        echo "  export PATH=\"${INSTALL_DIR}:\$PATH\""
    fi
}

install_binary
```

---

## 4. Homebrew Formula

```ruby
class Devkit < Formula
  desc "Developer toolkit for project scaffolding and deployment"
  homepage "https://github.com/devkit-org/devkit"
  version "1.0.0"

  on_macos do
    if Hardware::CPU.intel?
      url "https://github.com/devkit-org/devkit/releases/download/v1.0.0/devkit-v1.0.0-x86_64-apple-darwin.tar.gz"
      sha256 "abc123def456..."
    end
    if Hardware::CPU.arm?
      url "https://github.com/devkit-org/devkit/releases/download/v1.0.0/devkit-v1.0.0-aarch64-apple-darwin.tar.gz"
      sha256 "def789abc012..."
    end
  end

  on_linux do
    if Hardware::CPU.intel?
      url "https://github.com/devkit-org/devkit/releases/download/v1.0.0/devkit-v1.0.0-x86_64-unknown-linux-musl.tar.gz"
      sha256 "ghi345jkl678..."
    end
    if Hardware::CPU.arm?
      url "https://github.com/devkit-org/devkit/releases/download/v1.0.0/devkit-v1.0.0-aarch64-unknown-linux-musl.tar.gz"
      sha256 "mno901pqr234..."
    end
  end

  def install
    bin.install "devkit"
    generate_completions_from_executable(bin/"devkit", "completions", shells: [:bash, :zsh, :fish])
  end

  test do
    assert_match "devkit 1.0.0", shell_output("#{bin}/devkit --version")
  end
end
```

---

## 5. Profile Tuning for Size

```toml
[profile.release]
opt-level = "s"
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.release-debug]
inherits = "release"
debug = 1
```

---

## 6. npm Wrapper for JS Ecosystem

```json
{
  "name": "devkit-cli",
  "version": "1.0.0",
  "description": "Developer toolkit CLI via npm",
  "bin": {
    "devkit": "./run.js"
  },
  "os": ["linux", "darwin", "win32"],
  "cpu": ["x64", "arm64"],
  "scripts": {
    "postinstall": "node install.js"
  }
}
```

```js
// install.js
const { execSync } = require("child_process");
const os = require("os");
const path = require("path");
const fs = require("fs");

const VERSION = "1.0.0";
const BIN_DIR = path.join(__dirname, "bin");
const PLATFORM_MAP = {
  "linux-x64": "x86_64-unknown-linux-musl",
  "darwin-x64": "x86_64-apple-darwin",
  "darwin-arm64": "aarch64-apple-darwin",
  "win32-x64": "x86_64-pc-windows-msvc",
};

const platform = `${os.platform()}-${os.arch()}`;
const target = PLATFORM_MAP[platform];
const ext = os.platform() === "win32" ? ".exe" : "";

if (!target) {
  console.error(`Unsupported platform: ${platform}`);
  process.exit(1);
}

if (!fs.existsSync(BIN_DIR)) fs.mkdirSync(BIN_DIR, { recursive: true });

const url = `https://github.com/devkit-org/devkit/releases/download/v${VERSION}/devkit-v${VERSION}-${target}.tar.gz`;

execSync(`curl -fsSL "${url}" | tar xz -C "${BIN_DIR}"`, { stdio: "inherit" });
fs.chmodSync(path.join(BIN_DIR, `devkit${ext}`), 0o755);
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Recommended**: `strip = true` in release profile | Debug symbols should not ship to end users. Use `split-debuginfo = "packed"` when postmortem debugging is needed. |
| **Recommended**: `lto = true`, `codegen-units = 1` | Fat LTO produces smallest, fastest binaries. Accept longer CI build time; skip for rapid/dev builds. |
| **Preferred**: `musl` target for maximum Linux portability | Static linking avoids glibc version mismatches. glibc targets are acceptable for distro-specific packages where musl brings unnecessary complexity. |
| **Forbid** GitHub Release without checksums | Every binary archive must have a SHA256 checksum for integrity verification. |
| **Forbid** `curl | sh` without `set -eu` | Silent failures are indistinguishable from success. Fail explicitly. |

---

## References
- [cargo-dist book](https://opensource.axo.dev/cargo-dist/book/)
- [GitHub Actions matrix strategy](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idstrategy)
- [Homebrew formula cookbook](https://docs.brew.sh/Formula-Cookbook)
- [dtolnay/rust-toolchain action](https://github.com/dtolnay/rust-toolchain)