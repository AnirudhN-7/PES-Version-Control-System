# PES-Version-Control-System
PES-VCS is a mini version of Git, it tracks changes to your files over time so you can save snapshots and view history. 

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Prerequisites & Setup](#prerequisites--setup)
4. [Building the Project](#building-the-project)
5. [Phase 1 — Object Store](#phase-1--object-store)
6. [Phase 2 — Tree Objects](#phase-2--tree-objects)
7. [Phase 3 — Index (Staging Area)](#phase-3--index-staging-area)
8. [Phase 4 — Commits & History](#phase-4--commits--history)
9. [Full Integration Test](#full-integration-test)
10. [Analysis Questions](#analysis-questions)
11. [Submission Checklist](#submission-checklist)

---

## Project Overview

PES-VCS is a minimal, Git-like version control system implemented in C. It stores every piece of data (file contents, directory trees, commits) as a **content-addressable object** named by its SHA-256 hash — the same fundamental design used by Git.

**Core commands implemented:**

| Command | Description |
|---|---|
| `pes init` | Initialise a new repository |
| `pes add <file>...` | Stage files for the next commit |
| `pes status` | Show staged, unstaged, and untracked files |
| `pes commit -m "<msg>"` | Create a snapshot commit from staged files |
| `pes log` | Walk and display commit history |

---

## Repository Structure

```
.
├── pes.h             # Core data structures and constants (PROVIDED)
├── pes.c             # CLI entry point and command dispatch (PROVIDED)
├── object.c          # Content-addressable object store  ← TODO implemented
├── tree.c            # Tree object serialization          ← TODO implemented
├── index.c           # Staging area (index) logic         ← TODO implemented
├── commit.c          # Commit creation and history walk   ← TODO implemented
├── commit.h          # Commit interface header
├── index.h           # Index interface header
├── tree.h            # Tree interface header
├── test_objects.c    # Phase 1 unit tests
├── test_tree.c       # Phase 2 unit tests
├── test_sequence.sh  # End-to-end integration test
└── Makefile
```

---

## Prerequisites & Setup

### Install Dependencies (Ubuntu)

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Set Your Author Name

PES-VCS reads author info from the `PES_AUTHOR` environment variable. Set it before running any commands:

```bash
export PES_AUTHOR="Your Name <PESXUG24CS042>"
```

To make this permanent, add it to your `~/.bashrc`:

```bash
echo 'export PES_AUTHOR="Your Name <PESXUG24CS042>"' >> ~/.bashrc
source ~/.bashrc
```

---

## Building the Project

```bash
# Build only the main pes binary
make

# Build pes + both test binaries
make all

# Remove all compiled files and the .pes directory
make clean
```

---

## Phase 1 — Object Store

**File:** `object.c` — implements `object_write` and `object_read`

### What was implemented

**`object_write`** stores any data as a content-addressed object:
1. Builds a header string: `"<type> <size>\0"` (e.g. `"blob 16\0"`)
2. Concatenates header + data into a full buffer
3. SHA-256 hashes the full buffer to get the object ID
4. Deduplicates — if the hash already exists on disk, returns immediately
5. Creates the shard directory `.pes/objects/XX/` if needed
6. Writes to a temp file, `fsync()`s it, then `rename()`s atomically to the final path
7. `fsync()`s the shard directory to make the rename durable

**`object_read`** retrieves and verifies an object:
1. Builds the path using `object_path()`
2. Reads the entire file into memory
3. **Integrity check**: re-hashes the file and compares against the requested hash — returns `-1` on mismatch
4. Parses the header to extract type and size
5. Returns a `malloc`-ed copy of the data portion; caller must `free()` it

### Running Phase 1 Tests

```bash
make test_objects
./test_objects
```

>
> Expected output:
> ```
> Stored blob with hash: <64-char-hex>
> Object stored at: .pes/objects/XX/YYYYYY...
> PASS: blob storage
> PASS: deduplication
> PASS: integrity check
>
> All Phase 1 tests passed.
> ```

---

```bash
find .pes/objects -type f
```

---

## Phase 2 — Tree Objects

**File:** `tree.c` — implements `tree_from_index` (with `write_tree_level` recursive helper)

### What was implemented

**`tree_from_index`** builds a tree hierarchy from the current index:
1. Loads all staged index entries via `index_load()`
2. Calls `write_tree_level()` recursively
3. Flat entries (no `/` in path) become blob tree entries directly
4. Paths containing `/` are grouped by their first directory component; a sub-tree is written for each group (recursion)
5. Each level calls `tree_serialize()` then `object_write(OBJ_TREE, ...)`

The `tree_serialize()` and `tree_parse()` functions were already provided and sort entries alphabetically by name to guarantee deterministic hashing.

### Running Phase 2 Tests

```bash
make test_tree
./test_tree
```

>
> Expected output:
> ```
> Serialized tree: <N> bytes
> PASS: tree serialize/parse roundtrip
> PASS: tree deterministic serialization
>
> All Phase 2 tests passed.
> ```

---

```bash
# Pick a tree object from the store after running pes commit once
find .pes/objects -type f | head -5
xxd .pes/objects/XX/YYYY... | head -20
```

---

**File:** `index.c` — implements `index_load`, `index_save`, `index_add`

### What was implemented

**`index_load`** reads `.pes/index` line by line:
- Format per line: `<mode-octal> <64-hex-hash> <mtime> <size> <path>`
- Uses `fscanf` with `SCNu64` / `SCNu32` format macros (requires `#include <inttypes.h>`)
- If the file doesn't exist (first run), initialises an empty index — not an error

**`index_save`** writes the index atomically:
1. Sorts a copy of the entries by path using `qsort`
2. Writes to `.pes/index.tmp` using `fprintf` with `PRIu64` / `PRIu32` macros
3. `fflush()` → `fsync()` → `fclose()` → `rename()` to `.pes/index`

**`index_add`** stages a file:
1. Reads file contents into memory
2. Writes them as `OBJ_BLOB` via `object_write()`
3. Stats the file for `mode`, `mtime_sec`, and `size`
4. Updates the existing entry (via `index_find`) or appends a new one
5. Calls `index_save()` to persist atomically

### Running Phase 3 Tests

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index
```

>
> Expected `pes status` output:
> ```
> Staged changes:
>   staged:     file1.txt
>   staged:     file2.txt
>
> Unstaged changes:
>   (nothing to show)
>
> Untracked files:
>   (nothing to show)
> ```

---

```bash
cat .pes/index
```

---

## Phase 4 — Commits & History

**File:** `commit.c` — implements `commit_create`

### What was implemented

**`commit_create`**:
1. Calls `tree_from_index()` to snapshot the staged files as a tree object
2. Calls `head_read()` to get the parent commit hash — silently skips if no prior commits exist (`has_parent = 0`)
3. Fills the `Commit` struct: author from `pes_author()`, timestamp from `time(NULL)`, message from argument
4. Serialises the struct via `commit_serialize()` into the text format:
   ```
   tree <hex>
   parent <hex>        ← omitted for first commit
   author <name> <unix-ts>
   committer <name> <unix-ts>

   <message>
   ```
5. Writes the buffer as `OBJ_COMMIT` via `object_write()`
6. Atomically advances the branch pointer via `head_update()`

### Running Phase 4 Tests

```bash
make pes
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

>
> Expected format:
> ```
> commit <64-char-hex>
> Author: Your Name <PESXUG24CS042>
> Date:   <unix-timestamp>
>
>     Add farewell
>
> commit <64-char-hex>
> Author: Your Name <PESXUG24CS042>
> Date:   <unix-timestamp>
>
>     Add world
>
> commit <64-char-hex>
> ...
> ```

---


```bash
find .pes -type f | sort
```

---

### Screenshot 4C — Reference chain

```bash
cat .pes/HEAD
cat .pes/refs/heads/main
```

---

## Full Integration Test

```bash
make test-integration
```

This runs `test_sequence.sh` which exercises `init`, `add`, `status`, three consecutive commits, `log`, and verifies the object store and reference chain automatically.


---

## Analysis Questions

### Phase 5 — Branching and Checkout

**Q5.1 — How would you implement `pes checkout <branch>`?**

A branch is simply a file at `.pes/refs/heads/<branch>` containing a commit hash. Checking out involves:
1. Reading the target branch file to get its commit hash.
2. Reading the commit object to get its root tree hash.
3. Recursively reading all tree objects and writing every blob back out to the working directory at the correct paths — creating missing directories as needed.
4. Updating `.pes/HEAD` to `ref: refs/heads/<branch>`.
5. Rebuilding `.pes/index` from the checked-out tree so that the staging area matches HEAD.

The complexity comes from step 3: you must perform a full tree diff between the current HEAD tree and the target tree so you only touch changed files. Files present in the current tree but absent from the target must be deleted from disk. If any of those files have been modified in the working directory (i.e. their `mtime` or size diverges from the index), the checkout must be aborted to avoid silently discarding work.

**Q5.2 — Detecting a "dirty working directory" conflict**

For each entry in the current index, stat the file on disk and compare `st_mtime` and `st_size` against the stored `mtime_sec` and `size`. If they differ, the file is locally modified. For each such file, check whether the blob hash stored in the index differs from the blob hash in the target branch's tree for the same path. If both conditions are true — locally modified AND different between branches — refuse the checkout and report the conflicting paths. This requires no re-hashing: the fast metadata comparison (`mtime` + `size`) is sufficient for the dirty-check, matching exactly how Git's "index v2" works.

**Q5.3 — Detached HEAD and recovery**

In detached HEAD state, `.pes/HEAD` contains a raw commit hash instead of `ref: refs/heads/main`. New commits are written to the object store and HEAD is updated to point to them directly. However, no branch file is updated, so when you switch back to a named branch those new commits become unreachable — no reference points to them. To recover: while still in detached HEAD state, run `cat .pes/HEAD` to get the dangling commit hash, then manually create a branch file (`echo <hash> > .pes/refs/heads/recovery`). After that, the commits are reachable again via the new branch. Git provides `git reflog` for this purpose, which logs every HEAD movement including in detached state.

---

### Phase 6 — Garbage Collection

**Q6.1 — Algorithm to find and delete unreachable objects**

Use a **mark-and-sweep** approach:

1. **Root set**: collect all hashes currently referenced by any file in `.pes/refs/` (all branch tips) plus the hash in `.pes/HEAD` if it is detached.
2. **Mark phase**: starting from each root, read the commit object, add its hash to a `reachable` hash-set; follow its `tree` pointer, recursively visit every tree entry (adding tree and blob hashes); follow `parent` pointers to traverse the full history. A hash-set (`unordered_set` or a hash table) is ideal for O(1) lookup.
3. **Sweep phase**: enumerate every file under `.pes/objects/` and delete any whose hash is not in `reachable`.

For 100,000 commits with 50 branches, assume an average of 10 unique objects (blobs + trees) per commit: roughly **1,000,000 object visits** in the mark phase. The sweep phase touches every file in the object store — potentially millions of `stat()` calls, so batching directory reads with `opendir`/`readdir` is important.

**Q6.2 — GC race condition with a concurrent commit**

Consider this interleaving:

1. GC starts its mark phase and records all currently reachable hashes.
2. A concurrent `pes commit` runs `object_write(OBJ_BLOB, ...)`, writing a new blob to disk.
3. GC's mark phase finishes — it did **not** see the new blob because the commit hasn't called `head_update()` yet, so no reference points to it.
4. GC's sweep phase deletes the new blob (it is "unreachable").
5. The concurrent commit calls `object_write(OBJ_TREE, ...)` referencing the now-deleted blob → corrupt repository.

Git avoids this by maintaining a "recent objects grace period": any object younger than a configurable threshold (default 2 weeks, checked via `mtime`) is never deleted, even if unreachable. This ensures that objects written by in-progress operations are never swept before the commit that references them has landed and advanced a branch pointer. Additionally, Git's `gc` acquires a lock file (`.git/gc.pid`) so only one GC runs at a time, and `pack-refs` operations use similar lock files to serialise ref updates.

---

## Submission Checklist

| Phase | Screenshot ID | Command / What to Capture | Done? |
|---|---|---|---|
| 1 | 1A | `./test_objects` — all tests passing | ☐ |
| 1 | 1B | `find .pes/objects -type f` — sharded layout | ☐ |
| 2 | 2A | `./test_tree` — all tests passing | ☐ |
| 2 | 2B | `xxd .pes/objects/XX/YY...` \| `head -20` — raw binary | ☐ |
| 3 | 3A | `pes init` → `pes add` → `pes status` | ☐ |
| 3 | 3B | `cat .pes/index` — text-format index | ☐ |
| 4 | 4A | `pes log` — three commits with hashes and messages | ☐ |
| 4 | 4B | `find .pes -type f \| sort` — object growth | ☐ |
| 4 | 4C | `cat .pes/refs/heads/main` and `cat .pes/HEAD` | ☐ |
| Final | — | `make test-integration` full output | ☐ |

**Code files required:** `object.c`, `tree.c`, `index.c`, `commit.c`

**Analysis questions required:** Q5.1, Q5.2, Q5.3, Q6.1, Q6.2 (answered above)

---

