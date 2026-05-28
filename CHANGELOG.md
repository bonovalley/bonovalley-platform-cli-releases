<!-- SYNCED TO GitHub via scripts/upload-release.ps1. Source-of-truth is the Bitbucket repo. Edit there, then run: pwsh ./scripts/upload-release.ps1 -SyncDocsOnly -->

# Changelog

All notable changes to `bonovalley-platform-cli` are documented here.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Each entry's date is the date that release was **tagged + published to the GitHub releases page**, not the date the underlying work merged.

## [Unreleased]

## [v1.1.0] — 2026-05-28

### Changed

- `--help` usage text now reads `bonovalley-platform …` instead of `bonovalley …`,
  matching the installed binary name and the docs (the root command's `Use:` was
  aligned). No behavior change — only the help/usage strings.

### Added

- **`env:set` / `env:get` / `env:unset`** — manage integration-level environment
  variables & secrets, scoped per version. Reference a secret by name in your
  integration code (e.g. `bv.Secrets.GetSecret(ctx, "GOOGLE_CLIENT_SECRET")`) and
  submit the value out-of-band so it never lands in source, the binary, git, or
  the review bundle:
    - `bonovalley-platform env:set <version> KEY=VALUE [KEY=VALUE ...]`
    - `bonovalley-platform env:get <version>`
    - `bonovalley-platform env:unset <version> KEY [KEY ...]`
  The integration is taken from the project marker (`.bonovalley/integration.json`);
  the version is the first positional argument. Keys are normalised to
  `UPPER_SNAKE_CASE`. Values are stored **encrypted at rest** (AWS KMS envelope
  encryption) keyed to the integration + version; at deploy time the platform
  injects them into the running service, where integration code reads them via the
  standard secrets API. For local testing, put the same `KEY=VALUE` lines in a
  project-root `.env.development` file (loaded automatically in dev).

## [v1.0.2] — 2026-05-25

### Changed

- `init <name> --force` now **preserves any `.git/` directory** in the target folder. Previously the entire folder was wiped, which destroyed the developer's commit history when re-initing a project they had already put under version control. With the fix, `--force` wipes every entry in the target *except* `.git/`, so the dev can run `git status` / `git diff` after a re-init to see exactly what the template refresh changed. (Three deletion paths were involved — `preflightTarget`, `Extract`, and `ApplyIgnore` — all three now skip `.git/`.)

### Added

- `doctor` adds a 7th check: **"deployKey accepted by platform"**. The existing "Logged in" check only verifies that `~/.bonovalleyrc` exists on disk; the new check makes a small authenticated GET against the platform API to confirm the deployKey is still accepted. This catches the silent session-expiry state (the Ory access token wrapped inside the deployKey has ~1-hour TTL) BEFORE a partner spends 2-3 minutes on a `push` that will then 401. Fresh-install behaviour: 5/7 pass (checks 4 + 7 fail until `login`). Logged-in: 7/7.

## [v1.0.1] — 2026-05-24

### Fixed

- **Critical login-blocking bug.** `bonovalley-platform login` in v1.0.0 errored on first run with `open ./engine/tools/bv-cli-oauth2/oclient2/services_prod.json: The system cannot find the path specified.` Cause: the OAuth client read its config and HTML template from disk via paths that only exist when the binary runs from the source-repo root. Embedded both via `go:embed` so the binary is self-contained regardless of install location. Added `oclient2_test.go` to guard against future regressions of this pattern. Partners on v1.0.0 must upgrade.

## [v1.0.0] — 2026-05-24

First public release.

### Added — partner UX foundations

- `bonovalley-platform doctor` — pre-flight checklist (Go toolchain, goimports, CLI config, deployKey, default org, platform API reachability) so partners can diagnose environment issues in 2 seconds instead of waiting for a 2-minute build to fail.
- Auto log files at `~/.bonovalley/logs/<cmd>-YYYYMMDD-HHMMSS.log` for `push` (other commands to follow). Prune policy: 30 days OR 100 files, whichever hits first. Path printed in the final summary block. Log file always starts with a 3-line header identifying the CLI version, command, and UTC start time.
- `push` final summary block: succinct pass/fail line with duration + suggestion (on failure) + preserved temp-dir path (on failure) + log file path (always).
- `push` pre-flight auth check: verifies the deployKey and integration ownership in ~1.5 seconds *before* the 2-3 minute build, so expired tokens fail fast instead of after a wasted build cycle.
- Partner-friendly error translation in `bvapi.APIError` — common HTTP status codes (401, 403, 404, 413, 5xx) and known message patterns (version-already-used 400, auth-related 400) now produce a Friendly explanation + Suggestion line. Consistent across `push`, `link`, and `register`.
- Root-level persistent flags: `-v/--verbose` (show sub-step build chatter on terminal), `-q/--quiet` (only the summary), `--no-color` (disable ANSI colour; also disabled by `NO_COLOR` env var per no-color.org).
- ANSI colour for success/failure markers, "Suggestion:" labels, and dim secondary detail lines. Colour is stripped from the log file regardless so `cat`/`grep`/pagers stay clean.

### Added — core CLI surface

- Core commands: `login`, `logout`, `list`, `init`, `link`, `unlink`, `register`, `push`.
- Topic commands: `env`, `team`, `users`, `delete`.
- Scaffold commands: `scaffold create`, `scaffold resource`, `scaffold search`, `scaffold trigger`.
- Per-integration marker file (`.bonovalley/integration.json`) — identifies an integration's project folder and survives a `git clone` to a new machine.
- Local registry (`~/.bonovalley/integrations.json`) — `list` shows every integration tracked on the machine with on-disk path, status, and last-used timestamp.
- `push` workflow (SOP-12): scaffolded build via the integration template's `setup_parent.go`, multipart upload to `/upload/files` (with a generous 10-minute timeout for multi-MB binaries), automatic cleanup via the template's `remove_temp_dir.go` script.

### Added — release engineering

- `--version` flag prints `bonovalley-platform-cli vX.Y.Z (commit ..., built ...)`.
- Cross-platform release binaries for Windows (amd64), Linux (amd64), and macOS (amd64 + arm64), published to Bitbucket Downloads with a `SHA256SUMS` integrity manifest. Built via `scripts/build-release.ps1`.
- Override of backend API URL via `BV_INTEGRATIONS_CLI_REST_API_URL_DEV` / `_PROD` env vars (defaults baked into the binary; partners never need to set these).
- `docs/DEPLOYMENT_SOP.md` — canonical 15-step release procedure with explicit USER/CLAUDE role markings + release journal.

### Known limitations

- No auto-update — partners re-download manually when a new version is published.
- `delete` and `deprecate` topic commands are stubs (help text only) — full implementations are scheduled for a later release.
- Single Ory tenant baked in — override via `ORY_CLIENT_ID` env var if you need a different one.
