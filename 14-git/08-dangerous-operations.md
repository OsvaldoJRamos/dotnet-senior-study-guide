# Dangerous Operations

A senior should know which Git commands can lose work, what specifically gets lost, and how to recover. Most "Git destroyed my work" stories are recoverable via reflog if you act within 30-90 days; some aren't. Knowing the difference is part of the bar.

## The dangerous list

In rough order of how often these get people:

| Command | What it can lose | Recoverable? |
|---|---|---|
| `git reset --hard` | Uncommitted changes + commits "ahead" of target | Commits: yes via reflog. Uncommitted: NO. |
| `git checkout -- <file>` / `git restore <file>` | Uncommitted changes to that file | NO. |
| `git clean -fd` | Untracked files + directories | NO. |
| `git push --force` | Commits on the remote that you overwrite | Yes if anyone has them locally; otherwise NO. |
| `git branch -D <branch>` | The deleted branch | Yes via reflog (within window). |
| `git stash drop` / `git stash clear` | The stashed changes | Yes via reflog (within window) if you have the SHA. |
| `git rebase` (botched) | The pre-rebase state of your branch | Yes via reflog. |
| `git filter-repo` / `git filter-branch` | History rewriting at scale | Yes via reflog locally; on remote requires force-push and coordination. |
| `git gc --aggressive --prune=now` | Unreachable commits past the grace window | NO once pruned. |

The recurring theme: **committed work is almost always recoverable**, **uncommitted work is gone the moment a destructive command runs**.

## `git reset --hard`

The most dangerous command in routine use because it's also routinely useful.

```bash
$ git reset --hard HEAD~3   # I want to undo my last 3 commits
$ git reset --hard origin/main   # discard everything I've done locally
```

Both are valid. Both are also the canonical "I just nuked work" stories.

### What gets lost

`reset --hard <target>` does three things:
1. Move HEAD to `<target>`.
2. Reset the **index** to match.
3. Reset the **working tree** to match.

Step 3 is the killer. Any uncommitted changes (staged or unstaged) are **gone**, no reflog, no recovery.

Commits "ahead" of `<target>` are technically also dropped, BUT they live in the reflog for ~90 days (or unreachable-reflog for ~30) before garbage collection. Recovery:

```bash
$ git reset --hard HEAD~3   # 😱
$ git reflog
abc1234 HEAD@{0}: reset: moving to HEAD~3
def5678 HEAD@{1}: commit: Important work
$ git reset --hard def5678   # restored
```

### Safer alternatives

For "undo my last commit but keep the changes":

```bash
$ git reset --soft HEAD~1   # HEAD moves, index and working tree unchanged
$ git reset HEAD~1          # (mixed, default) HEAD + index move, working tree unchanged
```

For "throw away changes to ONE file":

```bash
$ git restore <file>        # modern (Git 2.23+)
$ git checkout -- <file>    # older equivalent
```

(Both lose uncommitted changes to that file unrecoverably.)

For "undo the last destructive operation":

```bash
$ git reset --hard ORIG_HEAD
```

`ORIG_HEAD` is auto-set by merge/reset/rebase to the commit before the operation.

## `git push --force`

Overwrites the remote branch with your local branch, regardless of what's there. If anyone else pushed in the meantime, **their commits disappear from the remote**. Their local clones still have them — but the next pull will create a tangle.

### Why it's bad on shared branches

```bash
# You: rebase your branch and force-push
$ git push --force origin feature/payment

# Teammate Alice was working on the same branch:
$ git pull
# Git: "Your branch and 'origin/feature/payment' have diverged"
# Alice now has to either:
#   - rebase her work onto your new history
#   - merge (creating a cycle of duplicate commits)
#   - reset --hard to your version (losing her work)
```

For your own feature branch (no one else pushes to it), force-push is fine. For shared branches, NEVER.

### `--force-with-lease` is the safer alternative

```bash
$ git push --force-with-lease origin feature/payment
```

This refuses if the remote was updated by anyone since your last fetch. Catches the "I forgot to fetch and overwrote teammate work" case.

Default to `--force-with-lease`. Use plain `--force` only when you've verified no one else is involved.

### Force-push to `main`

**Never.** Branch protection should prevent it; if it doesn't, fix the protection. The single most damaging Git operation in a team setting.

## `git clean -fd`

Removes untracked files and directories. Useful for cleaning up build artifacts that aren't gitignored:

```bash
$ git clean -fd          # remove untracked files and dirs
$ git clean -fdx         # also remove ignored files (dangerous: nukes node_modules, .vs, etc.)
```

The `-x` flag is the trap: it ignores `.gitignore` and removes IDE caches, dependency directories, build outputs — anything not tracked. On a fresh checkout, that's harmless. On a working machine, you might lose hours of "configured my IDE this way" state.

Always `git clean -nd` (`--dry-run`) FIRST to see what will be removed:

```bash
$ git clean -nd
Would remove tmp/
Would remove debug.log
$ git clean -fd
```

## `git stash drop` / `git stash clear`

The stash is a stack. `git stash drop` removes the top entry (or a specific `stash@{N}`). `git stash clear` removes all stashes.

Stashes ARE in the reflog (`refs/stash`), so recovery is possible if you act fast:

```bash
$ git fsck --unreachable | grep commit
unreachable commit abc1234
$ git stash apply abc1234
```

But "act fast" means within the reflog window before `git gc` removes unreachable objects. Stashes you've forgotten about for months are likely unrecoverable.

> Don't use stash for long-term storage. Commit WIP to a branch instead — branches are visible in `git log` and in your IDE's branch list.

## Recovery: the `reflog` workflow

Whenever you panic, check `git reflog` first:

```bash
$ git reflog
HEAD@{0}: reset: moving to HEAD~3       ← the "oh no" event
HEAD@{1}: commit: Add validation        ← what we want back
HEAD@{2}: commit: Refactor pipeline
...

$ git reset --hard HEAD@{1}
```

For a deleted branch:

```bash
$ git reflog --all   # search across all refs
# Find the SHA of the branch tip from before deletion
$ git checkout -b recovered-feature <sha>
```

For unreachable objects after `git gc`:

```bash
$ git fsck --unreachable
unreachable commit abc1234
unreachable blob def5678
```

`fsck` shows objects that exist in `.git/objects/` but aren't reachable from any ref. If they're still around (not yet pruned), you can resurrect them with `git update-ref` or a fresh branch.

## Reflog is local-only

Critical. **The reflog doesn't get pushed**. Recovery via reflog only works:

- On the machine where the destructive op happened.
- Within the 30-90 day expiration window.

If you force-pushed to a shared remote and a teammate cloned/pulled the new state before you noticed the mistake, **their machine doesn't have the reflog of your old work**. You can recover from your reflog and re-push, but anyone who already pulled will get the tangle.

This is why force-push to shared branches is so dangerous: the recovery story becomes a coordination problem, not a technical one.

## `git filter-repo` / `git filter-branch`

Tools for **history rewriting at scale** — removing a sensitive file from all of history, splitting a repo, etc.

`git filter-repo` (modern, Python-based) is preferred over `git filter-branch` (deprecated, slow, error-prone).

```bash
# Remove a file from ALL commits (e.g., leaked secrets)
$ git filter-repo --path secrets.txt --invert-paths
```

This rewrites every commit that touched `secrets.txt`, producing new SHAs throughout history. Consequences:

- All branches need to be force-pushed.
- All collaborators need to re-clone.
- Tags need to be re-signed.

**Coordinate with the team before running filter-repo on shared history.** It's the most disruptive Git operation; treat it like a database schema migration.

## What's truly unrecoverable

- **Uncommitted changes that get reset/checked-out**. Once `git reset --hard` runs, what was in your working tree (uncommitted) is gone.
- **Untracked files removed by `git clean`**. Not in any history.
- **Objects pruned by `git gc`** past the reflog expiration. After `gc` runs and unreachable objects are deleted, they're gone for real (unless you have a backup of `.git/objects/`).

For everything else, reflog within the window is your friend.

## Defensive habits

| Habit | Why |
|---|---|
| `git stash` (or commit-WIP) before risky operations | So `--hard` can't take uncommitted work |
| `--dry-run` (`-n`) first for `clean`, `rm`, etc. | See what would be deleted |
| `--force-with-lease` instead of `--force` | Refuses if remote diverged |
| Branch protection on `main` | Make force-push impossible at the platform layer |
| Push WIP to a remote backup branch | Reflog is local; remote is durable |
| Frequent `git fetch` | Keeps `--force-with-lease` honest |

## Senior-interview gotchas

- **`git reset --hard` deletes uncommitted work irrecoverably.** Commits go to reflog (recoverable for ~30-90 days).
- **Reflog is your safety net** — even after `reset --hard` or branch deletion, commits are usually recoverable.
- **Reflog is local-only.** It doesn't survive a re-clone or another machine.
- **`git clean -fdx`** removes ignored files including `node_modules`, `.vs/`, build caches. Always dry-run first.
- **`--force-with-lease`** beats `--force` for force-push of your own feature branch. Refuses on remote divergence.
- **Force-push to `main` is never OK.** Branch protection should make it impossible.
- **`git filter-repo`** is the modern history-rewrite tool (replaces deprecated `filter-branch`). Coordinate before running on shared history.
- **`ORIG_HEAD` auto-points to pre-operation state** for merge/reset/rebase — the easiest "undo last destructive op".

## Useful Links

- [`git-reset` docs — git-scm.com](https://git-scm.com/docs/git-reset) — `--soft`/`--mixed`/`--hard` semantics
- [`git-reflog` docs — git-scm.com](https://git-scm.com/docs/git-reflog) — selectors, expiration
- [`git-clean` docs — git-scm.com](https://git-scm.com/docs/git-clean) — `-x`, `-d`, `--dry-run`
- [`git filter-repo` — github.com/newren/git-filter-repo](https://github.com/newren/git-filter-repo) — modern history rewriting (replaces `filter-branch`)

---

[← Previous: Hooks](07-hooks.md) | [Back to index](README.md)
