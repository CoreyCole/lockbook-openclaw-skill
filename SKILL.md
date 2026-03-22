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

Lockbook = end-to-end encrypted file storage with real-time sync and built-in conflict resolution. The human shares their `.openclaw/` folder (write access) with the agent's lockbook account. Lockbook handles merge conflicts automatically for text files — both sides can edit simultaneously and sync will 3-way merge the result.

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

⚠️ **Prefer AUR/brew/snap/binaries over cargo** — cargo compiles from source and takes ~6 minutes. Use a prebuilt option when available.

The lockbook binary may land in `~/.cargo/bin/lockbook` (cargo) or `/usr/bin/lockbook` (AUR/snap). Use full path in exec commands if PATH isn't inherited.

---

## How sync works

`lockbook fs` mounts your entire lockbook tree as an NFS filesystem at `/tmp/lockbook`. Any file written there syncs to the lockbook server every 30 seconds. Lockbook handles merge conflicts natively:

- **Text files** (`.md`, `.txt`, `.json`, etc.) → 3-way merge, both edits preserved
- **Binary/other files** → duplicate created with incremented name on conflict

This means the agent and human can both edit the same files simultaneously. No custom conflict logic needed.

```
Agent edits /tmp/lockbook/.openclaw/workspace/MEMORY.md
                    ↓ lockbook fs syncs every 30s
Human edits MEMORY.md from phone
                    ↓ lockbook sync + 3-way merge
Both see merged result
```

---

## First-time setup

> **At the end of setup, always ask:** "Do you want me to set up `lockbook fs` so your workspace stays in sync automatically? Most people want this." If yes, follow Step 5.

Read `references/agent-accounts.md` for the full walkthrough. Summary below.

### Step 1 — Check if your human has a lockbook account

**Ask first — do not create a human account.**

Say something like:

> "Do you already have a lockbook account? If not, you'll need to create one on your device — I can't do it for you because lockbook generates a 24-word recovery phrase that only you should ever see. You can create in the native app on iOS, Android, Mac, Windows and Linux download at https://lockbook.net or via the CLI with `lockbook account new <username>`. Save that phrase somewhere safe — it's the only way to recover your account."

Wait for them to confirm they have an account before proceeding.

### Step 2 — Create the agent account

```bash
lockbook account new openclaw-<short-name>
# e.g. openclaw-ruby
```

Export and save the private key **outside** `~/.openclaw` — if it ends up inside the shared folder it will sync to the human:

```bash
echo "y" | lockbook account export > ~/.lockbook-agent-key
chmod 600 ~/.lockbook-agent-key
```

Sync to register with the server:

```bash
lockbook sync
```

Tell the human the agent username so they can share with it.

### Step 3 — Human creates and shares .openclaw

The human does this on their device (CLI or app):

```bash
lockbook new .openclaw/
lockbook share .openclaw/ openclaw-<agent-username> --mode=write
lockbook sync
```

Or via the desktop/mobile app: create `.openclaw` folder → right-click → Share → enter agent username → Write access.

### Step 4 — Agent accepts the share

```bash
lockbook sync
lockbook share pending       # confirm share is listed
```

Accept it — target must be `/` (root of agent's file tree):

```bash
lockbook share accept <pending-share-id> /
lockbook sync
lockbook list                # .openclaw/ should now appear
```

⚠️ **Known gotcha:** `lockbook share accept` requires two args: the share ID and a target path. Passing just the ID fails with "Missing required argument: target". Pass `/` to place it at root.

### Step 5 — Set up `lockbook fs` + sync watchers (strongly recommended)

`lockbook fs` is the correct sync solution. It mounts lockbook at `/tmp/lockbook` as an NFS filesystem — reads and writes go directly to lockbook, syncing every 30 seconds with built-in 3-way merge for text files.

> ⚠️ **Windows note:** `lockbook fs` requires NFS, which is only available on Windows Pro/Enterprise (not Windows Home). If your agent runs on Windows Home, `lockbook fs` will not work. On Linux and macOS it works out of the box.

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/lockbook-fs.service << 'EOF'
[Unit]
Description=Lockbook NFS filesystem
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c 'echo Y | /home/ruby/.cargo/bin/lockbook fs'
Restart=on-failure
RestartSec=10
StandardOutput=append:%h/.openclaw-sync.log
StandardError=append:%h/.openclaw-sync.log

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now lockbook-fs.service
```

⚠️ Update the lockbook binary path if it's not at `/home/ruby/.cargo/bin/lockbook` — check with `which lockbook`.

Then set up the sync watcher (requires `inotify-tools` — `yay -S inotify-tools`):

```bash
# Create the sync script
cat > ~/.local/bin/lockbook-sync.sh << 'SCRIPT'
#!/bin/bash
MOUNT=/tmp/lockbook/.openclaw
LOCAL=$HOME/.openclaw
LOG=$HOME/.openclaw-sync.log

if ! mountpoint -q /tmp/lockbook 2>/dev/null; then
  echo "$(date -u +%FT%TZ) lockbook not mounted, skipping" >> "$LOG"
  exit 0
fi

mkdir -p "$MOUNT"
rsync -a --delete \
  --exclude="*.sqlite*" --exclude="media/" \
  --exclude="agents/sessions/" --exclude="credentials/" \
  --exclude=".git/" \
  "$LOCAL/" "$MOUNT/" \
  && echo "$(date -u +%FT%TZ) sync ok" >> "$LOG" \
  || echo "$(date -u +%FT%TZ) sync failed" >> "$LOG"
SCRIPT
chmod +x ~/.local/bin/lockbook-sync.sh

# inotify watcher — syncs on every file change
cat > ~/.config/systemd/user/lockbook-watch.service << 'EOF'
[Unit]
Description=OpenClaw → Lockbook inotify watcher
After=lockbook-fs.service

[Service]
Type=simple
ExecStart=/bin/bash -c 'while inotifywait -r -e modify,create,delete,move \
  --exclude "(\.sqlite|media/|agents/sessions/|credentials/)" \
  $HOME/.openclaw/ 2>/dev/null; do \
  $HOME/.local/bin/lockbook-sync.sh; \
done'
Restart=on-failure
RestartSec=5
EOF

# Timer fallback — runs every 2 min regardless
cat > ~/.config/systemd/user/lockbook-sync.service << 'EOF'
[Unit]
Description=OpenClaw → Lockbook sync (timer fallback)
After=lockbook-fs.service

[Service]
Type=oneshot
ExecStart=/home/ruby/.local/bin/lockbook-sync.sh
EOF

cat > ~/.config/systemd/user/lockbook-sync.timer << 'EOF'
[Unit]
Description=OpenClaw → Lockbook sync timer

[Timer]
OnBootSec=60
OnUnitActiveSec=2min

[Install]
WantedBy=timers.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now lockbook-fs.service
systemctl --user enable --now lockbook-watch.service
systemctl --user enable --now lockbook-sync.timer
```

Verify:
```bash
ls /tmp/lockbook/.openclaw/     # should show workspace files
tail -f ~/.openclaw-sync.log    # watch sync activity
```

**macOS:** `lockbook fs` works the same. Use `fswatch` instead of `inotifywait` for the file watcher.

---

## Architecture: disk is primary, lockbook is the mirror

**`~/.openclaw/workspace/` is always the source of truth.** The agent reads and writes there directly. Lockbook is a best-effort mirror for sharing with the human — if the NFS mount goes down, the agent keeps running unaffected.

```
~/.openclaw/workspace/          ← primary (local disk, always available)
        ↓ rsync every 2 min (only if mounted)
/tmp/lockbook/.openclaw/workspace/  ← lockbook mirror (best-effort)
        ↓ lockbook fs syncs to server every 30s
Human's phone / desktop
```

Two systemd units keep the mirror current:
- **`lockbook-watch`** — uses `inotifywait` to trigger a sync immediately on any file change in `~/.openclaw/`
- **`lockbook-sync` timer** — fallback that runs every 2 minutes regardless

Both check mount health first and bail gracefully if lockbook is down. **The mount going down never affects the gateway or agent.**

### What gets synced

Everything except:
- `*.sqlite*` — databases (in use, can't rsync safely)
- `media/` — large inbound attachments
- `agents/sessions/` — huge session transcripts
- `credentials/` — sensitive keys

All of `workspace/`, `memory/`, `logs/`, `MEMORY.md`, `SOUL.md`, `HEARTBEAT.md` etc. sync in real time.

---

## Initial seeding (first-time only)

If you need to seed a fresh lockbook account from existing disk files, use `lockbook copy` once. After that, `lockbook fs` takes over.

```bash
# Create destination folder in lockbook first
lockbook new .openclaw/workspace/

# Copy once to seed
lockbook copy ~/.openclaw/workspace/ .openclaw/workspace/
lockbook sync
```

⚠️ **`lockbook copy` is not idempotent** — running it twice creates duplicates. Seed once, then switch to `lockbook fs`.

---

## CLI quick reference

```bash
lockbook fs                                            # Mount at /tmp/lockbook (primary sync method)
lockbook sync                                          # One-shot sync with server
lockbook list                                          # List all files/folders
lockbook list .openclaw/                               # List shared folder contents
lockbook export .openclaw/workspace/ ~/backup/         # One-time export to disk
lockbook copy /path/to/file.md .openclaw/              # Import a single file (one-shot only)
lockbook usage                                         # Check storage usage
lockbook share new .openclaw/ <username> --mode=write  # Share a folder
lockbook share pending                                 # List pending shares
lockbook share accept <id> /                           # Accept a share (target = /)
```

---

## Gotchas

- **`lockbook fs` is the correct sync solution** — it handles bidirectional sync and 3-way merge natively. `lockbook copy` is a one-shot import tool only.
- **`lockbook fs` requires NFS** — works on Linux and macOS out of the box. On Windows, requires Windows Pro/Enterprise (NFS not available on Windows Home)
- **`share accept` needs a target** — always pass `/` as the second arg or it errors with "Missing required argument: target"
- **Free tier is 25MB compressed** (~125MB of text). Run `lockbook usage` to check
- **Agent key must live outside `~/.openclaw`** — store at `~/.lockbook-agent-key` so it doesn't sync to the human
- **cargo install is slow** — ~6 minutes; prefer AUR/snap/brew
- **Binary path** — cargo installs to `~/.cargo/bin/lockbook`; AUR installs to `/usr/bin/lockbook`. Use full path in exec commands
- **`lockbook copy <dir>` wraps the directory** — always creates a new child inside dest, never syncs in place
