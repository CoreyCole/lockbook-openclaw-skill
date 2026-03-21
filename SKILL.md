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

Lockbook = end-to-end encrypted file storage with sharing. The human owns the `.openclaw/` folder in their lockbook and shares it (write access) to the agent's lockbook account. The server never sees plaintext — real E2E, not just access control.

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

## First-time setup

Read `references/agent-accounts.md` for the full walkthrough. Summary below.

### Step 1 — Check if your human has a lockbook account

**Ask first — do not create a human account.**

Say something like:

> "Do you already have a lockbook account? If not, you'll need to create one on your device — I can't do it for you because lockbook generates a 24-word recovery phrase that only you should ever see. You can create one at https://lockbook.net or via the CLI with `lockbook account new <username>`. Save that phrase somewhere safe — it's the only way to recover your account."

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

⚠️ **Known gotcha:** After accepting, `lockbook list .openclaw/` may show empty even if the share was successful — the folder is legitimately empty until the human puts files in it or the agent imports workspace files.

---

## Sync

All sync uses import/export. No NFS mount required — works everywhere.

```bash
# Pull lockbook → local disk
lockbook sync && lockbook export .openclaw/ ~/.openclaw/

# Push local disk → lockbook
lockbook import ~/.openclaw/ .openclaw/ && lockbook sync
```

**Before reading shared files** — always pull first.
**After writing files** — always push then sync.

---

## CLI quick reference

```bash
lockbook sync                                          # Sync with server
lockbook list                                          # List all files/folders
lockbook list .openclaw/                               # List contents of shared folder
lockbook export .openclaw/ ~/.openclaw/                # Pull lockbook → local
lockbook import ~/.openclaw/ .openclaw/                # Push local → lockbook
lockbook export .openclaw/somefile.md /tmp/somefile.md # Export single file
lockbook import /tmp/report.md .openclaw/reports/      # Import single file
lockbook usage                                         # Check storage usage
lockbook share new .openclaw/ <username> --mode=write  # Share a folder
lockbook share pending                                 # List pending shares
lockbook share accept <id> /                           # Accept a share (target = /)
```

---

## Gotchas

- **`share accept` needs a target** — always pass `/` as the second arg or it errors with "Missing required argument: target"
- **`lockbook export` overwrites local files** — pull before push if you have local-only changes
- **Last-write-wins** — no merge. Fine for config/logs where agent and human don't edit the same file simultaneously
- **Free tier is 25MB compressed** (~125MB of text). Run `lockbook usage` to check
- **Agent key must live outside `~/.openclaw`** — store at `~/.lockbook-agent-key` so it doesn't sync to the human
- **cargo install is slow** — ~6 minutes on a typical machine; prefer AUR/snap/brew
- **Binary path** — cargo installs to `~/.cargo/bin/lockbook`; AUR installs to `/usr/bin/lockbook`. Use full path in exec commands
