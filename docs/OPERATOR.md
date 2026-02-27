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

This guide assumes the operator is comfortable with Linux, SSH, Docker, and basic networking.

---

## 2. Architecture Overview

### High-level design

This document reflects the currently paired and validated production configuration.
If pairing breaks, restore from snapshot rather than improvising token resets.

- **Version**: OpenClaw v2026.2.22)
- **Host**: Hetzner Virtual Private Server (VPS) (Ubuntu 24.04)
- **Runtime**: Docker Compose
  - `openclaw-gateway`
  - `openclaw-node`
  - `openclaw-cli` (ephemeral container use)
- **Network access**:
  - Gateway bound to `127.0.0.1:18789` (host loopback only)
  - Access exposed via Tailscale Serve (HTTPS, MagicDNS)
- **Clients**:
  - macOS browser over tailnet HTTPS
  - CLI container on VPS (paired operator)
- **Nodes**:
  - Hetzner VPS node (runs inside Docker)

No public ports are exposed on the VPS.

### Actual Running Topology

```
Mac (Browser)
    ↓ HTTPS over Tailscale (MagicDNS)
tailscale serve (TLS termination)
    ↓
http://127.0.0.1:18789
    ↓
OpenClaw Gateway (Docker)
    ↑ WebSocket (127.0.0.1)
OpenClaw Node (Docker, same VPS)
```

All traffic remains:

- Bound to loopback
- Exposed only via tailnet
- Not reachable from public internet

The node connects outbound to the gateway over loopback WebSocket.
The gateway does not initiate execution connections.

### Trust boundaries

- Gateway runs server-side and owns:
  - execution context
  - agent coordination
- Device nodes (macOS) run:
  - screen, camera, notifications
  - local system actions

No public ports are exposed directly on the VPS.

---

## 3. Repository vs Runtime State (Critical)

### In Git (versioned)

- OpenClaw source code
- Docker configuration
- Documentation (`docs/`)
- Operator decisions and notes

### On VPS (persistent, NOT in git)

- `~/.openclaw/`
  - device identities
  - credentials
  - workspace files
- Docker volumes
- Tailscale state

⚠️ **Deleting `~/.openclaw` will invalidate device tokens and sessions.**

### Ephemeral / Safe to regenerate

- Dashboard URL token (`/#token=...`)
- SSH tunnels
- Browser session storage

---

## 4. Provisioning the VPS (Hetzner)

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

## 5. Docker & System Dependencies

### Required packages

- Docker Engine
- Docker Compose plugin
- Build tools (installed by OpenClaw installer)
- Node.js (installed by OpenClaw installer)

Docker is the only supported runtime for long-lived operation.

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

## 7. Network Access Model (Zero Public Ports)

### Gateway binding

- Gateway is bound to loopback:
  - `127.0.0.1:18789`
- Docker port mapping:
  - `127.0.0.1:18789 -> 18789/tcp`

### Exposure method

Access is provided via:

- `tailscale serve`
  - Terminates TLS
  - Enforces tailnet identity
  - Proxies to `http://127.0.0.1:18789`

The gateway is NOT:

- Bound to 0.0.0.0 publicly
- Exposed via public firewall ports
- Reachable via raw Tailscale IP + port

Only the MagicDNS HTTPS endpoint should be used.

---

## 8. Pairing Model (Current Working State)

As of 02/27/26, the system has:

### Paired Devices

1. **Hetzner Node device**
   - Roles: `node`
   - Runs inside Docker on VPS
   - Connected

2. **Operator device (Dashboard browser)**
   - Role: `operator`
   - Used for admin actions via UI

3. **CLI container (same device identity as Hetzner Node)**
   - Uses the same `~/.openclaw` identity as the node
   - Has `operator` scope attached to that identity
   - Required for CLI commands like:
     - `devices list`
     - `devices approve`
     - `nodes status`

### Important Principle

Each client context (browser, CLI container, node) must be paired individually.

Operator pairing is required for:

- Approving devices
- Viewing node status
- Admin actions
- Gateway access

Node pairing is required for:

- Execution capabilities
- Browser/system actions

⚠️ If `~/.openclaw` is wiped, all pairing must be redone.
Do not partially delete identity files unless intentionally resetting the environment.

---

## 9. Token & Credential Model (READ THIS FIRST)

There are **multiple token types**. Confusing them causes most failures.

### Dashboard token

- Embedded in UI URL
- Ephemeral
- Safe to ignore or regenerate

### Gateway auth token

- Used for initial client connection
- Must match on both sides
- Rotating it breaks all clients until updated

### Device / node tokens

- Issued after pairing
- Persisted in `~/.openclaw`
- Tied to specific device identity

### Provider API keys (e.g. VeniceAI)

- Independent of gateway auth
- Stored as environment variables or via onboarding
- Rotating does NOT require gateway reset

⚠️ Never rotate multiple token types at once.

---

## 10. Update & Maintenance Policy

### Updating OpenClaw

- Ensure git tree is clean
- Run `openclaw update`
- Verify gateway status after update

### SSH considerations

Long builds may cause SSH disconnects.
This does not necessarily indicate failure.

Always re-check status after reconnecting.

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

Pairing state, device tokens, and identities live in:

/home/taylor/.openclaw

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

## 13. Deployment Reference State (As of 02/26/26)

This section documents the validated, working steady-state configuration.
If the system drifts from this state, investigate before rotating tokens.

### Current State (✅ expected)

- Gateway running (Docker)
- Node running (Docker, same VPS)
- 1 paired node device (Hetzner VPS)
- CLI uses same VPS device identity
- 1 operator device (browser Dashboard)
- Gateway UI is reachable only at:
  - `https://<vps-hostname>.<tailnet>.ts.net/…` (MagicDNS over tailnet)
- Gateway bound to:
  - `127.0.0.1:18789`
- Dashboard shows:
  - Two paired devices
  - One node
  - Gateway token visible in UI (token itself stored only in `.env` / secrets)

The system is NOT designed to:

- Expose `:18789` publicly
- Bind gateway to `0.0.0.0`
- Be accessed via `http://<tailscale-ip>:18789`
- Use SSH tunnel as primary access method

All traffic remains:

- Loopback-bound
- Tailnet-restricted
- Identity-controlled via Tailscale
