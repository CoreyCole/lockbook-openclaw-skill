# RFC: `lockbook sync-dir` and `.lockbookignore`

## Summary

This proposal adds a new CLI workflow for Lockbook:

1. `lockbook sync-dir`: bidirectionally sync a real local directory with a Lockbook folder.
2. `.lockbookignore`: a `.gitignore`-style ignore file that excludes local files, directories, and globs from sync.

The goal is to support coding workspaces, agent sandboxes, automation pipelines, and other environments that need ordinary on-disk files rather than a mounted NFS view. This is a complement to `lockbook fs`, not a replacement for it.

## Motivation

Lockbook already has an experimental filesystem story via `lb-fs`, exposed in the CLI as `lockbook fs`. Today that path is NFS-backed. The current docs describe `lb-fs` as "an experimental virtual file system implementation backed by lockbook" and note that it currently uses NFS while platform-specific implementations may be explored later.

That direction makes sense for "mount my Lockbook like a drive" workflows, but it is a weak fit for modern coding and automation workloads:

- Git expects a normal local filesystem with stable inode and rename semantics.
- Build tools create large amounts of ephemeral output that should usually not sync.
- File watchers, language servers, package managers, and editor tooling often behave poorly over network filesystems.
- Agent workspaces and CI-like local automation want files to persist on disk even when the sync process is not running.
- NFS is especially awkward as a cross-platform long-term strategy.

Lockbook has already publicly signaled that this is an open design question. In the July 6, 2024 blog post "Multimedia Updates!", the team wrote that they were "pausing for some reflection" on whether `lb-fs` should keep investing in NFS, and suggested that higher-quality platform-specific interfaces may be preferable in the future:

- Blog post: https://lockbook.net/blog/multimedia-updates/
- Relevant repo copy: `public-site/content/blog/multimedia-updates.md`

This proposal addresses the same problem from a different angle: instead of projecting Lockbook into a mounted filesystem, let users designate a local directory as the working copy and have Lockbook sync to and from it.

## Goals

- Support coding workspaces and automation with ordinary local files.
- Preserve Lockbook as the durable encrypted sync system.
- Keep local disk as the primary working surface.
- Make startup and restart behavior deterministic.
- Avoid syncing obvious build artifacts and caches by default.
- Reuse existing `lb-rs` sync machinery as much as possible.

## Non-goals

- Replacing `lockbook fs` for every use case.
- Implementing a kernel virtual filesystem.
- Making ignored files available remotely.
- Solving every possible merge strategy beyond a safe conflict-copy model.

## Why this should exist alongside `lockbook fs`

`lockbook fs` and `lockbook sync-dir` solve different problems.

`lockbook fs` is still useful when the desired UX is:

- "mount my Lockbook and browse/open it like a drive"
- "let the desktop app expose files externally"
- "open a small number of non-native files through Finder/File Explorer"

`lockbook sync-dir` is better when the desired UX is:

- "treat this folder like a normal project checkout"
- "run git, cargo, npm, uv, make, watchers, and LSPs locally"
- "allow files to remain on disk when Lockbook is not running"
- "sync a workspace in the background instead of mounting a networked view"

In other words: `fs` remains the virtual mount story; `sync-dir` becomes the real-worktree story.

## Proposed CLI

Primary command:

```bash
lockbook sync-dir <lockbook-folder> <local-dir>
```

Example:

```bash
lockbook sync-dir /Projects/agent-workspace ~/work/agent-workspace
```

Suggested behavior:

- `<lockbook-folder>` must resolve to an existing Lockbook folder.
- `<local-dir>` is created if missing.
- The command starts a long-running sync loop and does not return until interrupted.
- Files remain on disk after the process exits.
- Restarting the command re-reconciles local state against the server and resumes syncing.

Suggested optional flags:

```bash
lockbook sync-dir <lockbook-folder> <local-dir> [--pull-interval 5s] [--no-watch]
```

Minimal initial flag set:

- `--pull-interval <duration>`: polling interval for remote changes, default `5s`
- `--no-watch`: disable filesystem watching and fall back to periodic local scans

Possible future flags, but not required for v1:

- `--one-shot`: reconcile once and exit
- `--state-dir <path>`: override where local sync state is stored
- `--exclude-defaults`: disable built-in ignore patterns for debugging

## Core semantics

### Working model

- Local disk is the primary editing surface.
- Lockbook is the sync target and source of truth for remote state.
- Sync is bidirectional while the process is running.
- When the process is not running, files simply remain on disk.

### Startup reconciliation

On startup, the command performs a full reconciliation before entering the watch loop.

Proposed rule: **server wins on startup**.

That means:

- If a path exists remotely and locally is unchanged since the last successful sync, materialize the remote version locally.
- If a path exists remotely and locally has also changed, keep the remote version at the canonical path and move the local version aside as a conflict copy.
- If a path exists only locally and collides with incoming remote state, move the local file aside as a conflict copy.
- If a path exists only remotely, create it locally.

Conflict sidecar naming:

```text
<name>.conflict-<timestamp>
```

Examples:

- `main.rs.conflict-2026-03-21T14-05-22Z`
- `notes.md.conflict-2026-03-21T14-05-22Z`

This startup rule is intentionally conservative and restart-safe. It guarantees that after process start, the visible workspace matches Lockbook state, while still preserving any conflicting local bytes for recovery.

### Live sync while running

While `sync-dir` is active:

- Local filesystem changes are detected with `inotify` on Linux and `FSEvents` on macOS.
- Detected local changes are debounced, fingerprinted, and uploaded into Lockbook.
- Remote changes are fetched by periodic Lockbook sync and then applied locally.

Conflict policy during runtime:

- If a remote change arrives for a path whose local content has not changed since the last synced baseline, apply the remote change in place.
- If both local and remote changed relative to the last synced baseline, write the remote version to the canonical path and preserve local content as a conflict sidecar.
- If only local changed, upload local.

This gives a clear rule set:

- unchanged local + changed remote -> remote wins
- changed local + unchanged remote -> local uploads
- changed local + changed remote -> keep both, canonical path gets remote, local copy becomes conflict sidecar

## Internal architecture

### Local sync state

`sync-dir` needs durable local metadata so that restart behavior is deterministic and conflict detection is cheap.

Suggested approach: store a small SQLite-backed sync manifest inside the local directory.

Example internal layout:

```text
<local-dir>/
  .lockbook/
    sync.sqlite
    config.json
    journal/
```

The `.lockbook/` directory should always be excluded from sync.

Suggested manifest tables:

- `entries`
  - relative path
  - lockbook file id
  - file type
  - last synced content hash
  - last synced remote version / metadata fingerprint
  - last observed local mtime / size
- `pending_local_ops`
  - queued creates / writes / deletes / renames
- `sync_session`
  - local root
  - lockbook root folder id
  - schema version
  - last successful full sync time

SQLite is a good fit because it provides:

- crash-safe local state
- cheap restart and resume
- simple path/file-id mapping
- a clear place to record conflict outcomes and pending operations

### Mapping model

Each synced path should be tracked by both:

- relative path within the local root
- Lockbook file/folder id

That avoids treating renames as delete-and-recreate whenever metadata allows better reconciliation. It also reduces ambiguity if a path is reused locally.

### Startup algorithm

High-level flow:

1. Load sync manifest from `.lockbook/sync.sqlite`, or create one.
2. Perform a normal Lockbook sync to fetch latest remote metadata and content handles.
3. Enumerate the Lockbook subtree rooted at `<lockbook-folder>`.
4. Enumerate local files under `<local-dir>`, applying built-in ignores and `.lockbookignore`.
5. Compare remote state, local state, and last synced manifest baseline.
6. Apply startup reconciliation:
   - materialize remote files/directories
   - move conflicting local-only or locally-modified files to sidecars
   - update manifest
7. Start watcher + periodic remote sync loop.

### Runtime loop

Two independent event sources feed the reconciler:

- local watcher events
- periodic remote sync ticks

Local path:

1. Watcher reports create/write/remove/rename.
2. Debounce event bursts.
3. Re-scan affected paths to compute stable state and content hash.
4. Filter ignored files.
5. Translate to Lockbook operations.
6. Commit to Lockbook and update manifest.

Remote path:

1. Timer triggers a Lockbook sync.
2. Fetch subtree changes for the bound folder.
3. Compare remote state against manifest baseline and current local state.
4. Apply remote updates to disk where safe.
5. If both sides changed, create conflict sidecar and keep both.
6. Update manifest.

### File operations

The implementation should support:

- file create/update/delete
- directory create/delete
- rename/move when detectable

Delete behavior should be explicit:

- a local delete propagates to Lockbook if the path is tracked and not ignored
- a remote delete removes the local file if the local path is unchanged since baseline
- if the local path changed and the remote side deleted it, preserve the local content as a conflict sidecar rather than silently discarding it

### Hashing and change detection

Manifest entries should store a content hash for the last synced version. That makes conflict detection straightforward:

- compare current local hash to last synced hash
- compare current remote version/hash to last synced remote fingerprint

This is more reliable than mtime-only logic and avoids false positives from editor save patterns.

### Platform support

Initial target:

- Linux: `inotify`
- macOS: `FSEvents`

Fallback:

- if watcher setup fails, optionally continue in scan/poll mode with a warning

Windows can be deferred until there is a clean `ReadDirectoryChangesW` or similar implementation.

## `.lockbookignore` specification

### Purpose

`.lockbookignore` tells `lockbook sync-dir` which local files and directories should not be uploaded to Lockbook and should not be managed as part of the synced working set.

This is necessary because coding workspaces generate a large amount of local-only state that should remain local:

- VCS data
- dependency caches
- compiler output
- SQLite side files
- editor cruft
- OS metadata

### Location

- `.lockbookignore` lives at the root of the synced local directory.
- Only the root `.lockbookignore` is required for v1.
- Nested `.lockbookignore` files could be added later, but are not necessary initially.

### Syntax

Use the same pattern syntax as `.gitignore`:

- blank lines ignored
- `#` starts a comment
- glob patterns supported
- trailing `/` matches directories
- leading `/` anchors to sync root
- `!` negates a previous pattern

Examples:

```gitignore
# Dependencies
node_modules/

# Rust build output
target/

# Local databases
*.sqlite
*.sqlite-shm
*.sqlite-wal

# Python cache
__pycache__/

# Keep one generated file
dist/
!dist/manifest.json
```

### Built-in default ignores

These should apply even if no `.lockbookignore` file exists:

```gitignore
.git/
*.sqlite
*.sqlite-shm
*.sqlite-wal
node_modules/
target/
__pycache__/
.DS_Store
```

Recommended additional implicit ignore:

```gitignore
.lockbook/
```

That entry is not user-facing project content and should never sync.

### Ignore semantics

Ignored paths should:

- not be uploaded to Lockbook
- not be deleted locally because of remote changes
- not appear in the sync manifest as tracked content, except possibly as cached ignore records

If a file was previously tracked and later becomes ignored, the behavior should be explicit. The safest v1 behavior is:

- stop syncing future local changes for that path
- leave the current remote copy untouched
- emit a warning that the file is now ignored locally but still exists remotely until manually deleted or unignored

This avoids surprising destructive deletes.

## UX details

### Console output

The command should be quiet by default but clear during important events:

- initial full reconcile summary
- local upload summary
- remote apply summary
- conflict notifications
- watcher fallback warnings

Example:

```text
sync-dir: bound /Projects/agent-workspace -> ~/work/agent-workspace
startup: fetched 184 remote paths
startup: restored 181 paths, wrote 3 conflict copies
watch: inotify active
remote: applied 2 updates
local: uploaded 5 changes
conflict: src/lib.rs -> src/lib.rs.conflict-2026-03-21T14-05-22Z
```

### Safety properties

The design should prioritize "never silently lose local bytes".

Concretely:

- conflicting local content is moved aside, not overwritten invisibly
- ignored files remain local
- stopping the process does not remove local files
- restart always replays a deterministic reconciliation process

## Why server-wins on startup is the right default

For this workflow, startup is the only moment where the process must reconcile an unknown amount of offline local drift against authoritative remote state. Favoring the server at that boundary provides predictable semantics:

- the canonical working tree after startup matches Lockbook
- remote edits from another machine are never shadowed by stale local files
- local unsynced changes are preserved as conflict copies instead of silently merged or dropped

This is stricter than typical local-first sync tools, but it matches the stated use case: ephemeral agent and coding workspaces that can be stopped and restarted frequently, where correctness and recoverability matter more than aggressive auto-merge.

## Implementation sketch

This seems implementable mostly in the CLI plus a reusable sync helper crate, with heavy reuse of `lb-rs` for:

- authentication and account state
- file tree enumeration
- content fetch/upload
- existing sync primitives

Rough layering:

- `clients/cli`
  - command parsing
  - console UX
- new sync-dir module / crate
  - watcher integration
  - manifest persistence
  - reconcile engine
  - `.lockbookignore` parsing
- `lb-rs`
  - remote sync, file metadata, content read/write

## Open questions

1. Should v1 require the remote folder to already exist, or create it if missing?
2. Should `.lockbookignore` use an existing Rust crate with gitignore-compatible behavior, or a smaller custom matcher?
3. Should conflict sidecars preserve the original extension exactly as `<name>.conflict-<timestamp>` or use `<stem>.conflict-<timestamp>.<ext>`?
4. Should remote polling eventually move from timer-based sync to push/event-driven sync if Lockbook infrastructure grows that capability?
5. Should there be a one-shot "materialize without watching" mode for scripting and backup use cases?

## Proposed rollout

### v1

- `lockbook sync-dir <lockbook-folder> <local-dir>`
- startup reconciliation with server-wins semantics
- local watcher on Linux/macOS
- periodic remote sync polling
- SQLite sync manifest
- `.lockbookignore` support
- built-in default ignores
- conflict-copy behavior

### Later

- Windows watcher support
- desktop app integration
- richer status output
- one-shot mode
- optional nested ignore files
- smarter rename tracking

## Closing

Lockbook should keep pursuing the mounted-filesystem story where it makes sense, but coding and automation workloads need a different primitive: a real local working tree with safe, deterministic background synchronization.

`lockbook sync-dir` plus `.lockbookignore` gives Lockbook that primitive without abandoning `lb-fs`. It fits the project's current architecture, matches the team's public reconsideration of NFS as the long-term answer, and opens Lockbook to a much stronger set of developer, agent, and automation workflows.
