<!-- SYNCED TO GitHub via scripts/upload-release.ps1. Source-of-truth is this Bitbucket repo. Edit here, then run: pwsh ./scripts/upload-release.ps1 -SyncDocsOnly -->

# Installing bonovalley-platform-cli

Partner-facing install + upgrade + uninstall + troubleshooting guide.

Releases live at: <https://github.com/bonovalley/bonovalley-platform-cli-releases/releases>

---

## 1. First-time install

### Windows

1. Download `bonovalley-platform-vX.Y.Z-windows-amd64.exe` from the [latest release](https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest).
2. Rename it to `bonovalley-platform.exe` (optional, but the rest of the docs assume this).
3. Place it on your `PATH`. Quickest setup:
   ```powershell
   # In an admin PowerShell:
   New-Item -ItemType Directory -Force -Path "$HOME\bin"
   [Environment]::SetEnvironmentVariable(
       "PATH",
       "$([Environment]::GetEnvironmentVariable('PATH','User'));$HOME\bin",
       "User")
   # Open a NEW terminal for the PATH update to take effect, then:
   Move-Item bonovalley-platform.exe "$HOME\bin\"
   ```
4. In a new terminal: `bonovalley-platform --version` should print the version.

If Windows SmartScreen blocks the download or first run, see [§6 Troubleshooting](#6-troubleshooting).

### macOS (Apple Silicon — M1/M2/M3/M4)

```bash
curl -L -o bonovalley-platform \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest/download/bonovalley-platform-darwin-arm64"
chmod +x bonovalley-platform
sudo mv bonovalley-platform /usr/local/bin/
bonovalley-platform --version
```

If macOS shows "unidentified developer" / "cannot be opened", see [§6 Troubleshooting](#6-troubleshooting).

### macOS (Intel)

Same as Apple Silicon — replace `darwin-arm64` with `darwin-amd64` in the URL.

### Linux

```bash
curl -L -o bonovalley-platform \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest/download/bonovalley-platform-linux-amd64"
chmod +x bonovalley-platform
sudo mv bonovalley-platform /usr/local/bin/
bonovalley-platform --version
```

---

## 2. Verifying the install

```bash
bonovalley-platform --version
# Expected: bonovalley-platform-cli vX.Y.Z (commit ..., built YYYY-MM-DD)

bonovalley-platform doctor
# Expected on fresh install: 5/6 pass (the "Logged in" check fails until you run `login`)
# Expected after login:      6/6 pass
```

---

## 3. First-time setup

After install, follow the quickstart in [README.md](../README.md#quickstart) — `login` → `init` → `register` → `push`.

---

## 4. Upgrading to a new version

When you receive a release announcement (or want to check for one yourself), upgrading is a one-step replacement of the binary.

### Steps

1. **Check your current version**:
   ```bash
   bonovalley-platform --version
   ```
2. **Download the new binary** for your OS from the release page in the announcement (or browse <https://github.com/bonovalley/bonovalley-platform-cli-releases/releases>).
3. **Replace your existing binary**:
   - **Windows**: drag the new `.exe` over the old one (close any terminal that's running it first). If you renamed it to `bonovalley-platform.exe` per [§1](#windows), keep the same name.
   - **macOS / Linux**:
     ```bash
     chmod +x <newly-downloaded-file>
     sudo mv <newly-downloaded-file> /usr/local/bin/bonovalley-platform
     ```
4. **Verify**:
   ```bash
   bonovalley-platform --version
   # Should now show the new version
   bonovalley-platform doctor
   # Should still be 6/6 pass
   ```

### Your config + integrations are preserved

Upgrading swaps only the binary. Your `~/.bonovalleyrc` (login state), `~/.bonovalley/` (config, registry, logs), and any per-project `<project>/.bonovalley/integration.json` markers are untouched. You don't need to re-login or re-link anything.

### Optional: verify integrity

Each release ships a `SHA256SUMS-vX.Y.Z` manifest. To verify the file you downloaded matches the file the maintainers built:

```bash
curl -L -o SHA256SUMS-vX.Y.Z \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/download/vX.Y.Z/SHA256SUMS-vX.Y.Z"
sha256sum -c SHA256SUMS-vX.Y.Z --ignore-missing
# Expected: <your-binary>: OK
```

If the line is missing or shows `FAILED`, re-download — the file may have been truncated in transit.

---

## 5. Uninstalling

### Light uninstall (keeps your login + integration state)

Just remove the binary. Your `~/.bonovalley/` state stays put so a future re-install picks up where you left off.

- **Windows**: delete `bonovalley-platform.exe` from wherever you placed it (e.g. `$HOME\bin\`)
- **macOS / Linux**: `sudo rm /usr/local/bin/bonovalley-platform`

### Full uninstall (also removes all state)

After removing the binary, also delete:

- **Windows PowerShell**:
  ```powershell
  Remove-Item -Recurse -Force "$HOME\.bonovalley", "$HOME\.bonovalleyrc"
  ```
- **macOS / Linux**:
  ```bash
  rm -rf ~/.bonovalley ~/.bonovalleyrc
  ```

Per-project markers (`<project>/.bonovalley/integration.json`) live inside each integration's source tree — they don't get removed here. Either keep them (so a future `link` finds the project automatically) or delete them with the rest of the project.

---

## 6. Troubleshooting

### "command not found" / "is not recognized"

The binary downloaded fine but isn't on your `PATH`. Print your PATH and check:

```bash
# Linux / macOS
echo $PATH
# Windows PowerShell
$env:PATH -split ';'
```

If the folder you placed the binary in isn't listed, either move the binary to a listed folder or add its folder to PATH (see [§1 Windows step 3](#windows)).

### macOS: "cannot be opened — unidentified developer" (Gatekeeper)

The CLI isn't signed by an Apple Developer account yet. To allow it:

1. Try to run it once — you'll see the block dialog
2. Open **System Settings → Privacy & Security**
3. Scroll to the "Security" section — "`bonovalley-platform` was blocked..." will be shown
4. Click **Allow Anyway**
5. Run the command again; click **Open** at the second prompt

Or via the terminal (clears the quarantine attribute, no UI):
```bash
sudo xattr -dr com.apple.quarantine /usr/local/bin/bonovalley-platform
```

### Windows: SmartScreen blocks the download or first run

1. On the download confirmation, click **More info** → **Keep**
2. On first run, click **More info** → **Run anyway**

### Garbled escape codes in terminal output (`[32m✔[0m`, `[1m...[0m`)

Your terminal doesn't render ANSI colour. Either:
- Use a modern terminal (Windows Terminal, PowerShell 7, iTerm2, etc.), or
- Add `--no-color` to every command, or set `NO_COLOR=1` in your environment

### "Your session has expired" (HTTP 401 from server)

The deployKey from `login` has aged out (1-hour Ory access-token TTL wrapped inside it). Run:
```bash
bonovalley-platform login
```
and retry the failing command.

### Corporate proxy / TLS errors

The CLI uses the OS's default TLS trust store. If your network injects custom CA certificates:
- **Windows / macOS**: usually picked up automatically when the CA is installed in the system store
- **Linux**: set `SSL_CERT_FILE=/path/to/your/ca-bundle.pem`

### Anything else

Run `bonovalley-platform doctor` and include the output when you reach out for help. Also attach the latest `push` log from `~/.bonovalley/logs/` (timestamped per run; safe to share — no secrets).

---

## 7. Where your config + logs live

| Path | Purpose | Safe to delete? |
|---|---|---|
| `~/.bonovalleyrc` | Encrypted deployKey from `login` | Yes — re-run `login` to recreate |
| `~/.bonovalley/config.json` | CLI preferences (default org, etc.) | Yes — recreated on next CLI run with defaults |
| `~/.bonovalley/integrations.json` | List of integrations you've linked on this machine | Yes — re-link each via `link <id>` afterwards |
| `~/.bonovalley/logs/*.log` | Per-command logs (timestamped) | Yes — auto-pruned (30 days OR 100 files) |
| `<project>/.bonovalley/integration.json` | Per-integration marker | **No** — deleting breaks the platform link; use `unlink` instead |

Windows: `~` = `C:\Users\<you>` (`$HOME` in PowerShell).

---

## 8. Getting help

Include all three of these in any bug report — they give the maintainers everything they need to reproduce:

1. `bonovalley-platform --version` output
2. `bonovalley-platform doctor` output
3. The latest log file from `~/.bonovalley/logs/` (the `push-YYYYMMDD-HHMMSS.log` for whichever command failed)

Per-command help: `bonovalley-platform <command> --help`.

See also: [`CHANGELOG.md`](../CHANGELOG.md) for per-release notes, and [`docs/USER_GUIDE.md`](USER_GUIDE.md) for the end-to-end usage walkthrough.
