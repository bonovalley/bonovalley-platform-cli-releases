# Credential Tracking

Live inventory of every secret/token the `bonovalley-platform-cli` release pipeline needs. Goal: **never surprised by an expired credential**.

---

## Active credentials

| # | Name | Used by | Lives in | Owner | Renew by | Expires | Procedure |
|---|---|---|---|---|---|---|---|
| 1 | `bv-platform-cli-build-uploads-gh` | `scripts/upload-release.ps1` — GitHub Releases API uploads | `<repo>/.env.bonovalley-release-credentials` → key `GH_TOKEN` | amit@bonovalley.com | **2027-04-24** | 2027-05-24 | [§3.1](#31-bv-platform-cli-build-uploads-gh) |
| 2 | `bv-platform-cli-build-uploads` | (currently unused) — Atlassian API token, kept for any future Bitbucket REST API needs | `<repo>/.env.bonovalley-release-credentials` → key `BB_API_TOKEN` | amit@bonovalley.com | **2027-04-24** | 2027-05-24 | [§3.2](#32-bv-platform-cli-build-uploads) |

Reading the table:
- **Renew by** = `Expires − 30 days`. Hit this date → rotate immediately.
- **Lives in** = exact path / env-var / secret-manager key the script reads.
- **Owner** = the Atlassian-account email tied to the token. If the owner leaves the org, all their tokens must be rotated by the new owner.

---

## 1. Review cadence

| When | What |
|---|---|
| 1st of every quarter (Jan/Apr/Jul/Oct) | Open this doc. Anything with **Renew by** within 60 days → schedule a rotation this month. |
| Any time **Renew by** drops below 30 days | Stop and rotate today. No exceptions. |
| When a maintainer joins/leaves | Audit ownership of every row. |

Setting a calendar reminder for each `Renew by` date when you add a credential is recommended.

---

## 2. Adding a new credential

When the pipeline starts needing a new secret:

1. Add a row to the **Active credentials** table above. Fill all 8 columns.
2. `Renew by` = `Expires − 30 days`. Set a calendar reminder for that date.
3. Add a `§3.x` section below describing the renewal procedure (specific to that credential's source — what UI page, what scopes, etc.).
4. If the credential is read from disk: add the file/path to `.gitignore` and update the script to read from the agreed location.
5. Commit with message `creds: add <name> credential (used by <X>)`.

---

## 3. Renewal procedures

One section per credential. Steps assume you're on the maintainer machine that holds the live token.

### 3.1 `bv-platform-cli-build-uploads-gh`

GitHub Personal Access Token used by `scripts/upload-release.ps1` to attach binary assets to GitHub Releases.

1. Open <https://github.com/settings/tokens>
2. Either path works:
   - **Classic PAT**: click **Generate new token (classic)**; scope = `repo` (full)
   - **Fine-grained PAT** (preferred — narrower blast radius): click **Generate new token**, set **Resource owner** to the releases-repo owner, set **Repository access** to just `<GH_REPO>`, grant **Repository permissions → Contents: Read and write**
3. **Note**: `bv-platform-cli-build-uploads-gh` (so you can identify it later)
4. **Expiration**: 12 months from today
5. Copy the new token *(shown only once)*
6. Open `<repo>/.env.bonovalley-release-credentials`; replace the `GH_TOKEN` value with the new token; save
7. Test by re-running an upload against any existing release:
   ```powershell
   pwsh ./scripts/upload-release.ps1 -Version <existing-version-tag>
   ```
   Existing assets show `SKIPPED`; if the token is bad you'll see a 401/403. A run with no failed assets confirms the new token works.
8. Once confirmed working: **revoke the old token** at the GitHub URL above
9. Update this doc's row 1: `Renew by` → new expiry − 30 days; `Expires` → new expiry
10. Commit:
    ```bash
    git commit -am "creds: rotated GH_TOKEN (release uploads; now expires YYYY-MM-DD)"
    ```

### 3.2 `bv-platform-cli-build-uploads`

Atlassian API token for Bitbucket. Currently unused (release uploads moved to GitHub; see §3.1). Kept on file in case future operations need Bitbucket REST access (PR comments, tags, etc.). Renewal procedure if/when it's needed again:

1. Open <https://id.atlassian.com/manage-profile/security/api-tokens>
2. Click **Create API token with scopes**
3. **App**: `Bitbucket`
4. **Scopes** (both required): `read:repository:bitbucket` AND `write:repository:bitbucket`
5. **Name**: `bv-platform-cli-build-uploads`
6. **Expiry**: 12 months from today
7. Copy the new token *(shown only once)*
8. Open `<repo>/.env.bonovalley-release-credentials`; replace the `BB_API_TOKEN` value with the new token; save
9. Once confirmed working: revoke the old token; update this doc's row 2 dates; commit `creds: rotated bv-platform-cli-build-uploads (now expires YYYY-MM-DD)`

---

## 4. Initial setup (one-time per maintainer)

The credentials file is **per-maintainer** — never committed, never shared via Slack/email. Two accepted locations:

### 4a. In-project dotenv (primary, recommended)

Lives at `<repo>/.env.bonovalley-release-credentials`. Matches the `.env.development` / `.env.production` convention used in the API repo. The repo ships a **committed template** at `<repo>/.env.bonovalley-release-credentials.example` — copy it, fill in your real values, and you're done. **Gitignored** — git refuses to add the real file. **pCloud-sync note**: if your repo is in a pCloud-synced folder, the file syncs to every pCloud-connected device — that's the desired behaviour if you want credentials available everywhere, a risk to weigh if any device isn't fully trusted.

1. Generate a token per [§3.1](#31-bv-platform-cli-build-uploads).
2. Copy the template to the live filename and edit it:
   ```powershell
   Copy-Item .env.bonovalley-release-credentials.example `
             .env.bonovalley-release-credentials
   # Then open the new file in your editor and replace the two placeholder
   # values (BB_EMAIL + BB_API_TOKEN).
   ```
3. Restrict permissions:
   - **Windows PowerShell**: `icacls .\.env.bonovalley-release-credentials /inheritance:r /grant:r "$($env:USERNAME):F"`
   - **macOS / Linux**: `chmod 600 .env.bonovalley-release-credentials`
4. Sanity check gitignore is doing its job:
   ```bash
   git check-ignore -v .env.bonovalley-release-credentials
   # expected: .gitignore:N:.env.bonovalley-release-credentials   .env.bonovalley-release-credentials
   git check-ignore -v .env.bonovalley-release-credentials.example
   # expected: (no output, exit 1) — the .example IS committed
   ```

### 4b. Per-user JSON in $HOME (fallback)

Lives at `~/.bonovalley-release-credentials.json` (JSON shape). Cross-project (any repo on this machine that uses it can read), but doesn't follow you across machines via pCloud sync.

1. Generate a token per [§3.1](#31-bv-platform-cli-build-uploads).
2. Create the file with content:
   ```json
   {
     "BB_EMAIL": "you@yourdomain.com",
     "BB_API_TOKEN": "<paste API token>"
   }
   ```
3. Permissions: `chmod 600` on Unix, restrict ACLs on Windows.

### Resolution order

`scripts/upload-release.ps1` looks up creds in this order; first hit wins per key:

1. Environment variables (`$env:BB_EMAIL`, `$env:BB_API_TOKEN`)
2. `<repo>/.env.bonovalley-release-credentials` (4a above)
3. `$HOME/.bonovalley-release-credentials.json` (4b above)

---

## 5. What NOT to do

- Don't paste tokens into chat, commit messages, issue trackers, or screenshots
- Don't email tokens or send via Slack — share them via a password manager
- Don't reuse a single token across unrelated tools (one purpose = one token)
- Don't extend `Renew by` without rotating the actual credential
- Don't delete this doc — even an empty entries table is useful (proves nothing's needed yet)
