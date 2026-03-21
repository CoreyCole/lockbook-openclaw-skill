# Agent Account Setup

Guide for setting up lockbook accounts and sharing between human and agent. Only needed once.

## Prerequisites

- Lockbook CLI installed on both human's device and agent's machine
- Human needs a lockbook account (or will create one during setup)

## 1. Human account

If the human doesn't have one yet:

```bash
lockbook create-account <their-chosen-username>
```

This returns a 24-word recovery phrase. There are no passwords, no email, no reset flow. That phrase is the only way to recover the account. Make sure they save it.

## 2. Agent account

On the agent's machine:

```bash
lockbook create-account openclaw-<short-identifier>
```

Save the 24-word phrase to `~/.lockbook-agent-key` (NOT inside `~/.openclaw` — it would get synced to the shared folder):

```bash
echo "<24 word phrase>" > ~/.lockbook-agent-key
chmod 600 ~/.lockbook-agent-key
```

Sync immediately:

```bash
lockbook sync
```

Tell the human the agent's username so they can share with it.

## 3. Human creates and shares .openclaw

The human does this on any lockbook client (CLI, desktop, or mobile):

```bash
lockbook create .openclaw/
lockbook share .openclaw/ openclaw-<agent-username> --mode=write
lockbook sync
```

Or via the app: right-click `.openclaw` → Share → enter agent username → Write access.

## 4. Agent accepts and pulls

On the agent's machine:

```bash
lockbook sync
lockbook list                          # verify .openclaw/ appears
mkdir -p ~/.openclaw
lockbook export .openclaw/ ~/.openclaw/
```

Shares appear in the agent's file tree after sync. If pending acceptance is required, check `lockbook pending-shares`.

## 5. Set up periodic sync (optional)

For always-on agents (VMs, servers), set up a cron job or systemd timer.

### Cron (Linux/macOS)

```bash
crontab -e
```

Add:

```
*/2 * * * * lockbook export .openclaw/ $HOME/.openclaw/ && lockbook import $HOME/.openclaw/ .openclaw/ && lockbook sync >> $HOME/.openclaw-sync.log 2>&1
```

### Systemd timer (Linux)

`~/.config/systemd/user/openclaw-sync.service`:

```ini
[Unit]
Description=OpenClaw Lockbook Sync

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'lockbook export .openclaw/ %h/.openclaw/ && lockbook import %h/.openclaw/ .openclaw/ && lockbook sync'
```

`~/.config/systemd/user/openclaw-sync.timer`:

```ini
[Unit]
Description=OpenClaw Lockbook Sync Timer

[Timer]
OnBootSec=30
OnUnitActiveSec=2min

[Install]
WantedBy=timers.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-sync.timer
```

### Windows Task Scheduler

Wrap in a `.bat` file or use `schtasks`:

```powershell
$action = New-ScheduledTaskAction -Execute "cmd" -Argument "/c lockbook export .openclaw/ %USERPROFILE%\.openclaw\ && lockbook import %USERPROFILE%\.openclaw\ .openclaw/ && lockbook sync"
$trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 2) -At "00:00" -Once
Register-ScheduledTask -TaskName "OpenClawSync" -Action $action -Trigger $trigger
```

## Troubleshooting

**Shared folder not appearing** — both accounts must sync. Human runs `lockbook sync`, then agent runs `lockbook sync`. May take a moment to propagate.

**Permission denied on import** — human shared read-only. Re-share with `--mode=write`.

**Storage full** — free tier is 25MB compressed (~125MB text). Check with `lockbook usage`. Upgrade to 30GB for $2.99/month if needed.

**Sync fails with network error** — lockbook server may be temporarily unreachable. Local files remain intact. Next sync catches up.

## Notes on encryption

Lockbook uses elliptic curve cryptography. When the human shares with the agent, lockbook re-encrypts the folder key for the agent's public key. The server never sees plaintext. This is real end-to-end encrypted sharing, not just access control.

## Install lockbook CLI

```bash
cargo install lockbook                          # Requires Rust toolchain
brew tap lockbook/lockbook && brew install lockbook  # macOS
snap install lockbook                           # Linux
yay -S lockbook                                 # Arch
nix-shell -p lockbook                           # Nix
```

Windows: download from https://github.com/lockbook/lockbook/releases

Full install options: https://lockbook.net/docs/installing.html
