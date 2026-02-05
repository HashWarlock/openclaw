# Fix CVM Config: Redpill-Only Agent Auth

**Date:** 2026-02-04
**Status:** Approved

## Problem

Production deployment on Phala Cloud CVM fails with five cascading errors during agent creation:

1. `gateway failed: raw required` -- config patch requests fail because the existing config is invalid
2. `config.patch INVALID_REQUEST` -- gateway refuses to patch an already-invalid config
3. `Unrecognized key: "userTokens"` -- root cause; `gateway.auth.userTokens` is not in the Zod schema
4. `Gateway restart is disabled` -- `commands.restart` not set
5. `No API key found for provider "anthropic"` -- orchestrator agent has no Anthropic credentials

Errors 1-3 are a cascade: the invalid `userTokens` key breaks config validation, which blocks all config patch operations. Error 5 is a separate auth issue where the orchestrator defaults to Anthropic but has no key.

## Root Cause

- `gateway.auth.userTokens` was added to the live config based on the multi-agent deployment design (`docs/plans/2026-02-04-multi-agent-deployment-design.md`), but the schema (`src/config/zod-schema.ts`) uses `.strict()` validation and only allows `mode`, `token`, `password`, and `allowTailscale`.
- The orchestrator agent tried to call Anthropic because no default model was configured, and no `ANTHROPIC_API_KEY` was provisioned.

## Solution

Three config-level changes. No code changes required.

### 1. Remove `gateway.auth.userTokens` from CVM config

The `userTokens` feature is not yet implemented in the schema. Remove it to restore config validity. The feature will be implemented as a separate task when multi-user routing is needed.

### 2. Add `commands.restart: true`

Allows agents to restart the gateway when needed (e.g., after config changes during agent setup).

### 3. Set default model to Redpill

Set `agents.defaults.model.primary` to `redpill/moonshotai/kimi-k2.5`. This ensures all agents (including orchestrator) use Redpill instead of Anthropic. The `REDPILL_API_KEY` env var is already in `docker-compose.yml` and the auth resolver picks it up automatically.

### Resulting config shape

```json5
{
  gateway: {
    auth: {
      mode: "token",
      token: "<existing-admin-token>"
    }
  },
  commands: {
    restart: true
  },
  agents: {
    defaults: {
      model: {
        primary: "redpill/moonshotai/kimi-k2.5"
      }
    }
  }
}
```

### What we do NOT need

- No `auth-profiles.json` per agent (env var fallback is sufficient)
- No `ANTHROPIC_API_KEY` env var
- No custom `models.providers.redpill` block (Redpill is a built-in provider with auto-discovery)
- No schema changes (userTokens deferred to a future task)

## Deployment Steps

1. SSH into CVM
2. Edit `/data/openclaw/openclaw.json`: remove `userTokens`, add `commands.restart`, set default model
3. Restart the gateway
4. Verify with `openclaw channels status --probe`
