# PES1UG24CS078 — PES-VCS Lab Report
**Name:** Archana Shivakumar  
**SRN:** PES1UG24CS078  
**Repository:** https://github.com/ArchanaShivakumar111/PES1UG24CS078-pes-vcs

---

## Phase 1: Object Storage Foundation

### Implementation
Implemented `object_write` and `object_read` in `object.c`.
- `object_write` builds a header (`"blob <size>\0"`), computes SHA-256 of the full object, shards into `.pes/objects/XX/` directories, and writes atomically using temp-file-then-rename.
- `object_read` reads the file, verifies integrity by recomputing SHA-256, parses the header, and returns the data portion.

### Screenshot 1A — test_objects passing
<img width="602" height="105" alt="1a" src="https://github.com/user-attachments/assets/c78d3037-eaf4-4b97-b0c1-e93e6e12ba4b" />


### Screenshot 1B — Sharded object directory structure
<img width="602" height="66" alt="1b" src="https://github.com/user-attachments/assets/9d021a02-82d3-4407-a86a-8ef4d5d6a2f2" />


---

## Phase 2: Tree Objects

### Implementation
Implemented `tree_from_index` in `tree.c` using a recursive helper `write_tree_level`.
- Handles nested paths by grouping entries sharing the same directory prefix at each depth level.
- Calls `tree_serialize` to convert the Tree struct to binary format, then `object_write` to store it.

### Screenshot 2A — test_tree passing
<img width="602" height="109" alt="2a" src="https://github.com/user-attachments/assets/04e41919-f3fe-45ca-8cd1-368bda6a5f7d" />


### Screenshot 2B — Raw binary tree object (xxd)
<img width="602" height="102" alt="2b" src="https://github.com/user-attachments/assets/3130c1af-db59-4fe6-bd30-20a5fbb4e769" />

---

## Phase 3: The Index (Staging Area)

### Implementation
Implemented `index_load`, `index_save`, and `index_add` in `index.c`.
- `index_load` reads `.pes/index` line by line using `fscanf` with format `%o %s %llu %u %s`.
- `index_save` uses heap-allocated copy of Index to avoid stack overflow, sorts entries by path, writes atomically with temp-file-then-rename.
- `index_add` reads file contents, writes blob to object store, then updates the index entry with mode, hash, mtime, and size.

### Screenshot 3A — pes init + pes add + pes status
<img width="602" height="336" alt="3a" src="https://github.com/user-attachments/assets/f0240396-f70e-4237-b9bf-dcd00053dd15" />

### Screenshot 3B — cat .pes/index
<img width="602" height="41" alt="3b" src="https://github.com/user-attachments/assets/b259fa77-8e4f-48ef-b4d2-d786a47a9229" />

---

## Phase 4: Commits and History

### Implementation
Implemented `commit_create` in `commit.c`.
- Calls `tree_from_index` to build the tree snapshot from staged files.
- Reads current HEAD as parent (skipped for first commit).
- Fills Commit struct with tree, parent, author, timestamp, and message.
- Serializes and writes commit object, then atomically updates HEAD via `head_update`.

### Screenshot 4A — pes log showing three commits
<img width="602" height="283" alt="4a" src="https://github.com/user-attachments/assets/752956ae-1f36-44f7-a517-fc7614491cd2" />

### Screenshot 4B — find .pes -type f showing object growth
<img width="602" height="179" alt="4b" src="https://github.com/user-attachments/assets/b80a56a8-a3c6-47be-bbdd-fb8e6d2035f1" />

### Screenshot 4C — cat .pes/refs/heads/main and cat .pes/HEAD
<img width="602" height="67" alt="4c" src="https://github.com/user-attachments/assets/e35e1230-604c-43cb-aa84-4fc4dfce760a" />

### Final Screenshot — make test-integration
<img width="602" height="483" alt="f1" src="https://github.com/user-attachments/assets/5fce9ef0-df95-4745-8943-b40f2dc17b91" />
<img width="602" height="473" alt="f2" src="https://github.com/user-attachments/assets/3879e95b-788a-4f66-9843-946ade492397" />
<img width="602" height="349" alt="f3" src="https://github.com/user-attachments/assets/d440f259-35de-47e8-a82c-b6b9a763c89b" />



---

## Phase 5: Branching and Checkout (Analysis)

**Q5.1:** To implement `pes checkout <branch>`, the following must happen:
1. Read the target branch ref from `.pes/refs/heads/<branch>` to get the commit hash.
2. Update `.pes/HEAD` to contain `ref: refs/heads/<branch>`.
3. Read the target commit's tree object recursively.
4. Update every file in the working directory to match the target tree — creating new files, deleting removed files, and overwriting modified files.

This operation is complex because it must handle conflicts between the working directory state and the target branch. If a file is modified locally and differs between branches, the checkout must refuse to proceed to avoid data loss.

**Q5.2:** To detect a dirty working directory conflict:
- For each file in the index, compare its hash against the corresponding entry in the HEAD tree and the target branch tree.
- If the file differs between branches AND the working directory version differs from the index (detected via mtime/size mismatch), the file is "dirty" and checkout must be refused.
- This can be done entirely using the index entries and object store without reading file contents — just comparing metadata and stored hashes.

**Q5.3:** In detached HEAD state, HEAD contains a commit hash directly instead of a branch reference. Any new commits are written but no branch pointer is updated, so they become unreachable once you switch away. To recover them, you would need the commit hash (visible in terminal history or via `pes log` before switching), then create a new branch pointing to it: `pes branch recovery-branch <hash>`. Without the hash, the commits are effectively lost unless you scan all objects in the store.

---

## Phase 6: Garbage Collection (Analysis)

**Q6.1:** Algorithm to find and delete unreachable objects:
1. Start from all branch refs in `.pes/refs/heads/`.
2. For each branch, walk the commit chain following parent pointers.
3. For each commit, recursively traverse its tree and all subtrees, collecting every referenced blob and tree hash.
4. Store all reachable hashes in a **HashSet** (O(1) lookup).
5. Scan all files in `.pes/objects/` and delete any whose hash is not in the HashSet.

For a repository with 100,000 commits and 50 branches: assuming each commit references ~50 objects on average, you'd visit approximately 5,000,000 objects during the reachability walk. A HashSet handles this efficiently with O(n) time and space.

**Q6.2:** Race condition between GC and commit:
- A concurrent commit first writes a new blob object, then writes the tree, then the commit, then updates HEAD.
- If GC runs its reachability scan between the blob write and the commit write, the blob exists in the object store but is not yet reachable from any ref.
- GC would mark it as unreachable and delete it, causing the commit to reference a missing object — corruption.

Git avoids this by using a **grace period**: objects newer than 2 weeks are never deleted by GC, regardless of reachability. This gives concurrent operations time to complete before GC considers the objects. Additionally, Git uses lock files to prevent concurrent writes to refs during GC.

---

## File Summary

| File | Description |
|------|-------------|
| `object.c` | Content-addressable object store with SHA-256 hashing |
| `tree.c` | Recursive tree building and binary serialization |
| `index.c` | Staging area with atomic saves and change detection |
| `commit.c` | Commit creation and history traversal |
| `pes.c` | CLI entry point (provided) |
