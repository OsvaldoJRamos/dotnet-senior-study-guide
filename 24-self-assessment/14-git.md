# Git

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the four Git object types and how do they fit together?

<details>
<summary>Reveal answer</summary>

| Object | Stores |
|---|---|
| **blob** | File content (no filename) |
| **tree** | A directory listing — maps names to blobs and other trees |
| **commit** | Snapshot: pointer to a root tree + parent commits + author + message |
| **tag** | Annotated tags carry metadata pointing to a commit; lightweight tags are just refs |

Git is **content-addressed** — each object's identity is the SHA-1 (or SHA-256 in newer repos) of its content. Two files with identical content share storage; one byte changed produces a completely different hash everywhere up the chain.

Crucially: **commits are full snapshots, not diffs**. Each commit references a complete tree of the repo state. Diffs are computed on demand by `git diff`. Git's packfile format does delta-compress similar trees on disk, but the conceptual model is "snapshot per commit".

A **branch is just a file** at `.git/refs/heads/<name>` containing a SHA. That's why creating one is O(1).

Deep dive: [Internals](../14-git/01-internals.md)

</details>

---

### 2. What is the difference between `git rebase` and `git merge`?

<details>
<summary>Reveal answer</summary>

**Merge** preserves history with both branches' commits intact and a merge commit at the join point (two parents).

**Rebase** rewrites history by replaying your commits **as new commits** on top of the target branch. New SHAs because parent pointers change.

Per the official `git-rebase` docs: *"Reapply commits on top of another base tip"*.

Decision rule:

- **Use merge** when integrating a long-lived branch back to main, or when working with public/shared branches.
- **Use rebase** when cleaning up local feature branches before opening a PR, or syncing your branch with the latest `main`.

The **golden rule**: *"Rebasing (or any other form of rewriting) a branch that others have based work on is a bad idea: anyone downstream of it is forced to manually fix their history."*

Never rebase shared branches. For your own feature branch, rebase + `--force-with-lease` is fine.

Deep dive: [Rebase vs Merge](../14-git/03-rebase-vs-merge.md)

</details>

---

### 3. Why use `git worktree` instead of `git stash` for parallel work?

<details>
<summary>Reveal answer</summary>

`git worktree` lets a single repository have **multiple checked-out branches simultaneously**, each in its own directory. Per the official docs: *"Manage multiple working trees attached to the same repository. A git repository can support multiple working trees, allowing you to check out more than one branch at a time."*

The canonical scenario from the docs: *"You are in the middle of a refactoring session and your boss comes in and demands that you fix something immediately... Instead, you create a temporary linked worktree to make the emergency fix, remove it when done, and then resume your earlier refactoring session."*

Versus stash:

- **Stash** dumps changes onto a stack, requires switching back, and is invisible from `git log`.
- **Worktree** keeps your work in place — open the editor in another directory, make commits there, never touch the original.

Versus re-cloning:

- A worktree shares `.git/objects` and refs with the parent — adding one is essentially free disk-wise.
- A clone duplicates the entire object database.

Key constraint: **a branch can only be checked out in ONE worktree at a time**. The official docs: *"By default, `add` refuses to create a new worktree when _<commit-ish>_ is a branch name and is already checked out by another worktree."*

Use cases beyond hotfixes: side-by-side branch comparison, long-running tests on one branch while developing on another, multi-version maintenance (LTS branches in their own worktrees).

Deep dive: [Worktree](../14-git/04-worktree.md)

</details>

---

### 4. How do you recover from `git reset --hard` that nuked work?

<details>
<summary>Reveal answer</summary>

**Committed work** is recoverable via `git reflog` for ~30-90 days. **Uncommitted work is gone** — `reset --hard` blows away the working tree and index unrecoverably.

Recovery for committed work:

```bash
$ git reset --hard HEAD~3   # 😱
$ git reflog
abc1234 HEAD@{0}: reset: moving to HEAD~3
def5678 HEAD@{1}: commit: Important work
$ git reset --hard def5678   # restored
```

The reflog records every move of `HEAD` (and per-branch reflogs record per-branch moves). Use `git reflog --all` to search across all refs (e.g., for a deleted branch).

`ORIG_HEAD` is auto-set by destructive ops (merge, reset, rebase) to the pre-operation state — `git reset --hard ORIG_HEAD` is the easiest "undo last destructive op".

Two critical caveats:

1. **The reflog is local-only.** It doesn't survive `re-clone` or another machine.
2. **Default expiration is 30-90 days** (`gc.reflogExpire` 90 days for reachable, `gc.reflogExpireUnreachable` 30 days). After `git gc` runs past the window, unreachable objects are pruned and gone.

For long-term archival of work-in-progress, push to a backup remote branch — don't rely on reflog.

Deep dive: [Stash, Cherry-pick, Reflog](../14-git/05-stash-cherry-pick-reflog.md), [Dangerous Operations](../14-git/08-dangerous-operations.md)

</details>

---

### 5. What's the right way to update a feature branch with the latest `main`?

<details>
<summary>Reveal answer</summary>

For a branch you own and haven't shared, **rebase**:

```bash
$ git fetch origin
$ git rebase origin/main
$ git push --force-with-lease
```

This replays your commits on top of the latest `main`, producing a clean linear history. Use `--force-with-lease` (not plain `--force`) — it refuses if someone else pushed to your branch in the meantime, preventing accidental overwrites.

For a shared branch (multiple people commit to it), **merge `main` in**:

```bash
$ git merge origin/main
```

This avoids rewriting history that others have based work on. The trade-off is "back-merge" commits in history.

The wrong default: `git pull` without thinking — by default, `git pull` does fetch + merge, which creates noisy merge commits even on your private feature branch. Configure `pull.rebase = true` to default to rebase, or always `git fetch && git rebase` explicitly.

Deep dive: [Rebase vs Merge](../14-git/03-rebase-vs-merge.md)

</details>

---

### 6. What's the difference between `git push --force` and `git push --force-with-lease`?

<details>
<summary>Reveal answer</summary>

`--force` overwrites the remote branch unconditionally with your local branch.

`--force-with-lease` overwrites **only if the remote hasn't moved since your last fetch**. If a teammate pushed to the same branch in the meantime, the push is refused.

```bash
# UNSAFE: overwrites whatever's there
$ git push --force origin feature/payment

# SAFE: refuses if remote diverged
$ git push --force-with-lease origin feature/payment
```

Use `--force-with-lease` by default. Plain `--force` is only safe when you've verified no one else is involved with the branch.

**Force-pushing to `main` is never OK.** Branch protection should make it impossible at the platform level. Force-pushing to a shared feature branch where teammates commit is also dangerous — their local clones get tangled and recovery is a coordination problem.

For your own private feature branch (rebase + force-push to clean up before merge), `--force-with-lease` is fine and standard.

Deep dive: [Dangerous Operations](../14-git/08-dangerous-operations.md), [Code Review Workflows](../14-git/06-code-review-workflows.md)

</details>

---

### 7. What does `git commit --fixup=<sha>` do, and how does `--autosquash` use it?

<details>
<summary>Reveal answer</summary>

`git commit --fixup=<sha>` creates a commit whose subject is `fixup! <original commit's subject>`. It's a marked commit meant to be combined with `<sha>` later via interactive rebase.

```bash
$ git commit --fixup=abc1234   # creates: "fixup! Add Order validation"
```

Then before merging:

```bash
$ git rebase -i --autosquash main
```

The editor opens with the fixup commit **already positioned** to squash into its target:

```
pick    abc1234 Add Order validation
fixup   9i0j1k2 fixup! Add Order validation
pick    def5678 Some other commit
```

Save+quit and the rebase squashes the fixup into the original, leaving a clean history.

The pattern is the standard idiom for "responding to PR review without polluting history with `address review` commits". Set `git config --global rebase.autosquash true` so every interactive rebase respects fixup markers.

`--squash` is similar but opens the message editor; `--fixup` keeps the original message unchanged.

Deep dive: [Code Review Workflows](../14-git/06-code-review-workflows.md)

</details>

---

### 8. Where do Git hooks live and why don't they sync between developers by default?

<details>
<summary>Reveal answer</summary>

Hooks live in `.git/hooks/` — and **`.git/` is not tracked**. So new developers don't get them automatically when they clone, and hook differences across machines are common.

Three solutions:

1. **`core.hooksPath`** — commit a `hooks/` directory in the repo, and each developer (or post-clone script) runs `git config core.hooksPath hooks`. Now the tracked `hooks/pre-commit` is the active hook.

2. **Husky (Node)** — auto-installs hooks via a `prepare` script in `package.json`. Hooks live in `.husky/`. Standard for JS/TS repos.

3. **`pre-commit` framework** (`pre-commit.com`, Python) — language-agnostic, configured via `.pre-commit-config.yaml`. Each dev runs `pre-commit install` once.

Critical rule: **client hooks are convenience, not security.** Anyone can `git commit --no-verify` to bypass them. For non-bypassable enforcement, use:

- **Branch protection on the platform** (GitHub/GitLab/Azure DevOps) — required PR, required CI green, no force-push, etc.
- **CI gates** — same checks as `pre-commit` but mandatory for merge.

Common hooks:

- `pre-commit` — fast lint/format/secret scan
- `commit-msg` — message format validation (Conventional Commits, length)
- `pre-push` — heavier checks (full test suite, prevent push to `main`)

Deep dive: [Hooks](../14-git/07-hooks.md)

</details>

---

### 9. When would you reach for `git cherry-pick` vs `git rebase`?

<details>
<summary>Reveal answer</summary>

**Cherry-pick** for a single commit you want to apply to a different branch:

```bash
# Hotfix on main, also needs to go to release/v2.x
$ git checkout release/v2.x
$ git cherry-pick <sha-from-main>
```

Result: a NEW commit on the target branch with the same content (different SHA, different parent).

**Rebase** for replaying a series of commits onto a different base. Cherry-pick is the right tool for "this one specific commit"; rebase is for "all my recent commits, replayed on top of <upstream>".

Cherry-pick gotchas:

- Use `-x` for backports — appends `(cherry picked from commit <sha>)` to the message for traceability.
- Cherry-picking a merge commit needs `-m <parent-number>` to choose a side.
- Cherry-picking many commits one by one defeats rebase machinery — if you find yourself doing it, you probably want rebase.

Deep dive: [Stash, Cherry-pick, Reflog](../14-git/05-stash-cherry-pick-reflog.md)

</details>

---

### 10. What's the senior-level explanation of "branches are cheap"?

<details>
<summary>Reveal answer</summary>

A branch in Git is literally **a 40-byte file** at `.git/refs/heads/<name>` containing a SHA. Creating one is O(1) — write the SHA to a file. Deleting one removes the file. The commits it pointed at remain in `.git/objects/`, reachable via reflog or other branches.

Commits are likewise cheap to make — but each commit contains a full tree snapshot. The reason it doesn't blow up disk: packfiles delta-compress similar trees, and identical content (across commits, branches, even unrelated files) is stored once via content-addressing.

So:

- Switching branches is `cd` + checkout the tree files. Fast.
- Creating branches is free.
- Diffing branches walks the DAG comparing trees. Linear in the diff size.
- Merging branches finds the common ancestor (LCA) and applies both diffs. Mostly fast unless conflicts need resolution.

This is the foundational reason Git workflows that involve **many short-lived branches** don't have a performance ceiling — you're moving labels along a graph, not copying data.

Deep dive: [Internals](../14-git/01-internals.md)

</details>

---

[Back to index](README.md)
