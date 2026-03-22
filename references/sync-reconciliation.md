# Lockbook Sync Reconciliation Research

## Summary

Lockbook does expose per-file `last_modified` metadata, but the current export path is not a reconciliation engine. It either refuses to overwrite existing files or truncates and overwrites them when `edit=true`. Lockbook's own internal sync solves merge against its own metadata/doc model, not against arbitrary disk state. For OpenClaw, the safest simple design is:

- split paths into `human-managed` and `agent-managed`
- never silently overwrite when both sides changed
- use Lockbook metadata plus a small local sync state file
- use disk `mtime` only as a cheap hint, not as the source of truth

## Findings

### 1. Does Lockbook track last-modified timestamps per file?

Yes.

- `File` includes `last_modified: u64` and `last_modified_by: Username` in [`libs/lb/lb-rs/src/model/file.rs:41`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/model/file.rs#L41).
- `list_metadatas()` returns `Vec<File>` in [`libs/lb/lb-rs/src/service/file.rs:123`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/file.rs#L123), so callers do get that field.
- The value comes from `meta.timestamped_value.timestamp` in [`libs/lb/lb-rs/src/model/core_ops.rs:31`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/model/core_ops.rs#L31).
- That timestamp is generated from wall-clock milliseconds since epoch via `get_time()` in [`libs/lb/lb-rs/src/model/clock.rs:10`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/model/clock.rs#L10).

Important nuance:

- this is a metadata timestamp, not a POSIX filesystem timestamp
- it updates when a new signed metadata/doc state is created
- it is useful for "has Lockbook changed?" but not strong enough by itself for cross-device overwrite decisions

### 2. Does export overwrite or skip existing files?

It depends on the `edit` flag.

In [`libs/lb/lb-rs/src/service/import_export.rs:266`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/import_export.rs#L266):

- if `edit == true`, documents are opened with `create(true)` and `truncate(true)` in lines 282-287, so existing files are overwritten
- if `edit == false`, documents are opened with `create_new(true)` in lines 288-292, so export fails if the file already exists

For folders:

- it calls `fs::create_dir(...)` in line 301, so existing directories also cause an error rather than being merged

So the export command itself is not doing selective reconciliation. It is either:

- strict no-clobber export, or
- clobber existing document files in place

### 3. Is there a way to know which files changed since last sync?

Internally: yes. Publicly as a simple file-list API: not really.

What exists:

- internal sync asks the server for updates with `GetUpdatesRequestV2 { since_metadata_version }` in [`libs/lb/lb-rs/src/service/sync.rs:67`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L67) and [`libs/lb/lb-rs/src/service/sync.rs:243`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L243)
- the request shape is defined in [`libs/lb/lb-rs/src/model/api.rs:277`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/model/api.rs#L277)
- `calculate_work()` exposes `SyncStatus { work_units, latest_server_ts }` and reports `WorkUnit::ServerChange(id)` / `WorkUnit::LocalChange(id)` in [`libs/lb/lb-rs/src/service/sync.rs:61`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L61) and [`libs/lb/lb-rs/src/model/work_unit.rs:6`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/model/work_unit.rs#L6)
- local persisted sync position is `db.last_synced` in [`libs/lb/lb-rs/src/service/sync.rs:1154`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L1154)

What does not exist in the code I found:

- no `list_files_changed_since(timestamp)` API returning `Vec<File>`
- no public `list remote changes since X` helper that already resolves ids to paths + metadata for you

So if you are building on `lb-rs`, the closest existing signal is:

- call `calculate_work()` before sync to get changed ids
- or maintain your own previous snapshot from `list_metadatas()` and diff by `id` / `last_modified`

### 4. How does Lockbook's own sync handle merge internally?

It does a real base/remote/local merge inside its own metadata + encrypted-doc model.

The main merge pipeline is in [`libs/lb/lb-rs/src/service/sync.rs:370`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L370).

High level:

- `base` = last synced local base metadata
- `remote` = server updates since `last_synced`
- `local` = unsynced local metadata/doc changes
- it constructs a merged change set and validates it in a loop, adding conflict-resolution constraints when needed

Conflict behavior:

- path conflicts are resolved by renaming one local side with incremented suffixes like `-1`, `-2` in [`libs/lb/lb-rs/src/service/sync.rs:830`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L830)
- tests confirm this behavior in [`libs/lb/lb-rs/tests/sync_service_tests.rs:8`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/tests/sync_service_tests.rs#L8) and [`libs/lb/lb-rs/tests/sync_service_path_conflict_resolution_tests.rs:20`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/tests/sync_service_path_conflict_resolution_tests.rs#L20)
- text documents use a 3-way merge via `Buffer::merge(local, remote)` in [`libs/lb/lb-rs/src/service/sync.rs:635`](./tmp/tmp.WF9H4PAsll/lockbook/libs/lb/lb-rs/src/service/sync.rs#L635)
- drawing documents also have specialized merge logic in lines 656-699
- other document types are not merged textually; Lockbook duplicates the file under a new incremented name in lines 700-747
- if only local changed, it overwrites remote/base with local in lines 749-757

This is useful as inspiration, but it is not directly reusable for disk reconciliation unless you are willing to:

- mirror Lockbook ids/hmacs/base state per file on disk
- keep base copies
- classify text vs drawing vs binary
- build a 3-way merge layer around arbitrary filesystem files

That is much more complex than your stated use case needs.

### 5. Could we use disk `mtime` vs Lockbook `last_modified` to detect conflicts?

Partially, but not as the primary decision rule.

Why it helps:

- both are time-ish signals in milliseconds
- it is cheap to see whether the disk file was touched since the last export/import

Why it is unsafe as the main rule:

- Lockbook `last_modified` comes from whichever device signed the metadata, so cross-device clock skew matters
- disk `mtime` can be preserved, rounded, rewritten, or touched without content change
- export/import itself will change disk timestamps unless you explicitly preserve them
- comparing `disk_mtime > lockbook_last_modified` assumes synchronized clocks, which you do not control

Best use:

- use `mtime` only as a fast candidate signal
- confirm with stored content hashes and stored last-seen Lockbook metadata

## Simplest Safe Reconciliation Strategy For OpenClaw

### Design goal

Do not build a generic bidirectional sync engine for the whole workspace. Split ownership.

That gives you the policy you asked for:

- human changes win for files they explicitly put in
- agent changes win for files the agent manages
- conflicts never silently drop data

### 1. Split the namespace

Use two explicit classes of paths inside the synced Lockbook folder:

- `inbox/` or `human/`: human-managed drop zone
- `workspace/` or selected subpaths: agent-managed mirror

Rules:

- humans are expected to add/edit files in `inbox/`
- agent-owned files live under `workspace/`
- if a human edits an agent-owned path directly in Lockbook, treat it as a conflict, not as an in-place overwrite

This is the biggest simplification. It avoids needing content-aware merges for logs, scratch files, memory files, and constantly-changing workspace artifacts.

### 2. Keep a small local sync state

Store a local state file outside the mirrored workspace, for example:

- `~/.openclaw/lockbook-sync-state.json`

Per tracked file, store at least:

- Lockbook file `id`
- Lockbook path
- last seen `last_modified`
- last exported/imported content hash
- last known disk path
- last known disk hash
- last known disk `mtime`
- ownership class: `human-managed` or `agent-managed`

This lets you answer:

- did Lockbook change since I last saw it?
- did disk change since I last saw it?
- are these the same bytes despite timestamp noise?

### 3. Detect remote changes by snapshot diff, not by timestamp comparison alone

At each cycle:

1. run Lockbook sync
2. call `list_metadatas()`
3. diff current metadata against your saved state by `id`

Treat a file as remotely changed if:

- it is new
- it was deleted
- its path changed
- its `last_modified` changed

This is simpler and safer than comparing raw disk `mtime` against Lockbook `last_modified`.

If you are embedding `lb-rs`, you can optionally use `calculate_work()` as a prefilter for changed ids, but the state-file snapshot diff is easier to reason about and sufficient.

### 4. Reconciliation rules

#### Human-managed paths

Policy: remote wins, but never destroy unsynced local data.

If Lockbook changed and disk did not:

- export with overwrite enabled

If Lockbook changed and disk also changed since last sync:

- keep the Lockbook version at the intended path
- move the local disk version to a conflict path, for example:
  - `foo.conflict-agent-2026-03-21T120000Z.txt`
- record the conflict in the state/log

Rationale:

- human intent should win in the inbox/drop zone
- agent data is still preserved

#### Agent-managed paths

Policy: local agent version wins, but human remote edits are preserved as conflict copies.

If disk changed and Lockbook did not:

- import/patch the agent version into Lockbook

If Lockbook changed and disk did not:

- do not overwrite the agent path in place unless the path is explicitly designated as remotely controlled
- instead export the Lockbook version as a sibling conflict file or into a `_from_lockbook_conflicts/` folder

If both changed:

- keep the disk file as authoritative at the main path
- export the Lockbook version as a conflict copy
- optionally create a small `.conflict.json` note with both hashes and timestamps

Rationale:

- the agent edits these files constantly
- surprise remote overwrite is the highest-risk failure mode
- preserving the human-provided version as a side file avoids data loss

### 5. Avoid using `export_file(..., edit=true)` blindly

Because export with `edit=true` truncates and overwrites existing disk files, it should only be used after your own conflict logic decides overwrite is safe.

Otherwise:

- export to a temp path first
- compare hashes
- then atomically replace or create a conflict copy

### 6. Handle deletions conservatively

For your use case, do not propagate deletes automatically across ownership boundaries.

Suggested rule:

- if a human deletes something in `inbox/`, reflect that locally
- if a human deletes an agent-managed file in Lockbook, do not delete the disk copy automatically; mark it as a conflict/tombstone event
- if the agent deletes one of its own files locally, propagate that to Lockbook for agent-managed paths

This avoids the most dangerous silent-loss case: a phone-side delete removing active workspace state.

## Recommended Minimal Algorithm

Per sync cycle:

1. Scan disk state for tracked files under `~/.openclaw/workspace/` and compute cheap metadata (`mtime`, size). Hash only candidate-changed files.
2. Run Lockbook sync.
3. Fetch Lockbook `list_metadatas()` for the synced folder.
4. Diff against saved sync state.
5. For each changed file, apply ownership rule:
   - `human-managed`: remote wins, local conflicting copy preserved
   - `agent-managed`: local wins, remote conflicting copy preserved
6. For safe overwrites, write via temp file + atomic rename.
7. Update sync state only after the file result is durable on both sides.

## Recommendation

The simplest workable design is not "true bidirectional merge" across the whole workspace. It is:

- ownership-based reconciliation
- snapshot diff using Lockbook `id` + `last_modified`
- local state file with hashes
- conflict copies instead of silent overwrite

Use `mtime` as an optimization, not as the authority.

If later you want something smarter, the next step would be:

- text-only 3-way merge for a very small allowlist such as `.md`, `.txt`, `.json`

I would not start there. The namespace split plus conflict-copy strategy is much smaller and much safer for an agent workspace.
