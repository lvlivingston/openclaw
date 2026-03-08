# OpenClaw Operator Guide (Local Deployment)

This document describes how **this repository’s OpenClaw deployment** is installed,
operated, secured, and maintained.

It is intentionally **deployment-specific** and does not duplicate upstream
OpenClaw documentation. For general product documentation, see:

- https://docs.openclaw.ai
- The upstream repositroy `README.md`

This guide acts as a **personal runbook for stable long-lived operation**.

---

## 1. Purpose & Scope

This document exists to:

- Prevent accidental token rotation or state loss
- Clarify what is safe vs unsafe to reset
- Make the system reproducible after downtime or VPS rebuild
- Act as a personal runbook for long-lived operation
- Document the validated steady-state configuration

This guide assumes the operator is comfortable with:

- Basic networking
- Docker
- Linux
- SSH
- TLS / HTTPS
- Virtual Private Servers

This deployment's goal is **security-first and intentionally non-public**.

---

## 2. Architecture Overview

### High-level design

This document reflects the **currently paired and validated production configuration**.

If pairing or runtime state breaks, restore from a snapshot rather than improvizing token resets.

- **Version**: OpenClaw v2026.3.2
- **Host**: Hetzner Virtual Private Server (VPS)
- **OS**: Ubuntu 24.04
- **Model Provider**: VeniceAI (Privacy focused)
- **Default Model (Main Agent)**: `venice/kimi-k2-5`
- **Runtime**: Docker Compose
  - `openclaw-gateway`
    - Primary runtime service. Responsibilities:
      - Agent orchestration
      - Channel ingestion
      - Model inference routing
      - Tool execution coordination
      - Sandbox spawning
    - Container:
      `openclaw-openclaw-gateway-1`
    - Ports:
      - 127.0.0.1:18789 → gateway UI / websocket
      - 127.0.0.1:18790 → bridge
      - Both ports are **loopback-only**.
  - `openclaw-cli` (ephemeral container use)
    - Used for:
      - device management
      - pairing
      - diagnostics
      - operator actions
      - The CLI runs as **ephemeral containers**. Example:
        - `docker compose run --rm openclaw-cli devices list`
- **Execution Model (Sandbox)**:
  - All agent tool execution occurs inside **ephemeral Docker sandbox containers**.
  - Configuration:
    - `agents.defaults.sandbox.mode = all`
  - Meaning:
    - All sessions sandboxed
    - Every tool execution is sandboxed
    - Containers are spawned dynamically
    - Containers terminate after session use
    - Example container:
      `openclaw-sbx-agent-default-*`
  - Requirements:
    - For sandboxing to function the gateway must be able to control Docker. The gateway container therefore has:
      - Docker socket
        - `/var/run/docker.sock` mounted into the container
      - Docker CLI
        - The gateway image is built with:
          - `OPENCLAW_INSTALL_DOCKER_CLI=1`
        - This installs Docker inside the container.
      - Docker group access
        - Container user:
          - `node`
        - Group membership includes the host docker group.
        - This allows sandbox containers to be created.
- **Ingress surfaces**:
  - Tailscale
    - Access exposed exclusively via Tailscale Serve (HTTPS + MagicDNS)
    - Gateway bound to `127.0.0.1:18789` (loopback only)
  - Telegram (polling mode)
    - Bot: @<botname>\_bot
    - dmPolicy: allowlist
    - groupPolicy: allowlist
    - allowFrom restricted to specific Telegram sender IDs
    - No webhook exposure (outbound polling only)
  - Google Workspace APIs (OAuth via gogcli)
    - Used for Google Calendar (and optionally Gmail)
    - Authentication performed via OAuth using a Desktop App client
    - Tokens stored in `~/.config/gogcli` and mounted into the gateway container
    - OAuth client secret stored locally in `~/.secrets` (never committed to Git)
    - Access occurs via outbound HTTPS to Google APIs only
    - No inbound ports or webhook exposure
- **Clients**:
  - macOS browser over tailnet HTTPS (operator role)
  - CLI container on VPS (uses node device identity)
- **Nodes**:
  - Single Hetzner VPS node (runs inside Docker)

No public inbound ports are exposed on the VPS firewall.

⚠️ Docker socket access is high-privilege interface and is acceptable only because:

- Gateway UI is not publicly exposed
- Gateway binds only to loopback-only
- Access is restricted via Tailscale identity
- Sandbox mode is enforced for all sessions
- Web tools are denied unless explicitly enabled

### Network Access Model

The gateway is **not exposed publicly**.

#### Local binding

Gateway ports:

`127.0.0.1:18789`
`127.0.0.1:18790`

Docker mapping:

`127.0.0.1:18789 → 18789`
`127.0.0.1:18790 → 18790`

No service binds to:

`0.0.0.0`

### Ingress via Tailscale

External access occurs **only via Tailscale Serve**.

Configuration:

`tailscale serve https / http://127.0.0.1:18789`

Result:

`https://<hostname>.<tailnet>.ts.net`

This endpoint is:

- HTTPS
- MagicDNS
- tailnet-restricted

Only **authorized tailnet devices** can access it.

### Actual Running Topology

```
MacOS (Browser - Operator)
    ↓ HTTPS over Tailscale Tailnet (MagicDNS)
tailscale serve (TLS termination)
    ↓
http://127.0.0.1:18789
    ↓
OpenClaw Gateway (Docker)
    ↓ /var/run/docker.sock + docker CLI
Docker daemon (on VPS)
    ↓
Sandbox Containers
```

All traffic remains:

- Loopback-bound
- Tailnet-restricted
- Not reachable from the public internet

The node connects outbound to the gateway over loopback WebSocket.
The gateway does not initiate execution connections.

- All services must mount:
  ` ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw`

⚠️ **Do not rerun onboarding after node pairing is stable.**

### Channel Ingress (Telegram)

- Telegram uses outbound polling only.
- No public webhook endpoint is exposed.
- No inbound firewall ports are opened for Telegram.

```
Telegram Bot API (Outbound HTTPS polling)
    ↓
OpenClaw Gateway (Docker container)
    ↓
Agent binding → default
```

### Outbound API Access (Google Workspace)

- Google services are accessed via outbound HTTPS using the `gog` CLI.
- Authentication is performed via OAuth.

```
OpenClaw Gateway (Docker container)
    ↓
gog CLI
    ↓
Google OAuth tokens
    ↓
Google APIs
```

Characteristics:

- No inbound listener
- No webhook exposure
- No firewall port opened
- Tokens stored locally in `~/.config/gogcli`
- OAuth client secret stored locally in `~/.secrets`
- Access originates from the gateway container only

### Time & Clock Integrity

The infrastructure, including the VPS, runs in UTC.

```Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
NTP service: active
```

- Application-layer scheduling may convert to operator-local time.
- NTP must remain enabled to prevent token expiry issues, cron drift, and inconsistent logs.

To verify:

```bash
timedatectl
```

---

## 3. Trust & Identity Model

### Single Intended Operator

This deployment is designed for **a single operator**.

| Device        | Role                  |
| ------------- | --------------------- |
| VPS           | execution environment |
| Browser       | operator              |
| CLI container | operator tooling      |

No multi-user or public access model is supported.

### Workspace Identity Files (Persistent State)

Runtime identity files exist inside:
`~/.openclaw/workspace/`

The following files define long-term identity and reasoning posture:

- `SOUL.md`
- `IDENTITY.md`
- `USER.md`
- `ROLE.md`
- `TOOLS.md`

These represent the **agent personality, reasoning state, behaviors, and tools**.

These files are part of runtime state, not Git state.

Deleting `~/.openclaw` removes:

- Device identities
- Pairing tokens
- Workspace files
- Session history

---

## 4. Repository vs Runtime State (Critical)

### In Git (versioned)

- OpenClaw source (code)
- Docker configuration (docker-compose.yml)
- Documentation (`docs/`)
- Architectural decisions (operational runbooks)

### On VPS (persistent, NOT in git)

- `~/.openclaw/`
  - device identity
  - pairing tokens
  - workspace files
  - sessions
  - memory
- Docker volumes
- Tailscale state
- Provider API keys (via environment variables)

⚠️ **Deleting `~/.openclaw` will invalidate device tokens and sessions.**

### Ephemeral / Safe to regenerate

- Dashboard URL token
- SSH tunnels
- Browser session storage

---

## 5. Network Access Model (Zero Public Ports)

Gateway binding:

- `127.0.0.1:18789`
- Docker mapping: `127.0.0.1:18789 -> 18789/tcp`

Exposure method:

- `tailscale serve`
  - Terminates TLS
  - Enforces tailnet identity
  - Proxies to `http://127.0.0.1:18789`

The gateway is NOT:

- Bound to `0.0.0.0`
- Exposed via public firewall ports
- Reachable via raw Tailscale IP + port
- Intended for SSH tunneling as primary access

Only MagicDNS HTTPS endpoint is supported.

---

## 6. Provisioning the VPS (Hetzner)

### Baseline assumptions

- Ubuntu 24.04 LTS
- Non-root user with sudo
- SSH key-based access only

### Security hardening (recommended)

- Disable password SSH
- Enable UFW
- Allow only:
  - SSH
  - Tailscale
- Optional: fail2ban

---

## 7. Docker & System Dependencies

### Required packages

- Docker Engine
- Docker Compose plugin
- Build tools (installed by OpenClaw installer)
- Node.js (installed by OpenClaw installer)

Docker is the only supported runtime for long-lived operation.

---

## 8. Tailscale

- Connect to Docker and device.
- Must be downloaded to individual devices

---

## 9. Pairing Model (Validated State)

As of 03/08/26:

### Devices

1. **Hetzner VPS Device**
   - Role: `node`
   - Runs inside Docker
   - Provides execution capability

2. **Browser Operator Device**
   - Role: `operator`
   - Used for admin and UI access

3. **CLI Container**
   - Uses same identity as VPS node
   - Has operator scope attached
   - Used for:
     - `devices list`
     - `devices approve`
     - `nodes status`

⚠️ If `~/.openclaw` is wiped, all pairing must be redone.

Do not partially delete identity files unless intentionally resetting the entire environment.

---

## 10. Token & Credential Model

There are multiple token types.

### Dashboard token

- Embedded in UI URL
- Ephemeral
- Safe to regenerate

### Gateway auth token

- Used for initial client connection
- Must match on both sides
- Rotating breaks clients until updated

### Device / node tokens

- Issued after pairing
- Persisted in `~/.openclaw`
- Tied to specific device identity

### Provider API keys (e.g., VeniceAI)

- Independent of gateway auth
- Stored as environment variables
- Rotating does NOT require gateway reset

### Telegram Bot Token

- Stored in `.env`
- Used by Telegram ingress adapter
- Never committed to Git

### Google OAuth Client Secret

- Stored locally in `~/.secrets/client_secret_*.json`
- Never committed to Git (GitHub push protection blocks these)
- Used only for initial `gog auth credentials` bootstrap
- If exposed, rotate immediately in Google Cloud Console

⚠️ Never rotate multiple token types at once.

---

## 11. OpenClaw Installation Strategy

### Chosen method

- **Installer**: official OpenClaw installer
- **Mode**: git checkout (not npm global)
- **Reason**:
  - visibility into source
  - predictable updates
  - easier debugging

### Canonical install steps

1. Clone repo
2. Run installer (`curl … | bash`)
3. Ensure PATH includes `~/.local/bin`
4. Run `pnpm install`
5. Run `openclaw update`
6. Start gateway

⚠️ `openclaw update` will refuse to run if the git working tree is dirty.

---

## 12. Update & Maintenance Policy

### Updating OpenClaw

- Ensure git tree is clean
- Run `openclaw update`
- Verify gateway and node status
- Confirm dashboard loads correctly
- Confirm devices and node still paired

Long builds may cause SSH disconnects.
Always verify system state after reconnecting.

---

## 13. Backups & Recovery

### Code Snapshot (Git)

Always checkpoint working state:

```bash
git add -A
git commit -m "checkpoint: <description>"
git tag checkpoint-$(date +%Y%m%d-%H%M)
git push origin HEAD
git push origin checkpoint-YYYYMMDD-HHMM
```

### Runtime State Snapshot (Critical)

Persistent state:
`/home/taylor/.openclaw`

#### Backup:

```bash
tar -czf /home/<sudo>/openclaw-state-$(date +%Y%m%d-%H%M).tgz \
  -C /home/<sudo> .openclaw
```

#### Restore:

```bash
docker compose down
rm -rf /home/<sudo>/.openclaw
tar -xzf openclaw-state-YYYYMMDD-HHMM.tgz -C /home/<sudo>
docker compose up -d
```

⚠️ Never copy .openclaw between different machines.
Each deployment must generate its own pairing state.

---

## 14. References

- Community Hetzner + Tailscale OpenClaw guide: https://gist.github.com/thedudeabidesai/eb9490c031d869313142368150a060e9
- Pulumi cloud deployment ideas: https://www.pulumi.com/blog/deploy-openclaw-aws-hetzner/
- Zero-trust Tailscale deployment patterns: https://alirezarezvani.medium.com/i-deployed-openclaw-with-zero-public-ports-here-is-the-tailscale-setup-that-actually-works-86f8c9e6f158

---

## 15. Deployment Reference State (As of 03/08/26)

This section documents the validated, working steady-state configuration.
If the system drifts from this state, investigate before rotating tokens.

### Current State (✅ expected)

- Gateway running in Docker
- `/var/run/docker.sock` mounted
- Docker CLI installed inside gateway image
- Gateway user `node` in docker group
- Sandbox mode `"all"` enabled
- Execution environment: Hetzner VPS Docker host
- Sandbox containers spawned dynamically by gateway
- One paired operator (browser)
- CLI uses VPS identity
- One main agent configured
- Default model: VeniceAI `venice/kimi-k2-5`
- Google Workspace integration enabled via `gog` CLI
- OAuth tokens persisted in `~/.config/gogcli`
- OAuth client secret stored locally in `~/.secrets`
- Google API access is outbound HTTPS only
- Gateway bound to `127.0.0.1:18789`
- Access only via MagicDNS HTTPS over tailnet
- Gateway UI is reachable only at:
  - `https://<vps-hostname>.<tailnet>.ts.net/…` (MagicDNS over tailnet)
- No public ports exposed
- Gateway token and VeniceAI API key only availabe in `.env`

### System is NOT designed to:

- Expose `:18789` publicly
- Bind gateway to `0.0.0.0`
- Use raw Tailscale IP access
- Be accessed via `http://<tailscale-ip>:18789`
- Support multi-operator public access

All traffic remains:

- Loopback-bound
- Tailnet-restricted
- Identity-controlled
- Outbound API access via HTTPS only

### Operator Note for next session

Google integration is authenticated through the `gog` CLI on the gateway container.

The Executive Assistant agent runs inside an OpenClaw sandbox.
The sandbox does not contain the `gog` binary, so direct CLI access from the agent fails.

Future work will introduce **host-side wrapper commands** that safely expose calendar and Gmail functionality without granting the sandbox direct access to `gog`.

Baseline snapshot for this state:

Git tag:
baseline-pre-wrapper-20260308-1150

Local backups:
~/openclaw/local-backups/openclaw.json.baseline._
~/openclaw/local-backups/workspace-executive-assistant._
