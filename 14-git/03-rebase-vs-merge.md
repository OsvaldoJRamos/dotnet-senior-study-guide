# Rebase vs Merge

Both integrate changes from one branch into another, but they produce **different histories** and have **different safety profiles**. Knowing when to use which is one of the most-asked Git questions in senior interviews.

## What `merge` does

`git merge <branch>` creates a **merge commit** that has two parents: the tip of your current branch and the tip of `<branch>`. History is preserved as-is — both branches' commits remain, the merge commit shows where they came together.

```
Before merge:
       A — B — C       (master)
            \
             D — E     (feature)

After git checkout master && git merge feature:
       A — B — C — M   (master)
            \     /
             D — E     (feature)
```

`M` is the merge commit. It has parents `C` and `E`. The commits `D` and `E` keep their original SHAs.

### Fast-forward merge

If `<branch>` is a strict descendant of the current branch, Git skips the merge commit and just moves the pointer forward. No new commit is created.

```
Before:
       A — B — C       (master)
                \
                 D — E (feature)

After merge (no merge commit needed):
       A — B — C — D — E   (master, feature)
```

You can force a real merge commit even for fast-forward cases with `git merge --no-ff <branch>`. Some teams prefer that to make the branch boundary visible in history.

## What `rebase` does

Per the official `git-rebase` docs:

> "Reapply commits on top of another base tip"

`git rebase <upstream>` takes your branch's commits, replays them as **new commits** on top of `<upstream>`, and moves your branch pointer to the new tip. The original commits become garbage-collectable (they still exist in reflog for ~90 days, but aren't reachable from any branch).

```
Before:
       A — B — C        (master)
            \
             D — E      (feature)

After git checkout feature && git rebase master:
       A — B — C        (master)
                \
                 D' — E'  (feature, NEW SHAs)
```

`D'` and `E'` have the same content as `D` and `E` but different parent pointers, so different SHAs.

### Interactive rebase

`git rebase -i <commit>` opens an editor with the commits to be replayed. You can reorder, edit, drop, or combine them:

| Command | Action |
|---|---|
| `pick` | Use commit as-is |
| `reword` | Use commit, but edit the message |
| `edit` | Use commit, but pause for amending |
| `squash` | Combine into the previous commit, opens editor for combined message |
| `fixup` | Like `squash` but discard this commit's message |
| `drop` | Remove the commit entirely |
| `exec <cmd>` | Run a shell command between commits (e.g., `exec npm test`) |
| `break` | Pause; resume with `git rebase --continue` |

This is the foundation of "clean up my branch before the PR" workflows. See [Code Review Workflows](06-code-review-workflows.md) for the `--autosquash` and fixup conventions.

## When to merge

- **Integrating a long-lived branch back to main** — preserve history showing the branch existed.
- **Public/shared branches** — never rewrite history that others have based work on.
- **You want history to reflect *what happened*** — branches as concurrent work streams.

## When to rebase

- **Cleaning up a feature branch before opening a PR** — squash WIP commits into logical units.
- **Updating your feature branch with the latest `main`** — instead of merging `main` into your branch (creates noisy back-merges), rebase your branch onto `main` so your work appears on top of the latest.
- **You want history to reflect *what should be remembered*** — coherent logical commits, not the messy reality of writing them.

## The golden rule

Per the official `git-rebase` docs:

> "Rebasing (or any other form of rewriting) a branch that others have based work on is a bad idea: anyone downstream of it is forced to manually fix their history."

In English: **never rebase commits that have been pushed and that others may have pulled.**

If you rebase a shared branch and force-push, anyone with the old commits in their local clone has to do non-trivial recovery (their `git pull` will create a tangled merge or refuse outright). This is the most common cause of "git destroyed our history" stories.

Safe rule of thumb:

| Branch state | Rebase OK? |
|---|---|
| Local-only (never pushed) | Yes, freely |
| Pushed but only YOU work on it (a feature branch you own) | Yes, but force-push with care |
| Shared with the team / part of trunk | NO |

For "rebase and force-push your own feature branch", prefer `git push --force-with-lease` over `git push --force` — it refuses if someone else pushed in the meantime, preventing accidental overwrites of teammate work.

## Merge vs rebase decision tree

```
Are these commits public AND shared with others?
├─ Yes → MERGE (no history rewriting)
└─ No  → Are you cleaning up before merging to main?
         ├─ Yes → REBASE -i (interactive, squash/fixup)
         └─ No  → Are you syncing your feature branch with latest main?
                  ├─ Yes → REBASE (clean linear history)
                  └─ No  → MERGE
```

Most teams converge on: **rebase locally before opening / updating a PR; merge (or squash-merge) into `main`**. The PR target's preferred merge strategy is set on the platform side.

## "Squash and merge" on the platform

Most platforms (GitHub, GitLab, Azure DevOps) offer "Squash and merge" as a merge strategy. It:

1. Combines all the PR's commits into one.
2. Creates that single commit on `main`.
3. The original branch's commits are NOT preserved on `main`.

This is essentially a managed rebase + merge. Pros: clean linear `main`, one commit per feature. Cons: the original commit-by-commit history is lost (still on the branch / closed PR, but not on `main`).

Whether to use squash-merge is a team policy choice. The trade-off:

| Strategy | History on `main` | When |
|---|---|---|
| Plain merge | Full PR commits + merge commit | When PR commits are individually meaningful |
| Squash merge | One commit per PR | When PR commits are noisy (WIP, fixup, ...) |
| Rebase merge | PR commits replayed onto main, no merge commit | Linear, but each PR commit is on `main` |

The project's own convention matters more than the "right" answer.

## Conflicts

Both merge and rebase can hit conflicts. The difference:

- **Merge conflicts** — resolved once, in the merge commit.
- **Rebase conflicts** — can recur for each commit being replayed (commit `D` conflicts, fix it, then `E` conflicts on top of the resolved tree, fix it again).

Rebase conflicts are more painful when there are many commits. Use `git rebase --abort` to back out, or `--continue` after resolving each conflict step.

`git rerere` (reuse recorded resolution) caches your conflict resolutions so identical conflicts during subsequent rebases are auto-resolved. Enable with `git config --global rerere.enabled true`.

## Senior-interview gotchas

- **Merge preserves history; rebase rewrites it.** New SHAs after rebase.
- **Never rebase shared branches.** The golden rule.
- **Fast-forward merges don't create a merge commit.** Some teams force `--no-ff` for visibility.
- **Interactive rebase (`-i`) is the cleanup tool**: squash, fixup, reorder, drop.
- **Use `--force-with-lease` instead of `--force`** when force-pushing your own feature branch.
- **`git rerere`** caches conflict resolutions for repeated rebases.
- **Squash-merge on the platform** = clean `main` history at the cost of preserving PR commits.

## Useful Links

- [`git-rebase` docs — git-scm.com](https://git-scm.com/docs/git-rebase) — rebase mechanics, interactive mode, golden rule
- [`git-merge` docs — git-scm.com](https://git-scm.com/docs/git-merge) — merge strategies, fast-forward, --no-ff
- [Atlassian — Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing) — illustrated comparison

---

[← Previous: Branching Strategies](02-branching-strategies.md) | [Back to index](README.md) | [Next: Worktree →](04-worktree.md)
