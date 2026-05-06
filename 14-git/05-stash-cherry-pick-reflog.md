# Stash, Cherry-pick, Reflog

Three commands that handle "I have changes I'm not ready to commit", "I want this one commit on a different branch", and "I broke something — can I recover?". A senior should reach for them naturally.

## `git stash` — temporary work shelf

`git stash` saves your working-tree and index changes onto an internal stack and reverts the working tree to a clean state. You can switch branches, work on something else, then come back and restore.

```bash
$ git status
modified: src/Order.cs
modified: src/OrderService.cs

$ git stash push -m "WIP: refactoring order pipeline"
Saved working directory and index state On main: WIP: refactoring order pipeline

$ git status
nothing to commit, working tree clean

$ git stash list
stash@{0}: On main: WIP: refactoring order pipeline

# Later:
$ git stash pop      # apply + remove from stack
# or:
$ git stash apply    # apply but keep on stack
```

### Useful flags

| Command | Effect |
|---|---|
| `git stash push -m "msg"` | The modern form (preferred over bare `git stash`) |
| `git stash push -u` | Include untracked files (default excludes them) |
| `git stash push -a` | Include untracked AND ignored files |
| `git stash push <pathspec>` | Stash only specific files |
| `git stash list` | Show stash stack |
| `git stash show -p stash@{N}` | Show the diff of a specific stash |
| `git stash pop` | Apply top stash + drop |
| `git stash apply stash@{N}` | Apply a specific stash without dropping |
| `git stash drop stash@{N}` | Discard a specific stash |
| `git stash clear` | Discard all stashes |
| `git stash branch <name>` | Create a branch from a stash (useful when stash conflicts on pop) |

### When stash hurts

- **Long-lived stashes** — stashes are invisible from `git log`; if you forget about them, the work is essentially lost. Use `git stash list` periodically.
- **Frequent context switches** — if you're stashing daily, you probably want **worktrees** instead (see [Worktree](04-worktree.md)).
- **Stash conflicts on pop** — if you've changed the same files since stashing, popping conflicts. `git stash branch <name>` creates a branch from the stash to resolve in isolation.

> Modern alternative for "I'm switching branches but have changes": `git switch <branch>` will refuse if your changes conflict, prompting you to commit or stash. For "save and switch", `git stash push` is fine. For "I'll come back to this in a few hours/days", **commit a WIP** instead — it's visible, named, and survives.

## `git cherry-pick` — apply a specific commit elsewhere

`git cherry-pick <commit>` takes the diff of `<commit>` and applies it as a **new commit** on the current branch. Different SHA, same content (assuming no conflicts).

```bash
# I'm on main, want commit abc123 from feature branch
$ git checkout main
$ git cherry-pick abc123
[main def456] Add validation to Order
```

### When to cherry-pick

- **Hotfix on `main`** that needs to also go to `release/v2.x` — cherry-pick the hotfix commit onto the release branch.
- **Recover a single commit** from a deleted/abandoned branch.
- **Backport** a bugfix from `main` to an LTS branch.

### When NOT to cherry-pick

- **Catching up a branch with `main`** — use `git rebase` or `git merge`. Cherry-picking many commits one by one defeats the rebase machinery.
- **Sharing work that requires its full context** — cherry-picking a fix without the prior commits it depends on creates broken history.

### Cherry-pick gotchas

- **`-x` flag** appends `(cherry picked from commit <sha>)` to the message — useful for backporting so reviewers can find the original.
- **Cherry-picking a merge commit** requires `-m <parent-number>` to pick a side; otherwise Git refuses.
- **Conflicts** work like merge conflicts: resolve, `git cherry-pick --continue`, or `--abort`.
- **Same-content cherry-picks aren't deduplicated** — if you cherry-pick `abc123` to `main`, then later merge the source branch, you'll see two commits with the same diff (because they have different SHAs and parents). Git's merge logic handles this gracefully (no double-application of changes), but the history shows both.

## `git reflog` — your safety net

The **reflog** is a per-branch history of where each ref pointed over time. Even commits that aren't reachable from any current branch (because of `reset --hard`, force-push, or rebase) live in the reflog for **~90 days by default** (`gc.reflogExpire`) before garbage collection finally removes them.

This is what makes Git operations recoverable.

```bash
$ git reflog
a1b2c3d HEAD@{0}: rebase: Add Order model
e5f6g7h HEAD@{1}: checkout: moving from feature/payment to main
9i0j1k2 HEAD@{2}: commit: WIP refactor
3l4m5n6 HEAD@{3}: pull: Fast-forward
...
```

`HEAD@{N}` means "where HEAD pointed N moves ago". You can also use time-based selectors:

```bash
$ git reflog HEAD@{yesterday}
$ git reflog HEAD@{2.hours.ago}
```

### Recovering "lost" work

The classic recovery scenarios:

#### Recovering from `git reset --hard`

```bash
$ git reset --hard HEAD~3   # 😱 deleted 3 commits

$ git reflog
abc1234 HEAD@{0}: reset: moving to HEAD~3
def5678 HEAD@{1}: commit: Important work I just blew away

$ git reset --hard def5678   # restore
```

#### Recovering from a deleted branch

```bash
$ git branch -D feature/important   # 😱
$ git reflog
abc1234 HEAD@{0}: ...
def5678 HEAD@{N}: checkout: moving from feature/important to main
# def5678 was the branch tip
$ git checkout -b feature/important def5678   # restore branch
```

#### Recovering from a botched rebase

```bash
$ git rebase main   # 😱 everything's a mess

$ git reflog
# Find the entry just before the rebase started
abc1234 HEAD@{N}: rebase: starting

$ git reset --hard HEAD@{N+1}   # back to pre-rebase state
```

`ORIG_HEAD` is also automatically set by destructive operations (merge, reset, rebase) to the commit BEFORE the operation:

```bash
$ git reset --hard ORIG_HEAD   # undo last destructive op
```

### Reflog is local-only

Critical caveat: **reflog is not pushed**. Only your local clone has it. If you force-push and someone else pulls before you notice the mistake, the reflog on your machine still has the old SHAs but the remote doesn't. Recovery via reflog only works on the machine where the destructive op happened.

This is also why force-push to a shared branch is dangerous — collaborators don't have your reflog, and their clones may have already been overwritten.

### Reflog expiration

Default expiration:

| Config | Default | Meaning |
|---|---|---|
| `gc.reflogExpire` | 90 days | Reachable reflog entries (still on a branch) |
| `gc.reflogExpireUnreachable` | 30 days | Unreachable entries (commit no longer on any branch) |

After expiration, the next `git gc` removes them. So your safety net has a 30-90 day window depending on the type of action.

For long-term archival of work-in-progress, **don't rely on reflog** — push to a remote backup branch.

## Senior-interview gotchas

- **Stash is a stack** — it can have many entries. Don't lose track. Prefer commit-WIP for anything beyond minutes.
- **`git stash branch <name>`** is the right tool when stash pop conflicts.
- **Cherry-pick creates NEW commits** with new SHAs — not references to the original.
- **Cherry-pick a merge commit** requires `-m <parent>` to pick a side.
- **Use `-x`** when cherry-picking for traceability in the message.
- **Reflog is the universal safety net** — even after `reset --hard`, branch deletion, or botched rebase, the old commits live on for 30-90 days.
- **Reflog is local-only.** It doesn't survive a re-clone or move to another machine.
- **`ORIG_HEAD`** points to the pre-operation state for merge/reset/rebase — handy for "undo the last destructive thing".

## Useful Links

- [`git-stash` docs — git-scm.com](https://git-scm.com/docs/git-stash) — push/pop/apply, branch, untracked
- [`git-cherry-pick` docs — git-scm.com](https://git-scm.com/docs/git-cherry-pick) — `-x`, `-m`, conflict handling
- [`git-reflog` docs — git-scm.com](https://git-scm.com/docs/git-reflog) — selectors, expiration

---

[← Previous: Worktree](04-worktree.md) | [Back to index](README.md) | [Next: Code Review Workflows →](06-code-review-workflows.md)
