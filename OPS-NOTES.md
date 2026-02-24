## 2026-02-24 — Secure baseline established

- Gateway token auth enabled
- Strong random token generated
- Node auth mismatch resolved
- Gateway + node stable under Docker Compose
- `openclaw security audit --deep`: 0 critical

Notes:
- Gateway token lives in `.env` (gitignored)
- Do not rotate token without restarting both gateway + node

Agents:
- main     → stable / default
- sandbox  → experimental (tools, browser control, model tests)
