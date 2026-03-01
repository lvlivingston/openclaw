# OpenClaw Operator Guide (Local Deployment)

This document describes how **this repository’s OpenClaw deployment** is installed,
operated, secured, and maintained.

It is intentionally **deployment-specific** and does not duplicate upstream
OpenClaw documentation. For general product documentation, see the root `README.md`
and https://docs.openclaw.ai.

---

## 1. Purpose & Scope

This document exists to:

- Prevent accidental token rotation or state loss
- Clarify what is safe vs unsafe to reset
- Make the system reproducible after downtime or VPS rebuild
- Act as a personal runbook for long-lived operation
- Document the validated steady-state configuration

This guide assumes the operator is comfortable with Virtual Private Servers, Linux, SSH, Docker, TLS, and basic networking.

This deployment is security-first and intentionally non-public.

---

## 2. Architecture Overview

### High-level design

This document reflects the currently paired and validated production configuration.
If pairing breaks, restore from snapshot rather than improvizing token resets.

- **Version**: OpenClaw v2026.2.26
- **Host**: Hetzner Virtual Private Server (VPS) (Ubuntu 24.04)
- **Runtime**: Docker Compose
  - `openclaw-gateway`
  - `openclaw-node`
  - `openclaw-cli` (ephemeral container use)
- **Model Provider**: VeniceAI (Privacy focused)
- **Default Model (Main Agent)**: `venice/llama-3.3-70b`
- **Sandboxing**:
  - agents.defaults.sandbox.mode = all
  - All sessions sandboxed
  - Gateway allowed to spawn sandbox containers via Docker socket
    - `/var/run/docker.sock` mounted into gateway
    - `/usr/bin/docker` mounted read-only
    - gateway container user `node` in docker group
- **Network access**:
  - Gateway bound to `127.0.0.1:18789` (loopback only)
  - Access exposed exclusively via Tailscale Serve (HTTPS + MagicDNS)
- **Clients**:
  - macOS browser over tailnet HTTPS (operator role)
  - CLI container on VPS (uses node device identity)
- **Nodes**:
  - Single Hetzner VPS node (runs inside Docker)

No public ports are exposed on the VPS.

⚠️ Docker socket access is high-privilege interface and is acceptable only because:

- Gateway UI is loopback-only + Tailscale-only
- No public ingress
- Sandbox mode is enforced for all sessions
- Web tools are denied unless explicitly enabled

### Actual Running Topology

```
Mac (Browser - Operator)
    ↓ HTTPS over Tailscale (MagicDNS)
tailscale serve (TLS termination)
    ↓
http://127.0.0.1:18789
    ↓
OpenClaw Gateway (Docker, run as user: node)
    ↓ /var/run/docker.sock + docker CLI
Host Docker daemon (on VPS)
    ↓
Ephemeral Sandbox Containers (per-session tool runs)
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

This deployment is designed for a single human operator.

There is:

- One paired operator device (browser)
- One paired node device (VPS)
- CLI uses the VPS device identity

No multi-user or public access model is supported.

### Workspace Identity Files (Persistent State)

The following files define long-term identity and reasoning posture:

- `SOUL.md`
- `IDENTITY.md`
- `USER.md`

These are stored inside:
`~/.openclaw/workspace/`

These files are part of runtime state, not Git state.

Deleting `~/.openclaw` removes:

- Device identities
- Pairing tokens
- Workspace files
- Session memory

---

## 4. Repository vs Runtime State (Critical)

### In Git (versioned)

- OpenClaw source code
- Docker configuration
- Documentation (`docs/`)
- Architectural decisions

### On VPS (persistent, NOT in git)

- `~/.openclaw/`
  - device identities
  - pairing state
  - workspace files
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

## 8. Tailscale

- Connect to Docker and device.
- Must be downloaded to individual devices

---

## 9. Pairing Model (Validated State)

As of 02/28/26:

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

## 8. Token & Credential Model

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

⚠️ Never rotate multiple token types at once.

---

## 6. OpenClaw Installation Strategy

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

## 10. Update & Maintenance Policy

### Updating OpenClaw

- Ensure git tree is clean
- Run `openclaw update`
- Verify gateway and node status
- Confirm dashboard loads correctly
- Confirm devices and node still paired

Long builds may cause SSH disconnects.
Always verify system state after reconnecting.

---

## 11. Backups & Recovery

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
tar -czf /home/taylor/openclaw-state-$(date +%Y%m%d-%H%M).tgz \
  -C /home/taylor .openclaw
```

#### Restore:

```bash
docker compose down
rm -rf /home/taylor/.openclaw
tar -xzf openclaw-state-YYYYMMDD-HHMM.tgz -C /home/taylor
docker compose up -d
```

⚠️ Never copy .openclaw between different machines.
Each deployment must generate its own pairing state.

---

## 12. References

- Community Hetzner + Tailscale OpenClaw guide: https://gist.github.com/thedudeabidesai/eb9490c031d869313142368150a060e9
- Pulumi cloud deployment ideas: https://www.pulumi.com/blog/deploy-openclaw-aws-hetzner/
- Zero-trust Tailscale deployment patterns: https://alirezarezvani.medium.com/i-deployed-openclaw-with-zero-public-ports-here-is-the-tailscale-setup-that-actually-works-86f8c9e6f158

---

## 13. Deployment Reference State (As of 02/28/26)

This section documents the validated, working steady-state configuration.
If the system drifts from this state, investigate before rotating tokens.

### Current State (✅ expected)

- Gateway running in Docker
- `/var/run/docker.sock` mounted
- `/usr/bin/docker` mounted read-only
- Gateway user `node` in docker group
- Sandbox mode `"all"` enabled
- One paired node (VPS)
- One paired operator (browser)
- CLI uses VPS identity
- One main agent configured
- Default model: VeniceAI `venice/llama-3.3-70b`
- Gateway bound to `127.0.0.1:18789`
- Access only via MagicDNS HTTPS over tailnet
- Gateway UI is reachable only at:
  - `https://<vps-hostname>.<tailnet>.ts.net/…` (MagicDNS over tailnet)
- No public ports exposed
- Gateway token and VeniceAI API key only availabe in `.env`

System is NOT designed to:

- Expose `:18789` publicly
- Bind gateway to `0.0.0.0`
- Use raw Tailscale IP access
- Be accessed via `http://<tailscale-ip>:18789`
- Support multi-operator public access

All traffic remains:

- Loopback-bound
- Tailnet-restricted
- Identity-controlled
