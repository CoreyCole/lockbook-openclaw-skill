# `--patch` brainstorm

## High-confidence simplifications

### 1. `list_metadatas` + `list_paths_with_ids` looks redundant
`import_files_patch` currently does two full-tree reads and then joins them:
- `list_metadatas()` to build `HashMap<Uuid, File>`
- `list_paths_with_ids(None)` to get `(id, path)`
- then a second map from `path -> File`

That can likely collapse to just `list_paths_with_ids(None)` plus a smaller lookup shape.

Options:
- If patch mode only needs `id` and "is this a folder?", build `HashMap<String, (Uuid, bool)>` directly from `list_paths_with_ids(None)` plus one cheap metadata lookup helper, or better:
- Add a dedicated lb-rs helper that returns `Vec<(File, String)>` or `HashMap<String, File>` in one read transaction.

The second option is the cleanest API. Right now the patch implementation is compensating for the fact that there is no single "list files with resolved paths" helper.

### 2. Prefer a dedicated helper over open-coding path indexing in `import_export.rs`
The path-index setup in `import_files_patch` is a fairly specific tree query. It does not really belong inline in import logic.

A helper like one of these would shrink the method substantially:
- `Lb::files_by_path() -> LbResult<HashMap<String, File>>`
- `Lb::list_metadatas_with_paths() -> LbResult<Vec<(File, String)>>`
- `Lb::get_by_path_maybe(path: &str) -> LbResult<Option<File>>`

The first two preserve the current "snapshot the tree once, then do lookups cheaply" design. The third would allow a simpler recursive implementation without the upfront map, but at different performance cost.

### 3. Passing full `lb_path` through recursion is simpler
`import_file_recursively_patch` currently takes:
- `dest: Uuid`
- `dest_lb_path: &str`
- then recomputes `name`
- then rebuilds `lb_path = format!("{}/{}", dest_lb_path, name)`

That can be simplified by making the recursive parameter the full target lockbook path for the current disk node.

That would remove the repeated `dest_lb_path + name` reconstruction and make the function easier to reason about:
- caller computes the top-level target path once
- recursive calls append child names to the current `lb_path`
- lookup is directly based on `lb_path`

You would still need a parent id for `create_file`, unless you switch creation to `create_at_path`.

### 4. `Arc<HashMap<...>>` is cleaner for top-level concurrency
At the top level, `path_map` is deeply cloned once per source because the outer loop owns a `HashMap<String, File>` and each `async move` takes ownership.

That is unnecessary duplication. `Arc<HashMap<String, File>>` would avoid copying the whole map per source.

Important nuance:
- The inner recursive `let path_map = path_map.clone();` is already cheap today because `path_map` there is `&HashMap<...>`, so that clone is only cloning the reference.
- The expensive clone is only the top-level one in `import_files_patch`.

So the simplification is specifically: wrap the precomputed map once in `Arc`, not because recursion needs it, but because top-level `FuturesUnordered` does.

### 5. The same applies to `dest_lb_path`
`dest_lb_path.clone()` per source is cheap relative to the map clone, but it could still become `Arc<str>` or just be folded away if recursion receives full target paths. Not a big win by itself, but it disappears naturally if you pass full `lb_path`.

## Existing lb-rs methods worth considering

### 6. `create_at_path` could replace `create_file(name, parent, file_type)` in patch mode
There is already a path-based creator:
- `Lb::create_at_path(&str)`

If `import_file_recursively_patch` took the full target path, then the miss case could become:
- document: `create_at_path("/a/b/file.md")`
- folder: `create_at_path("/a/b/folder/")`

That removes the need to thread `dest: Uuid` through the recursive API.

Tradeoff:
- `create_at_path` creates missing intermediate folders. That may be acceptable here, but it is slightly looser semantics than "create under this already-resolved parent id".
- Since patch mode already walks top-down from an existing destination, this is probably fine, but it is a behavior choice worth making explicitly.

### 7. `get_by_path` could replace the upfront path map entirely
There is also:
- `Lb::get_by_path(&str)`

So the patch algorithm could be much smaller:
1. compute current target `lb_path`
2. `get_by_path(lb_path)`
3. if found, validate type and reuse it
4. if not found, create it

That would delete all of the precomputation in `import_files_patch`.

Tradeoff:
- Simpler code, but potentially much slower for large imports because every node would re-traverse the tree.
- If `--patch` is mostly for small targeted updates, this may be an acceptable simplification.
- If performance matters, the current snapshot-map design is still the better shape, but it should probably be hidden behind a helper.

### 8. A hybrid helper would be ideal
The best balance may be a new helper that gives patch mode what it really wants in one call, for example:
- `Lb::files_by_path()`

Then `import_files_patch` keeps O(1) lookup behavior without manually stitching together two APIs.

## Other simplifications

### 9. The type-checking / lookup logic could be factored out
This block is its own unit of behavior:
- normalize path for folder vs document
- check `path_map`
- validate file-vs-folder match
- otherwise create

That could be a helper like:
- `ensure_patch_target(path, parent_id, file_type, path_index) -> LbResult<File>`

That would make the recursive function mostly about:
- status updates
- reading/writing document bytes
- descending into children

### 10. Normalizing folder lookup keys deserves a helper
This code relies on the convention that folders are looked up with a trailing slash:
- document key: `lb_path`
- folder key: `format!("{lb_path}/")`

That convention is easy to get subtly wrong. A tiny helper like `path_lookup_key(lb_path, file_type)` would reduce duplication and make intent clearer.

### 11. Top-level concurrency may not buy much in patch mode
`import_files` and `import_files_patch` both use `FuturesUnordered` over sources. If `sources` is usually a single path from the CLI, the concurrency machinery is mostly overhead.

This is not necessarily worth changing, but it does make the ownership story more complicated (`path_map` cloning, `dest_lb_path` cloning). If patch mode is only ever called with one or a few sources, a simple sequential loop could make the implementation smaller.

### 12. CLI wiring is already minimal; only naming/message polish stands out
`clients/cli/src/imex.rs` and `main.rs` are pretty lean already.

Minor simplifications only:
- `copy(patch: bool, ...)` could dispatch to a function pointer or local async closure instead of an `if patch { ... } else { ... }`, but that is stylistic.
- The progress string switches between `patching` and `importing`; that is fine as-is.
- The main CLI flag description is short and clear enough.

I do not see a meaningful CLI-side simplification opportunity beyond keeping the service API cleaner so the CLI does not need to care which import variant it is calling.

## Recommended direction

If the goal is to simplify without regressing performance much, the cleanest path looks like:
1. Add an lb-rs helper that builds a path-index in one place (`files_by_path` or equivalent).
2. Change recursive patch import to pass full target `lb_path`.
3. Wrap the path index in `Arc` so top-level concurrent imports do not deep-clone it.
4. Factor the lookup/type-check/create block into a helper.

If the goal is to minimize code even more and performance is secondary, then:
1. Drop the upfront path map entirely.
2. Use `get_by_path` + `create_at_path` inside recursion.
3. Pass only full `lb_path` through recursion.

That version would be much shorter, but it would trade some efficiency for readability.
