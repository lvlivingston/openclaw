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

- **Host**: Hetzner VPS (Ubuntu 24.04)
- **Runtime**: Docker (OpenClaw gateway + supporting services)
- **Network access**:
  - Gateway bound to loopback (`127.0.0.1`)
  - Access via:
    - SSH tunnel (local dashboard access)
    - Tailscale Serve/Funnel (TLS + identity)
- **Clients**:
  - macOS (primary)
  - CLI
  - Web UI (via SSH tunnel or Tailscale)

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

## 7. Network Access Model

### Gateway binding

- Gateway is bound to loopback only
- Never exposed directly on public interfaces

### Access methods

#### SSH Tunnel (default)

Used for:

- dashboard access
- debugging
- local-only exposure

#### Tailscale Serve / Funnel

Used for:

- TLS
- identity-based access
- remote management without open ports

Gateway auth mode must be compatible with Serve/Funnel rules.

---

## 8. Token & Credential Model (READ THIS FIRST)

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

## 9. Update & Maintenance Policy

### Updating OpenClaw

- Ensure git tree is clean
- Run `openclaw update`
- Verify gateway status after update

### SSH considerations

Long builds may cause SSH disconnects.
This does not necessarily indicate failure.

Always re-check status after reconnecting.

---

## 10. Backups & Recovery

### What to back up

- `~/.openclaw`
- Docker volumes (if applicable)

Example:

```bash
tar -czf openclaw-state-$(date +%F).tgz ~/.openclaw

---

## 11. References

- Community Hetzner + Tailscale OpenClaw guide: https://gist.github.com/thedudeabidesai/eb9490c031d869313142368150a060e9
- Pulumi cloud deployment ideas: https://www.pulumi.com/blog/deploy-openclaw-aws-hetzner/
- Zero-trust Tailscale deployment patterns: https://alirezarezvani.medium.com/i-deployed-openclaw-with-zero-public-ports-here-is-the-tailscale-setup-that-actually-works-86f8c9e6f158
```
