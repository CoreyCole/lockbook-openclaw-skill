# NFS, Git, and a Possible `lockbook fs --sync` Mode

Date: 2026-03-21

## Executive summary

For Git-heavy agent workspaces, NFS is generally a poor fit unless the NFS stack is unusually optimized and the workload is mostly large sequential IO instead of lots of metadata lookups and small-file updates. This is not just "bad NFS implementations"; Git's own documentation says a repository on NFS is often slower than the same repository accessed over SSH on the same server, because SSH lets Git operate on local disks while shared-mount access keeps every file operation on the network path.

Lockbook's current `lb-fs` is a localhost NFSv3 server implemented in userspace. Based on the code and on Linux NFS performance docs, that is very likely worse for Git than a normal local directory: even on loopback, NFS still pays protocol, RPC/XDR, caching, and mount semantics costs unless the client/server can use kernel-only optimizations such as Linux `LOCALIO`. `lb-fs` also has some implementation details that are especially unfriendly to Git workloads: a global mutex around metadata state, repeated full-document reads/writes on file writes, and known issues around file watching.

For agent workspaces, a "real local directory + background bidirectional sync" design looks like the better fit. It matches what Linux tools usually do when they care about local performance: keep the working tree on a native local filesystem, then propagate changes with `rsync`/`lsyncd`, `unison`, `syncthing`, or custom watcher-plus-reconcile logic. If Lockbook added `lockbook fs --sync`, that mode would likely be a better default than NFS for coding agents.

## 1. Is Git slow on NFS in general, or only on some NFS implementations?

### Short answer

Both are true:

- Git on NFS is *often* slow in general, because Git does lots of metadata-heavy filesystem work.
- The severity depends heavily on the specific NFS implementation, client/server tuning, caching behavior, network latency, and whether the client and server can bypass parts of the normal NFS RPC path.

### What the sources say

Git's own book is explicit:

- "A repository on NFS is often slower than the repository over SSH on the same server."  
  Source: Git Book, "The Protocols"  
  https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols.html

Why this happens is consistent with Git's other performance docs:

- `git update-index` documents fsmonitor as a way to avoid `lstat()`-ing every file in large working trees. That is a strong signal that Git status-like operations are dominated by filesystem metadata scans when nothing better is available.  
  https://git-scm.com/docs/git-update-index/2.16.6.html

- `git-fsmonitor--daemon` refuses network-mounted repos by default and says support there is experimental.  
  https://git-scm.com/docs/git-fsmonitor--daemon.html

The Linux kernel docs also explain why loopback NFS can still be materially slower than true local access:

- Linux added `NFS LOCALIO` so a client and server on the same host can bypass network RPC/XDR for reads, writes, and commits, because that makes operations "operate faster." The examples in the doc show very large gains, especially for small IO.  
  https://www.kernel.org/doc/html/v6.12/filesystems/nfs/localio.html

Community/operator reports line up with that:

- A Server Fault benchmark for a Git-like workload on NFS showed a large gap vs local drive, with NFS latency far worse than local. This is anecdotal, but it is directionally consistent with the official docs above.  
  https://serverfault.com/questions/580924/how-to-get-decent-nfs-performance-for-workloads-like-git

### Bottom line

It is not accurate to say "Git is always slow on NFS no matter what." But it *is* accurate to say:

- Git is unusually sensitive to metadata latency and small-file overhead.
- NFS commonly makes those costs visible.
- For coding workloads, "local working tree, remote sync" is usually a safer performance model than "mounted remote filesystem."

## 2. Is Lockbook's NFS implementation (`lb-fs`) a userspace NFS server? Would that make it slower than kernel NFS for Git?

### What `lb-fs` is

Yes. `lb-fs` is a userspace NFS server.

Evidence from the Lockbook repo:

- Lockbook's docs say `lb-fs` "uses nfs."  
  https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/docs/extending.md#L23-L28

- The Lockbook blog says the prototype was built on the `xetdata/nfsserve` crate.  
  https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/public-site/content/blog/multimedia-updates.md#L32-L44

- The crate itself describes itself as a "user-mode filesystem API" implemented by creating a localhost NFSv3 server you mount.  
  https://github.com/xetdata/nfsserve

- `lb-fs` binds `127.0.0.1:11111` with `NFSTcpListener::bind(...)`, then mounts that export.  
  https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/lib.rs#L69-L108

- The Linux CLI warns that this mode syncs on startup and every 30 seconds, and that file watching may miss updates.  
  https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/clients/cli/src/lb_fs.rs#L46-L61

### Would that likely be slower than kernel NFS for Git?

Probably yes, though this is partly an inference rather than a universal rule.

Why that inference is reasonable:

1. Linux `LOCALIO` only helps when the NFS client and server can cooperate in-kernel to bypass normal NFS RPC/XDR paths. A userspace localhost server will not get that optimization.  
   https://www.kernel.org/doc/html/v6.12/filesystems/nfs/localio.html

2. Mature kernel NFS servers/clients have decades of optimization around page cache interaction, request handling, and locality. Userspace NFS servers exist and can be good enough, but they usually pay extra context switches and data movement unless they do something special.

3. `lb-fs` has implementation details that are especially unfriendly to Git:
   - A global mutex protects the in-memory entry map.  
     https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/fs_impl.rs#L19-L37
   - `lookup()` often calls `get_children()` and linearly scans by name.  
     https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/fs_impl.rs#L81-L116
   - `read()` loads the whole document through Lockbook core, then slices it.  
     https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/fs_impl.rs#L125-L146
   - `write()` reads the whole document, mutates it in memory, then writes the whole document back.  
     https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/fs_impl.rs#L245-L273
   - Sync runs every 30 seconds in the background instead of being integrated with a robust watcher/change journal.  
     https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/lib.rs#L87-L95

For Git, those are bad signs because Git tends to do:

- many `stat`/lookup/readdir operations,
- many small file opens,
- lockfile/rename/create/delete sequences,
- bursts of tiny writes across many files.

### Bottom line

`lb-fs` is a userspace localhost NFS server. Even before comparing it with kernel NFS in the abstract, the current `lb-fs` implementation is very likely slower than a native local directory for Git workloads, and likely worse than a well-tuned kernel NFS path on the same host.

## 3. What is the standard Linux pattern for "sync a local directory with a server as source of truth"?

### There is no single standard

Linux does not have one standard answer. The common patterns are:

1. Local directory + one-way mirror
2. Local directory + bidirectional reconciliation
3. Peer-to-peer sync with policy controls
4. Application-specific custom watcher/journal/snapshot sync

### Common tool patterns

#### A. `rsync` / `lsyncd`: common for server-authoritative mirroring

`lsyncd` watches local directories with `inotify`/`fsevents`, batches events, and usually calls `rsync`. Its own README describes it as synchronizing local directories with remote targets and notes that it does not hamper local filesystem performance because the working tree stays local.  
https://github.com/lsyncd/lsyncd

This is a standard answer for:

- "I want fast local edits"
- "I mainly want to push changes outward"
- "The remote side is authoritative or mostly authoritative"

It is less ideal for true multi-writer bidirectional sync with rich conflict handling.

#### B. `unison`: classic bidirectional sync of two replicas

Unison's README describes the canonical two-replica model: two copies may be modified separately, then reconciled; non-conflicting changes propagate automatically and conflicting changes are detected explicitly. It also emphasizes a user-level design with normal syscalls rather than a special filesystem.  
https://github.com/bcpierce00/unison

This is the classic answer for:

- bidirectional sync,
- laptop/workstation sync,
- "keep local speed, sync later or continuously."

Its downside is operational complexity and the fact that it is not the default modern answer teams reach for first.

#### C. `syncthing`: modern continuous sync with folder roles

Syncthing documents folder roles directly:

- `sendreceive`: accept and propagate changes,
- `sendonly`: do not let Syncthing modify this device,
- `receiveonly`: do not propagate this device's changes.

It also supports filesystem watchers and rescans.  
https://docs.syncthing.net/v1.21.0/users/config.html

This is a common modern answer when users want:

- continuous sync,
- local-first performance,
- offline tolerance,
- explicit policy about who can push/pull.

### What people usually avoid for coding work

For interactive coding, people often avoid "live mounted remote filesystem as the main workspace" unless they specifically need shared storage semantics, because:

- editor/tooling latency becomes network/protocol latency,
- file watching gets tricky,
- lockfile/rename/fsync semantics vary,
- Git and build tools are happier on native local filesystems.

So the practical Linux pattern is usually:

- keep the workspace on local disk,
- use a sync tool or app-specific replicator,
- treat remote storage as replication/backup/collaboration state, not as the active working tree.

## 4. Would `lockbook fs --sync` be a better fit than NFS for agent workspaces? What would it look like?

### Recommendation

Yes, for agent workspaces, a new `lockbook fs --sync` mode looks like a better fit than NFS.

I would not frame that as "NFS is wrong." NFS still makes sense for:

- ad hoc "open this obscure file type in a desktop app",
- light document editing,
- cross-platform mount UX where a mounted drive is the main feature.

But for agents running Git, shells, compilers, and file watchers all day, a local-directory sync mode is the better architecture.

### Why it fits agent workspaces better

1. Git and build tools would operate on a native local filesystem.
2. File watching would work with normal Linux/macOS watcher APIs instead of through NFS edge cases.
3. Lockbook can choose conflict policy explicitly instead of inheriting NFS client behavior.
4. You can optimize for the coding workload directly:
   - debounce bursts,
   - hash changed files,
   - batch uploads,
   - exclude `.git/`, `target/`, `node_modules/`, etc,
   - preserve rename/move intent where possible.

### High-level implementation sketch

#### Mode shape

Example CLI:

```bash
lockbook fs --sync /path/to/worktree --root /lockbook/path
```

Behavior:

- On startup, reconcile local directory with Lockbook root.
- Server wins on startup conflicts, as requested.
- After startup, run bidirectional sync continuously.
- Keep the working tree as ordinary local files.

#### Internal model

Maintain a local sync database, for example SQLite, storing per-path:

- Lockbook file ID
- relative path
- file type
- last synced content hash
- local inode/mtime/size snapshot
- last seen remote version / server version
- tombstone state

This gives you a durable basis for conflict detection and incremental scans.

#### Startup algorithm

1. Pull the remote tree metadata from Lockbook.
2. Scan the local directory.
3. Build path maps on both sides.
4. Apply startup policy:
   - remote exists, local differs: overwrite local from remote
   - remote deleted, local exists: delete local or move aside based on policy
   - local exists, remote missing: upload local only if policy allows bootstrap import; otherwise remove/move aside
5. Persist the resulting baseline into the sync DB.

Because you explicitly want "server wins on conflicts at startup", the simplest safe policy is:

- treat remote state as authoritative baseline,
- materialize that baseline locally,
- only then begin continuous bidirectional sync.

That avoids ambiguous startup merges.

#### Continuous sync loop

Use two inputs:

- local watcher events (`inotify` on Linux, FSEvents on macOS),
- periodic remote poll or incremental remote sync API.

For local changes:

1. debounce a short window,
2. rescan only affected paths,
3. hash changed files,
4. upload changed content,
5. record new remote version and local hash.

For remote changes:

1. fetch changed metadata/content,
2. if local path unchanged since last sync, apply remote update locally,
3. if both changed since baseline, use conflict policy.

#### Conflict policy

Startup:

- remote wins.

After startup:

- either "last writer wins" with conflict copies,
- or "remote wins but preserve local as `*.conflict-<timestamp>`",
- or "surface conflict and pause path."

For agent workspaces, I would prefer:

- remote wins only at startup,
- after startup, preserve both sides on conflict by writing a conflict copy locally and in Lockbook metadata/logs.

That is safer than silently discarding agent edits.

#### Path and content policy

Add ignores, ideally with defaults:

- `.git/`
- build outputs (`target/`, `dist/`, `build/`)
- package caches
- editor temp files

This matters a lot. If the intent is "workspace contents tracked in Lockbook", syncing `.git/` itself is usually unnecessary and expensive. A better model is:

- sync source/worktree files,
- let Git talk to its real remotes normally.

If you do want `.git/`, make it opt-in.

#### Atomicity and correctness

Use temp files + atomic rename for local materialization.

Keep a write-intent guard so remote-applied local writes do not loop back into immediate re-upload.

Persist tombstones so deletes can sync cleanly.

If Lockbook's API can expose stable per-file content hashes/version vectors, use them. If not, add them; they make sync logic much simpler and more reliable.

### Recommended product positioning

My recommendation would be:

- keep `lockbook fs` as the experimental mounted-filesystem mode,
- add `lockbook fs --sync` or `lockbook sync-dir` as the coding/automation mode,
- present the latter as the preferred mode for Git repos, agents, builds, and editor-heavy workflows.

That split matches the two very different use cases:

- mounted-drive UX,
- high-performance local workspace replication.

## Overall answer

For agent workspaces, yes: a local-directory sync mode is the better fit than NFS.

The main reason is not just "userspace NFS is slower than kernel NFS." The stronger point is:

- Git wants a real local filesystem,
- Lockbook's current `lb-fs` adds extra costs that Git is sensitive to,
- Linux sync tools generally solve this class of problem by keeping the active tree local and syncing in the background,
- Lockbook can do the same with clearer semantics and better performance.

## Sources

- Git Book, "The Protocols": https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols.html
- Git `git-fsmonitor--daemon`: https://git-scm.com/docs/git-fsmonitor--daemon.html
- Git `git-update-index`: https://git-scm.com/docs/git-update-index/2.16.6.html
- Linux kernel docs, `NFS LOCALIO`: https://www.kernel.org/doc/html/v6.12/filesystems/nfs/localio.html
- Server Fault benchmark/discussion: https://serverfault.com/questions/580924/how-to-get-decent-nfs-performance-for-workloads-like-git
- Lsyncd README: https://github.com/lsyncd/lsyncd
- Unison README: https://github.com/bcpierce00/unison
- Syncthing config docs: https://docs.syncthing.net/v1.21.0/users/config.html
- `xetdata/nfsserve`: https://github.com/xetdata/nfsserve
- Lockbook `docs/extending.md`: https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/docs/extending.md#L23-L28
- Lockbook blog on `lb-fs`: https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/public-site/content/blog/multimedia-updates.md#L32-L44
- Lockbook `libs/lb-fs/src/lib.rs`: https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/lib.rs#L69-L108
- Lockbook `libs/lb-fs/src/fs_impl.rs`: https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/libs/lb-fs/src/fs_impl.rs
- Lockbook `clients/cli/src/lb_fs.rs`: https://github.com/lockbook/lockbook/blob/d38962b25cd901689b26c26f673b9a924a5ffb75/clients/cli/src/lb_fs.rs#L46-L61
