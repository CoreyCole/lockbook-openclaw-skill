---
name: lockbook
description: Work with lockbook encrypted file storage and sync the ~/.openclaw directory between agent and human. Use when reading/writing shared files, syncing with lockbook, or setting up agent accounts. Trigger on "lockbook", "sync openclaw folder", "share files with agent", or any file operation that should persist to the human's lockbook.
metadata:
  clawdbot:
    emoji: "🔒"
    requires:
      bins: ["lockbook"]
    install:
      - id: aur
        kind: aur
        package: lockbook
        bins: ["lockbook"]
        label: "Install lockbook CLI (AUR)"
      - id: cargo
        kind: cargo
        crate: lockbook
        bins: ["lockbook"]
        label: "Install lockbook CLI (cargo)"
      - id: brew
        kind: brew
        formula: lockbook/lockbook/lockbook
        bins: ["lockbook"]
        label: "Install lockbook CLI (brew)"
      - id: snap
        kind: snap
        snap: lockbook
        bins: ["lockbook"]
        label: "Install lockbook CLI (snap)"
---

# Lockbook

Lockbook = end-to-end encrypted file storage with real-time sync and built-in conflict resolution. The human shares a `.agents/` folder (write access) to the agent's lockbook account. The agent uses `lockbook sync-dir` to bidirectionally sync `~/.openclaw` into `.agents/<agent-name>/`.

## How it works

```
~/.openclaw/          ← agent's local files
    ↕ lockbook sync-dir (bidirectional, content-hash based)
.agents/<name>/       ← shared lockbook folder (human can browse on any device)
    ↕ lockbook server (E2E encrypted)
Human's phone / desktop
```

`sync-dir` runs a reconciliation cycle: scan local files → diff against last-agreed state → push changes to lockbook → sync with server → pull remote changes to disk. It detects conflicts via SHA-256 content hashing and saves local versions as `.conflict-<timestamp>` sidecars when both sides change the same file.

### Modes

- **`--once`** — single reconciliation cycle, then exit
- **Long-running** (default) — watches filesystem + polls remote on an interval
- **`--no-watch`** — disable filesystem watcher, poll only
- **`--pull-interval 30s`** — remote poll interval (default 5s)

## Install options

Official install paths from https://lockbook.net:

```bash
yay -S lockbook                                         # AUR (Arch Linux, preferred)
brew tap lockbook/lockbook && brew install lockbook      # brew (macOS)
snap install lockbook                                    # Snap
cargo install lockbook                                   # Cargo (compiles from source, ~6 min)
nix-shell -p lockbook                                    # Nix
# Binaries for macOS, Windows & Linux: https://github.com/lockbook/lockbook/releases
```

## First-time setup

Read `references/agent-accounts.md` for the full walkthrough. Summary:

1. **Human creates account** (if needed) — `lockbook account new <username>` on their device
2. **Agent creates account** — `lockbook account new openclaw-<name>`
3. **Human shares `.agents/`** — `lockbook share new .agents/ openclaw-<name> --mode=write`
4. **Agent accepts share** — `lockbook share accept <id> /`
5. **Test the share** — upload a `hello.txt` test file, ask human to verify they see it
6. **Full sync** — `lockbook sync-dir .agents/<name> ~/.openclaw --once` (warn human: takes a few minutes)
7. **Set up systemd service** for persistent sync (see agent-accounts.md)

## CLI quick reference

```bash
lockbook sync-dir <lb-folder> <local-dir> --once     # one-shot sync
lockbook sync-dir <lb-folder> <local-dir>             # long-running sync
lockbook sync                                          # sync metadata with server
lockbook list                                          # list all files/folders
lockbook list <folder>                                 # list folder contents
lockbook stream out <file>                             # print file to stdout
lockbook account status                                # check storage usage
lockbook share new <file> <user> --mode=write          # share with another user
lockbook share pending                                 # list pending shares
lockbook share accept <id> /                           # accept a share into root
```

## Gotchas

- **sync-dir is the primary sync solution** — `lockbook fs` (NFS mount) exists but is unstable
- **Agent syncs into `.agents/<name>/`** — not directly into the human's `.openclaw/`, to avoid overwriting
- **`share accept` needs two args** — the pending ID and a target path (always `/`)
- **Agent key must stay outside `~/.openclaw/`** — store at `~/.lockbook-agent-key`
- **Free tier is 25MB compressed** (~125MB text). Run `lockbook account status` to check
- **Initial sync is slow** — files are written sequentially; subsequent syncs are fast
- **sync-dir only deletes previously-tracked files** — won't delete local files on first run against a new remote folder
- **`.lockbookignore`** — auto-generated on first sync, supports gitignore syntax; `.git/`, `node_modules/`, `target/`, etc. are ignored by default
