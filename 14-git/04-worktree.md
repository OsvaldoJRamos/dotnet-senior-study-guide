# Worktree

`git worktree` is the underused power tool of the senior Git toolkit. The official one-line summary:

> "Manage multiple working trees attached to the same repository. A git repository can support multiple working trees, allowing you to check out more than one branch at a time." — `git-worktree` docs

It lets a single Git repository have **multiple checked-out branches simultaneously**, each in its own directory. No stash. No switch. No re-cloning. Each worktree shares the same `.git` directory — so objects, refs, and config are all common — but each has its own working tree files and HEAD.

## The problem it solves

You're deep in a feature branch with uncommitted changes. Someone reports a production bug that needs an immediate fix. Without worktrees you can:

1. **Stash** your work, checkout `main`, fix, commit, push, switch back, pop the stash. Risk: stash conflicts when you pop, especially if your fix touched related files.
2. **Commit WIP**, switch, fix, switch back, reset to undo the WIP commit. Cleaner but adds a fake commit to your branch's reflog.
3. **Re-clone** the repo somewhere else. Wasteful — duplicates the entire object database (potentially gigabytes).

With worktrees:

```bash
$ git worktree add ../v2-hotfix main
$ cd ../v2-hotfix
# fix, commit, push from here — your original directory untouched
$ cd -
$ git worktree remove ../v2-hotfix
```

The original directory's HEAD and uncommitted changes never moved.

The Git docs themselves use this as the canonical example:

> "You are in the middle of a refactoring session and your boss comes in and demands that you fix something immediately... Instead, you create a temporary linked worktree to make the emergency fix, remove it when done, and then resume your earlier refactoring session."

## The commands

### `git worktree add <path> [<commit-ish>]`

Create a new worktree at `<path>` checked out to `<commit-ish>` (a branch, tag, or commit SHA).

```bash
# Add a worktree on an existing branch
git worktree add ../my-feature feature/payment

# Create a NEW branch at the same time
git worktree add -b feature/spike ../my-spike main

# Detached HEAD (for throwaway exploration)
git worktree add --detach ../tmp 1a2b3c4
```

If `<commit-ish>` is a branch already checked out in another worktree, `add` refuses unless you pass `--force`:

> "By default, `add` refuses to create a new worktree when _<commit-ish>_ is a branch name and is already checked out by another worktree." — `git-worktree` docs

This is the **one-branch-per-worktree** rule and it's the most common surprise. You can't have two worktrees both checking out `main`. The reason: a branch is a single ref, and two trees pointing at it could diverge silently.

### `git worktree list`

Show all attached worktrees:

```bash
$ git worktree list
/repos/myproject              1a2b3c4 [main]
/repos/myproject-feature      5d6e7f8 [feature/payment]
/repos/myproject-hotfix       9a0b1c2 [hotfix/critical-bug]
```

The `[bare]`, `[detached HEAD]`, `[locked]`, and `[prunable]` annotations appear for special states.

### `git worktree remove <path>`

Removes the worktree. Refuses if the worktree has uncommitted/unpushed changes; pass `--force` to override.

```bash
git worktree remove ../my-feature
```

### `git worktree prune`

Cleans up administrative metadata for worktrees that were deleted manually (rm -rf) without `git worktree remove`. Run periodically; `git worktree list` will show stale entries marked `[prunable]` until you prune.

### `git worktree lock <path>` / `unlock <path>`

Prevents pruning. Useful when a worktree lives on removable media or a network mount that may be unavailable temporarily — `prune` would otherwise consider it stale.

### `git worktree move <path> <new-path>`

Move a worktree to a new path on disk (updates Git's tracking metadata accordingly).

### `git worktree repair`

Repair broken worktree linkage if administrative files got corrupted or paths changed unexpectedly.

## The shared `.git` and per-worktree state

A worktree is **not a clone**. The data layout:

```
/repos/myproject/.git/                    ← main worktree's full Git directory
                /index                    ← main worktree's index
                /HEAD                     ← main worktree's HEAD
                /worktrees/
                    myproject-feature/
                        index             ← feature worktree's index
                        HEAD              ← feature worktree's HEAD
                        ...

/repos/myproject-feature/.git             ← FILE (not directory) pointing back
                                             to /repos/myproject/.git/worktrees/myproject-feature
```

What's **shared**:

- `.git/objects/` — all blobs, trees, commits, packfiles. Adding a worktree adds zero disk cost for shared objects.
- `.git/refs/` (most refs) — branches, tags, remote-tracking branches.
- `.git/config` — repo-level config.
- `.git/hooks/` — same hooks apply to all worktrees.

What's **per-worktree**:

- `HEAD` — each worktree has its own current branch.
- `index` — each has its own staging area.
- Reflog of `HEAD` and per-branch reflogs.
- Working tree files themselves.

This is why worktrees are essentially free: a new worktree adds only the per-worktree state files and the working tree's checked-out files.

## Practical patterns

### 1. Hotfix while developing

The canonical case. Keep your feature work in `~/repos/proj`, spin up `~/repos/proj-hotfix` for an urgent fix, remove when done.

### 2. Compare branches side-by-side

Have `main` and `feature/v2` checked out in two directories. Open both in the same IDE / diff tool. No more `git diff main..feature/v2` for visual review of large changes.

### 3. Run a long-running test on one branch while developing on another

```bash
# Worktree 1: work on feature
cd ~/repos/proj
git checkout feature/payment

# Worktree 2: long-running test on a different branch
cd ~/repos/proj-test
git worktree add ../proj-test some-branch
npm test  # slow integration suite, leave running

# Back to worktree 1, keep coding
cd ~/repos/proj
```

The test in worktree 2 won't be disrupted by your branch switches in worktree 1.

### 4. Build artifacts in a parallel worktree

Some build systems pollute the working tree with caches, generated files, or `node_modules` that conflict with branch switches. A separate worktree per active branch keeps each build's cache scoped.

### 5. Multi-version maintenance

Maintaining multiple released versions (v1.x, v2.x) becomes trivial: one worktree per version branch. No constant context-switching.

## The `.git/worktrees` directory and CI

Some CI systems clone a fresh repo per build. If your CI uses persistent workspaces with worktrees (e.g., Jenkins agent caching), be aware that:

- Worktrees can leave stale entries that need pruning.
- A misplaced `git worktree remove --force` on a CI agent can wipe an active build's directory.

Most CI templates avoid worktrees for simplicity. Use them in the dev workflow, not on shared infra.

## Common mistakes

- **Trying to check out the same branch in two worktrees** — refused by default. Use `--force` only if you know what you're doing (and accept that two trees can write to the same branch and confuse each other).
- **Deleting a worktree directory with `rm -rf` instead of `git worktree remove`** — leaves stale entries. Run `git worktree prune` to clean them up.
- **Confusing the `.git` *file* in a worktree with a real `.git` directory** — in non-main worktrees, `.git` is just a tiny file pointing back to the main repo. Don't move/rename it.
- **Storing worktrees inside the repo's own working tree** (e.g., `git worktree add ./tmp ...`) — this works but pollutes status output and complicates `.gitignore`. Prefer external paths (`../tmp`).

## When NOT to use worktrees

- **Shared/CI environments** where lifecycle management is harder.
- **Cross-machine workflows** where you'd actually need a separate clone.
- **Tools that don't understand worktree linkage** (rare; most modern tools handle it fine, but very old Git GUIs may not).

## Senior-interview gotchas

- **One branch can be checked out in only ONE worktree at a time.** Default refusal protects you from data races.
- **Worktrees share `.git/objects` and refs** — adding a worktree is essentially free disk-wise.
- **Each worktree has its OWN HEAD and index** — that's how they don't interfere.
- **Use `git worktree remove`, not `rm -rf`** — `rm` leaves Git's metadata stale and you'll need `worktree prune`.
- **The canonical use case** is "urgent task interrupting current work" — worktrees beat stash-and-switch.
- **Hooks are shared** across worktrees (configured via `.git/hooks/` once).

## Useful Links

- [`git-worktree` docs — git-scm.com](https://git-scm.com/docs/git-worktree) — full subcommand reference, REFRESHING-STATE notes, examples
- [Pro Git book — Multiple working trees (older intro)](https://git-scm.com/book/en/v2) — Chapter on advanced features

---

[← Previous: Rebase vs Merge](03-rebase-vs-merge.md) | [Back to index](README.md) | [Next: Stash, Cherry-pick, Reflog →](05-stash-cherry-pick-reflog.md)
