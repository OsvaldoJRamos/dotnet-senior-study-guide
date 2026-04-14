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
| Remove untracked files | `git clean -fd` | Deletes untracked files and directories |

> `reset` rewrites history — never use it on commits that have been pushed and pulled by others. Use `revert` instead.

### Amend — common pitfall

`git commit --amend` rewrites the previous commit. If that commit was already pushed, a subsequent push requires `--force-with-lease`. Don't amend commits others are building on.

### Revert — safe undo

```bash
git revert abc1234       # creates a new commit that inverts abc1234
```

Safe for public history because it **adds** a new commit instead of rewriting old ones.

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

## What to read in an execution plan, err, a diff

- `git diff` — working tree vs index
- `git diff --staged` — index vs HEAD
- `git diff main...feature` — commits on `feature` not in `main` (three dots = since divergence)
- `git log --oneline --graph --all` — visualize all branches
- `git blame <file>` — who last changed each line
- `git bisect` — binary search commits to find the one that introduced a bug

---

[← Previous: Azure Pipelines](05-azure-pipelines.md) | [Back to index](README.md)
