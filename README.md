# OpenBao Secrets Automation — API Key Rotation

> Automating API key rotation with an open-source Vault fork.

## Problem

API keys were rotated manually every time they expired — the CEO was doing it himself in the OpenRouter dashboard. No audit trail, no automation, no scoped access. Each rotation touched production tenants directly.

## What I Built

`provision-openrouter-key.sh` — a Bash script that provisions budget-capped API keys via the OpenRouter API and stores them in [OpenBao](https://openbao.org) (open-source HashiCorp Vault fork).

### Features

- **Two modes**: `--manual` (paste an existing key) and `--auto` (mint a new budget-capped key via OpenRouter's provisioning API)
- **Budget caps**: `--limit` flag sets per-key credit limits
- **OpenBao integration**: Keys stored in OpenBao's KV v2 engine with full versioning
- **Rollback**: `kv rollback` to any prior version if a key is bad
- **Pipeline integration**: Per-instance `SECRET_BACKEND` flag flip migrates a tenant from Infisical to OpenBao — no script rewrite needed

### Security Design

- Secrets passed via stdin/pipes, never as command arguments
- Trap-based cleanup on exit
- No secrets written to disk
- Scoped to `bao-developer` role (operator-platform policy, 4h sessions)

## Architecture

```
provision-openrouter-key.sh
          │
          ├── --auto ──► OpenRouter Provisioning API
          │                    │
          │              (HTTP 201 + new key)
          │                    │
          └── --manual ──┐     │
                         ▼     ▼
                    OpenBao KV v2
                    (weown/platform/<tenant>)
                         │
                    Versioned storage
                    + audit logging
                         │
                    ┌────▼─────┐
                    │ Tenants  │
                    │ flip     │
                    │ SECRET_  │
                    │ BACKEND  │
                    └──────────┘
```

## Testing

- Tested against demo stack (both modes confirmed working)
- Tested against production `chattest2` tenant: manual mode (v30 → v31) and auto mode (HTTP 201 mint, v32 → v33)
- Rollback verified (kv rollback to prior version)
- All 4 containers (anythingllm, dashboard, embed-filter, caddy) booted healthy after each rotation
- Production tenant restored to original state after testing

## Outcome

Key rotation went from a manual CEO task to a single command with audit logging. Merged to the OpenBao repo (PR #126) with two additional security fixes from the CTO's review.

## Stack

`OpenBao` · `Vault` · `Bash` · `OpenRouter API` · `Secrets Management` · `KV v2`

## Context

Built as Platform Engineer at [WeOwn](https://weown.net) (2026). Script lives in the private OpenBao repo at `git.weown.tools`.
