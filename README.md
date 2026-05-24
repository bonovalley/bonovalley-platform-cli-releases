# bonovalley-platform-cli — binary releases

This repository hosts **prebuilt CLI binaries only**. The source code lives on Bitbucket:
<https://bitbucket.org/bonovalley/bonovalley-platform-cli>

The CLI lets Bonovalley integration partners build, register, and `push` integrations to the platform from the command line. Pre-built binaries are published here so partners can download and use the CLI without needing Go installed.

---

## Install

Pick the file that matches your OS from the [latest release](https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest):

| OS | File |
|---|---|
| Windows (64-bit) | `bonovalley-platform-vX.Y.Z-windows-amd64.exe` |
| macOS (Intel) | `bonovalley-platform-vX.Y.Z-darwin-amd64` |
| macOS (Apple Silicon, M1/M2/M3/M4) | `bonovalley-platform-vX.Y.Z-darwin-arm64` |
| Linux (64-bit) | `bonovalley-platform-vX.Y.Z-linux-amd64` |

### Windows

1. Download the `-windows-amd64.exe` file.
2. Rename to `bonovalley-platform.exe`.
3. Place it on your PATH (e.g. `C:\Users\<you>\bin\`).
4. New terminal → `bonovalley-platform --version`.

### macOS (Apple Silicon)

```bash
curl -L -o bonovalley-platform \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest/download/bonovalley-platform-darwin-arm64"
chmod +x bonovalley-platform
sudo mv bonovalley-platform /usr/local/bin/
bonovalley-platform --version
```

(Intel Macs: replace `darwin-arm64` with `darwin-amd64`.)

### Linux

```bash
curl -L -o bonovalley-platform \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest/download/bonovalley-platform-linux-amd64"
chmod +x bonovalley-platform
sudo mv bonovalley-platform /usr/local/bin/
bonovalley-platform --version
```

### Verify

```bash
bonovalley-platform --version
# Expected: bonovalley-platform-cli vX.Y.Z (commit ..., built ...)

bonovalley-platform doctor
# Expected on fresh install: 5/6 pass (the "Logged in" check fails until you run `login`)
```

---

## Upgrade to a new version

When a new release is announced:

1. Download the new binary for your OS from the [latest release](https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/latest).
2. Replace your existing binary in the same location.
3. Verify: `bonovalley-platform --version` should now show the new version.

Your login state, config, and integration registry are preserved across upgrades — they live outside the binary, in `~/.bonovalleyrc` and `~/.bonovalley/`.

---

## Verify integrity (optional)

Each release ships a `SHA256SUMS-vX.Y.Z` manifest:

```bash
curl -L -o SHA256SUMS-vX.Y.Z \
  "https://github.com/bonovalley/bonovalley-platform-cli-releases/releases/download/vX.Y.Z/SHA256SUMS-vX.Y.Z"
sha256sum -c SHA256SUMS-vX.Y.Z --ignore-missing
```

---

## First-time usage

```bash
bonovalley-platform login                   # OAuth via browser
bonovalley-platform init my-integration     # scaffold a new project
cd ~/bonovalley-integrations/my-integration
bonovalley-platform register                # register with the platform (interactive)
# ... write your integration code ...
bonovalley-platform push                    # build + ship to the platform
```

Run `bonovalley-platform <command> --help` for any command's options, or `bonovalley-platform doctor` to check your environment.

---

## Full docs (in the source repo)

- **[User guide](https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/docs/USER_GUIDE.md)** — end-to-end usage walkthrough (setup, configuration, every command by task)
- [Install / upgrade / uninstall / troubleshooting](https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/docs/INSTALL.md) — comprehensive partner install reference
- [Quickstart + overview](https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/README.md)
- [Changelog](https://bitbucket.org/bonovalley/bonovalley-platform-cli/src/master/CHANGELOG.md) — per-release notes

---

## Reporting bugs

Include all three of these when you reach out:

1. `bonovalley-platform --version`
2. `bonovalley-platform doctor`
3. The relevant log from `~/.bonovalley/logs/` (the `push-YYYYMMDD-HHMMSS.log` for whichever command failed)

Contact your Bonovalley point of contact.

---

*This README is auto-published from the source repo by `scripts/upload-release.ps1`. Edits made directly on GitHub will be overwritten on the next release. To change it, edit `docs/GITHUB_RELEASES_README.md` in the Bitbucket source repo.*
