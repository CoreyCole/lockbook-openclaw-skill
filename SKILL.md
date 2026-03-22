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

Lockbook = end-to-end encrypted file storage with real-time sync and built-in conflict resolution. The human shares their `.openclaw/` folder (write access) to the agent's lockbook account. With `lockbook fs` running as a systemd service, the entire `.openclaw/` directory is always available at `/tmp/lockbook/.openclaw/` and syncs bidirectionally with the human's devices.

## Install options

Official install paths from https://lockbook.net:

```bash
cargo install lockbook                                   # Cargo (compiles from source, ~6 min)
nix-shell -p lockbook                                    # Nix
yay -S lockbook                                         # AUR (Arch Linux)
brew tap lockbook/lockbook && brew install lockbook      # brew (macOS)
snap install lockbook                                    # Snap
# Binaries for macOS, Windows & Linux: https://github.com/lockbook/lockbook/releases
```

⚠️ **Prefer AUR/brew/snap/binaries over cargo** — cargo compiles from source and takes ~6 minutes.

The lockbook binary may land in `~/.cargo/bin/lockbook` (cargo) or `/usr/bin/lockbook` (AUR/snap). Use full path in exec commands if PATH isn't inherited.

> ⚠️ **Windows note:** `lockbook fs` requires NFS, only available on Windows Pro/Enterprise. Not available on Windows Home.

---

## How it works

Run `lockbook fs` as a persistent systemd service. It mounts your lockbook as an NFS filesystem at `/tmp/lockbook` and syncs to the server every 30 seconds. With `Restart=always`, systemd keeps it running.

```
/tmp/lockbook/.openclaw/    ← NFS mount, always available
        ↓↑ auto-sync every 30s + built-in 3-way merge
Human's phone / desktop
```

**Text files** get 3-way merged when both sides edit simultaneously. **Binary files** get duplicated with an incremented name on conflict. The agent and human can both edit files freely.

---

## First-time setup

> **At the end of setup, always ask:** "Do you want me to set up `lockbook fs` so your `.openclaw/` is always in sync? Most people want this." If yes, follow Step 5.

Read `references/agent-accounts.md` for the full walkthrough. Summary below.

### Step 1 — Check if your human has a lockbook account

**Ask first — do not create a human account.**

> "Do you already have a lockbook account? If not, you'll need to create one on your device — I can't do it for you because lockbook generates a 24-word recovery phrase that only you should ever see. Create in the native app on iOS, Android, Mac, Windows and Linux — download at https://lockbook.net — or via the CLI: `lockbook account new <username>`. Save that phrase somewhere safe — it's the only way to recover your account."

### Step 2 — Create the agent account

```bash
lockbook account new openclaw-<short-name>    # e.g. openclaw-ruby
```

Save the private key **outside** `~/.openclaw` so it doesn't sync:

```bash
echo "y" | lockbook account export > ~/.lockbook-agent-key
chmod 600 ~/.lockbook-agent-key
lockbook sync
```

Tell the human the agent username so they can share.

### Step 3 — Human creates and shares .openclaw

Human does this from their device (CLI or app):

```bash
lockbook new .openclaw/
lockbook share .openclaw/ openclaw-<agent-username> --mode=write
lockbook sync
```

### Step 4 — Agent accepts the share

```bash
lockbook sync
lockbook share pending                     # confirm share appears
lockbook share accept <pending-id> /       # target must be /
lockbook sync
lockbook list                              # .openclaw/ should appear
```

⚠️ `share accept` requires two args — the ID and a target path. Always pass `/`. Passing only the ID fails with "Missing required argument: target".

### Step 5 — Run lockbook fs as a systemd service

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/lockbook-fs.service << 'EOF'
[Unit]
Description=Lockbook NFS filesystem
After=network-online.target

[Service]
Type=simple
ExecStart=/bin/bash -c 'echo Y | /path/to/lockbook fs'
Restart=always
RestartSec=5
StandardOutput=append:%h/.openclaw-sync.log
StandardError=append:%h/.openclaw-sync.log

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now lockbook-fs.service
```

⚠️ Replace `/path/to/lockbook` with the actual binary path (`which lockbook`).

Verify:
```bash
ls /tmp/lockbook/.openclaw/    # should show your files
```

If the mount shows a stale file handle after a restart, clear it:
```bash
sudo umount -f /tmp/lockbook   # or umount -l if -f fails
sudo rm -rf /tmp/lockbook && sudo mkdir -p /tmp/lockbook && sudo chown $USER /tmp/lockbook
systemctl --user restart lockbook-fs.service
```

**macOS:** `lockbook fs` works the same — run it as a launchd service or keep it in a terminal.

---

## Symlink ~/.openclaw/workspace to the mount

Once `lockbook fs` is running and the shared folder is populated, symlink the workspace so the agent reads/writes directly through lockbook:

```bash
mv ~/.openclaw/workspace ~/.openclaw/workspace.local.bak
ln -s /tmp/lockbook/.openclaw/workspace ~/.openclaw/workspace
```

Now all agent reads/writes go directly to the lockbook NFS mount. Human edits on their phone appear immediately at `~/.openclaw/workspace/` on the agent's machine.

---

## Initial seeding (first-time only)

If you need to push existing disk files into a fresh lockbook account, use `lockbook copy` once:

```bash
lockbook new .openclaw/
lockbook copy ~/.openclaw/ .openclaw/
lockbook sync
```

⚠️ **`lockbook copy` is not idempotent** — running it twice creates duplicates. Seed once, then let `lockbook fs` handle everything.

---

## CLI quick reference

```bash
lockbook fs                               # Mount at /tmp/lockbook (primary)
lockbook sync                             # One-shot sync with server
lockbook list                             # List all files/folders
lockbook usage                            # Check storage usage
lockbook share new .openclaw/ <user> --mode=write
lockbook share pending
lockbook share accept <id> /              # Accept share (target must be /)
```

---

## Gotchas

- **`lockbook fs` is the correct sync solution** — bidirectional, 3-way merge for text, auto-managed by systemd
- **`lockbook fs` requires NFS** — Linux/macOS work out of the box; Windows requires Pro/Enterprise
- **Stale mount after restart** — `umount -f /tmp/lockbook`, delete, recreate, restart service
- **Only one `lockbook fs` process at a time** — kill any orphans (`ps aux | grep lockbook`) before restarting
- **`share accept` needs a target** — always pass `/` as second arg
- **Agent key must stay outside `~/.openclaw/`** — store at `~/.lockbook-agent-key`
- **Free tier is 25MB compressed** (~125MB text). Run `lockbook usage` to check
- **cargo install is slow** — ~6 min; prefer AUR/snap/brew/binaries
