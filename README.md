Release binaries for

# Vord

**Virtual Orchestration & Remote Deployment**

Deploy Docker containers to any VM. No SSH after setup, no YAML playbooks, no runtime dependencies.

```
your machine         vord deploy         your servers
  vord CLI   ────── HTTPS + token ──────►  vord-agent ──► Docker
```

One binary on your machine, one binary on each server. The CLI reads `vord.toml` from your project, talks to agents over HTTPS, and agents manage Docker. No control plane, no central server, no Ruby, no Python.

## Install

**macOS / Linux:**

```sh
curl -fsSL https://raw.githubusercontent.com/Damarus12/vord-releases/main/install.sh | bash
```

**Windows (PowerShell):**

```powershell
irm https://raw.githubusercontent.com/Damarus12/vord-releases/main/install.ps1 | iex
```

Verify:

```sh
vord version
```

To update later: `vord update`

## Quick start

### 1. Initialize your project

```sh
cd my-app
vord init
```

This creates:

| File                | What it is        | Commit?             |
| ------------------- | ----------------- | ------------------- |
| `vord.toml`         | Project config    | Yes                 |
| `.vord/secrets.enc` | Encrypted secrets | Yes                 |
| `.vord/key`         | Encryption key    | **No** (gitignored) |
| `.vord/token`       | Deploy token      | **No** (gitignored) |

### 2. Add a server

You need a Linux server with Docker installed. Vord handles the rest.

```sh
vord host add prod --address 203.0.113.10
```

This SSHs into the server **once** to:

1. Upload the agent binary
2. Set up the deploy token
3. Generate a TLS certificate
4. Create a systemd service
5. Write the host config + TLS fingerprint to `vord.toml`

**After this step, SSH is never used again.** All future communication is over HTTPS.

### 3. Configure a service

Edit `vord.toml`:

```toml
[project]
name = "my-app"

[hosts.prod]
address = "203.0.113.10"
tags = ["prod"]

[services.api]
image = "ghcr.io/myorg/api:latest"
hosts = "tag:prod"
ports = { "http" = "8080:8080" }

[services.api.health]
path = "/health"
port = 8080
```

### 4. Deploy

```sh
vord deploy
```

Vord pulls the image, stops the old container, starts the new one, and verifies the health check. If the health check fails, it **rolls back to the previous version automatically**.

### 5. Check on things

```sh
vord status                     # fleet overview
vord logs api -f                # stream logs
vord exec api -- env            # run a command in the container
```

## Commands

| Command                        | What it does                                             |
| ------------------------------ | -------------------------------------------------------- |
| `vord init`                    | Initialize a new project (creates vord.toml, token, key) |
| `vord deploy [service]`        | Deploy all services (or one) to target hosts             |
| `vord status`                  | Show all hosts and running services                      |
| `vord logs <service> [-f]`     | Stream container logs (auto-finds the host)              |
| `vord exec <service> -- <cmd>` | Run a command inside a container                         |
| `vord host add <name>`         | Add a server and bootstrap the agent                     |
| `vord host list`               | List configured hosts                                    |
| `vord host remove <name>`      | Remove a host from vord.toml                             |
| `vord secret set <KEY=VALUE>`  | Set an encrypted secret                                  |
| `vord secret list`             | List secret keys (values are never shown)                |
| `vord secret import .env`      | Bulk import secrets from a .env file                     |
| `vord proxy status`            | Show reverse proxy status on all hosts                   |
| `vord proxy sync`              | Force-sync proxy config to all hosts                     |
| `vord agent update`            | Push a new agent binary to all hosts                     |
| `vord update`                  | Update the CLI itself                                    |
| `vord setup`                   | Onboard a teammate (they paste the shared key)           |
| `vord drain <service>`         | Gracefully stop a service                                |

## Configuration

Everything lives in `vord.toml` in your project root. Here's a complete example:

```toml
[project]
name = "my-app"

# ── Proxy (optional) ──────────────────────────────────────────
# Enables auto-managed Caddy with Let's Encrypt.
# Only needed if any service has a "domain" field.
[proxy]
email = "devops@example.com"

# ── Registry (optional) ───────────────────────────────────────
# For private Docker images. Password is loaded from secrets.
[registry]
server = "ghcr.io/myorg"
username = "deploy-bot"

# ── Deploy defaults ───────────────────────────────────────────
[deploy]
strategy = "rolling"          # rolling | immediate
health_timeout = "90s"

# ── Hosts ─────────────────────────────────────────────────────

[hosts.prod-us]
address = "203.0.113.10"
tags = ["prod", "us"]
# fingerprint is captured automatically on first connect

[hosts.prod-eu]
address = "203.0.113.20"
tags = ["prod", "eu"]
vars = { REGION = "eu-west-1" }     # per-host env vars

# ── Tag variables ─────────────────────────────────────────────
# Env vars applied to all hosts with a matching tag.

[vars.us]
REGION = "us-east-1"

[vars.prod]
LOG_LEVEL = "warn"

# ── Services ──────────────────────────────────────────────────

[services.api]
image = "my-app:latest"
hosts = "tag:prod"                    # deploy to all "prod" hosts
domain = "api.example.com"            # auto-configures HTTPS proxy
ports = { "http" = "8080:8080" }
volumes = ["app_data:/data"]
command = ["node", "server.js"]       # override container CMD
env = { NODE_ENV = "production", HOST = "{{.Host.Name}}" }
extra_hosts = ["myhost:10.0.0.1"]     # --add-host entries
docker_args = ["--memory=512m"]       # passthrough to docker run

[services.api.health]
path = "/health"
port = 8080

[services.api.deploy]
strategy = "rolling"
health_timeout = "60s"
```

### Host selectors

The `hosts` field on each service controls where it deploys:

```toml
hosts = "tag:prod"          # all hosts tagged "prod"
hosts = "prod-us,prod-eu"   # specific hosts by name
```

### Environment variable priority

When the same key is defined in multiple places, highest priority wins:

1. **Service `env`** — explicit values and Go templates
2. **Secrets** — from `.vord/secrets.enc`
3. **Host `vars`** — per-host overrides
4. **Tag `vars`** — tag-based defaults

Templates have access to host metadata:

```toml
env = { HOST_NAME = "{{.Host.Name}}", ADDR = "{{.Host.Address}}" }
```

## Reverse proxy (HTTPS)

Add a `domain` to any service and Vord manages Caddy automatically:

```toml
[proxy]
email = "you@example.com"

[services.api]
domain = "api.example.com"
ports = { "http" = "8080:8080" }

[services.web]
domain = "app.example.com"
ports = { "http" = "3000:3000" }
```

On deploy, Vord:

1. Generates a Caddyfile from all domain-bearing services on the host
2. Starts a Caddy container (or reloads if already running)
3. Caddy handles **Let's Encrypt certificates**, HTTPS, and HTTP-to-HTTPS redirects

You get production HTTPS with zero cert management. Point your DNS to the server and deploy.

To check or force a sync: `vord proxy status` / `vord proxy sync`

## Secrets

Secrets are encrypted at rest using [age](https://age-encryption.org/) (X25519 + ChaCha20-Poly1305).

```sh
vord secret set DATABASE_URL=postgres://...    # inline
vord secret set API_KEY                        # prompts for value (hidden)
vord secret import .env.production             # bulk import from file
vord secret list                               # show keys (not values)
```

Secrets are automatically injected as environment variables on every deploy. The encrypted file (`.vord/secrets.enc`) is safe to commit. The key (`.vord/key`) is gitignored.

### Team sharing

One key unlocks everything. Share it once, securely (Slack DM, 1Password, etc.):

```sh
# Project owner
cat .vord/key          # copy this

# Teammate
vord setup             # paste the key — decrypts token, ready to deploy
```

## Health checks and rollback

Configure a health endpoint and Vord verifies it after every deploy:

```toml
[services.api.health]
path = "/health"
port = 8080

[services.api.deploy]
health_timeout = "60s"
```

Vord polls the endpoint with exponential backoff. If it doesn't return 200 within the timeout, Vord **automatically rolls back** to the previous image. The old image is already cached on the host, so rollback takes seconds.

During a rolling deploy, rollback happens per-host — if host 2 of 5 fails, only host 2 is rolled back and the remaining hosts are left untouched.

## Deploy strategies

**Rolling** (default) — one host at a time, health check between each. Safest option.

**Immediate** — all hosts concurrently. Use for workers or non-critical services.

```toml
[services.worker]
deploy = { strategy = "immediate" }
```

## Private registries

```toml
[registry]
server = "ghcr.io/myorg"
username = "deploy-bot"
```

```sh
vord secret set REGISTRY_PASSWORD=ghp_xxxxx
```

Vord authenticates with the registry before pulling. The password is loaded from secrets automatically.

## CI/CD

No SSH keys needed. Just a token.

**GitHub Actions:**

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Vord
        run: curl -fsSL https://raw.githubusercontent.com/Damarus12/vord-releases/main/install.sh | bash

      - name: Deploy
        run: vord deploy --skip-build
        env:
          VORD_TOKEN: ${{ secrets.VORD_TOKEN }}
```

More examples in [`examples/github-actions/`](examples/github-actions/) — including build+push+deploy and encrypted secrets in CI.

**Any CI system:**

```sh
export VORD_TOKEN="..."
curl -fsSL .../install.sh | bash
vord deploy --skip-build
```

## Multiple projects, same server

The agent is project-agnostic. If you add the same server to two different `vord.toml` files (with the same token), both projects deploy to it independently. Services are isolated by name via Docker labels.

## Updating

```sh
vord update              # update the CLI
vord agent update        # push new agent to all hosts
```

Agent updates are zero-downtime — the new binary is swapped and the service restarts automatically.

## How it works

```
┌──────────────┐         HTTPS + bearer token        ┌──────────────┐
│   vord CLI   │ ──────────────────────────────────►  │  vord-agent  │
│  (your box)  │   JSON requests, chunked streams     │  (each VM)   │
└──────────────┘                                      └──────┬───────┘
       │                                                     │
  reads vord.toml                                     manages Docker
  encrypts secrets                                    TLS self-signed cert
  orchestrates deploys                                lease-based locking
```

- **TLS TOFU** — The agent generates a self-signed certificate on setup. The fingerprint is stored in `vord.toml` and verified on every connection.
- **Bearer token auth** — A shared token authenticates all requests.
- **Lease locking** — The CLI acquires an exclusive lease before deploying. Two deploys to the same host can't run simultaneously.
- **Container labels** — All containers get `vord.managed=true` labels so the agent tracks only what it owns.

## Comparison

See [WHY.md](WHY.md) for detailed comparisons with Kamal, Coolify, CapRover, Ansible, and docker-compose.

**tl;dr** — Vord is for teams that want to deploy containers to VMs without SSH, without a control plane, without Ruby/Python, and with secrets and proxy management built in.

## Contributing

Requires Go 1.21+ and [Task](https://taskfile.dev).

```sh
git clone https://github.com/soncraftbot/vord
cd vord
task build       # build both binaries
task test        # run tests
```

See [CLAUDE.md](CLAUDE.md) for architecture details and package structure.

## License

MIT
