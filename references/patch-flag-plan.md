# Implementation Plan: `--patch` flag for `lockbook copy`

## Summary

Add a `--patch` flag to the `lockbook copy` CLI command that changes import semantics from "always create new (auto-rename on conflict)" to "update existing files in-place, create only if missing."

---

## Files to Change

### 1. `libs/lb/lb-rs/src/service/import_export.rs`

**Goal:** Add a parallel `import_files_patch` / `import_file_recursively_patch` path that uses "find-or-create" semantics.

#### New public method: `import_files_patch`

```rust
#[instrument(level = "debug", skip(self, update_status), err(Debug))]
pub async fn import_files_patch<F: Fn(ImportStatus)>(
    &self, sources: &[PathBuf], dest: Uuid, update_status: &F,
) -> LbResult<()> {
    update_status(ImportStatus::CalculatedTotal(get_total_child_count(sources)?));

    let parent = self.get_file_by_id(dest).await?;
    if !parent.is_folder() {
        return Err(LbErrKind::Validation(ValidationFailure::NonFolderWithChildren(dest)))?;
    }

    let import_file_futures = FuturesUnordered::new();

    for source in sources {
        let lb = self.clone();
        import_file_futures.push(async move {
            lb.import_file_recursively_patch(source, dest, update_status).await
        });
    }

    import_file_futures
        .collect::<Vec<LbResult<()>>>()
        .await
        .into_iter()
        .collect::<LbResult<()>>()
}
```

#### New private method: `import_file_recursively_patch`

This replaces the rename-loop with a "find existing child by name, or create" approach:

```rust
async fn import_file_recursively_patch<F: Fn(ImportStatus)>(
    &self, disk_path: &Path, dest: Uuid, update_status: &F,
) -> LbResult<()> {
    update_status(ImportStatus::StartingItem(format!("{}", disk_path.display())));

    if !disk_path.exists() {
        return Err(LbErrKind::DiskPathInvalid.into());
    }

    let name = disk_path
        .file_name()
        .and_then(|name| name.to_str())
        .ok_or(LbErrKind::DiskPathInvalid)?
        .to_string();

    let file_type = if disk_path.is_file() { FileType::Document } else { FileType::Folder };

    // Patch semantics: look for an existing child with same name
    let existing = self.get_children(&dest).await?
        .into_iter()
        .find(|f| f.name == name);

    let file = match existing {
        Some(f) => {
            // Validate type match: can't patch a folder over a document or vice versa
            if f.is_folder() != disk_path.is_dir() {
                return Err(LbErrKind::Validation(
                    ValidationFailure::FileNotDocument
                ).into());
            }
            f
        }
        None => {
            // No existing file — create as usual
            self.create_file(&name, &dest, file_type).await?
        }
    };

    match file_type {
        FileType::Document => {
            let content = fs::read(disk_path).map_err(LbErr::from)?;
            self.write_document(file.id, content.as_slice()).await?;
            update_status(ImportStatus::FinishedItem(file));
        }
        FileType::Folder => {
            let id = file.id;
            update_status(ImportStatus::FinishedItem(file));

            let disk_children = fs::read_dir(disk_path).map_err(LbErr::from)?;
            let import_file_futures = FuturesUnordered::new();

            for disk_child in disk_children {
                let child_path = disk_child.map_err(LbErr::from)?.path();
                let lb = self.clone();
                import_file_futures.push(async move {
                    lb.import_file_recursively_patch(&child_path, id, update_status).await
                });
            }

            import_file_futures
                .collect::<Vec<LbResult<()>>>()
                .await
                .into_iter()
                .collect::<LbResult<()>>()?;
        }
        FileType::Link { .. } => {
            error!("links should not be interpreted!")
        }
    }

    Ok(())
}
```

**Key differences from `import_file_recursively`:**
- Instead of the `loop` with `create_file` + rename retry, we call `get_children(&dest)` and search by name.
- If found → reuse the existing `File` (for documents this means `write_document` overwrites content).
- If not found → `create_file` as before (no rename loop needed since there's no conflict).
- Type mismatch (file on disk is a dir, lockbook has a document with same name, or vice versa) returns an error.

#### Alternative: Single method with a `patch: bool` parameter

Instead of two separate methods, you could refactor `import_files` to accept a `patch: bool` parameter and branch internally. This is simpler API-wise but makes the function signature change, requiring updates to all callers. The two-method approach is cleaner for backwards compatibility.

**Recommended:** Two separate methods. Less churn, callers of the original API are unaffected.

---

### 2. `clients/cli/src/main.rs`

**Goal:** Add `--patch` flag to the `copy` subcommand and pass it to the handler.

#### Changes to the `copy` command definition (~line 76-82):

```rust
// Before:
.subcommand(
    Command::name("copy").description("import files from your file system into lockbook")
        .input(Arg::<PathBuf>::name("disk-path").description("path of file on disk"))
        .input(Arg::<FileInput>::name("dest")
               .description("the path or id of a folder within lockbook to place the file.")
               .completor(|prompt| input::file_completor(prompt, Some(Filter::FoldersOnly))))
        .handler(|disk, parent| imex::copy(disk.get(), parent.get()))
)

// After:
.subcommand(
    Command::name("copy").description("import files from your file system into lockbook")
        .input(Flag::bool("patch").description("update existing files instead of creating duplicates"))
        .input(Arg::<PathBuf>::name("disk-path").description("path of file on disk"))
        .input(Arg::<FileInput>::name("dest")
               .description("the path or id of a folder within lockbook to place the file.")
               .completor(|prompt| input::file_completor(prompt, Some(Filter::FoldersOnly))))
        .handler(|patch, disk, parent| imex::copy(patch.get(), disk.get(), parent.get()))
)
```

Note: `Flag` inputs are added before `Arg` inputs based on the existing pattern (see `delete`, `list` commands). The handler closure parameters match the declaration order.

---

### 3. `clients/cli/src/imex.rs`

**Goal:** Accept the `patch` flag and call the appropriate import method.

```rust
// Before:
pub async fn copy(disk: PathBuf, parent: FileInput) -> CliResult<()> {
    let lb = &core().await?;
    ensure_account_and_root(lb).await?;

    let parent = parent.find(lb).await?.id;

    let total = Cell::new(0);
    let nth_file = Cell::new(0);
    let update_status = move |status: ImportStatus| match status {
        ImportStatus::CalculatedTotal(n_files) => total.set(n_files),
        ImportStatus::StartingItem(disk_path) => {
            nth_file.set(nth_file.get() + 1);
            print!("({}/{}) importing: {}... ", nth_file.get(), total.get(), disk_path);
            io::stdout().flush().unwrap();
        }
        ImportStatus::FinishedItem(_meta) => println!("done."),
    };

    lb.import_files(&[disk], parent, &update_status).await?;

    Ok(())
}

// After:
pub async fn copy(patch: bool, disk: PathBuf, parent: FileInput) -> CliResult<()> {
    let lb = &core().await?;
    ensure_account_and_root(lb).await?;

    let parent = parent.find(lb).await?.id;

    let total = Cell::new(0);
    let nth_file = Cell::new(0);
    let update_status = move |status: ImportStatus| match status {
        ImportStatus::CalculatedTotal(n_files) => total.set(n_files),
        ImportStatus::StartingItem(disk_path) => {
            nth_file.set(nth_file.get() + 1);
            let mode = if patch { "patching" } else { "importing" };
            print!("({}/{}) {}: {}... ", nth_file.get(), total.get(), mode, disk_path);
            io::stdout().flush().unwrap();
        }
        ImportStatus::FinishedItem(_meta) => println!("done."),
    };

    if patch {
        lb.import_files_patch(&[disk], parent, &update_status).await?;
    } else {
        lb.import_files(&[disk], parent, &update_status).await?;
    }

    Ok(())
}
```

---

### 4. `libs/lb/lb-rs/tests/import_export_file_tests.rs`

**Goal:** Add tests for patch semantics.

#### New tests to add:

```rust
#[tokio::test]
async fn import_patch_updates_existing_document() {
    let core = test_core_with_account().await;
    let tmp = tempfile::tempdir().unwrap();
    let root = core.root().await.unwrap();

    let name = "test.txt";
    let doc_path = tmp.path().join(name);

    // First import: creates the file
    std::fs::write(&doc_path, b"version 1").unwrap();
    core.import_files(&[doc_path.clone()], root.id, &|_| {}).await.unwrap();

    let file = core.get_by_path(&format!("/{name}")).await.unwrap();
    let content1 = core.read_document(file.id, false).await.unwrap();
    assert_eq!(content1, b"version 1");

    // Patch import: overwrites the content
    std::fs::write(&doc_path, b"version 2").unwrap();
    core.import_files_patch(&[doc_path], root.id, &|_| {}).await.unwrap();

    // Should still be only one file with this name
    let children = core.get_children(&root.id).await.unwrap();
    let matching: Vec<_> = children.iter().filter(|f| f.name == name).collect();
    assert_eq!(matching.len(), 1);

    // Content should be updated
    let content2 = core.read_document(matching[0].id, false).await.unwrap();
    assert_eq!(content2, b"version 2");
}

#[tokio::test]
async fn import_patch_creates_when_missing() {
    let core = test_core_with_account().await;
    let tmp = tempfile::tempdir().unwrap();
    let root = core.root().await.unwrap();

    let name = Uuid::new_v4().to_string();
    let doc_path = tmp.path().join(&name);
    std::fs::write(&doc_path, b"new file").unwrap();

    core.import_files_patch(&[doc_path], root.id, &|_| {}).await.unwrap();

    let file = core.get_by_path(&format!("/{name}")).await.unwrap();
    let content = core.read_document(file.id, false).await.unwrap();
    assert_eq!(content, b"new file");
}

#[tokio::test]
async fn import_patch_recursive_folder() {
    let core = test_core_with_account().await;
    let tmp = tempfile::tempdir().unwrap();
    let root = core.root().await.unwrap();

    let folder_name = "myfolder";
    let folder_path = tmp.path().join(folder_name);
    std::fs::create_dir(&folder_path).unwrap();
    std::fs::write(folder_path.join("a.txt"), b"aaa").unwrap();
    std::fs::write(folder_path.join("b.txt"), b"bbb").unwrap();

    // First import
    core.import_files(&[folder_path.clone()], root.id, &|_| {}).await.unwrap();

    // Modify a.txt, add c.txt on disk
    std::fs::write(folder_path.join("a.txt"), b"aaa-updated").unwrap();
    std::fs::write(folder_path.join("c.txt"), b"ccc").unwrap();

    // Patch import
    core.import_files_patch(&[folder_path], root.id, &|_| {}).await.unwrap();

    let lb_folder = core.get_by_path(&format!("/{folder_name}")).await.unwrap();
    let children = core.get_children(&lb_folder.id).await.unwrap();

    // Should have 3 children: a.txt (updated), b.txt (unchanged), c.txt (new)
    assert_eq!(children.len(), 3);

    let a = children.iter().find(|f| f.name == "a.txt").unwrap();
    let content = core.read_document(a.id, false).await.unwrap();
    assert_eq!(content, b"aaa-updated");

    let c = children.iter().find(|f| f.name == "c.txt").unwrap();
    let content = core.read_document(c.id, false).await.unwrap();
    assert_eq!(content, b"ccc");

    // Should NOT have duplicate folder
    let root_children = core.get_children(&root.id).await.unwrap();
    let matching_folders: Vec<_> = root_children.iter().filter(|f| f.name == folder_name).collect();
    assert_eq!(matching_folders.len(), 1);
}
```

---

## Edge Cases and Considerations

### 1. Type mismatch (folder on disk vs document in lockbook, or vice versa)
If a file named `foo` exists as a document in lockbook but `foo/` is a directory on disk (or vice versa), the patch import should return an error. The plan handles this with a type check before reuse.

**Relevant validation failure to use:** `ValidationFailure::FileNotDocument` (for the case where we expect a document but find a folder). May need to check which validation variants exist and potentially use a more appropriate one or add a new one. Fallback: use `LbErrKind::Unexpected` with a descriptive message.

### 2. Concurrent modifications
`get_children` + `write_document` is not atomic, but this matches the existing import pattern (which also does `create_file` then `write_document` non-atomically). Since the CLI is single-user, this is acceptable.

### 3. Files that exist in lockbook but NOT on disk
Patch mode does **not** delete lockbook files that are missing from disk. It's purely additive/update — not a full sync/mirror. This keeps the behavior safe and predictable.

### 4. Symlinks on disk
The existing code uses `disk_path.is_file()` / `disk_path.is_dir()` which follow symlinks. The patch code inherits this behavior — no change needed.

### 5. Empty documents
`write_document` with empty content on an existing file is valid and already handled by the documents service. No special case needed.

### 6. Binary files
`fs::read` returns raw bytes; `write_document` accepts `&[u8]`. Binary files work the same as text — no special handling.

### 7. Other callers of `import_files`
Searching the codebase shows `import_files` is called from:
- `clients/cli/src/imex.rs` (the CLI — we're modifying this)
- `clients/cli/src/migrate.rs` (Bear migration — unchanged, doesn't need patch)
- Various test files (unchanged)
- FFI layers (`lb-c`, `lb-java`, Swift bindings)

The FFI layers (`lb-c/src/lib.rs`, `lb-java/src/lib.rs`, Swift) would need new bindings if we want patch semantics available outside the CLI. **For this initial implementation, only the CLI is in scope.** FFI bindings can be added later.

---

## No New API Methods Needed Beyond `import_files_patch`

The implementation relies entirely on existing `Lb` methods:
- `get_file_by_id` — validate destination
- `get_children` — find existing files by name
- `create_file` — create when missing
- `write_document` — overwrite content of existing documents

No changes to the core data model, tree operations, or sync logic.

---

## Optimization: Build path map upfront with `list_metadatas`

The original plan calls `get_children()` at every directory level during the recursive walk — O(N) round trips for N directories.

**Better approach:** call `list_metadatas()` once before the walk, build a `HashMap<String, File>` keyed by full lockbook path, then do O(1) lookups during the recursive import.

### `list_metadatas` signature

```rust
// libs/lb/lb-rs/src/service/file.rs:123
pub async fn list_metadatas(&self) -> LbResult<Vec<File>>
```

Returns a flat `Vec<File>`. Each `File` has `id`, `parent` (UUID), `name`, `file_type` — no pre-computed path.

### Build the path map

```rust
async fn build_path_map(&self) -> LbResult<HashMap<String, File>> {
    let all_files = self.list_metadatas().await?;
    let by_id: HashMap<Uuid, &File> = all_files.iter().map(|f| (f.id, f)).collect();

    let mut path_map = HashMap::new();
    for file in &all_files {
        let mut parts = vec![file.name.clone()];
        let mut cur = file;
        // Walk parent chain to root
        while let Some(parent) = by_id.get(&cur.parent) {
            parts.push(parent.name.clone());
            cur = parent;
        }
        parts.reverse();
        let path = format!("/{}", parts.join("/"));
        path_map.insert(path, file.clone());
    }
    path_map
}
```

### Updated `import_file_recursively_patch`

Replace `get_children()` lookup with path map lookup:

```rust
async fn import_file_recursively_patch<F: Fn(ImportStatus)>(
    &self,
    disk_path: &Path,
    dest: Uuid,
    dest_lb_path: &str,        // ← pass current lockbook path during recursion
    path_map: &HashMap<String, File>,
    update_status: &F,
) -> LbResult<()> {
    let name = disk_path.file_name().and_then(|n| n.to_str())
        .ok_or(LbErrKind::DiskPathInvalid)?.to_string();
    let lb_path = format!("{}/{}", dest_lb_path.trim_end_matches('/'), name);

    let file_type = if disk_path.is_file() { FileType::Document } else { FileType::Folder };

    let file = match path_map.get(&lb_path) {
        Some(existing) => {
            if existing.is_folder() != disk_path.is_dir() {
                return Err(LbErrKind::Validation(ValidationFailure::FileNotDocument).into());
            }
            existing.clone()
        }
        None => self.create_file(&name, &dest, file_type).await?,
    };

    // ... rest of write_document / recurse logic unchanged
}
```

`import_files_patch` calls `build_path_map()` once, then passes `path_map` and the current lockbook path down through the recursion.

## Implementation Order

1. Add `import_files_patch` and `import_file_recursively_patch` to `import_export.rs`
2. Update `clients/cli/src/main.rs` to add `--patch` flag
3. Update `clients/cli/src/imex.rs` to accept and use the flag
4. Add tests in `import_export_file_tests.rs`
5. Verify with `cargo build` and `cargo test`
