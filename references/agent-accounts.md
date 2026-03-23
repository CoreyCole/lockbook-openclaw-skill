# Agent Account Setup

Guide for setting up lockbook accounts and sharing between human and agent. Only needed once.

## Prerequisites

- Lockbook CLI installed on the agent's machine (see install options below)
- Human needs a lockbook account (or will create one during setup)

## 1. Human account

If the human doesn't have one yet, they create it on any device — CLI, iOS, Android, Mac, Windows, or Linux. Download at https://lockbook.net.

This returns a 24-word recovery phrase. There are no passwords, no email, no reset flow. That phrase is the only way to recover the account. Make sure they save it.

**Do not create the human's account for them** — the recovery phrase should only be seen by the human.

## 2. Agent account

On the agent's machine:

```bash
lockbook account new openclaw-<short-identifier>   # e.g. openclaw-ruby
```

Export and save the private key **outside** `~/.openclaw` (so it doesn't get synced):

```bash
echo "y" | lockbook account export > ~/.lockbook-agent-key
chmod 600 ~/.lockbook-agent-key
lockbook sync
```

Tell the human the agent's username so they can share.

## 3. Human creates shared directory and shares it

The human does this on any lockbook client (CLI, desktop, or mobile):

```bash
lockbook new .agents/                                        # create the shared folder
lockbook share new .agents/ openclaw-<agent-username> --mode=write
lockbook sync
```

Or via the app: right-click `.agents/` → Share → enter agent username → Write access.

The agent syncs into `.agents/<agent-name>/` (e.g. `.agents/ruby/`).

## 4. Agent accepts and syncs

On the agent's machine:

```bash
lockbook sync
lockbook share pending                     # confirm share appears
lockbook share accept <pending-id> /       # target must be /
lockbook sync
lockbook list                              # .agents/ should appear
```

⚠️ `share accept` requires two args — the pending share ID and a target path. Always pass `/`.

Verify the share works by uploading a quick test file:

```bash
lockbook new .agents/<agent-name>/hello.txt
echo "hello from openclaw-<agent-name>" | lockbook stream in .agents/<agent-name>/hello.txt
lockbook sync
```

Ask the human to check their lockbook — they should see `.agents/<name>/hello.txt` with the greeting. This confirms the share is working. Once confirmed, proceed to the full sync.

⚠️ **The full sync takes a few minutes.** Tell the human: "The share is working — you should see my hello.txt. Now I'm going to sync my full config directory (~2000 files). This will take a few minutes because lockbook writes files sequentially. I'll let you know when it's done."

```bash
lockbook sync-dir .agents/<agent-name> ~/.openclaw --once
```

The test file will remain in lockbook (sync-dir won't delete it since it wasn't previously tracked locally). Clean it up after the full sync completes:

```bash
echo "y" | lockbook delete .agents/<agent-name>/hello.txt
```

## 5. Set up persistent sync (systemd)

For always-on agents, run `sync-dir` as a systemd service:

```bash
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/lockbook-sync-dir.service << 'EOF'
[Unit]
Description=Lockbook sync-dir (.openclaw)
After=network-online.target

[Service]
Type=simple
ExecStart=/path/to/lockbook sync-dir .agents/<agent-name> %h/.openclaw --pull-interval 30s
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now lockbook-sync-dir.service
```

⚠️ Replace `/path/to/lockbook` with the actual binary path (`which lockbook`) and `<agent-name>` with the agent's short name.

Verify:
```bash
systemctl --user status lockbook-sync-dir.service
```

## Troubleshooting

**Shared folder not appearing** — both accounts must sync. Human runs `lockbook sync`, then agent runs `lockbook sync`. May take a moment to propagate.

**`share accept` missing target error** — always pass two args: `lockbook share accept <id> /`

**PathConflict on accept** — the agent already has a folder with the same name. Delete it first: `echo "y" | lockbook delete <conflicting-folder>`, then accept.

**Storage full** — free tier is 25MB compressed (~125MB text). Check with `lockbook account status`. Upgrade to 30GB for $2.99/month if needed.

**Sync fails with network error** — lockbook server may be temporarily unreachable. Local files remain intact. Next sync catches up.

**sync-dir deletes local files** — only files previously tracked in `.sync-dir-state` can be deleted. On a first run against a new remote folder, nothing gets deleted. But be careful syncing against the wrong remote path.

## Notes on encryption

Lockbook uses elliptic curve cryptography. When the human shares with the agent, lockbook re-encrypts the folder key for the agent's public key. The server never sees plaintext. This is real end-to-end encrypted sharing, not just access control.

## Install lockbook CLI

```bash
yay -S lockbook                                         # AUR (Arch Linux, preferred)
brew tap lockbook/lockbook && brew install lockbook      # macOS
snap install lockbook                                    # Linux (snap)
cargo install lockbook                                   # Cargo (compiles from source, ~6 min)
nix-shell -p lockbook                                    # Nix
```

Windows/Linux binaries: https://github.com/lockbook/lockbook/releases

⚠️ **Prefer AUR/brew/snap/binaries over cargo** — cargo compiles from source and takes ~6 minutes.
