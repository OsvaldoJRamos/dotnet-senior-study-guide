# Internals

Most senior questions on Git are diagnostic: "what does this command actually do?" Knowing the **object model** turns commands like `rebase`, `cherry-pick`, and `reset` from incantations into clear operations. Everything in Git is a small set of primitives.

## The DAG

A Git repository is a **directed acyclic graph (DAG)** of commits. Each commit points to its parent(s); merge commits have multiple parents. There are no cycles, and no commit is ever modified — operations like `rebase` create *new* commits with different hashes.

```
        A — B — C — D       (master)
                 \
                  E — F     (feature)
```

That's the conceptual mental model. Branches are just **labels** that move along the graph; everything else (history, merges, rebases) is graph-walking.

## Four object types

The Git book describes the four object types stored in `.git/objects/`:

| Type | Stores |
|---|---|
| **blob** | File content (no filename) |
| **tree** | A directory listing — maps names to blobs and other trees |
| **commit** | A snapshot: pointer to a root tree + parent commits + author + message |
| **tag** | A named pointer to a specific commit (annotated tags carry metadata; lightweight tags are just refs) |

Each object is **content-addressed** by SHA-1 (or SHA-256 in newer repos). The hash is computed over the object's content (with a small header), so identical content → identical hash → deduplicated storage.

```bash
$ echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```

The hash is stored as `.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4` (first 2 chars are subdirectory, remaining 38 are filename, content is zlib-compressed).

> **Why this matters:** content-addressing is what makes Git fast and trustworthy. Two files with identical content share storage. A commit's hash *is* a cryptographic fingerprint of everything reachable from it — change any byte anywhere and every descendant hash changes too.

## A commit is a snapshot, not a diff

A common misconception: Git stores diffs between commits. **It doesn't.** A commit references a complete tree (full snapshot of the repo at that point). Diffs are computed on the fly when you run `git diff`, `git show`, etc.

```
commit C
  ├─ tree (root directory snapshot)
  │   ├─ blob src/Program.cs
  │   ├─ tree src/Models/
  │   │   └─ blob Order.cs
  │   └─ blob README.md
  ├─ parent: B
  ├─ author: ...
  └─ message: "Add Order model"
```

This is why Git can switch branches instantly — no replay of diffs, just "point HEAD at this commit and lay out its tree".

The packfile format optimizes storage of similar trees (delta-compression between objects), but conceptually each commit holds a full snapshot.

## Refs

A **ref** is a name → SHA mapping. They live under `.git/refs/`. Common kinds:

| Ref | Path | Purpose |
|---|---|---|
| Branch | `.git/refs/heads/main` | Local branch tip |
| Remote-tracking branch | `.git/refs/remotes/origin/main` | What `origin/main` was at last fetch |
| Tag | `.git/refs/tags/v1.0.0` | Named commit (release marker) |
| HEAD | `.git/HEAD` | Current branch (or detached commit) |
| Stash | `.git/refs/stash` | Top of the stash stack |

A branch is literally just a file containing a SHA-1. `main` doesn't "contain" commits — it points to one, and the one it points to has parent pointers, and so on. That's why creating a branch is O(1) and free of disk cost.

## HEAD

`HEAD` is the **special ref pointing to the currently checked-out branch** (or commit, in detached state). Two forms:

```
# Normal: HEAD is a "symbolic ref" pointing to a branch
$ cat .git/HEAD
ref: refs/heads/main

# Detached: HEAD is a raw SHA
$ cat .git/HEAD
a1b2c3d4...
```

When you make a commit on `main`, Git updates `HEAD` indirectly — `HEAD` keeps pointing at `refs/heads/main`, and `refs/heads/main` is what gets the new SHA.

`HEAD~N` means "N commits back from HEAD via first parent". `HEAD^N` means "the Nth parent of HEAD" (matters for merge commits with two parents). Combine: `HEAD~3^2` = "third commit back, then take its second parent".

## The index (staging area)

Between your working tree and the next commit, there's the **index** (a.k.a. the staging area, a.k.a. the cache). It's a binary file at `.git/index` listing the files that will be in the next commit and their SHAs.

The flow:

```
Working tree           Index             Last commit
   (files)         (staging area)        (snapshot)
       |                  |                  |
       |── git add ──────►|                  |
       |                  |── git commit ───►|
```

- `git add file.cs` → take the working tree's `file.cs`, write a blob, store the blob's SHA in the index.
- `git commit` → write a tree from the index, write a commit pointing to that tree, advance HEAD's branch.
- `git diff` → working tree vs index.
- `git diff --staged` (or `--cached`) → index vs HEAD.

This three-stage model is why `git reset --soft HEAD~1` "uncommits" without losing changes (HEAD moves, index stays), `git reset HEAD~1` (mixed, default) keeps changes in the working tree but unstages them, and `git reset --hard HEAD~1` blows away everything (HEAD, index, and working tree all move together).

## Where commands fit

Knowing the object model makes commands concrete:

- `git checkout <branch>` — read the branch's tree into the working tree, update HEAD.
- `git merge <branch>` — find the merge base (lowest common ancestor), apply diffs from both sides, write a merge commit with two parents.
- `git rebase <upstream>` — replay each of your commits as new commits on top of `<upstream>`. New SHAs because the parent chain changed.
- `git cherry-pick <sha>` — apply the diff of one commit as a new commit on the current branch.
- `git tag -a v1.0` — create an annotated tag object pointing at the current commit.
- `git push` — send objects + refs to a remote.
- `git fetch` — receive objects + remote refs from a remote (no merge).

Every "fancy" Git operation is one of these primitives at heart.

## Why `git gc`, packfiles, and `--prune` exist

Loose objects (one file per object) waste disk space and inode count. Periodically `git gc` packs many objects into a single **packfile** (`.git/objects/pack/*.pack`) with delta compression. After a pack, the loose objects are eligible for pruning.

`git fsck` checks the integrity of the object database — broken refs, dangling objects, missing parents.

You don't usually need to invoke `git gc` manually; Git runs it automatically when certain thresholds are reached.

## Senior-interview gotchas

- **Commits are snapshots, not diffs.** Git stores full trees per commit; diffs are computed on demand.
- **Content addressing via SHA** means identical content is stored once, and any tampering changes hashes globally.
- **A branch is a file with a SHA.** Creating one is free; merging it is just walking the graph.
- **HEAD points to the current branch (or commit when detached).** `HEAD~N` walks first parents; `HEAD^N` picks parents.
- **The index is a real, physical staging area** — files become commits via `add` → `commit`.
- **`git reset` operates on three layers** (HEAD, index, working tree) with `--soft`/`--mixed`/`--hard` controlling which move.
- **Rebase and cherry-pick produce NEW commits** with different SHAs — the originals still exist until garbage-collected.

## Useful Links

- [Pro Git book — Git Internals: Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects) — blob, tree, commit, tag with examples
- [Pro Git book — Git Internals: Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References) — refs, HEAD, branches
- [`git-cat-file` docs](https://git-scm.com/docs/git-cat-file) — inspect raw objects in a repo

---

[Back to index](README.md) | [Next: Branching Strategies →](02-branching-strategies.md)
