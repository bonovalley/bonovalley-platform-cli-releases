# Deployment SOP — bonovalley-platform-cli

This document is the **canonical procedure** for cutting a new tagged release of `bonovalley-platform-cli`, publishing the cross-compiled binaries to Bitbucket Downloads, and verifying that integration partners can actually download, install, and use the binary against the live platform.

Two audiences read this doc:

- **You** (the human maintainer) — to know what's happening, give the go-ahead, and provide credentials when asked.
- **Claude** (me, the AI assistant) — to execute the steps in order and report progress.

Every step below is marked **(USER)**, **(CLAUDE)**, or **(BOTH)** so neither party guesses what the other is doing.

---

## 0. Overview

Cutting a release is a single sequence with **15 numbered steps** (sections 4-18 below). Most steps run on a maintainer's machine; a couple require Bitbucket UI clicks or a one-time app password. The whole sequence is **15-25 minutes** of wall time end-to-end, the bulk of which is the cross-compile + smoke tests.

The artifacts a partner ends up with are the same regardless of who in the team runs the SOP:

```
Bitbucket Downloads
    └── bonovalley-platform-windows-amd64.exe   (Windows x64)
    └── bonovalley-platform-linux-amd64         (Linux x64)
    └── bonovalley-platform-darwin-amd64        (macOS Intel)
    └── bonovalley-platform-darwin-arm64        (macOS Apple Silicon)
    └── SHA256SUMS                              (integrity manifest)
```

Each binary self-identifies via `bonovalley-platform --version` so the partner can include their build in any bug report.

---

## 1. Lifecycle (text diagram)

```
                       ┌─────────────────────────────┐
                       │   USER says "deploy vX.Y.Z" │
                       └────────────┬────────────────┘
                                    │
                            (CLAUDE executes §4-§14)
                                    │
                                    ▼
        ┌──────────────────────────────────────────────────┐
        │  master clean? tests pass? doctor green?         │   §4 pre-flight
        ├──────────────────────────────────────────────────┤
        │  CHANGELOG.md: move Unreleased → vX.Y.Z          │   §5
        │  git commit "Release vX.Y.Z"                     │
        ├──────────────────────────────────────────────────┤
        │  git tag -a vX.Y.Z -m "Release vX.Y.Z"           │   §6
        ├──────────────────────────────────────────────────┤
        │  scripts/build-release.ps1 -Version vX.Y.Z       │   §7
        │  dist/vX.Y.Z/ now holds 4 binaries + SHA256SUMS  │
        ├──────────────────────────────────────────────────┤
        │  smoke: dist/.../*.exe --version, doctor         │   §8
        ├──────────────────────────────────────────────────┤
        │  git push origin master                          │   §9
        │  git push origin vX.Y.Z                          │
        ├──────────────────────────────────────────────────┤
        │  upload 5 files to Bitbucket Downloads (§10)     │   §10
        │    REST API path (preferred) if app pw provided  │
        │    UI drag-drop fallback                          │
        ├──────────────────────────────────────────────────┤
        │  verify public URL serves the binary             │   §11
        ├──────────────────────────────────────────────────┤
        │  simulate partner journey                        │   §12
        │    curl download → --version → doctor            │
        │    (login + push require USER hands)              │
        ├──────────────────────────────────────────────────┤
        │  append release journal entry (§13)              │   §13
        │  commit the journal entry, push                  │
        └──────────────────────────────────────────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────────┐
                       │  partner: download → run    │
                       │  --version / login / push   │
                       └─────────────────────────────┘
```

---

## 2. How you (USER) request a deploy

When you want a release cut, say one of:

- **"Deploy v1.0.0"** — exact semver tag including the leading `v`
- **"Cut release v1.0.0"** — same effect

I (Claude) will:

1. Echo back the version + the commits that will be included (so you can sanity-check before anything is committed).
2. Wait for your **"go"** before I touch `CHANGELOG.md`, tag, or push.
3. Execute steps 4 through 13 in order; pause for your input at step 10 if I don't have Bitbucket credentials cached.
4. Report end-of-run with the journal entry.

If you want a **dry run** (everything except the push + upload + journal commit), say so explicitly: "Deploy v1.0.0 — dry run". I'll build artifacts under `dist/`, smoke-test them, and stop short of any irreversible action.

---

## 3. Release artifacts

Per release we publish to **GitHub Releases** on the sibling repo (Bitbucket source repo + GitHub releases repo — Bitbucket Free plan doesn't support Repository Downloads as of 2025). Concretely:

- **Source code** lives at <https://bitbucket.org/bonovalley/bonovalley-platform-cli>
- **Release binaries** live at <https://github.com/{GH_REPO}/releases> (the repo name is read from the maintainer's `GH_REPO` env var or credentials file; see §5)

Every filename embeds the version. Pattern: `bonovalley-platform-vX.Y.Z-<os>-<arch>[.exe]` — same convention as `gh`, `terraform`, `hugo`, `kubectl`.

| File (example for v1.0.0) | Target | Notes |
|---|---|---|
| `bonovalley-platform-v1.0.0-windows-amd64.exe` | Windows x64 | Most common partner platform today |
| `bonovalley-platform-v1.0.0-linux-amd64` | Linux x64 | For partners on WSL2 or native Linux dev boxes |
| `bonovalley-platform-v1.0.0-darwin-amd64` | macOS Intel | |
| `bonovalley-platform-v1.0.0-darwin-arm64` | macOS Apple Silicon (M1/M2/M3/M4) | |
| `SHA256SUMS-v1.0.0` | Integrity manifest | `sha256sum -c SHA256SUMS-v1.0.0` validates downloads |

Each binary is stripped (`-s -w`), reproducible-path (`-trimpath`), and self-identifies via `--version` (commit SHA + UTC build date embedded at link time via the `engine/tools/bvversion` package).

Multiple versions coexist on the Downloads page without collision (`bonovalley-platform-v1.0.0-windows-amd64.exe` and `bonovalley-platform-v1.0.1-windows-amd64.exe` are different objects). Partners find the current version from `README.md`'s install table, which is updated per release; advanced partners can also `git ls-remote --tags origin` for the newest tag.

---

## 4. Versioning

Semver: `vMAJOR.MINOR.PATCH` (and optional `-rc1`, `-beta1`, etc. for pre-releases).

| Change | Bump |
|---|---|
| Breaking change to command names, flag semantics, marker file shape, or registered API contract | MAJOR |
| New command, new optional flag, new output line | MINOR |
| Bug fix, perf, log/output tweaks, doc-only | PATCH |

**Reuse rule.** Never reuse a tag. Once `v1.0.0` is pushed, even a deletion + re-push is not allowed — partners may have cached the old artifact. Bump to `v1.0.1` instead.

The very first public release is **v1.0.0**. Pre-1.0 work happened on `master` as untagged commits and is not eligible for partner download.

---

## 5. Prerequisites

| Required | Owner | Notes |
|---|---|---|
| Go 1.22+ installed and on PATH | CLAUDE machine | I check this myself before building |
| PowerShell 7+ (`pwsh`) on PATH | CLAUDE machine | Required by `scripts/build-release.ps1` |
| git push access to `bitbucket.org/bonovalley/bonovalley-platform-cli` | CLAUDE machine (uses the cached credentials of the local checkout) | If push fails on credentials, USER provides them |
| Bitbucket account with write access to repo's Downloads page | USER | The repo owner controls this. Settings → Repository details → User and group access |
| **GitHub PAT** (`GH_TOKEN`) — Classic with `repo` scope, OR fine-grained with `Contents: write` on the releases repo | USER provides ONCE; reusable across releases | Needed for the GitHub Releases upload in §9. Create at <https://github.com/settings/tokens>. Stored in `<repo>/.env.bonovalley-release-credentials` (gitignored). |
| **GitHub releases repo** (`GH_REPO`, e.g. `bonovalley/bonovalley-platform-cli-releases`) | USER creates ONCE | The sibling repo that hosts the binary Releases. Source stays on Bitbucket. Same env-file row as the PAT. |
| Fresh CLI login (deployKey valid) on CLAUDE machine | CLAUDE | Required for the partner-journey simulation in §12 |

No CI is wired up today — every release is a manual run of `scripts/build-release.ps1` from a maintainer's machine. **Future work:** wire Bitbucket Pipelines to automate steps 7-11 on tag push.

---

## 6. Release procedure (steps 1-13)

Each step shows its owner in **(USER)** / **(CLAUDE)** / **(BOTH)**, the estimated time, and the exact commands.

### Step 1 — **(USER)** Request the release · *< 10s*

You say: **"Deploy v1.0.0"** (substitute the real version).

### Step 2 — **(CLAUDE)** Pre-flight verification · *~30s*

I run the following without modifying anything:

```bash
git status                # must be clean
git log --oneline -10     # show what's been merged since the last tag
go test ./...             # must pass (no tests yet → reports 0 tests)
go build ./...            # must compile cleanly
go run main.go doctor     # all 6 checks must pass
```

Then I list the commits since the last tag (or all commits, for v1.0.0) so you can confirm the scope is right.

> **CHECKPOINT** — I'll show you the commit list and the proposed version number. Reply **"go"** to continue, or push back if anything's off.

### Step 3 — **(CLAUDE)** Update CHANGELOG.md · *~10s*

I move every line from `## [Unreleased]` into a new dated section `## [v1.0.0] — YYYY-MM-DD`. Categories used (only non-empty ones are kept): `Added`, `Changed`, `Fixed`, `Deprecated`, `Removed`, `Security`. The `## [Unreleased]` heading stays in place (empty) for the next cycle.

If `## [Unreleased]` is empty (no changes since last tag), I refuse to proceed — there's nothing to release.

### Step 4 — **(CLAUDE)** Commit the release · *~5s*

```bash
git add CHANGELOG.md
git commit -m "Release v1.0.0"
```

### Step 5 — **(CLAUDE)** Tag the release · *~5s*

Annotated tag (carries author + message; lightweight tags don't):

```bash
git tag -a v1.0.0 -m "Release v1.0.0"
```

The tag is **local only** at this point — not pushed yet (so a failed build doesn't leave a tag pointing at nothing).

### Step 6 — **(CLAUDE)** Build cross-platform binaries · *~3-5 minutes*

```powershell
.\scripts\build-release.ps1 -Version v1.0.0
```

The script:
- Refuses if the working tree is dirty (guardrail against unreproducible builds)
- Reads `git rev-parse --short HEAD` and `Get-Date -AsUTC` for the embedded `Commit` + `BuildDate`
- Builds 4 binaries (windows/amd64, linux/amd64, darwin/amd64, darwin/arm64) with `-trimpath -ldflags="-s -w -X bonovalley-platform-cli/engine/tools/bvversion.Version=v1.0.0 ..."`
- Writes `dist/v1.0.0/SHA256SUMS`
- Prints a table of artifact sizes

### Step 7 — **(CLAUDE)** Smoke-test the built binary · *~10s*

```powershell
.\dist\v1.0.0\bonovalley-platform-windows-amd64.exe --version
# expected: bonovalley-platform-cli v1.0.0 (commit abc123, built 2026-05-24)

.\dist\v1.0.0\bonovalley-platform-windows-amd64.exe doctor
# expected: 6/6 pass, exit 0
```

If either fails, I stop, delete the local tag (`git tag -d v1.0.0`), and report back so we can fix on master.

### Step 8 — **(CLAUDE)** Push commit + tag to origin · *~5s*

```bash
git push origin master       # the release commit
git push origin v1.0.0       # the tag itself
```

This is the **first irreversible step**. Up to here, everything is local; from here on, rollback requires cutting a new patch (see §15).

### Step 9 — **(CLAUDE)** Upload binaries to GitHub Releases · *~30s*

> **Distribution-host note.** Bonovalley's Bitbucket workspace is on the Free
> plan, which (since 2025) returns HTTP 402 on any Repository Downloads
> upload — API or UI. Source code stays on Bitbucket; binary releases live
> on a sibling GitHub repo (one-time setup; see `docs/CREDENTIAL_TRACKING.md`
> §4a). The auth credential is a GitHub PAT, not an Atlassian token.

Credentials come from `<repo>/.env.bonovalley-release-credentials` (gitignored). The two keys read: `GH_TOKEN` and `GH_REPO`.

```powershell
pwsh ./scripts/upload-release.ps1 -Version v1.0.0
```

The script:
- Reads `GH_TOKEN` + `GH_REPO` (env vars first, then the in-project dotenv file, then the legacy `$HOME` JSON)
- Looks up the release for the given tag; **creates** it if missing (GitHub auto-creates the matching tag on the repo's default branch)
- PATCHes the release body to the current rich template (per-OS install + verify + integrity steps, with the version + repo substituted in). Template lives inline in `Get-ReleaseBody` in the script — edit there to change every future release page in one place. Idempotent: if the body already matches, skipped.
- POSTs every file in `dist/v1.0.0/` as a release asset via `Invoke-RestMethod -InFile` (streams, no memory load). Prints `OK` / `SKIPPED` / `FAILED` per file; existing assets are SKIPPED (delete in the GitHub UI first if you need to replace one).
- **Syncs the GitHub repo's `README.md`** from `docs/GITHUB_RELEASES_README.md` in the source repo. One-way sync (source repo wins; any direct edits on GitHub get overwritten). Idempotent: only commits if content differs. This is what partners see when they land on `github.com/<GH_REPO>` — so the install steps stay accurate without anyone remembering to update the GitHub repo manually.
- Exits non-zero on any FAILED asset upload so the SOP stops here.

Output looks like:

```
Looking up release 'v1.0.0' on https://github.com/<GH_REPO> ...
  No release for v1.0.0 yet. Creating one ...
  Created release id=12345678 at https://github.com/<GH_REPO>/releases/tag/v1.0.0

Uploading 5 file(s) from C:\...\dist\v1.0.0
  SHA256SUMS-v1.0.0                                            OK
  bonovalley-platform-v1.0.0-darwin-amd64                      OK
  bonovalley-platform-v1.0.0-darwin-arm64                      OK
  bonovalley-platform-v1.0.0-linux-amd64                       OK
  bonovalley-platform-v1.0.0-windows-amd64.exe                 OK

Done. Release page:
  https://github.com/<GH_REPO>/releases/tag/v1.0.0
```

No UI fallback today — GitHub Releases' web UI uploads work fine, but if the API is failing it's almost always either a token-scope issue or a missing `GH_REPO`, both of which are faster to fix than dragging files through a browser.

### Step 10 — **(CLAUDE)** Verify the public URL serves the binary · *~5s*

```bash
curl -sSIL "https://github.com/<GH_REPO>/releases/download/v1.0.0/bonovalley-platform-v1.0.0-windows-amd64.exe" \
  | grep -iE "^HTTP|^content-length"
# expected: HTTP/1.1 200 OK (after a 302 to objects.githubusercontent.com)
#           content-length: ~8.4 MB
```

If the URL 404s, the upload didn't take — retry §9 (the script is idempotent; existing assets are skipped, missing ones get re-tried).

### Step 11 — **(CLAUDE)** Simulate partner install + first-run journey · *~30s*

This is the new step that didn't exist in the old SOP. It catches the most common "the binary works on my machine but partners can't use it" class of bugs.

```bash
# Pretend we're a partner with no local checkout
TMP=$(mktemp -d)
cd "$TMP"

# Download the binary as a partner would (note the GitHub Releases URL)
curl -sSL -o bonovalley-platform.exe \
  "https://github.com/<GH_REPO>/releases/download/v1.0.0/bonovalley-platform-v1.0.0-windows-amd64.exe"

# Verify it self-identifies correctly
./bonovalley-platform.exe --version
# expected: bonovalley-platform-cli v1.0.0 (commit ..., built ...)

# Verify doctor passes from a fresh machine perspective
./bonovalley-platform.exe doctor
# expected: 6/6 pass (or 5/6 if not logged in — that's also fine, partner runs login next)

# Clean up
cd / && rm -rf "$TMP"
```

I report each line's result. If `doctor` shows anything unexpected (other than "not logged in"), I stop and we investigate before declaring the release done.

**Note:** the full partner journey (`login` → `init` → `register` → `push`) needs interactive user input (OAuth browser flow). That's outside this SOP's automation; it'd be done by a partner whenever they pick up the new binary.

### Step 12 — **(CLAUDE)** Append release journal entry · *~10s*

I append a new section to the **Release Journal** (see §14 below) with:

- Version tag
- Date (UTC) it shipped
- The commit list included
- Build-script output summary (artifact sizes)
- Smoke-test result (§7 + §11)
- Bitbucket Downloads URL count (5 files expected)
- Any deviations / surprises during the run

Then I commit the journal:

```bash
git add docs/DEPLOYMENT_SOP.md
git commit -m "Release journal: v1.0.0"
git push origin master
```

### Step 13 — **(CLAUDE)** Report completion · *~5s*

I say something like:

> **Release v1.0.0 shipped.** 5 files at https://bitbucket.org/bonovalley/bonovalley-platform-cli/downloads/, all smoke tests passed, journal entry committed. Total wall time: 18m 22s.

You acknowledge or flag any issues. SOP ends.

---

## 7. Rollback

If a release is found broken **after step 8 (push)**:

1. **Pull the binaries** from Bitbucket Downloads (UI: each row has a delete icon, or `DELETE /2.0/repositories/.../downloads/<filename>` via REST).
2. **Do NOT delete the git tag** — it stays for traceability. Partners who already downloaded have it; deleting the tag would just confuse the history.
3. **Cut the next patch** (`v1.0.x+1`) following the normal procedure. Make the CHANGELOG entry explicit: `"Reverts <bug> introduced in v1.0.x"`.
4. **Append a rollback note** to that version's journal entry so future-you knows why the prior version got pulled.

Tag deletion + re-push is forbidden because cached partner downloads cannot be invalidated.

---

## 8. Quick cheat sheets

### USER (you) cheat sheet

```text
1. Pick a version per the semver table (§4).
2. Tell Claude: "Deploy vX.Y.Z"
3. Confirm the commit list at the §6 step-2 checkpoint with "go"
4. (Optional) Provide a Bitbucket app password at §6 step-9a
5. Done — Claude reports completion at §6 step-13.
```

### CLAUDE cheat sheet (the full sequence as one block)

```powershell
# from repo root, master clean
$ver = "v1.0.0"

# §6 step-2: pre-flight
git status; git log --oneline -10
go test ./...; go build ./...; go run main.go doctor

# §6 step-3: changelog
notepad CHANGELOG.md   # move [Unreleased] into [$ver] — YYYY-MM-DD

# §6 steps 4-5: commit + tag
git commit -am "Release $ver"
git tag -a $ver -m "Release $ver"

# §6 step-6: build
.\scripts\build-release.ps1 -Version $ver

# §6 step-7: smoke
.\dist\$ver\bonovalley-platform-windows-amd64.exe --version
.\dist\$ver\bonovalley-platform-windows-amd64.exe doctor

# §6 step-8: push
git push origin master
git push origin $ver

# §6 step-9a: REST upload (loop over the 5 files)
foreach ($f in Get-ChildItem ".\dist\$ver") {
    curl -X POST -u "$env:USERNAME`:$env:APP_PASSWORD" `
         -F "files=@$($f.FullName)" `
         https://api.bitbucket.org/2.0/repositories/bonovalley/bonovalley-platform-cli/downloads
}

# §6 step-10: verify URL
curl -sSI https://bitbucket.org/bonovalley/bonovalley-platform-cli/downloads/bonovalley-platform-windows-amd64.exe | head -3

# §6 step-11: partner journey
$TMP = New-TemporaryFile | %{ Remove-Item $_; New-Item -Type Directory -Path $_ }
Push-Location $TMP
curl -sSL -o bonovalley-platform.exe https://bitbucket.org/bonovalley/bonovalley-platform-cli/downloads/bonovalley-platform-windows-amd64.exe
.\bonovalley-platform.exe --version
.\bonovalley-platform.exe doctor
Pop-Location
Remove-Item -Recurse -Force $TMP

# §6 step-12: journal
notepad docs\DEPLOYMENT_SOP.md  # append entry under §14
git commit -am "Release journal: $ver"
git push origin master
```

---

## 9. Partner install + use (for cross-reference)

After §10 succeeds, integration partners can use the new release. This section mirrors the README's partner-facing instructions so the round-trip story lives here too.

### Install per OS

| OS | Step |
|---|---|
| Windows | Download `bonovalley-platform-windows-amd64.exe` from Bitbucket Downloads, rename to `bonovalley-platform.exe`, place on PATH |
| macOS (Apple Silicon) | `curl -L -o bonovalley-platform https://bitbucket.org/.../bonovalley-platform-darwin-arm64 && chmod +x bonovalley-platform && sudo mv bonovalley-platform /usr/local/bin/` |
| macOS (Intel) | Same as Apple Silicon but `darwin-amd64` |
| Linux | Same pattern as macOS but `linux-amd64` |

### Verify integrity (optional, recommended)

```bash
curl -L -o SHA256SUMS https://bitbucket.org/.../downloads/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

### First-run sequence

```bash
bonovalley-platform doctor       # confirm environment is healthy
bonovalley-platform login        # browser opens, OAuth flow, deployKey saved
bonovalley-platform init <name>  # scaffold a new integration project
cd ~/bonovalley-integrations/<name>
bonovalley-platform register     # interactive: name, key, etc.
# (write integration code)
bonovalley-platform push         # build + upload + deploy
```

Full docs at <https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/docs/>.

---

## 10. When to update THIS doc

| Trigger | What to update |
|---|---|
| Add a new release target (e.g. windows-arm64) | Build matrix in `scripts/build-release.ps1` AND the §3 release-artifacts table |
| Move to automated releases via Bitbucket Pipelines | Replace the manual §6 steps 6-10 with a `git push origin vX.Y.Z` → Pipeline trigger; keep the manual fallback documented |
| Switch artifact host (e.g., to GitHub Releases) | URLs throughout; the build script stays the same |
| Change in versioning policy | §4 versioning section |
| Add a new partner OS | §9 install table |
| The SOP itself trips someone up | Add to a "Common surprises" section at the bottom (create it on first use) |

---

## 11. Partner announcement template

Copy + fill in per release. Paste into your normal partner-comms channel (Slack, email).

```text
Subject: bonovalley-platform-cli vX.Y.Z is out

Hi all,

bonovalley-platform-cli vX.Y.Z is now available.

What changed (top items from CHANGELOG.md → [vX.Y.Z] section):
  • <bullet 1 — most user-visible change>
  • <bullet 2>
  • <bullet 3>

Download:
  https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/tag/vX.Y.Z

How to upgrade (2 min):
  1. Download the file for your OS from the link above
  2. Replace your existing binary (same location you placed it last time)
  3. Verify:  bonovalley-platform --version   → should show vX.Y.Z

Full changelog:     https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/CHANGELOG.md
Install/upgrade:    https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/docs/INSTALL.md

Hit any issues? Reply with the output of `bonovalley-platform doctor`
and the latest log from `~/.bonovalley/logs/`.
```

### How to scope the message per release type

| Bump | What to call out |
|---|---|
| **Patch** (v1.0.X → v1.0.Y) | One-line "what changed" is enough. Skip the bullets section if a single line covers it. |
| **Minor** (v1.X.Y → v1.X+1.0) | 3-5 bullets focusing on new commands / new flags / observable behavior changes. |
| **Major** (vX.Y.Z → vX+1.0.0) | Explicit `Breaking changes` section under the bullets, with migration steps + a fair lead time before partners are expected to upgrade. |

Always link to the full changelog so anyone wanting more detail has it; never paste the whole changelog in the message.

---

## 12. Release journal

One entry per release, newest at the top. Each entry follows the template below. **Both** USER and CLAUDE can append; in practice CLAUDE writes the entry at step 12 of the deploy run, and USER may add notes later (e.g., "partner X reported they couldn't run on cmd.exe — issue tracked at …").

### Template

```markdown
### vX.Y.Z — YYYY-MM-DD UTC

- **Triggered by:** USER name / chat session
- **Commits included** (since previous tag):
  - `<shortsha>` short message
  - ...
- **Build:** 4 binaries, sizes XX-YY MB each, SHA256SUMS written
- **Smoke (§6 step-7):** --version OK, doctor 6/6 OK
- **Push:** master + vX.Y.Z pushed cleanly
- **Upload path:** REST API (or "UI by USER")
- **URL verify (§6 step-10):** HTTP 200, content-length matches build
- **Partner journey (§6 step-11):** download → --version → doctor all OK
- **Surprises / deviations:** none (or describe)
- **Rollback (if applicable):** N/A
```

### Entries

### v1.0.0 — 2026-05-24 UTC

- **Triggered by:** USER (Amit Arora) / chat session
- **Source tag:** `bitbucket.org/bonovalley/bonovalley-platform-cli` @ commit `c280c26`
- **Release page:** <https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/tag/v1.0.0>
- **Commits included** (20, since repo creation — first release):
  - `4daddee` re-submit from new device
  - `8abcff5` Update Go version to 1.22 in go.mod
  - `d2060fe` Add CLAUDE.md, plus initial dev_start command and docker-compose
  - `82295d8` Implement bootstrap, bvapi, and 7 commands (login/init refactor + 5 new)
  - `1776807` Fix bvtemplate ignore.go to handle directory-only patterns
  - `c18b5bd` Implement register command (interactive prompts + API + marker/registry writes)
  - `99eef8e` Implement link command (no-args + by-id forms, ownership-verified)
  - `4c9b624` Update SOP with Slice B Tier 2 verified state
  - `449ea38` Add SOP-12 (push workflow) and close $11 item #9
  - `bb807a9` Implement push command per SOP-12 (build + upload + cleanup)
  - `54b0122` push: route temp-dir cleanup through template's remove_temp_dir.go
  - `598b650` bvapi: 10-min upload timeout + env-var override for backend URL
  - `4cd5cff` release: --version flag, build script, deployment SOP, partner README, changelog
  - `ec14ad0` push: auto log files, error translation, final summary block
  - `6576013` ux: verbosity flags, color output, doctor command
  - `b7f1c46` ux: log-file version header + pre-flight wording fix
  - `14014c8` bvapi: translate 400-with-auth-keywords as a login-required error
  - `9faa8ef` ux: consistent auth-error UX across push/link/register + push fail-fast
  - `a739c10` docs: rewrite DEPLOYMENT_SOP for the partner-facing release loop
  - `f1a1b00` chore: pre-release housekeeping — vet fixes + carryover WIP
- **Build:** 4 binaries (8.0-8.4 MB each) + `SHA256SUMS-v1.0.0`. Embedded `Version=v1.0.0`, `Commit=ae32c5f` (post-tag commit — see Surprises #4), `BuildDate=2026-05-24`.
- **Smoke (§6 step-7):** `--version` reports `v1.0.0 (commit ae32c5f, built 2026-05-24)` OK; `doctor` 6/6 OK.
- **Push (§6 step-8):** `master` + tag `v1.0.0` pushed to `origin` cleanly.
- **Upload path:** GitHub Releases REST API via `scripts/upload-release.ps1`. All 5 assets uploaded `OK`.
- **URL verify (§6 step-10):** HTTP 302 → 200, content-length 8,705,024 bytes (matches Windows binary).
- **Partner journey (§6 step-11):** curl-download to a temp dir → `--version` → `doctor` — all OK on a clean filesystem.
- **Surprises / deviations** (first-deploy stumbles, all flagged so v1.0.1+ avoids them):
  1. Pre-flight failed initially on two pre-existing `vet` warnings (`oclient2.go:105` `fmt.Printf` with no format directives; `files_uploader.go:117` `log.Printf %w` on a string) that had been latent since `82295d8` / `4daddee`. Fixed in commit `f1a1b00` alongside 3 carryover dirty WIP files from prior sessions.
  2. First upload attempt to Bitbucket Downloads (the original plan) returned **HTTP 402** — Bitbucket Free plan no longer supports Repository Downloads (workspace-level block, both API and UI). Pivoted mid-deploy to a sibling GitHub repo, `bonovalley/bonovalley-platform-cli-releases`, for binary distribution. Source stays on Bitbucket.
  3. Bitbucket app passwords were deprecated mid-session (Atlassian announcement: removal on 2026-07-28). Upload script's auth was migrated `BB_APP_PW` → Atlassian API token → GitHub PAT during the session.
  4. Embedded commit SHA in the binary is `ae32c5f` while the tag `v1.0.0` points to `c280c26`. Cause: build-script versioning changes (commit `ae32c5f`) landed on master AFTER the tag was created. For v1.0.1+, the SOP should commit any final build-script changes BEFORE running `git tag`.
  5. Credentials file location debated: initial design `$HOME/.bonovalley-release-credentials.json`, then moved to in-project `<repo>/.env.bonovalley-release-credentials` per user preference. Script now resolves env vars → in-project dotenv → `$HOME` JSON.
- **Rollback:** N/A
- **Follow-ups for v1.0.1:**
  - Reorder: build-script tweaks → commit → tag → build → upload
  - Consider upgrading Bitbucket workspace to Standard if you want Downloads back as an option
  - Add `Renew by` date to docs/CREDENTIAL_TRACKING.md row 1 (GitHub PAT) once you know the expiry of the token you generated
  - Revoke the leaked BB_API_TOKEN at Atlassian (token value was shown to CLAUDE via system Read tool earlier in this session)

