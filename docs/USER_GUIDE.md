<!-- SYNCED TO GitHub via scripts/upload-release.ps1. Source-of-truth is this Bitbucket repo. Edit here, then run: pwsh ./scripts/upload-release.ps1 -SyncDocsOnly -->

# bonovalley-platform-cli — user guide

End-to-end guide for integration partners using the CLI. If you're trying to **install or upgrade**, see [`docs/INSTALL.md`](INSTALL.md) instead — this guide assumes you already have the binary.

## Table of contents

1. [Quickstart (5 commands)](#1-quickstart-5-commands)
2. [Initial setup](#2-initial-setup)
3. [Configuration](#3-configuration)
4. [Your first integration (happy path)](#4-your-first-integration-happy-path)
5. [Working with multiple integrations](#5-working-with-multiple-integrations)
6. [Daily workflow](#6-daily-workflow)
7. [Where things live on your machine](#7-where-things-live-on-your-machine)
8. [Command reference](#8-command-reference)
9. [Root-level flags](#9-root-level-flags)
10. [Troubleshooting](#10-troubleshooting)
11. [Getting help](#11-getting-help)

---

## 1. Quickstart (5 commands)

If you've used CLIs like `gh` or `aws` before, this is all you need:

```bash
bonovalley-platform login
bonovalley-platform init my-integration       # scaffolds at <project_parent_dir>/my-integration
cd ~/bonovalley-integrations/my-integration   # (your project_parent_dir, default ~/bonovalley-integrations)
bonovalley-platform register                  # interactive
bonovalley-platform push                      # build + ship to platform
```

The rest of this guide explains each step, what they do, and how to recover when something doesn't go to plan.

---

## 2. Initial setup

### 2.1 Install the binary

See [`docs/INSTALL.md`](INSTALL.md) for per-OS install + PATH setup. Confirm with:

```bash
bonovalley-platform --version
# Expected: bonovalley-platform-cli vX.Y.Z (commit ..., built YYYY-MM-DD)
```

### 2.2 Check your environment

Before logging in, run the diagnostics:

```bash
bonovalley-platform doctor
```

Expected on a fresh install: **5 of 7 checks pass**. The two that fail (`Logged in (deployKey present)` and `deployKey accepted by platform`) are expected — you haven't logged in yet. After step 2.3 below they both go green.

If anything else fails, fix it before continuing — `doctor` tells you exactly what to do.

### 2.3 Log in

```bash
bonovalley-platform login
```

This:
1. Opens your default browser to the Bonovalley OAuth page
2. You sign in with your Bonovalley account
3. The browser redirects to a local URL (`http://127.0.0.1:8080/redirect`) and the CLI captures the result
4. An encrypted **deployKey** is saved to `~/.bonovalleyrc`. Every subsequent CLI command sends this key automatically.
5. Your **default organization** is saved to `~/.bonovalley/config.json` (one-time auto-populate; the CLI uses this for API calls that need an org context).

Verify:

```bash
bonovalley-platform doctor      # should now be 7/7 ✔
bonovalley-platform config get  # see what login set up
```

### 2.4 Pick where new projects should land

By default, `bonovalley-platform init <name>` creates new integration projects under `~/bonovalley-integrations/<name>`. To change that location, see [§3.2](#32-changing-project_parent_dir).

You're ready. Jump to [§4](#4-your-first-integration-happy-path) to scaffold your first integration.

---

## 3. Configuration

### 3.1 Where settings live

| Setting | File | Set by |
|---|---|---|
| `project_parent_dir` | `~/.bonovalley/config.json` | `login` (default) or `config set` |
| `default_organization_id` | `~/.bonovalley/config.json` | `login` (auto-populated) or `config set` |
| `api_env` (`dev` / `prod`) | `~/.bonovalley/config.json` | `login` (default `prod`) or `config set` |
| deployKey | `~/.bonovalleyrc` (encrypted) | `login` only |

### 3.2 Changing `project_parent_dir`

Look at the current value:

```bash
bonovalley-platform config get project_parent_dir
# C:\Users\you\bonovalley-integrations
```

Change it:

```bash
# Linux / macOS
bonovalley-platform config set project_parent_dir ~/work/integrations

# Windows (use the full path)
bonovalley-platform config set project_parent_dir "C:\Users\you\work\integrations"
```

Next time you run `bonovalley-platform init my-thing`, the new project will land under the new path. Existing projects stay where they are (the local registry tracks each project's full path independently — see [§5](#5-working-with-multiple-integrations)).

### 3.3 Changing the default organization

If you belong to multiple Bonovalley organizations and want to switch which one is the default for API calls:

```bash
bonovalley-platform config get default_organization_id
bonovalley-platform config set default_organization_id <new-org-uuid>
```

You can also use the `env` / `team` topic commands for a friendlier interface (`bonovalley-platform env --help`).

### 3.4 Switching between prod and a local API server

For local development of the Bonovalley platform itself (rare — partners don't normally need this):

```bash
bonovalley-platform config set api_env dev   # talks to http://127.0.0.1:8000
bonovalley-platform config set api_env prod  # talks to https://cli-api.bonovalley.com (default)
```

Advanced env-var overrides (for CI, corporate proxies, staging endpoints) exist via `BV_INTEGRATIONS_CLI_REST_API_URL_DEV` / `_PROD` — contact your Bonovalley point of contact if you need to point the CLI at a non-default URL.

### 3.5 Edit the JSON directly (escape hatch)

If the `config set` command ever misbehaves, you can edit `~/.bonovalley/config.json` directly. It's a plain JSON object with the keys above. Restart any in-flight CLI command for changes to take effect.

---

## 4. Your first integration (happy path)

End-to-end: scaffold a new project, claim it on the platform, write your code, ship it.

### 4.1 Scaffold the project — `init`

```bash
bonovalley-platform init my-integration
```

Before creating anything, `init` shows the full target path and asks you to confirm:

```
Your new project will be created at:
  C:\Users\you\bonovalley-integrations\my-integration
  (parent from "project_parent_dir" in your config)

Create it here? [Y/n]:
```

Press Enter (or `y`) to proceed. Answer **n** to stop without creating anything — the CLI then shows how to pick a different location. To send a single project somewhere else without changing your default, pass `-d` / `--dir`:

```bash
bonovalley-platform init my-integration --dir D:\work\integrations
```

(`-y` / `--yes` skips the confirmation; it's also skipped automatically in non-interactive / CI runs.)

What this does:
- Downloads the latest **integration template** from the Bonovalley template repo
- Extracts it into `<project_parent_dir>/my-integration` (e.g. `~/bonovalley-integrations/my-integration`)
- Strips template-developer-only files (per `.bvtemplate-ignore`)

If the target folder already exists and is non-empty, `init` refuses unless you pass `--force` (which wipes the folder contents first, **preserving any `.git/` directory** so your existing git history survives a re-init — handy for refreshing template files in a project you've already put under version control).

After `init`, `cd` into the project:

```bash
cd ~/bonovalley-integrations/my-integration
```

### 4.2 Claim it on the platform — `register`

```bash
bonovalley-platform register
```

This prompts you for 7 fields:

| Field | Example | Note |
|---|---|---|
| `key` | `my-integration` | URL-safe slug; partners can't change after register |
| `name` | `My Integration` | Display name on the marketplace |
| `description` | `Does X for Y.` | Required |
| `website` | `https://example.com/my-integration` | Required |
| `category` | `productivity` | Pick from the list shown |
| `intended_audience` | `business` | Pick from the list shown |
| `relationship_with_app` | `bv_employee` or `bv_partner` | Pick from the list shown |

After confirming, the CLI:
- POSTs to the platform (creates an `integrations` row)
- Writes `.bonovalley/integration.json` in the project (the **marker file** — identifies which platform integration this folder represents)
- Adds an entry to your local registry (`~/.bonovalley/integrations.json`) so future commands can find this project by id or name

If a registration with the same key already exists on the platform, register refuses (you'd `link` it instead — see [§5.2](#52-link-pick-up-an-existing-integration-on-a-new-machine)).

### 4.3 Write your integration code

The scaffold under `<project>/creates/`, `<project>/triggers/`, `<project>/searches/`, etc. is where your code lives. See the template's `README.md` and `CLAUDE.md` (inside your project folder) for the framework's conventions.

### 4.4 Ship it — `push`

When your code is ready:

```bash
bonovalley-platform push
```

Before the build, push asks whether you've set this integration's environment
variables — answer **yes**, **no**, or **not-required**. Answering **no** stops
immediately (before the long build) and shows the `env:set` commands to run, so
a missing secret doesn't turn into a broken deploy (see §6.5 below). Add
`-y` / `--yes` to skip the question; it's also skipped automatically in
non-interactive / CI runs.

The CLI then runs a 7-phase pipeline:

```
→ Pre-flight: integration <id> (<key>), bv_organization_id <uuid>
→ Verifying deployKey + integration ownership      ← fails fast in ~1.5s if your login expired
✔ Auth + ownership OK
→ Verifying Go toolchain + goimports                ← auto-installs goimports if missing
✔ Toolchain ready
→ Installing project dependencies (go mod tidy)
✔ Dependencies up to date
→ Building integration (setup_parent.go)            ← ~30s-3min depending on cache
✔ Build complete in 47.297s
→ Uploading 2 files (hello + integrationdefinition.json)
✔ Upload accepted: All Files Uploaded
→ Cleaning up temp directory (remove_temp_dir.go)
✔ Cleaned

✔ Push succeeded in 1m31.279s.
   Log: ~/.bonovalley/logs/push-20260524-153247.log
```

Every push run also writes a timestamped log file under `~/.bonovalley/logs/` — handy for retrospective debugging or attaching to bug reports.

If push **fails** at any phase, the summary block tells you exactly what to do next:

```
✗ Push failed after 35s.
   Suggestion: Bump the 'version' field in package.json (semver: 1.0.1 → 1.0.2) and retry.
   Detail: HTTP 400 — Version Number ( "1.0.1" ) mentioned in package file already used.
   Log: ~/.bonovalley/logs/push-20260524-153247.log
```

Common push failures:

| Symptom | Cause | Fix |
|---|---|---|
| `Auth check failed: Your deployKey is invalid, malformed, or expired` | Login session aged out | `bonovalley-platform login` then re-push |
| `Upload failed: That version number is already published` | `version` in `package.json` already used | Bump semver in `package.json`, push again |
| Build phase fails with validation errors | Schema / JSON-tag mismatches in your code | Read the inline errors in the build output (or `bonovalley-platform push -v` for full detail); fix; re-push |

---

## 5. Working with multiple integrations

### 5.1 `list` — what's tracked on this machine

```bash
bonovalley-platform list
```

```
ID     KEY              NAME              PATH                                     STATUS  LAST USED
13021  google-gemini    Google Gemini     G:\...\googlegemini_vid_600_52           ok      2026-05-24 10:41:10
13037  push-smoke-once  Push Smoke Once   C:\Users\you\work\bv-test\push-smoke-once  stale   2026-05-23 21:25:55
```

`stale` means the folder no longer exists on disk (moved, deleted, or external drive unmounted). Stale rows aren't auto-cleaned — use `unlink <id>` if you're sure.

### 5.2 `link` — track an existing integration (clone, new machine, or attach an init'd project)

Use `link` when an integration **already exists** on the platform and you want this machine/folder to point at it — you cloned the repo onto a new machine, you lost your local registry, or you just `init`ed a fresh project and want to attach it to an existing integration **without** running `register` (which would create a *new* one).

```bash
# From inside the project folder (reads .bonovalley/integration.json):
cd "G:\…\7_Integrations\google-gemini"
bonovalley-platform link

# By integration id — link works out which folder to attach:
bonovalley-platform link 13021

# Point it at a specific folder (no prompts; use this in scripts / CI):
bonovalley-platform link 13021 --dir "G:\…\7_Integrations\google-gemini"
```

When you pass an `<id>`, `link` decides which project folder to attach, in this order:

1. `-d` / `--dir <path>` if you gave one;
2. a marker already present in the current folder (or an ancestor);
3. the current folder, if it looks like an integration project;
4. otherwise it **asks** — listing the projects under your `project_parent_dir`:

```
Integration 13021 (google-gemini) isn't linked to a project on this machine yet.
Which project should it link to?
Projects in G:\…\7_Integrations:
  [1] google-gemini      (not linked yet)
  [2] another-proj       (linked to 13055)
  [c] enter a custom path
  [q] cancel
Choose:
```

It then writes `.bonovalley/integration.json` into the folder you choose and records it in your registry. Safeguards: it **never** writes into the CLI's own `~/.bonovalley` state folder, and the target must look like a real integration project.

Both forms check that you actually **own** the integration on the platform (403 if not — so an integration created by a teammate can't be linked yet).

### 5.3 `unlink` — stop tracking on this machine

```bash
bonovalley-platform unlink 13021
```

Removes the registry entry. **Does not** delete the project folder or change anything on the platform. To re-track later, just `link` again.

---

## 6. Daily workflow

### 6.1 The push loop

```
edit code → bonovalley-platform push → fix any errors from the summary block → repeat
```

If push succeeded but you want to redeploy without changes, **bump the version in `package.json`** first — the platform rejects duplicate versions.

### 6.2 `doctor` — quick health check

Run before push if anything feels off:

```bash
bonovalley-platform doctor
```

7 checks, ~2 seconds. Catches environment problems before they bite a 2-minute build. The 7th check (`deployKey accepted by platform`) hits an authenticated endpoint to catch silent session-expiry — a deployKey file can sit on disk indefinitely, but the access token wrapped inside it has a ~1-hour TTL, so the file-existence check alone isn't enough to know your session still works.

### 6.3 Reading logs

Every `push` writes a log to `~/.bonovalley/logs/push-YYYYMMDD-HHMMSS.log`. To find the latest:

```bash
# Linux / macOS
ls -t ~/.bonovalley/logs/ | head -1

# Windows PowerShell
Get-ChildItem $HOME\.bonovalley\logs -File | Sort-Object LastWriteTime -Desc | Select -First 1
```

Logs auto-prune after 30 days OR 100 files (whichever hits first). Safe to share — no secrets are logged.

### 6.4 Logout

```bash
bonovalley-platform logout
```

Removes the deployKey. Your registry + per-project markers stay. Next `login` brings auth back.

### 6.5 Environment variables & secrets — `env:set` / `env:get` / `env:unset`

Reference app-level credentials (OAuth client secret, API keys, signing secrets)
**by name** in your integration code instead of hardcoding them:

```go
secret, _ := bv.Secrets.GetSecret(ctx, "GOOGLE_CLIENT_SECRET")
clientSecret := secret.Value
```

Then submit the actual values out-of-band, scoped to a specific version, so they
never end up in your source, the binary, git, or the review bundle. Values are
stored encrypted at rest and injected into your running integration at deploy time.

```bash
# Set one or more. Version is the first argument; the integration is read from
# the project marker, so run these from inside the project folder.
bonovalley-platform env:set 1.0.0 GOOGLE_CLIENT_SECRET=abc123
bonovalley-platform env:set 1.0.0 API_KEY=xyz WEBHOOK_TOKEN=t0ken

# List what's set for a version (values shown)
bonovalley-platform env:get 1.0.0

# Remove one or more
bonovalley-platform env:unset 1.0.0 API_KEY WEBHOOK_TOKEN
```

Notes:
- Keys are normalised to `UPPER_SNAKE_CASE` (e.g. `google.client-secret` →
  `GOOGLE_CLIENT_SECRET`), matching how your code reads them.
- **Secrets are bound at deploy time.** A change via `env:set` takes effect on your
  **next `push`/deploy** of that version, not on an already-running service.
- **Local testing:** put the same `KEY=VALUE` lines in a `.env.development` file in
  your project root — they're loaded automatically when you run the integration in dev.
- **`push` reminds you.** Before each build, `push` asks whether your env vars are
  set. Answer **no** to bail out and set them first; pass `-y` / `--yes` (or run in
  CI) to skip the prompt.

---

## 7. Where things live on your machine

| Path | Purpose | Safe to delete? |
|---|---|---|
| `~/.bonovalleyrc` | Encrypted deployKey from `login` | Yes — re-run `login` to recreate |
| `~/.bonovalley/config.json` | `project_parent_dir`, `default_organization_id`, `api_env` | Yes — recreated on next CLI run with defaults |
| `~/.bonovalley/integrations.json` | List of integrations tracked on this machine | Yes — re-`link` each one afterwards |
| `~/.bonovalley/logs/*.log` | One log file per `push` run | Yes — auto-pruned (30 days / 100 files) |
| `<project>/.bonovalley/integration.json` | Per-project marker (integration id + key + name) | **No** — deleting breaks the platform link; use `unlink` instead |
| `<project>/BonoValley-d-temp-<digits>/` | Build scratch space; only present if push failed before cleanup | Yes — the next push wipes it via the template's `remove_temp_dir.go` |

Windows: `~` = `C:\Users\<you>` (`$HOME` in PowerShell).

---

## 8. Command reference

Run `bonovalley-platform <command> --help` for full flag detail on any of these.

### Working commands

| Command | What it does |
|---|---|
| `login` | OAuth flow; saves deployKey to `~/.bonovalleyrc` and default org to config |
| `logout` | Removes deployKey |
| `doctor` | 7-check pre-flight; safe to run anywhere |
| `config get [key]` | Print one or all config settings |
| `config set <key> <value>` | Update a config setting |
| `init <name>` | Scaffold a new integration project from the template (confirms the location first; `-d` / `--dir <path>` overrides where, `-y` / `--yes` skips the prompt) |
| `register` | Interactive: claim a new integration on the platform; writes marker + registry |
| `link [<id>]` | Track an existing integration on this machine; `link <id>` works out (or asks) which project folder to attach (`-d` / `--dir <path>` to set it explicitly) |
| `unlink <id>` | Stop tracking an integration on this machine |
| `list` | Print every integration tracked on this machine |
| `push` | Build + validate + upload the current integration (asks you to confirm env vars are set first; `-y` / `--yes` skips) |
| `env:set <version> KEY=VALUE …` | Set integration env vars / secrets for a version (encrypted at rest) |
| `env:get <version>` | List the env vars / secrets set for a version |
| `env:unset <version> KEY …` | Remove env vars / secrets from a version |
| `--version` | Print the CLI version, embedded commit, and build date |

### Commands shown in `--help` but not yet fully implemented

These appear in `--help` but are placeholders today. Avoid relying on them; they'll get real implementations in later releases:

`delete`, `deprecate`, `migrate`, `promote`, `analytics`, `autocomplete`, `convert`, `describe`, `history`, `integrations`, `jobs`, `logs`, `path`, `scaffold`, `status`, `test`, `upload`, `validate`, `versions`.

Track release notes in [`CHANGELOG.md`](../CHANGELOG.md) for when each goes live.

---

## 9. Root-level flags

These work on every command:

| Flag | Effect |
|---|---|
| `-v` / `--verbose` | Show sub-step output on terminal (e.g. full `push` build chatter). Log file always has full detail regardless. |
| `-q` / `--quiet` | Suppress phase markers on terminal; only the final summary prints. Ignored if `--verbose` is also set. |
| `--no-color` | Disable ANSI colour. Also disabled by `NO_COLOR=1` env var. |
| `-h` / `--help` | Help for any command. |

Example — get the full build chatter on a `push`:

```bash
bonovalley-platform push --verbose
```

---

## 10. Troubleshooting

Most "I can't get the CLI to work" issues are install / environment — those live in [`docs/INSTALL.md` §6](INSTALL.md#6-troubleshooting) (PATH, macOS Gatekeeper, Windows SmartScreen, ANSI escapes, expired session, corporate proxy).

CLI-usage issues:

| Symptom | What it usually means |
|---|---|
| `not inside a bonovalley integration project (no .bonovalley/integration.json...)` on `push` | You're not in the right folder. `cd` into your integration project. |
| `default_organization_id is not set in config; run 'bonovalley-platform login' first` | Login state lost. Run `login`. |
| `Push failed after Xs. Suggestion: ... Run login` | DeployKey expired. Run `login`, re-push. |
| `Push failed after Xs. Suggestion: Bump the 'version' field in package.json` | Platform already has this version. Bump `version` in `package.json` and re-push. |
| `Auth check failed: integration <id> exists but is owned by another user` | You're trying to push someone else's integration. Verify with `list`. |
| `multiple BonoValley-d-temp-* folders found` | Previous push was interrupted. Remove the older temp dir(s) manually, retry. |

For anything else, run `doctor` and include its output + the latest log when you reach out — see [§11](#11-getting-help).

---

## 11. Getting help

Include all three in any bug report — they give the maintainers everything they need to reproduce:

1. `bonovalley-platform --version` — which build you're on
2. `bonovalley-platform doctor` — your environment state
3. The relevant log file from `~/.bonovalley/logs/` — what happened on the failing run

Per-command help: `bonovalley-platform <command> --help`.

Contact your Bonovalley point of contact.

### See also

- [`docs/INSTALL.md`](INSTALL.md) — install / upgrade / uninstall / troubleshooting
- [`README.md`](../README.md) — quickstart + overview
- [`CHANGELOG.md`](../CHANGELOG.md) — per-release notes
