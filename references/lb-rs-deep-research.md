# lb-rs deep research: `import_export --patch` simplification candidates

Scope reviewed:
- `libs/lb/lb-rs/src/service/import_export.rs`
- `libs/lb/lb-rs/src/service/path.rs`
- `libs/lb/lb-rs/src/service/file.rs`
- `libs/lb/lb-rs/src/service/documents.rs`
- `libs/lb/lb-rs/src/service/sync.rs`
- `libs/lb/lb-rs/src/model/path_ops.rs`
- `libs/lb/lb-rs/src/model/tree_like.rs`
- Supporting lower-level ops in `libs/lb/lb-rs/src/model/core_ops.rs`, `lazy.rs`, `staged.rs`

Patch code under review:
- `Lb::import_files_patch`
- `Lb::import_file_recursively_patch`
- File: `libs/lb/lb-rs/src/service/import_export.rs`

## Executive summary

There is no existing single method in `lb-rs` that does "resolve path -> return metadata -> create if missing -> write content" in one call.

The strongest existing replacements/simplifiers are:
- `Lb::get_by_path(&self, path: &str) -> LbResult<File>`
- `Lb::create_at_path(&self, path: &str) -> LbResult<File>`
- `Lb::write_document(&self, id: Uuid, content: &[u8]) -> LbResult<()>`

That combination can remove the current upfront `list_metadatas + list_paths_with_ids` path map entirely, at the cost of doing per-file path lookups and handling `PathConflict` / `FileNonexistent` at call sites.

There is no existing write-by-path method, no upsert-by-path method, and no service-layer merge helper intended for import patching.

---

## 1. Methods that return files with resolved paths

### `pub async fn get_by_path(&self, path: &str) -> LbResult<File>`
- File: `libs/lb/lb-rs/src/service/path.rs`
- Internals:
  - builds a lazy tree from base + local metadata
  - resolves `path` via `tree.path_to_id(path, root, &self.keychain)?`
  - decrypts the resulting metadata into a `File`
- What it returns:
  - a `File` struct with metadata (`id`, `parent`, `name`, `file_type`, owner, shares, size, etc.)
  - not the path string itself
- Could replace part of patch implementation:
  - Yes. It can replace the current precomputed `path_map` lookup for existence + metadata.
  - Pattern would be: compute `lb_path`, call `get_by_path(lb_path_or_lb_path_slash)`, then `write_document(existing.id, ...)` or `create_at_path` if not found.
- Caveats:
  - one lookup per imported item instead of one global scan
  - folder paths appear to require trailing `/` for folder semantics in some APIs
  - returns `File`, but not `(File, resolved_path)`

### `pub async fn get_file_by_id(&self, id: Uuid) -> LbResult<File>`
- File: `libs/lb/lb-rs/src/service/file.rs`
- What it returns:
  - full `File` metadata only
- Could replace part of patch implementation:
  - Only indirectly. Useful once you already have an id.
  - Not useful for path-based existence checks.
- Caveats:
  - no path resolution

### `pub fn decrypt(&mut self, keychain: &Keychain, id: &Uuid, public_key_cache: &LookupTable<Owner, String>) -> LbResult<File>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- What it returns:
  - full `File` metadata from an id
- Could replace part of patch implementation:
  - Only in a custom helper that resolves path to id inside one transaction, then decrypts.
  - Not directly callable as a service API.
- Caveats:
  - low-level tree API, still needs explicit `path_to_id`

### `pub fn decrypt_all<I>(&mut self, keychain: &Keychain, ids: I, public_key_cache: &LookupTable<Owner, String>, skip_invisible: bool) -> LbResult<Vec<File>> where I: Iterator<Item = Uuid>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- Could replace part of patch implementation:
  - Only if you already have ids. Not a path lookup tool.
- Caveats:
  - no path output

### Conclusion for item 1
- There is no existing method returning `Vec<(File, path)>` or `File` plus resolved path in one call.
- Closest existing service API is `get_by_path`, which already combines path resolution and metadata fetch for a single item.

---

## 2. `get_by_path`, `get_file_by_path`, or similar path-based lookup

### `pub async fn get_by_path(&self, path: &str) -> LbResult<File>`
- File: `libs/lb/lb-rs/src/service/path.rs`
- Best direct match for "get file by path"
- Could replace part of patch implementation:
  - Yes, likely the cleanest replacement for `path_map.get(...)`

### `pub fn path_to_id(&mut self, path: &str, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Behavior:
  - walks each path component from `root`
  - skips deleted children
  - compares names using `name_using_links`
  - if a child is a link, follows the link target
- Could replace part of patch implementation:
  - Yes, but only inside a new service helper. `get_by_path` already wraps it.
- Caveats:
  - returns only `Uuid`
  - requires an already-built tree and root id
  - link-following behavior matters; this is not a raw path table lookup

### `pub async fn get_path_by_id(&self, id: Uuid) -> LbResult<String>`
- File: `libs/lb/lb-rs/src/service/path.rs`
- Behavior:
  - reverse lookup via `tree.id_to_path`
- Could replace part of patch implementation:
  - Already used for `dest_lb_path`
  - Not helpful for existence checks

### `pub fn id_to_path(&mut self, id: &Uuid, keychain: &Keychain) -> LbResult<String>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Behavior:
  - reconstructs full path
  - adds trailing `/` for folders
  - handles links specially
- Caveats:
  - returns only string path

### Missing APIs
- No `get_file_by_path`
- No `maybe_get_by_path`
- No bulk path lookup helper taking many paths

---

## 3. `write_document` variants: any upsert or write-by-path methods?

### `pub async fn write_document(&self, id: Uuid, content: &[u8]) -> LbResult<()>`
- File: `libs/lb/lb-rs/src/service/documents.rs`
- Behavior:
  - opens write tx
  - resolves links to target
  - `tree.update_document(&id, content, &self.keychain)?`
  - writes encrypted doc blob to docs store
  - emits events/activity
- Could replace part of patch implementation:
  - Already used
  - Still requires prior lookup or creation
- Caveats:
  - id-based only
  - not an upsert
  - not path-based

### `pub async fn safe_write(&self, id: Uuid, old_hmac: Option<DocumentHmac>, content: Vec<u8>) -> LbResult<DocumentHmac>`
- File: `libs/lb/lb-rs/src/service/documents.rs`
- Behavior:
  - compare-and-swap style write
  - errors with `ReReadRequired` if hmac changed
- Could replace part of patch implementation:
  - Probably not for import patch.
  - Useful only if patching should avoid overwriting concurrent local edits.
- Caveats:
  - still id-based
  - caller must already know id and old hmac

### `pub fn update_document(&mut self, id: &Uuid, document: &[u8], keychain: &Keychain) -> LbResult<EncryptedDocument>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- Low-level version underlying `write_document`
- Caveats:
  - not a service helper
  - not path-based
  - not an upsert

### `pub fn update_document_unvalidated(&mut self, id: &Uuid, document: &[u8], keychain: &Keychain) -> LbResult<EncryptedDocument>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- Used heavily in sync merge construction
- Could replace part of patch implementation:
  - Only inside a custom transaction-aware helper
  - Not appropriate directly from service/import code unless intentionally bypassing validation
- Caveats:
  - unvalidated
  - still id-based
  - not path-based

### Conclusion for item 3
- No write-by-path method
- No create-or-write method
- No upsert document API

---

## 4. `create_at_path`: what exactly does it do, does it handle conflicts?

### Service method
### `pub async fn create_at_path(&self, path: &str) -> LbResult<File>`
- File: `libs/lb/lb-rs/src/service/path.rs`
- Behavior:
  - opens tx
  - builds staged lazy tree
  - resolves root
  - calls model `tree.create_at_path(path, root, &self.keychain)?`
  - decrypts created id into `File`
- Could replace part of patch implementation:
  - Yes. It can replace `create_file(name, parent, file_type)` plus some manual parent-path bookkeeping.
  - Especially useful if patch code is rewritten around full lockbook paths instead of `(dest id, name)`.
- Caveats:
  - creates intermediate folders automatically
  - returns error on conflicts; it is not an upsert

### Model method
### `pub fn create_at_path(&mut self, path: &str, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Behavior:
  - validates path
  - infers final type from trailing slash:
    - `.../` => folder
    - otherwise => document
  - splits path and delegates to `create_at_path_helper`
- Conflict behavior:
  - yes, it explicitly detects conflict on the final path component and returns:
    - `LbErrKind::Validation(ValidationFailure::PathConflict(HashSet<Uuid>))`
- Intermediate path behavior:
  - if a matching intermediate child exists:
    - `Document` => error `NonFolderWithChildren`
    - `Folder` => descend
    - `Link { target }` => descends into target if write access is available, otherwise `InsufficientPermission`
  - if intermediate child does not exist:
    - creates a folder automatically
- Could replace part of patch implementation:
  - Yes, for creation logic.
  - No, for "get existing or create" because conflict is treated as error.

### `fn create_at_path_helper(&mut self, file_type: FileType, path_components: Vec<&str>, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Important detail:
  - this is where the conflict behavior lives
  - it is pure create, not upsert

---

## 5. `list_paths_with_ids`: what exactly does it return, is there a version with metadata?

### `pub async fn list_paths_with_ids(&self, filter: Option<Filter>) -> LbResult<Vec<(Uuid, String)>>`
- File: `libs/lb/lb-rs/src/service/path.rs`
- Behavior:
  - opens read tx
  - builds lazy tree
  - calls `tree.list_paths(filter, &self.keychain)?`
- Return shape:
  - `Vec<(Uuid, String)>`
  - path string includes trailing `/` for folders via `id_to_path`
- Could replace part of patch implementation:
  - It is already part of the current patch implementation.
  - Useful for bulk indexing when you truly need the whole namespace once.
- Caveats:
  - no metadata attached
  - requires separate `list_metadatas` join to build `HashMap<String, File>`

### `pub fn list_paths(&mut self, filter: Option<Filter>, keychain: &Keychain) -> LbResult<Vec<(Uuid, String)>>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Behavior:
  - optional filtering:
    - `DocumentsOnly`
    - `FoldersOnly`
    - `LeafNodesOnly`
    - `None`
  - filters invisible ids via `is_invisible_id`
  - converts each id to path using `id_to_path`
- Caveats:
  - same as above, ids + paths only

### `pub async fn list_metadatas(&self) -> LbResult<Vec<File>>`
- File: `libs/lb/lb-rs/src/service/file.rs`
- Current companion for `list_paths_with_ids`
- Caveats:
  - metadata only, no path

### Is there a version including file metadata?
- No existing service or model method found that returns:
  - `Vec<(File, String)>`
  - `Vec<(Uuid, String, File)>`
  - `HashMap<String, File>`

---

## 6. Any method combining path resolution + file metadata in one call

### Yes, single-item only: `get_by_path`
- Signature:
  - `pub async fn get_by_path(&self, path: &str) -> LbResult<File>`
- File:
  - `libs/lb/lb-rs/src/service/path.rs`
- Why it matters:
  - this is the only existing service-level API that already combines:
    - path resolution
    - id lookup
    - metadata decryption
- Could replace part of patch implementation:
  - Yes, strongly.

### No bulk version found
- No `list_files_with_paths`
- No `list_paths_with_metadatas`
- No `get_by_paths`

### Custom-helper opportunity
- A new helper in `service/path.rs` or `service/import_export.rs` could do this in one read tx:
  - build tree once
  - for each requested path:
    - `path_to_id`
    - `decrypt`
  - return `Vec<(String, File)>` or a `HashMap<String, File>`
- That would preserve the current "bulk preload" strategy but avoid the awkward `list_metadatas + list_paths_with_ids` join.

---

## 7. Existing `sync` / `merge` helpers in the service layer

### Public sync entrypoints

### `pub async fn calculate_work(&self) -> LbResult<SyncStatus>`
- File: `libs/lb/lb-rs/src/service/sync.rs`
- Purpose:
  - reports pending local + remote work units
- Could replace part of patch implementation:
  - No.

### `pub async fn sync(&self, f: Option<Box<dyn Fn(SyncProgress) + Send>>) -> LbResult<SyncStatus>`
- File: `libs/lb/lb-rs/src/service/sync.rs`
- Purpose:
  - full sync pipeline
- Calls internal helpers:
  - `setup_sync`
  - `prune`
  - `fetch_meta`
  - `populate_pk_cache`
  - `fetch_docs`
  - `merge`
  - `push_meta`
  - `push_docs`
  - `commit_last_synced`
- Could replace part of patch implementation:
  - No. This is remote sync logic, not local import reconciliation.

### Internal merge helper
### `async fn merge(&self, ctx: &mut SyncContext) -> LbResult<()>`
- File: `libs/lb/lb-rs/src/service/sync.rs`
- What it does:
  - computes a merge between base, remote, and local trees
  - uses low-level unvalidated ops:
    - `create_unvalidated`
    - `move_unvalidated`
    - `rename_unvalidated`
    - `delete_unvalidated`
    - `add_share_unvalidated`
    - `delete_share_unvalidated`
    - `update_document_unvalidated`
  - resolves validation failures by retrying with constraints
  - does document-level 3-way merge for text and drawings
- Could replace part of patch implementation:
  - Not directly.
  - It is too specialized to sync state and assumes base/local/remote trees and sync context.
- Caveats:
  - private method
  - deeply tied to remote metadata and docs store semantics

### Conclusion for item 7
- There are service-layer sync/merge helpers, but none are general-purpose import merge helpers.
- The sync merge code is useful as precedent for using lower-level unvalidated tree ops, not as a drop-in replacement.

---

## 8. Full transaction/tree API in `model/path_ops.rs` and `model/tree_like.rs`; any upsert ops?

## `model/path_ops.rs`

### `pub fn path_to_id(&mut self, path: &str, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - path lookup primitive
- Caveats:
  - id only

### `pub fn id_to_path(&mut self, id: &Uuid, keychain: &Keychain) -> LbResult<String>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - reverse path construction

### `pub fn list_paths(&mut self, filter: Option<Filter>, keychain: &Keychain) -> LbResult<Vec<(Uuid, String)>>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - bulk path enumeration

### `pub fn create_link_at_path(&mut self, path: &str, target_id: Uuid, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - not relevant for import patch

### `pub fn create_at_path(&mut self, path: &str, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - creation by path

### `fn create_at_path_helper(&mut self, file_type: FileType, path_components: Vec<&str>, root: &Uuid, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/path_ops.rs`
- Could help:
  - internal create logic only

### Upsert operations in `path_ops.rs`
- None found.
- There is no `maybe_create_at_path`, `get_or_create_at_path`, `upsert_at_path`, or `resolve_or_create`.

## `model/tree_like.rs`

### Trait: `pub trait TreeLike: Sized`
- File: `libs/lb/lb-rs/src/model/tree_like.rs`
- Public API:
  - `fn ids(&self) -> Vec<Uuid>;`
  - `fn maybe_find(&self, id: &Uuid) -> Option<&Self::F>;`
  - `fn find(&self, id: &Uuid) -> LbResult<&Self::F>`
  - `fn maybe_find_parent<F2: FileLike>(&self, file: &F2) -> Option<&Self::F>`
  - `fn find_parent<F2: FileLike>(&self, file: &F2) -> LbResult<&Self::F>`
  - `fn all_files(&self) -> LbResult<Vec<&Self::F>>`
  - `fn stage<Staged>(&self, staged: Staged) -> StagedTree<&Self, Staged>`
  - `fn to_staged<Staged>(self, staged: Staged) -> StagedTree<Self, Staged>`
  - `fn as_lazy(&self) -> LazyTree<&Self>`
  - `fn to_lazy(self) -> LazyTree<Self>`
- Could replace part of patch implementation:
  - only as low-level building blocks for a new helper

### Trait: `pub trait TreeLikeMut: TreeLike`
- File: `libs/lb/lb-rs/src/model/tree_like.rs`
- Public API:
  - `fn insert(&mut self, f: Self::F) -> LbResult<Option<Self::F>>;`
  - `fn remove(&mut self, id: Uuid) -> LbResult<Option<Self::F>>;`
  - `fn clear(&mut self) -> LbResult<()>;`

### Is there any upsert in `tree_like.rs`?
- Sort of, but only at generic container level:
  - `insert` on `Vec<F>` replaces an existing same-id element, otherwise pushes
  - `insert` on `HashMap<Uuid, F>` is standard hash-map insert by id
- Why this does not help patch implementation:
  - this is not semantic file upsert
  - it is keyed by `Uuid`, not path
  - it does not validate parents, names, conflicts, or file types

---

## Other lower-level ops relevant to a possible rewrite

These are not in `path_ops.rs` / `tree_like.rs`, but they matter if patching is rewritten around one custom transaction helper.

### `pub fn create(&mut self, id: Uuid, key: AESKey, parent: &Uuid, name: &str, file_type: FileType, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- Validated create by parent id
- Could help:
  - current `create_file` already wraps this

### `pub fn create_unvalidated(&mut self, id: Uuid, key: AESKey, parent: &Uuid, name: &str, file_type: FileType, keychain: &Keychain) -> LbResult<Uuid>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`
- Used by sync merge only
- Caveat:
  - bypasses validation

### `pub fn update_document(&mut self, id: &Uuid, document: &[u8], keychain: &Keychain) -> LbResult<EncryptedDocument>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`

### `pub fn update_document_unvalidated(&mut self, id: &Uuid, document: &[u8], keychain: &Keychain) -> LbResult<EncryptedDocument>`
- File: `libs/lb/lb-rs/src/model/core_ops.rs`

### `pub fn name_using_links(&mut self, id: &Uuid, keychain: &Keychain) -> LbResult<String>`
- File: `libs/lb/lb-rs/src/model/lazy.rs`

### `pub fn linked_by(&mut self, id: &Uuid) -> LbResult<Option<Uuid>>`
- File: `libs/lb/lb-rs/src/model/lazy.rs`

### `pub fn children(&mut self, id: &Uuid) -> LbResult<HashSet<Uuid>>`
- File: `libs/lb/lb-rs/src/model/lazy.rs`

### `pub fn children_using_links(&mut self, id: &Uuid) -> LbResult<HashSet<Uuid>>`
- File: `libs/lb/lb-rs/src/model/lazy.rs`

### `pub fn descendants_using_links(&mut self, id: &Uuid) -> LbResult<HashSet<Uuid>>`
- File: `libs/lb/lb-rs/src/model/lazy.rs`

These support custom path/metadata traversal but do not provide upsert semantics.

---

## Practical replacement assessment for `import_files_patch`

## Best existing simplification

Use per-item service APIs instead of the upfront bulk `path_map`:

1. Build destination root path once with:
   - `get_path_by_id(dest)`
2. For each item:
   - compute full LB path
   - try `get_by_path(full_path_or_folder_path_with_slash)`
   - if found, verify type and reuse `File`
   - if not found, call `create_at_path(full_path_or_folder_path_with_slash)`
   - for documents, call `write_document(file.id, bytes)`

Why this is simpler:
- removes the brittle join of `list_metadatas()` and `list_paths_with_ids()`
- uses existing service APIs that already encode path resolution
- avoids carrying a cloned `HashMap<String, File>` through recursive calls

Caveats:
- more tree lookups overall
- more transactions overall
- still no true atomic "lookup or create and write" helper

## Best custom-helper opportunity

If you want both simplicity and bulk efficiency, the missing helper is something like:
- `get_by_path_maybe`
- or `list_files_with_paths`
- or `get_or_create_at_path`

None of those exist today.

The highest-value new helper would likely be:
- `pub async fn get_or_create_at_path(&self, path: &str) -> LbResult<(File, bool)>`
or
- `pub async fn list_paths_with_files(&self, filter: Option<Filter>) -> LbResult<Vec<(String, File)>>`

---

## Bottom line

Existing methods that can materially simplify the current patch implementation:
- `get_by_path`
- `create_at_path`
- `write_document`
- `get_path_by_id`

Existing methods that are useful only as lower-level building blocks:
- `path_to_id`
- `id_to_path`
- `list_paths`
- `create`
- `update_document`
- `create_unvalidated`
- `update_document_unvalidated`

Important absences:
- no `get_file_by_path` alias beyond `get_by_path`
- no `maybe_get_by_path`
- no write-by-path API
- no upsert-by-path API
- no bulk path+metadata API
- no import-specific merge helper in the service layer
