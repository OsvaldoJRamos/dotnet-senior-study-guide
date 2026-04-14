# Git Essentials

Every senior developer is expected to be fluent in Git — not just committing, but rewriting history safely, resolving conflicts, and knowing which "undo" to reach for.

## Merge vs Rebase

Both integrate changes from one branch into another, but they produce very different histories.

### Merge

Creates a **merge commit** that joins two histories. Original commits are preserved unchanged.

```
* M   merge feature into main   (merge commit)
|\
| * C   feature
| * B   feature
|/
* A   main
```

```bash
git checkout main
git merge feature
```

- **Preserves real history** — you can see exactly when and how the branch was integrated.
- Non-destructive — no commits are rewritten.
- History becomes harder to read on busy repos ("railroad" graph).

### Rebase

Moves your branch commits to sit **on top of** another branch. The original commits are **replaced** by new ones with the same content but new SHAs.

```
Before rebase:                After `git rebase main`:
* C   feature                 * C'   feature   (new SHA)
* B   feature                 * B'   feature   (new SHA)
| * D main                    * D    main
|/                            * A
* A
```

```bash
git checkout feature
git rebase main
```

- **Linear history** — easier to read, bisect, and blame.
- **Rewrites commits** — the originals are gone; only the new ones remain.
- Requires a force push (`--force-with-lease`) if the branch was already pushed.

### When to use which

| Use case | Choose |
|---|---|
| Integrating a finished feature branch into main | **Merge** (optionally squash merge for a clean history) |
| Keeping your feature branch up to date with main | **Rebase** |
| Cleaning up local commits before pushing/PR | **Interactive rebase** |
| Shared/public branch that others work on | **Never rebase** — use merge |

> **Golden rule**: never rebase commits that other people have already pulled. Rewriting shared history forces everyone else to fix their clones.

## Undoing things

Git has several "undo" commands — picking the right one matters.

| Scenario | Command | Effect |
|---|---|---|
| Fix the last commit message | `git commit --amend` | Rewrites the last commit (new SHA) |
| Add a forgotten file to the last commit | `git add file && git commit --amend --no-edit` | Same commit, with the new file |
| Undo last commit, keep changes staged | `git reset --soft HEAD~1` | Commit is gone, changes remain staged |
| Undo last commit, keep changes unstaged | `git reset --mixed HEAD~1` (default) | Commit is gone, changes in working tree |
| Undo last commit, **throw away** changes | `git reset --hard HEAD~1` | Commit and changes are gone locally |
| Undo a commit that was already pushed | `git revert <sha>` | Creates a new commit that reverses `<sha>` |
| Discard unstaged changes in a file | `git restore <file>` (or `git checkout -- <file>`) | File reverts to HEAD |
| Unstage a file | `git restore --staged <file>` (or `git reset <file>`) | File leaves the index |
| Remove untracked files | `git clean -fd` (preview first with `git clean -nd`) | Deletes untracked files and directories |

> `reset` rewrites history — never use it on commits that have been pushed and pulled by others. Use `revert` instead.

### Amend — common pitfall

`git commit --amend` rewrites the previous commit. If that commit was already pushed, a subsequent push requires `--force-with-lease`. Don't amend commits others are building on.

### Revert — safe undo

```bash
git revert abc1234       # creates a new commit that inverts abc1234
```

Safe for public history because it **adds** a new commit instead of rewriting old ones.

## `git switch` and `git restore` — the modern alternatives to `checkout`

`git checkout` was historically overloaded — it changed branches **and** restored files, which was a common source of confusion and destructive mistakes. Git 2.23 split its responsibilities:

```bash
# Change branches
git switch main                     # replaces: git checkout main
git switch -c feat/pricing          # replaces: git checkout -b feat/pricing

# Restore files
git restore <file>                  # replaces: git checkout -- <file>
git restore --staged <file>         # replaces: git reset <file>
git restore --source=HEAD~3 <file>  # bring a file back from a past commit
```

`checkout` still works — but `switch`/`restore` make your intent explicit and are harder to misuse.

## Interactive rebase — cleaning up history

Rewrite the last N commits: reorder, squash, edit messages, drop commits.

```bash
git rebase -i HEAD~5
```

Editor opens with:

```
pick  a1b2c3 Add login endpoint
pick  d4e5f6 Fix typo
pick  g7h8i9 WIP
pick  j1k2l3 Address review feedback
pick  m4n5o6 Fix another typo
```

Change keywords:

- `pick` → keep as-is
- `reword` → change commit message
- `squash` (`s`) → merge into the previous commit, combining messages
- `fixup` (`f`) → merge into previous commit, **discard** this message
- `drop` → delete the commit
- `edit` → pause the rebase to amend that commit

**Typical cleanup before opening a PR**: squash "fix typo" and "WIP" commits into meaningful units with clear messages.

### `git rebase --onto` — transplanting a branch

Use when you need to move a range of commits onto a different base — e.g., you branched `feat/pricing` off `feat/invoices`, but `feat/invoices` got abandoned and you want your commits to sit on `main` instead.

```bash
# Move commits that are on feat/pricing but not on feat/invoices, onto main
git rebase --onto main feat/invoices feat/pricing
```

Reads as: "Take everything between `feat/invoices` (exclusive) and `feat/pricing` (inclusive) and replay it on top of `main`."

## `git fetch` vs `git pull`

- **`git fetch`** — downloads commits, branches, and tags from the remote into your local **remote-tracking branches** (`origin/main`, etc.) but **does not touch your working branch**. Safe; nothing is merged into your code.
- **`git pull`** — runs `git fetch` **and then** integrates the fetched changes into your current branch. By default this is a merge; with `git pull --rebase`, it's a rebase.

```bash
git fetch origin                    # see what's on the server, no merge
git log HEAD..origin/main           # what's new upstream
git merge origin/main               # apply when you are ready

# Or in one shot
git pull                            # fetch + merge
git pull --rebase                   # fetch + rebase (cleaner linear history)
```

**Rule of thumb**: `fetch` before making decisions (inspect, compare), `pull` when you just want to update a clean branch. Set `pull.rebase = true` globally if you prefer linear history by default.

## Cherry-pick

Apply a specific commit from another branch onto the current one.

```bash
git cherry-pick abc1234
git cherry-pick abc1234..def5678    # a range
```

Common uses:
- Port a bug fix from `main` to a release branch.
- Grab a single commit from a feature branch without merging the whole thing.

If the commit conflicts, resolve the conflict and run `git cherry-pick --continue`.

## Stash

Save uncommitted changes for later without committing them.

```bash
git stash push -m "WIP on pricing"    # save
git stash list                        # see stashes
git stash pop                         # apply and remove
git stash apply stash@{2}             # apply without removing
git stash drop stash@{0}              # discard a stash
```

Useful when you need to switch branches quickly but aren't ready to commit.

## Resolving merge conflicts

Git pauses a merge/rebase when it can't combine two changes automatically. Conflict markers are inserted:

```
<<<<<<< HEAD
public int Price => BasePrice * 2;
=======
public int Price => BasePrice + Surcharge;
>>>>>>> feature/surcharge
```

Process:

1. Open each conflicted file, decide the correct resolution, remove the markers.
2. `git add <file>` to mark as resolved.
3. Continue:
   - After `git merge`: `git commit`
   - After `git rebase`: `git rebase --continue`
   - After `git cherry-pick`: `git cherry-pick --continue`
4. To bail out: `git merge --abort` / `git rebase --abort` / `git cherry-pick --abort`.

Tips:
- Use a GUI/merge tool (VS Code, Beyond Compare) for complex conflicts.
- `git log --merge -p <file>` shows the relevant history.
- Conflicts during rebase can happen **per commit** — you may resolve several in a row.

## Squash merge — combining approaches

```bash
git checkout main
git merge --squash feature
git commit -m "feat: add pricing engine"
```

All commits from `feature` land as a **single** commit on `main`. No merge commit, no per-commit history from the feature branch. Popular for keeping a clean main branch while still using "messy" feature branches locally.

GitHub's "Squash and merge" button does exactly this.

## Force-push safely

Rewriting a branch (rebase, amend, reset) means the remote no longer matches and a normal push is rejected. Force-pushing is required — but do it safely:

```bash
git push --force-with-lease       # ✅ aborts if someone else pushed in the meantime
git push --force                  # ❌ blindly overwrites — can destroy others' work
```

Never force-push to shared branches like `main`/`master`/`develop`. Branch protection rules should forbid it at the server level.

## Recovery — `git reflog`

`reflog` is your safety net. It records every change `HEAD` has made locally, including ones removed by reset/rebase.

```bash
git reflog
# e3f1a2b HEAD@{0}: reset: moving to HEAD~5
# 7a9c4d0 HEAD@{1}: commit: important work
# ...

git checkout 7a9c4d0       # inspect
git reset --hard 7a9c4d0   # recover that state
```

As long as a commit existed locally, `reflog` can find it for ~90 days (default garbage collection window).

## Everyday workflow — fast reference

```bash
# Start a branch
git checkout -b feat/pricing

# Work, commit often
git add . && git commit -m "feat: add pricing model"

# Keep up to date with main
git fetch origin
git rebase origin/main           # or: git merge origin/main

# Clean up commits before pushing
git rebase -i origin/main

# Push (first push sets upstream)
git push -u origin feat/pricing

# Update a pushed branch after amend/rebase
git push --force-with-lease

# After PR merge, clean up
git checkout main
git pull
git branch -d feat/pricing
git fetch -p                     # prune deleted remote branches
```

## Tags — annotated vs lightweight

Tags mark a specific commit (typically a release). Two flavors:

```bash
# Lightweight — just a named pointer to a commit
git tag v1.2.0

# Annotated — full object with tagger, date, message, signable with GPG
git tag -a v1.2.0 -m "Release 1.2.0"
git tag -s v1.2.0 -m "Release 1.2.0"   # signed

# Tags are NOT pushed by default
git push origin v1.2.0
git push origin --tags                  # push all tags
```

**Rule of thumb**: use **annotated tags for releases** — they carry metadata and can be signed. Use lightweight tags for ephemeral markers (CI checkpoints, personal bookmarks).

## Reading diffs and history

- `git diff` — working tree vs index
- `git diff --staged` — index vs HEAD
- `git diff main...feature` — changes introduced on `feature` since it diverged from `main` (three dots in `diff` = "since divergence")
- `git log --oneline --graph --all` — visualize all branches
- `git blame <file>` — who last changed each line
- `git bisect` — binary search commits to find the one that introduced a bug

---

[← Previous: Azure Pipelines](05-azure-pipelines.md) | [Back to index](README.md)
