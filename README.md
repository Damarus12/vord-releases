# Vord Releases

Pre-built binaries for [Vord](https://github.com/soncraftbot/vord) — deploy Docker containers to any VM. No SSH, no YAML, no nonsense.

## Install

**macOS / Linux:**

```sh
curl -fsSL https://github.com/Damarus12/vord-releases/releases/latest/download/install.sh | bash
```

**Windows (PowerShell):**

```powershell
irm https://github.com/Damarus12/vord-releases/releases/latest/download/install.ps1 | iex
```

Restart your terminal, then verify:

```sh
vord version
```

### Install options

| Variable           | Default       | Description                                |
| ------------------ | ------------- | ------------------------------------------ |
| `VORD_VERSION`     | `latest`      | Install a specific version (e.g. `v0.2.0`) |
| `VORD_INSTALL_DIR` | `~/.vord/bin` | Custom install directory                   |

Example:

```sh
VORD_VERSION=v0.2.0 curl -fsSL .../install.sh | bash
```

### Update

```sh
vord update
```

## Available binaries

| Binary                   | OS      | Arch          | Description             |
| ------------------------ | ------- | ------------- | ----------------------- |
| `vord-darwin-amd64`      | macOS   | Intel         | CLI                     |
| `vord-darwin-arm64`      | macOS   | Apple Silicon | CLI                     |
| `vord-linux-amd64`       | Linux   | x86_64        | CLI                     |
| `vord-linux-arm64`       | Linux   | ARM64         | CLI                     |
| `vord-windows-amd64.exe` | Windows | x86_64        | CLI                     |
| `vord-agent-linux-amd64` | Linux   | x86_64        | Agent (runs on servers) |
| `vord-agent-linux-arm64` | Linux   | ARM64         | Agent (runs on servers) |

## Verify release integrity

Every binary is SHA256 checksummed and the checksum file is signed with [cosign](https://github.com/sigstore/cosign) (Sigstore keyless). The install scripts verify checksums automatically.

To verify manually:

```sh
# Download release files
VERSION=v0.1.0
BASE=https://github.com/Damarus12/vord-releases/releases/download/$VERSION

# Verify checksums
sha256sum --check checksums.txt

# Verify signature (requires cosign)
cosign verify-blob \
  --signature checksums.txt.sig \
  --certificate checksums.txt.pem \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'github.com/soncraftbot/vord' \
  checksums.txt
```

## Documentation

Full documentation, configuration reference, and source code: **[github.com/soncraftbot/vord](https://github.com/soncraftbot/vord)**
