# Environment Variables — bonovalley-platform-cli

This document is the canonical reference for every environment variable the CLI reads, their defaults, and the override-precedence rules.

## Quick reference

| Variable | Used by | Default (seeded by `main.go` if unset) | Override at runtime? |
|---|---|---|---|
| `APP_ENV` | Process-wide mode flag (`dev` / `prod`). Today this is informational; the active backend URL is picked by `api_env` in `~/.bonovalley/config.json`, not by `APP_ENV`. | `dev` | Yes — `export APP_ENV=prod` |
| `ORY_CLIENT_ID` | Ory OAuth2 PKCE client used by `bonovalley-platform login`. | `3ad08fa5-6e81-4370-869a-bad2dbdf78aa` | Yes — for a different Ory tenant set your own |
| `BV_INTEGRATIONS_CLI_REST_API_URL_DEV` | Backend REST API URL when `api_env=dev`. | `http://127.0.0.1:8000` | Yes — point at any reachable host:port |
| `BV_INTEGRATIONS_CLI_REST_API_URL_PROD` | Backend REST API URL when `api_env=prod`. | `https://cli-api.bonovalley.com` | Yes — point at a staging or PR-preview deploy |

## Precedence (URL resolution)

The CLI resolves the backend URL in this order:

1. **Environment variable** `BV_INTEGRATIONS_CLI_REST_API_URL_DEV` or `BV_INTEGRATIONS_CLI_REST_API_URL_PROD` — whichever matches the current `api_env`.
2. **Hardcoded fallback** `BaseURLDev` / `BaseURLProd` in `engine/tools/bvapi/bvapi.go`.

`main.go` seeds the env vars with the defaults from the table above *only if the caller hasn't already set them* (`if os.Getenv(...) == ""`). So:

- Shell export, CI variable, `.env` exported before `go run`, etc. — wins, because `main.go` sees the value already set and skips its seeding.
- Nothing exported — `main.go` seeds the default, and `resolveBaseURL` reads that value.
- Both env var and hardcoded constant differ — env var wins (the hardcoded constant only matters if the env var is somehow unset after `main.go` ran, which shouldn't happen in normal flow but is preserved as a defensive fallback).

## Which URL the CLI hits

Picked by **`api_env`** in `~/.bonovalley/config.json` (NOT by `APP_ENV` despite the name overlap):

```json
{
  "project_parent_dir": "...",
  "default_organization_id": "...",
  "api_env": "prod"          // or "dev"
}
```

- `api_env=dev` → reads `BV_INTEGRATIONS_CLI_REST_API_URL_DEV`
- `api_env=prod` → reads `BV_INTEGRATIONS_CLI_REST_API_URL_PROD`
- `api_env` missing/empty → treated as `prod` so a fresh install talks to production by default

Switch with a direct edit of `~/.bonovalley/config.json` (no `bonovalley-platform config set` command exists today).

## Common scenarios

### Test against production from your laptop

Already the default after `bonovalley-platform login` (which sets `api_env=prod` if you log in against the prod Ory tenant). To be sure:

```jsonc
// ~/.bonovalley/config.json
"api_env": "prod"
```

No env-var work needed — `main.go` seeds `BV_INTEGRATIONS_CLI_REST_API_URL_PROD=https://cli-api.bonovalley.com` by default.

### Test against a locally-running API

```jsonc
// ~/.bonovalley/config.json
"api_env": "dev"
```

Start the API on port 8000. CLI's default `BV_INTEGRATIONS_CLI_REST_API_URL_DEV=http://127.0.0.1:8000` will hit it.

### Test against a staging / PR-preview deploy

```bash
# PowerShell
$env:BV_INTEGRATIONS_CLI_REST_API_URL_PROD = "https://pr-42.cli-api.bonovalley.com"
bonovalley-platform.exe push

# bash / zsh / Git Bash
export BV_INTEGRATIONS_CLI_REST_API_URL_PROD="https://pr-42.cli-api.bonovalley.com"
bonovalley-platform push
```

With `api_env=prod` in the config and this env var exported, the CLI hits the preview URL — without rebuilding.

### Use a non-default Ory tenant

```bash
export ORY_CLIENT_ID="<your-client-id>"
bonovalley-platform login
```

## Where each value is read in the code

| Variable | First-read location |
|---|---|
| `APP_ENV` | `main.go` (seed only) |
| `ORY_CLIENT_ID` | `engine/tools/bv-cli-oauth2/...` during login |
| `BV_INTEGRATIONS_CLI_REST_API_URL_DEV` | `engine/tools/bvapi/bvapi.go::resolveBaseURL` |
| `BV_INTEGRATIONS_CLI_REST_API_URL_PROD` | `engine/tools/bvapi/bvapi.go::resolveBaseURL` |

## What's NOT an environment variable today

These live in `~/.bonovalley/config.json` (Viper-style) — not env vars:

- `project_parent_dir` — where `bonovalley-platform init <name>` creates new projects
- `default_organization_id` — auto-populated by `login`
- `api_env` — `dev` / `prod` switch
