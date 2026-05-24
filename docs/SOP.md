# `bonovalley-cli` — Design & SOP

> Living document. Captures every design decision made for the CLI, the rationale, and the step-by-step workflows for every situation a developer may encounter. Both Claude (in future sessions) and the human developer should be able to read this and immediately get oriented. Sections marked **OPEN** are not yet decided.

> **Names:** the project is `bonovalley-cli`; the binary the developer installs and invokes is `bonovalley-platform`. Every command example below uses that binary name.

> **Status snapshot — 2026-05-24.**
> Live in production at `https://cli-api.bonovalley.com` (API revision `bv-integrations-cli-rest-api-prod-00012-b5g`, image `1.0.10`). CLI master at commit `99eef8e`.
>
> | Command | End-to-end verified |
> |---|---|
> | `login` (writes `default_organization_id` inline) | ✓ |
> | `init <name>` (template download + `.bvtemplate-ignore` apply) | ✓ |
> | `register` (8 prompts → API → marker + registry) | ✓ |
> | `link` / `link <id>` (ownership-verified marker + registry write) | ✓ |
> | `config get/set`, `status`, `list`, `path`, `unlink` | ✓ |
>
> **Known limitations** (tracked in §11): deployKey effectively expires every ~1 hour (Ory access_token TTL — item #14); `pci`/`hipaa` security tiers not yet supported (item #11, deferred); 403-on-foreign-owner branch in `link` not live-tested (item #15); key regex disallows digits (item #16).

---

## 1. System overview

`bonovalley-cli` is one of three cooperating pieces:

| # | Component | Location | Role |
|---|---|---|---|
| 1 | `bonovalley-cli` (this repo) | `bonovalley-platform-cli/` | The Go CLI the developer installs on their laptop. |
| 2 | `bv-integrations-cli-rest-api` | `2_APIs/0_Graphjin_Graphql_Backend_API/bv-integrations-cli-rest-api/` | REST API the CLI calls for all DB operations (register integration, fetch deploy key, upload files, etc.). |
| 3 | Integration project | Anywhere on the developer's disk, e.g. `7_Integrations/googlegemini_vid_600_52/` | A single integration's source code. Has its own git repo. Many of these per developer. |

The CLI never touches the database directly. Everything DB-related goes through the API.

---

## 2. Core design principle — authority vs discoverability

Two layers, deliberately separated:

| Layer | Source of truth for… | Lives in | Per-machine? |
|---|---|---|---|
| **Authority** | "What integration is this folder?" | `./.bonovalley/integration.json` (committed to project's git) | No — travels with the repo |
| **Discoverability** | "Where on this disk does integration X live?" | `~/.bonovalley/integrations.json` (per-machine registry) | Yes — each machine has its own |
| **Identity** | "Who is the user?" | `~/.bonovalleyrc` (deployKey, per user) | Yes |
| **CLI settings** | "How does this CLI behave on this machine?" | `~/.bonovalley/config.json` | Yes |

Rule of thumb:
- The **project marker** is the truth. It survives `git clone` to any machine on any OS.
- The **registry** is a path cache. If it's lost, no integrations are lost — just re-link from the project folders.

---

## 3. File layout reference

### 3.1 Home directory (per-user, per-machine)

```
~/.bonovalleyrc                       # JSON. deployKey. Created on `login` success.
~/.bonovalley/                        # CLI state directory.
├── config.json                       # CLI settings (project_parent_dir, default_org, api_env, ...).
└── integrations.json                 # Local registry: integrations on this machine + their paths.
```

**Cross-platform resolution** via `os.UserHomeDir()`:

| OS | `~/` resolves to |
|---|---|
| macOS | `/Users/<u>/` |
| Linux | `/home/<u>/` |
| Windows | `C:\Users\<u>\` |

### 3.2 Default content of `~/.bonovalley/config.json`

```json
{
  "project_parent_dir": "<absolute path: ~/bonovalley-integrations>",
  "default_organization_id": null,
  "api_env": "prod"
}
```

- `project_parent_dir` is stored **absolute** (resolved at write time). On Windows it stores `C:\Users\<u>\bonovalley-integrations`. No tildes persisted.
- The directory itself is **NOT** pre-created — it's `mkdir -p`'d lazily on first `register`.

### 3.3 Default content of `~/.bonovalley/integrations.json`

```json
{
  "integrations": []
}
```

Entries are added by `register` / `link`, removed by `unlink`. Shape of an entry:

```json
{
  "id": "13021",
  "key": "google-drive",
  "name": "Google Drive",
  "path": "/Users/amit/bonovalley-integrations/google-drive",
  "registered_at": "2026-05-23T10:00:00Z",
  "last_used_at": "2026-05-23T10:00:00Z"
}
```

`key` and `name` are cached here as a developer convenience so `list` doesn't need to read every project's marker. Both are authoritative in the API; if they ever drift from the marker, the marker wins.

### 3.4 Project directory (per-integration, lives with the source code)

```
<integration-root>/
├── .bonovalley/
│   ├── integration.json              # COMMITTED. The marker file. Identity of this folder.
│   └── cache/                        # gitignored. Future use: build cache, upload logs.
├── .gitignore                        # contains: .bonovalley/* then !.bonovalley/integration.json
├── main.go
├── app-config.go
├── ... (rest of the integration's source)
```

### 3.5 Content of `./.bonovalley/integration.json`

The marker holds **stable identity only** — fields that don't change during the integration's normal lifecycle. Mutable marketplace metadata (`description`, `website`, `category`, `intended_audience`, `security_tier`, `icon`) stays in the DB; fetch on demand via `bonovalley-platform info`. This avoids marker-vs-DB drift.

```json
{
  "integration_id": "13021",
  "key": "google-drive",
  "name": "Google Drive",
  "relationship_with_app": "employee",
  "created_at": "2026-05-23T10:00:00Z"
}
```

- `integration_id` — immutable PK from the DB.
- `key` — user-chosen slug, validated against `^[a-z]+(-[a-z]+)*$`. Immutable per schema.
- `name` — display name. Mutable but slow-changing; kept here for human readability when grepping marker files.
- `relationship_with_app` — `employee` / `consultant` / `no_affiliation` / `bv_employee`. One of the 4 enum values from `relationship_with_app_type`. Technically mutable but rare.
- `created_at` — UTC ISO 8601.

Path-free by design. Same file works on macOS, Linux, Windows; same file works for every developer who clones the repo.

**Not stored in the marker** (derived server-side or fetched on demand):
- `owner_id` — derived by API from authenticated deployKey.
- `bv_organization_id` — derived by API; only non-null when `relationship_with_app = bv_employee`.
- `built_type` — derived by API from `relationship_with_app` (`bv_employee` → `bonovalley_tool`; else → `external_partner_integration`).
- `approval_status` — owned by the API; flows through `under_development → under_review → approved/rejected`.
- All marketplace metadata (`icon`, `category`, `intended_audience`, `security_tier`, `description`, `website`).

### 3.6 Legacy files (to be deprecated)

| File | Status | Plan |
|---|---|---|
| `~/.bonovalleyapprc` | Currently used as a global "active integration" pointer. Bad pattern. | Stop writing it. Add a one-time migration on first `login` after upgrade: if present, read its `id`, find the matching folder via API, write a registry entry, then delete the file. |
| `~/.bonovalley-platform-cli.yaml` | Viper config from cobra template. Mostly unused. | Drop. Fold any real settings into `~/.bonovalley/config.json`. |

---

## 4. Defaults and per-OS values

| Setting | Default | macOS | Linux | Windows |
|---|---|---|---|---|
| `project_parent_dir` | `~/bonovalley-integrations` | `/Users/<u>/bonovalley-integrations` | `/home/<u>/bonovalley-integrations` | `C:\Users\<u>\bonovalley-integrations` |
| `api_env` | `prod` | same | same | same |
| `default_organization_id` | `null` | same | same | same |

Why a visible (non-hidden) folder for `project_parent_dir`? Source code shouldn't live in a dotfile dir. The dev will `cd` into it and edit code daily — that belongs in a visible folder. Hidden dirs are for CLI state.

---

## 5. Bootstrap lifecycle

### 5.1 When `~/.bonovalley/` is created

Two triggers, both safe (idempotent):

1. **Primary — on `bonovalley-platform login`:** before OAuth runs, ensure `~/.bonovalley/`, `~/.bonovalley/config.json` (with defaults), and `~/.bonovalley/integrations.json` (empty registry) exist.
2. **Lazy fallback — on any command:** every command first runs a `ensureBootstrap()` check. If the CLI dir is missing, create it with defaults. This way `bonovalley-platform config set project_parent_dir <path>` works even before login.

### 5.2 When `~/.bonovalleyrc` is created

Only on **successful** OAuth login. Contains the deployKey. Format already established by the existing code.

### 5.3 When `~/bonovalley-integrations/` (or whatever `project_parent_dir` is) is created

**Lazy.** Only `mkdir -p` on the first `register` that needs it. Avoids leaving an empty directory in `~` if the dev changes `project_parent_dir` before ever registering.

### 5.4 When `./.bonovalley/integration.json` is created

Written by `register` (after API returns the new `integration_id`) and by `link` (after API confirms the supplied id is valid and owned by this user).

---

## 6. Command catalog

| Command | Hits API? | Writes home files | Writes project files | Purpose |
|---|---|---|---|---|
| `login` | Yes (OAuth + deploy key fetch) | `~/.bonovalleyrc`, bootstraps `~/.bonovalley/*` | — | First step. Authenticates the user and fetches the deployKey. |
| `logout` | Optional (revoke?) | clears `~/.bonovalleyrc` | — | Wipes the deployKey. Registry and config stay. |
| `config get/set <key>` | No | `~/.bonovalley/config.json` | — | Read/write CLI settings (e.g. `project_parent_dir`). |
| `register [--yes]` | Yes (POST `/integrations`) | adds entry to `integrations.json` | writes `.bonovalley/integration.json` in cwd | Creates a brand-new integration row in the DB. **Runs INSIDE an existing project folder** — typically the one created by `init`, or a pre-existing source folder being onboarded. Collects field values interactively (see §6.1 for the API contract). `--yes` skips the proceed prompt for scripts. Idempotent on conflict — see §8. |
| `init <name> [--force]` | Yes (GET `/template`) | — | creates `<project_parent_dir>/<name>`, downloads + extracts template, applies `.bvtemplate-ignore` | Bootstraps a new integration project folder by downloading the latest template from the platform. **Does NOT register with the API or write the marker file** — that is `register`'s job. Refuses if target exists and is non-empty unless `--force` (which wipes). |
| `link [<id>]` | Yes (verifies id ownership) | adds entry to `integrations.json` | writes marker if missing | Bring an existing project under tracking on this machine. |
| `unlink [<id>]` | No | removes entry from `integrations.json` | leaves marker untouched | Stop tracking on this machine. Doesn't delete the integration in DB. |
| `list [--remote]` | `--remote` hits API | reads `integrations.json`; validates paths exist | — | Show integrations. With `--remote`, cross-check against the API's view of integrations the user owns. |
| `path <id>` | No | reads registry | — | Print the absolute path. For `cd $(bonovalley-platform path 13021)`. |
| `status` | No | — | reads marker walking up from cwd | "What integration am I in?" |
| `rename <new-name>` | Yes (PATCH) | updates registry entry | updates marker `name` | Change the integration's name in DB + local. |
| `delete <id>` | Yes (DELETE) | removes from registry | optionally rm -rf project (prompted) | Hard removal. **OPEN** — confirm semantics. |
| `push [--ascii]` | Yes (POST `/upload/files`) | — | reads marker from cwd; spawns + cleans up `BonoValley-d-temp-*` | Build + validate + upload the integration's two artifacts (`hello`, `integrationdefinition.json`) to the platform. Full flow in SOP-12. Thin orchestrator over the integration repo's own `setup_parent.go` which does the actual build and JSON-schema validation. |
| `build` | No (or yes for remote build) | — | reads marker | Build artifact. **OPEN** — local vs server-side. |
| `logs` | Yes | — | reads marker | Tail server-side logs. |

---

## 6.1 API contract: `POST /integrations` (called by `register`)

The full field-by-field contract for the `register` API call. Closes Open Item #1.

For the cross-cutting API client basics — base URLs, Bearer auth, error envelope, and the full endpoint catalog — see §13.

### 6.1.1 Fields the CLI collects from the developer

Asked in this order during the interactive `register` prompt. The CLI does client-side validation on each before sending the request.

| Order | Field | DB column | Required? | CLI validation | Example |
|---|---|---|---|---|---|
| 1 | Integration name | `integration_name` | Yes | non-empty, trimmed | `Google Drive` |
| 2 | Key (slug) | `key` | Yes | `^[a-z]+(-[a-z]+)*$` — lowercase letters and single hyphens; must start and end with a letter | `google-drive` |
| 3 | Relationship with app | `relationship_with_app` | Yes | must be one of the 4 enum values (interactive picker) | `employee` |
| 4 | Category | `category` | Yes | non-empty | `Storage` |
| 5 | Intended audience | `intended_audience` | Yes | `public` or `private` (interactive picker) | `private` |
| 6 | Security tier | `security_tier` | Yes (default `standard`) | `standard` / `pci` / `hipaa` (interactive picker, default `standard`) | `standard` |
| 7 | Description | `description` | No | free-text (skippable) | `Sync files with Google Drive` |
| 8 | Website | `website` | No | URL-ish (skippable) | `https://drive.google.com` |

**Security tier prompt UX:** all 3 options are shown so the developer knows the platform supports them, but `standard` is the default (Enter accepts). About 99% of integrations are `standard`. The prompt also notes the implication of picking `pci`/`hipaa` — see §11 open item #11 for what happens when the dev picks a non-standard tier (requires collecting `sensitive_fields`).

`relationship_with_app` enum (from `tables/integrations.SQL`):

| Value | When to pick |
|---|---|
| `employee` | Builder is an employee of the company whose app is being integrated (e.g., a Salesforce employee building the Salesforce integration). |
| `consultant` | External consultant building an integration for a client company. |
| `no_affiliation` | Builder has no formal relationship with the app. Common for open API integrations. |
| `bv_employee` | Builder is a Bonovalley team member. Used for `bonovalley_tool` integrations. |

### 6.1.2 Fields derived by the API server (CLI does NOT send these)

| DB column | Source | Reason |
|---|---|---|
| `owner_id` | Authenticated user (deployKey → `integration_partners.id`) | Security: never trust the CLI to send identity. |
| `bv_organization_id` | Server-derived; only set when `relationship_with_app = bv_employee`. Enforced by `integrations_bv_org_only_for_bv_employees` CHECK. | Same reason as `owner_id`. |
| `built_type` | Server-derived: `bv_employee` → `bonovalley_tool`; else → `external_partner_integration` | One source of truth; avoids CLI/server disagreement. |
| `approval_status` | Always `under_development` on INSERT | Defined by the integration lifecycle. |
| `created_at`, `updated_at` | DB default `now() AT TIME ZONE 'UTC'` | Always server-clock truth. |

### 6.1.3 Fields deferred to a later step

Three classes of fields are NOT collected at `register` and are filled in before the dev transitions the integration to `under_review`. All require coordinated schema changes — listed below by field. **OPEN** — confirm all schema changes with the API team before implementing.

#### `icon` — currently `NOT NULL` with no default

Make nullable while `approval_status = 'under_development'`; enforce non-null on transition to `under_review` via a NEW CHECK constraint:

```sql
ALTER TABLE public.integrations ALTER COLUMN icon DROP NOT NULL;

ALTER TABLE public.integrations ADD
    CONSTRAINT integrations_icon_required_after_dev CHECK (
        approval_status = 'under_development'
        OR (icon IS NOT NULL AND icon <> '')
    );
```

Filled in later via `bonovalley-platform upload-icon` (TBD command) or a generic update flow.

#### `sensitive_fields` — existing CHECK constraints need to be relaxed

If the dev picks `pci` or `hipaa` at `register`, the existing `integrations_pci_requires_sensitive_fields` and `integrations_hipaa_requires_sensitive_fields` CHECK constraints will block the INSERT because `sensitive_fields` is empty.

Plan: drop the existing constraints and re-add them with an `under_development` escape clause:

```sql
ALTER TABLE public.integrations DROP CONSTRAINT integrations_pci_requires_sensitive_fields;
ALTER TABLE public.integrations DROP CONSTRAINT integrations_hipaa_requires_sensitive_fields;

ALTER TABLE public.integrations ADD
    CONSTRAINT integrations_pci_requires_sensitive_fields CHECK (
        approval_status = 'under_development'
        OR security_tier != 'pci'
        OR (sensitive_fields IS NOT NULL AND array_length(sensitive_fields, 1) > 0)
    );

ALTER TABLE public.integrations ADD
    CONSTRAINT integrations_hipaa_requires_sensitive_fields CHECK (
        approval_status = 'under_development'
        OR security_tier != 'hipaa'
        OR (sensitive_fields IS NOT NULL AND array_length(sensitive_fields, 1) > 0)
    );
```

This means: during `under_development`, the dev can pick any tier without declaring sensitive fields. On transition to `under_review`, the constraint fires and the API rejects the transition until the dev has provided `sensitive_fields` (for `pci`: an explicit list of dot-notation paths; for `hipaa`: typically `["*"]`).

Filled in later via a TBD command — likely `bonovalley-platform sensitive-fields add <path>` or part of a `submit-for-review` pre-flight that prompts for them.

#### Other marketplace metadata

Same deferred-until-review treatment is reasonable for: `compliance_flags`, `supports_test_mode`, `test_credentials_url`. None of these have NOT NULL constraints today, so no schema change is needed — just CLI commands to set them later before review submission.

### 6.1.4 Pre-flight checks (CLI-side, before hitting the API)

These are the checks `register` runs. `init`'s separate pre-flight (target-dir doesn't exist or is empty, unless `--force`) is documented in §6 catalog.

1. `~/.bonovalleyrc` exists and has a deployKey → else refuse with "run `bonovalley-platform login`".
2. `~/.bonovalley/config.json` `default_organization_id` is non-null → else refuse with "your local config is missing `default_organization_id`; run `bonovalley-platform login` again".
3. Current cwd does NOT already contain a `.bonovalley/integration.json` (would mean another integration is already registered here). **Note:** register operates on cwd; it does NOT create a new directory. The folder must already exist (created by `init` or pre-existing for SOP-3).
4. All collected fields pass client-side validation (regex on `key`, enum membership for the picker fields, etc.).
5. Show proceed prompt `Proceed? [Y/n]` (skippable with `--yes`).

### 6.1.5 API behavior on conflict — idempotent return

If an integration with the same `key` OR `integration_name` already exists (server-side uniqueness check):

- API returns HTTP 200 (not 4xx) with `{ "status": "already_exists", "integration": { ...existing row... } }`.
- CLI prints: `Integration with key 'google-drive' already exists (id=13021, owner=<user|other>).`
- If the existing integration is owned by **this** user: CLI asks `Link this folder to integration 13021 instead? [Y/n]`. On `Y`, behaves like `link 13021`.
- If owned by **another** user: CLI exits without modifying anything on disk.
- No DB row is created in either case.

### 6.1.6 Response shape on successful create

```json
{
  "status": "created",
  "integration": {
    "id": "13021",
    "key": "google-drive",
    "integration_name": "Google Drive",
    "relationship_with_app": "employee",
    "built_type": "external_partner_integration",
    "owner_id": "<uuid>",
    "bv_organization_id": null,
    "category": "Storage",
    "intended_audience": "private",
    "security_tier": "standard",
    "approval_status": "under_development",
    "description": "...",
    "website": "...",
    "created_at": "2026-05-23T10:00:00Z"
  }
}
```

CLI uses fields from this response to write `./.bonovalley/integration.json` (per §3.5) and the registry entry in `~/.bonovalley/integrations.json`.

---

## 7. Standard Operating Procedures (SOPs)

### SOP-1 — First-time developer setup

Audience: dev who just installed `bonovalley-cli` and has never used it.

1. Install the CLI (binary on PATH).
2. Run `bonovalley-platform login`.
   - Bootstraps `~/.bonovalley/` (config.json, integrations.json) with defaults.
   - Opens browser → OAuth → returns to local callback → fetches deployKey via API → writes `~/.bonovalleyrc`.
3. (Optional) Inspect / customize settings:
   - `bonovalley-platform config get project_parent_dir`
   - `bonovalley-platform config set project_parent_dir /Volumes/work/bv` (if they want integrations elsewhere)
4. Done. Ready to `register` or `link`.

### SOP-2 — Create a brand-new integration (no local code yet)

Audience: dev wants to start a new integration from scratch.

The flow is two commands run back-to-back: `init` creates the folder and pulls templates; `register` registers it with the API. They are intentionally separate (see §6 catalog notes).

1. **Run `bonovalley-platform init <key>`**, e.g. `bonovalley-platform init google-drive`.
   - CLI resolves target = `<project_parent_dir>/<key>`.
   - **Pre-flight**: refuses if target already exists and is non-empty, unless `--force` is passed (which wipes it).
   - `mkdir -p` the target.
   - Calls `GET /template` with Bearer deployKey — server-proxies the template repo as a `.tar.gz` (§13.3 / §13.4 + the API project's `controllers/template/`).
   - Extracts into target, stripping Bitbucket's wrapper directory.
   - Applies `.bvtemplate-ignore` (if present in the template) — drops template-developer-only files (`CLAUDE.md`, internal docs, etc.).
   - Prints: `Initialized template at /Users/amit/bonovalley-integrations/google-drive` plus a "next steps" hint pointing at register.

2. **`cd` into the new directory.**
   ```
   cd /Users/amit/bonovalley-integrations/google-drive
   ```

3. **Run `bonovalley-platform register`** to create the DB row and write the marker.
   - **Pre-flight checks** (§6.1.4): logged in? `default_organization_id` set? cwd does NOT already have a marker file?
   - **Interactive prompts** (see §6.1.1 for the full field list):
     ```
     Integration name:                Google Drive
     Key (lowercase, a-z and -):      google-drive   # default = folder basename
     Relationship with app:           [1-4]
     Category:                        Storage
     Intended audience:               [1-2]
     Security tier:                   [1-3] (default standard)
     Description (optional):          Sync files with Google Drive
     Website (optional):              https://drive.google.com
     ```
   - **Proceed prompt**: `Proceed? [Y/n]` — Enter accepts. `--yes` skips it for scripts.
   - **API call**: POST `/integrations` with the collected fields. Server derives `owner_id`, `bv_organization_id`, `built_type`, sets `approval_status = under_development`, returns the created row (see §6.1.6).
   - **Conflict handling** (§6.1.5): if key or name already exists, no DB row is created; CLI offers to `link` (if the dev owns it) or exits.
   - **Local writes** on success (inside cwd, NOT inside `project_parent_dir/<key>` — they are the same path in this flow but conceptually it's the cwd that matters):
     - Write `./.bonovalley/integration.json` with the stable identity fields (see §3.5).
     - Append entry to `~/.bonovalley/integrations.json` with cwd as the path.
     - Print: `Created integration 13021 (key=google-drive)`.

4. Start coding.

**Note on the project directory name**: by convention `init <key>` makes the folder name match the key, so when `register` runs from inside, its key-prompt default = `filepath.Base(cwd)` and the two stay aligned. Devs can override the key during the prompt but doing so creates a key/folder mismatch — the marker is still the source of truth for discoverability, but it's visually confusing.

### SOP-3 — Existing local code, not yet in DB

Audience: dev wrote code outside the platform; now wants to register it.

1. `cd existing-project-folder`.
2. **Skip `init`** — running it would wipe (with `--force`) or refuse (without). The dev's code is already here; they only need to register it.
3. Run `bonovalley-platform register`.
   - **Same interactive prompt sequence as SOP-2 step 3** (name, key, relationship, category, audience, security tier, description, website). See §6.1.1.
   - **Same pre-flight checks** (§6.1.4) — including: cwd does NOT already have `.bonovalley/integration.json` (would mean already registered).
   - **Proceed prompt**: `Proceed? [Y/n]`. Same as SOP-2.
   - **Same API call + conflict handling** (§6.1.5, §6.1.6).
   - **Local writes** on success: writes `./.bonovalley/integration.json` in cwd and appends to the registry.

**Difference from SOP-2**: no template download, no involvement of `project_parent_dir`. The existing folder is used as-is — its name does NOT need to match the integration key. The marker file (committed inside the folder) is what makes it discoverable, not the folder name.

### SOP-4 — Already-registered integration, new machine

Audience: dev set up the integration on laptop A, now on laptop B.

1. `git clone <integration-repo>` into wherever (cwd or under `project_parent_dir`, dev's choice).
2. `cd <integration-repo>`.
3. The committed marker file `.bonovalley/integration.json` is already there with the `integration_id`.
4. Run `bonovalley-platform link`.
   - CLI reads marker, calls API to verify the user owns this `integration_id`.
   - Adds entry to this machine's `~/.bonovalley/integrations.json` with the current absolute path.
5. Start working. **No `init`** — code is already there.

### SOP-5 — Re-open an integration (forgot where it lives)

Audience: dev wants to come back to integration `13021` and isn't sure of the path.

Three lookups, in order of effort:

1. `bonovalley-platform list` — table of all integrations tracked on this machine.
2. `bonovalley-platform path 13021` — prints just the absolute path. Use `cd $(bonovalley-platform path 13021)` to jump.
3. `bonovalley-platform list --remote` — hits API for the user's full integration list (truth). Each row marks whether it's linked locally on this machine.

### SOP-6 — Moved a project folder

Audience: dev physically moved `~/bv/foo` → `~/work/foo`.

1. `cd ~/work/foo`.
2. Run **any** CLI command (e.g. `bonovalley-platform status`).
3. CLI reads cwd, walks up to find marker, finds `integration_id=13021`, looks up 13021 in registry, sees stored path differs from cwd, **silently updates registry path** (logs one line: `Updated path for integration 13021`).
4. Done. No explicit command needed.

Explicit alternative: `bonovalley-platform link --update` from inside the new location.

### SOP-7 — Multiple integrations in parallel

Audience: dev wants to work on integration A and B at the same time.

1. Terminal 1: `cd <path-to-A>` — all commands target A.
2. Terminal 2: `cd <path-to-B>` — all commands target B.
3. No "active" flag, no conflict. `cwd` is the switch.

### SOP-8 — Change `project_parent_dir`

Audience: dev wants future integrations to land in a different default folder.

1. `bonovalley-platform config set project_parent_dir /new/parent`.
2. CLI prints: "New integrations will be created under /new/parent. Existing integrations are unaffected."
3. **Existing** integrations' paths in the registry are not modified. They stay where they physically are.
4. To physically move an existing integration too, see SOP-6.

### SOP-9 — Recover from a wiped/broken local registry

Audience: dev deleted `~/.bonovalley/integrations.json` (or it got corrupted, or fresh install).

1. Delete or rename the bad file (if not already gone).
2. Run any command — CLI auto-recreates an empty registry via lazy bootstrap.
3. For each integration folder on disk: `cd <folder> && bonovalley link`.
4. To find folders you might have forgotten: `bonovalley-platform list --remote` shows everything the API knows you own; cross-reference with disk.

### SOP-10 — Stop tracking locally but keep the integration

Audience: dev doesn't want this integration showing up in `bonovalley-platform list` on this machine.

1. `bonovalley-platform unlink <id>`.
2. Removes registry entry only. Marker stays in the project folder. API row stays. `link` re-enables tracking later.

### SOP-11 — Delete an integration entirely

Audience: dev wants the integration gone from DB and disk.

1. `bonovalley-platform delete <id>`.
2. CLI prompts for confirmation (with the name and path).
3. API call to delete the DB row.
4. Registry entry removed.
5. Project folder: CLI asks `Also delete the local folder at <path>? (y/N)`. Default `N`.

### SOP-12 — Push the integration to the platform

Audience: dev wants to ship the current state of their integration. Runs `setup_parent.go` from the integration's template, packages the resulting artifacts, uploads to the API, cleans up.

The heavy lifting (building, validation, codegen, schema check) lives in `bv-integration-platform-core/pckgs/setupprep/setup_parent.go` inside the integration repo itself. The CLI is a thin orchestrator + uploader.

**Pre-conditions:**
- Run from inside a registered integration folder (marker file present in cwd or ancestor).
- Logged in (deployKey in `~/.bonovalleyrc`, `default_organization_id` in config).
- Go toolchain on PATH.

**Steps:**

1. **Pre-flight.** Resolve `integration_id` from the marker file; resolve `default_organization_id` from `~/.bonovalley/config.json`. Refuse if either missing.

2. **Verify Go toolchain + goimports.** `go version` must succeed. If `goimports` is missing from PATH, **CLI auto-installs it** via:
   ```
   go install golang.org/x/tools/cmd/goimports@latest
   ```
   (setup_parent.go requires goimports at its Step-6; pre-installing avoids a confusing late failure.)

3. **Install project dependencies.** Always run `go mod tidy` (fast no-op when everything is already in `go.sum`). Skips the surprise of a stale dependency tripping up `go run` later.

4. **Build + validate via `setup_parent.go`.** Single `go run` call:
   ```
   go run bv-integration-platform-core/pckgs/setupprep/setup_parent.go
   ```
   Stream stdout/stderr to the dev as it runs — this is where they see the build progress, schema-validation results, etc. Internally setup_parent.go:
   - Removes any prior `BonoValley-d-temp-*` folder
   - Creates a fresh `BonoValley-d-temp-<random-digits>` folder
   - Runs `setupprep1.go`, `setupprep2.go`, `setupprep3.go` from inside the temp folder
   - Runs `goimports -w .`
   - Runs schema validation tests (`jsonvalidator.InitializeValidationTests`)
   - Runs `integration-builder.go` which produces the two artifacts the CLI will upload
   .
   On non-zero exit: push fails. Preserve the temp folder for debugging and print its absolute path.

5. **Locate artifacts.** After setup_parent.go succeeds, find the folder matching the pattern `BonoValley-d-temp-*` (the `-*` suffix is a random number that varies every run; the prefix `BonoValley-d-temp-` is stable). Inside that folder, the two upload targets are:
   - `hello` — the compiled integration binary
   - `integrationdefinition.json` — generated metadata (lowercase filename; the API doc comment's `IntegrationDefinition.json` is stale)

6. **Upload via multipart POST.**
   ```
   POST https://cli-api.bonovalley.com/upload/files?orgId=<orgId>&integrationId=<integrationId>
   Authorization: Bearer <deployKey>
   Content-Type: multipart/form-data
   ```
   Both files attach under the **same form field name `"file"`** (the server reads `r.MultipartForm.File["file"]` as a slice). Field shape:
   ```
   --boundary
   Content-Disposition: form-data; name="file"; filename="hello"
   <binary bytes>
   --boundary
   Content-Disposition: form-data; name="file"; filename="integrationdefinition.json"
   <json bytes>
   --boundary--
   ```
   Auth via Bearer deployKey — same scheme as every other authenticated endpoint (§13.2).

7. **Surface server response.** The server runs additional validation (version-number qualification per `integrations_versions.approval_status`; rejects pushes to versions already in `approved`/`under_review`/`rejected`). On 200: report the accepted version. On 4xx: surface the server's error message verbatim so the dev sees the actual reason.

8. **Cleanup.**
   - On success → `rm -rf BonoValley-d-temp-*`.
   - On failure → leave the temp folder intact and print its absolute path so the dev can inspect.

**Progress display.** Use unicode marks (`✔ ✗ →`) by default — Windows 10+, Mac, and modern Linux terminals all render these natively. ASCII fallback (`[ok] [x] >`) available via `--ascii` flag for older terminals or scripts piping output.

Sample happy-path output:

```
$ bonovalley-platform push
→ Pre-flight: integration 13031 (google-drive), org abc123-...
→ Verifying Go toolchain + goimports
✔ Toolchain ready
→ Installing project dependencies (go mod tidy)
✔ Dependencies up to date
→ Building integration (setup_parent.go)
  · Cleaning previous temp directory
  · Created BonoValley-d-temp-3583389725
  · Running setupprep1, setupprep2, setupprep3
  · Running goimports
  · Running schema validation tests
  · Building integration binary
✔ Build complete in 28.4s
→ Uploading 2 files (hello + integrationdefinition.json) to cli-api.bonovalley.com
✔ Upload accepted: integration 13031 version 1.2.3
→ Cleaning up temp directory
✔ Cleaned

Pushed integration 13031 (google-drive) version 1.2.3.
```

Sample failure output (setup step):

```
$ bonovalley-platform push
→ Pre-flight: integration 13031 (google-drive), org abc123-...
→ Verifying Go toolchain + goimports
✔ Toolchain ready
→ Installing project dependencies (go mod tidy)
✔ Dependencies up to date
→ Building integration (setup_parent.go)
  · Cleaning previous temp directory
  · Created BonoValley-d-temp-3583389725
  · Running setupprep1, setupprep2, setupprep3
✗ Build failed: setupprep2 returned exit code 1

The temp directory was preserved for inspection:
  /Users/amit/.../BonoValley-d-temp-3583389725

Re-run after fixing the underlying issue, or 'rm -rf BonoValley-d-temp-*'
to discard.
```

**Guard rails:**
- Refuse if cwd has no marker (use the same "walk-up like git" logic as `status`).
- Refuse if `setup_parent.go` not found at the expected path inside the project — that's a structural problem with the integration repo, not something push can fix.
- Refuse on multiple `BonoValley-d-temp-*` folders after the run (shouldn't happen because setup_parent.go's Step-0 deletes old ones, but defensive — if it does, print all of them and abort rather than guessing).

**OPEN** — does the API hard-delete or soft-delete? Is there a `deprecate` middle ground?

---

## 8. Guard rails (refuse-loudly behavior)

| Situation | Behavior |
|---|---|
| `register` in folder that already has `.bonovalley/integration.json` | Refuse: "already linked to integration X. Use `unlink` first if you really want to register a new one here." |
| `init <name>` target dir (`<project_parent_dir>/<name>`) already exists and is non-empty | Refuse unless `--force` (which wipes). |
| `register` where API reports `key` or `integration_name` already exists | API returns `status=already_exists` with the existing row (HTTP 200, not 4xx). CLI: if owned by this user → offer to `link` to that integration; if owned by another user → print and exit. No row created, no local files written. See §6.1.5. |
| `register` `key` doesn't match `^[a-z]+(-[a-z]+)*$` (CLI-side regex check) | Refuse before hitting API: "Invalid key. Use only lowercase letters (a-z) and single hyphens between letters." |
| `register` with `default_organization_id == null` in local config | Refuse: "Run `bonovalley-platform login` first." See §6.1.4. |
| `init` in folder with no marker | Refuse: "no integration in this folder. Run `register` or `link` first." |
| `init` in folder that already has scaffolded source (e.g. `main.go` present) | Refuse unless `--force`. |
| `link` in folder with no marker AND no `<id>` argument | Refuse: "no marker found; pass an integration id: `bonovalley-platform link <id>`." |
| `link <id>` where the user doesn't own that id (API check) | Refuse. |
| `link <id>` where the registry already has that id at a different path | Refuse: "13021 is already linked at X. Use `unlink` first or pass `--update` to move it." |
| Any command requiring auth, run before `login` | Refuse with: "not logged in. Run `bonovalley-platform login` first." |
| `config set project_parent_dir <path>` where path is unwritable | Refuse with permission error. |

---

## 9. Self-healing behaviors

| Trigger | Action |
|---|---|
| `~/.bonovalley/` missing on any command | Auto-recreate with defaults. |
| `~/.bonovalley/config.json` missing | Auto-recreate with defaults. |
| `~/.bonovalley/integrations.json` missing | Auto-recreate as empty registry. |
| Project's path in registry doesn't match where the dev is running the command | Silently update the path field. Log one line. |
| Registry has an entry whose path no longer exists on disk | `list` flags it with `[stale]`. Don't auto-delete (could be temporarily unmounted external drive / pCloud not synced). |

---

## 10. Cross-platform & multi-machine considerations

- **Path strings differ across OSes** — store paths in the registry exactly as the OS returns them (`os.Getwd()`). Don't normalize to forward slashes. Don't try to be portable across OSes for paths.
- **The project marker is path-free** — that's what makes the project itself portable.
- **Same user, multiple machines** — each machine has its own registry. Truth lives in the project repos. Onboarding a new machine: `git clone` each integration, run `bonovalley-platform link` inside each. Or run `bonovalley-platform list --remote` first to remember what to clone.
- **pCloud / cloud-synced home directories** — if the dev's `$HOME` is itself synced (e.g. pCloud), DO NOT also sync `~/.bonovalley/integrations.json` between machines: it would carry paths that don't exist on the other side. If unavoidable, document the caveat — re-link on each machine after sync.

---

## 11. Open / undecided items

Tracked here so neither Claude nor the dev forgets them.

| # | Item | Why it matters |
|---|---|---|
| 1 | ~~`register` API contract — mandatory fields, optional fields, defaults, validation rules~~ | **CLOSED.** See §6.1 for the full contract: collected fields, server-derived fields, deferred fields, pre-flight checks, conflict handling, response shape. |
| 2 | ~~Integration naming convention~~ | **CLOSED.** Key format `^[a-z]+(-[a-z]+)*$`, uniqueness enforced server-side on `key` and `integration_name`. Project directory name = `key`. See §6.1.1. |
| 3 | Authorization model | One deployKey per user is decided. But: does the API enforce that the user behind the deployKey owns the requested integration_id on **every** command (`push`, `logs`, `link`, etc.)? Required to prevent marker-file tampering. To be confirmed by reading `bv-integrations-cli-rest-api`. |
| 4 | ~~Organization scoping~~ | **CLOSED.** `default_organization_id` is populated by `login` and can never be null after a successful login. If null on any command → "run `bonovalley-platform login`". `bv_organization_id` on the integrations table is server-derived from auth context (only set when `relationship_with_app = bv_employee`). |
| 5 | `delete` semantics | Hard delete vs soft delete vs `deprecate`? |
| 6 | ~~`init` template content~~ | **CLOSED.** Templates live in the `bv-integrations-platform` Bitbucket repo. `init` fetches the latest as a tarball via the API's `/template` endpoint and applies `.bvtemplate-ignore` to drop template-developer-only files (CLAUDE.md, internal docs, etc.). Change templates by editing that repo; no CLI redeploy needed. |
| 7 | Migration from `~/.bonovalleyapprc` | Old single-integration global file. Plan: on first `login` after upgrade, migrate then delete. Deferred — only matters for devs upgrading from a CLI version that wrote this file. Worth ~30 lines in `bvbootstrap`; will land if a real upgrader hits friction. |
| 8 | ~~`~/.bonovalley-platform-cli.yaml` (Viper config)~~ | **CLOSED.** Dropped entirely. Viper plumbing removed from `cmd/root.go` in commit `82295d8`; `go.mod` collapsed from ~20 deps to 3 direct + 4 indirect. |
| 9 | ~~`push` flow detail~~ | **CLOSED.** Full workflow in SOP-12. Push is a thin orchestrator over the integration repo's own `setup_parent.go`. Only 2 files get uploaded (`hello` + `integrationdefinition.json`) — both produced inside `BonoValley-d-temp-*`. No `.bonovalley/cache/` exclusion question because we don't upload anything from the project directory itself. |
| 10 | Whether `bonovalley-platform new <name>` (register + init combined) is worth adding as a convenience wrapper | After we see if `register && init` is the common case. |
| 11 | ~~What happens when the dev picks `pci` or `hipaa` at register~~ | **CLOSED.** Deferred like `icon`. CHECK constraints relaxed to only fire when leaving `under_development`. See §6.1.3 for the schema changes (DROP + re-ADD of both constraints with an `approval_status = 'under_development'` escape clause). |
| 12 | The `submit-for-review` / `transition out of under_development` command | The deferral pattern in §6.1.3 means there's an implicit "you've finished development, now your icon / sensitive_fields / etc. must be present" gate. Need a dedicated command (e.g., `bonovalley-platform submit-for-review`) that runs all the pre-checks the relaxed constraints will enforce on the API side. Otherwise the dev hits a wall of DB errors on review submission. |
| 13 | ~~API server: `/get-deploy-token` doesn't return `default_organization_id` inline yet~~ | **CLOSED (2026-05-23).** `bv-integrations-cli-rest-api` 1.0.3 deployed to Cloud Run (revision `00003-rx9`) with: a new `GetUserPrimaryOrgID` method on `UserServices` that queries `integration_partners.bv_organization_ids` via GraphJin; auth middleware now sets `ContextUserIDForFirstGetDeployKeyRequest` + `ContextDefaultOrgIDForFirstGetDeployKeyRequest`; `model.DeployToken` extended; handler includes both inline. End-to-end verified — `default_organization_id` populates correctly in `~/.bonovalley/config.json` after `bonovalley-platform login`. "Primary org" today = first element of `bv_organization_ids`; if a real "default org" concept lands later, replace `GetUserPrimaryOrgID`'s body in `data/postgress/users.go`. |
| 14 | DeployKey effective TTL is ~1 hour because it's a thin wrapper around the underlying Ory access_token | Symptom: every authenticated endpoint (`/template`, `/integrations` POST + GET) starts returning 401 ~1 hour after login with "invalid deployKey. Your Authentication Expired." Cause: `engine/auth.go`'s middlewares decrypt the deployKey to recover the access_token and re-validate against Ory userinfo on every request — when Ory says expired, the deployKey is effectively dead. Hit 4-5 times during 2026-05-23 and 2026-05-24 testing. **Two fixes worth exploring:** (a) server-side refresh-token flow — exchange the access_token for an Ory refresh_token at `/get-deploy-token` time, store it server-side keyed by deployKey, refresh transparently on auth check (Ory's services config already has `refresh_allowed: true`); (b) client-side — CLI catches 401, prompts to re-login automatically. (a) is the right architecture; (b) is a stopgap UX win. Until either lands, devs must `bonovalley-platform login` every ~hour. |
| 15 | `link <id>` 403 (foreign owner) branch not live-tested | The CLI's `link` command and server's `GET /integrations/{id}` both have ownership-check paths that return HTTP 403 if the authenticated user doesn't own the requested integration. The code path is a single equality (`integration.OwnerID != userID`). Not exercised end-to-end because we'd need a second authenticated user account with an integration to point at. Should be tested whenever a second test account becomes available. |
| 16 | Key regex disallows digits — design choice to revisit | Current rule: `^[a-z]+(-[a-z]+)*$` (lowercase letters with single hyphens between letter groups, no digits). Bit us in test setup — `link-smoke-1` rejected. Real-world slugs commonly include digits (`oauth-v2`, `s3-bucket-watch`). The user's original spec said "only lowercase a-z and -" (literal) — relaxing to allow digits is a one-line change to `cmd/commands/register/register.go` keyRegex (e.g., `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`). Confirm intent before changing. |

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **deployKey** | Per-user secret returned by the API after OAuth login. Stored in `~/.bonovalleyrc`. Used in `Authorization` header for every API call. |
| **integration_id** | DB primary key of a row in the `integrations` table. Created by `register`. Lives in the marker file. |
| **marker** | The `./.bonovalley/integration.json` file inside a project folder. Identifies the folder as a bonovalley integration. |
| **registry** | `~/.bonovalley/integrations.json` — the local list mapping `integration_id` → on-disk path for this machine. |
| **project_parent_dir** | The default folder under which new integrations are created. Configurable. |
| **bootstrap** | The one-time-ish creation of `~/.bonovalley/` + default config + empty registry. Happens on `login` or lazily on first command. |

---

## 13. API client basics

The CLI talks to one backend — the integrations CLI REST API. Every API-touching command (register, link, push, login, logs, etc.) goes through one Go package: `engine/tools/bvapi`. See §6.1 for the `register` endpoint's field-level contract; this section covers the cross-cutting basics.

### 13.1 Base URLs

| `api_env` value | Base URL |
|---|---|
| `dev` | `http://localhost:8000` |
| `prod` | `https://cli-api.bonovalley.com` |

The CLI resolves the base URL once at `NewClient` time by reading `api_env` from `~/.bonovalley/config.json`. Default on a fresh install is `prod`; devs override with `bonovalley-platform config set api_env dev` for local backend work.

### 13.2 Authentication

After successful `bonovalley-platform login`, the user's deployKey is stored in `~/.bonovalleyrc`. Every authenticated API call sends it as a Bearer token:

```
Authorization: Bearer <deployKey>
```

The CLI never sends raw OAuth `access_token`s to the production API beyond the initial exchange. `login` swaps the access_token for the long-lived deployKey at `/get-deploy-token` once; every subsequent command carries the deployKey.

### 13.3 File-upload pattern (for `push` and similar)

Most CLI ↔ API traffic is JSON. The file-upload endpoints are the exception — they use multipart form-data so they can carry one or more files plus query-param metadata. Standard shape:

```http
POST /upload/files?orgId=<org>&integrationId=<id> HTTP/1.1
Host: cli-api.bonovalley.com
Authorization: Bearer <deployKey from ~/.bonovalleyrc>
Content-Type: multipart/form-data; boundary=...

--boundary
Content-Disposition: form-data; name="files"; filename="hello"
Content-Type: application/octet-stream

<file bytes>
--boundary
Content-Disposition: form-data; name="files"; filename="integrationdefinition.json"
Content-Type: application/json

<file bytes>
--boundary--
```

Components:

- **Query params**: `orgId` (from `default_organization_id` in `~/.bonovalley/config.json`) and `integrationId` (from the project's marker file `.bonovalley/integration.json`).
- **Auth**: Bearer deployKey, as §13.2 — no special handling for multipart.
- **Body**: one form part per file. Filename and content-type set per-file. The example above shows two: a generic `hello` file (octet-stream) and an `integrationdefinition.json` (application/json).

### 13.4 Endpoint catalog (initial scope)

| Method | Path | Auth | Used by | Notes |
|---|---|---|---|---|
| `POST` | `/get-deploy-token` | OAuth access_token in body (one-shot, not deployKey) | `login` | Exchanges access_token for deployKey. Returns `{ deploy_key, user_id, default_organization_id }` inline so `login` only needs one round-trip to populate both `~/.bonovalleyrc` and `~/.bonovalley/config.json`. |
| `GET` | `/me` | Bearer deployKey | future commands (`whoami`, etc.) | Returns full user/profile info. **Not needed by `login`** — org id comes from `/get-deploy-token` above. Reserved for later commands that need richer profile data. |
| `POST` | `/integrations` | Bearer deployKey | `register` | **LIVE** (image 1.0.10). Field contract: §6.1. Idempotent on `key` or `integration_name` collision (returns existing row with `status: already_exists`). Server-derives owner_id, bv_organization_id, built_type, security_tier (forced "standard" in v1), approval_status. |
| `GET` | `/integrations/{id}` | Bearer deployKey | `link <id>`, ownership checks | **LIVE** (image 1.0.10). 200 with `model.Integration`, 404 if missing, 403 if owned by another user, 400 if id not numeric. Auth via `bearerOnlyAuthMiddleware` (same as POST /integrations; no integrationId in query). |
| `GET` | `/integrations` | Bearer deployKey | `list --remote` (Tier 2 polish) | Not implemented. |
| `POST` | `/upload/files?orgId=&integrationId=` | Bearer deployKey | `push` | Multipart form-data per §13.3. |

### 13.5 Error envelope (OPEN — confirm with API)

Assumed shape on 4xx/5xx for the `bvapi.APIError` Go type:

```json
{
  "code": "validation_failed",
  "message": "Human-readable explanation.",
  "details": { ... optional structured detail ... }
}
```

Confirm what `bv-integrations-cli-rest-api` actually returns so the client's error parsing matches.
