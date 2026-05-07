# Code Review Workflows

How you shape a branch's history before opening (and after updating) a PR is one of the strongest signals of seniority on a team. The toolkit: interactive rebase, fixup commits with autosquash, and signed commits for trustable history.

## The goal: clean, reviewable history

A good PR has:

- **Logical, atomic commits.** Each commit does one thing and has a meaningful message.
- **No "WIP" / "fix typo" / "address review" noise** in the final history (squash-merge or interactive rebase to clean up).
- **A linear, conflict-free relationship to `main`** at merge time (rebase your branch on `main` before requesting review).

Ugly history slows reviewers and obscures intent. Clean history makes the diff narrative obvious.

## `git commit --amend` — fix the last commit

When you spot a typo or want to add a forgotten file IMMEDIATELY after committing:

```bash
$ git commit -m "Add Order validation"
$ # Oops, forgot a file
$ git add forgotten_file.cs
$ git commit --amend --no-edit
```

`--amend` replaces the last commit with a new one (different SHA) including the staged changes. `--no-edit` keeps the original message; omit to edit the message too.

**Don't amend pushed commits** unless you own the branch and will force-push. Same risk as rebasing public history.

## Fixup commits and `--autosquash`

The pattern for "I'm responding to review comments and want to keep the history clean":

```bash
# Reviewer asks for a change to the commit "Add Order validation" (SHA abc1234)
$ git add updated_file.cs
$ git commit --fixup=abc1234

# Now your branch has:
#   abc1234 Add Order validation
#   def5678 Some other commit
#   9i0j1k2 fixup! Add Order validation       ← marked for autosquash

# Before merging the PR:
$ git rebase -i --autosquash main
# Editor opens with the fixup already positioned to squash:
#   pick    abc1234 Add Order validation
#   fixup   9i0j1k2 fixup! Add Order validation     ← squashes into abc1234
#   pick    def5678 Some other commit
```

The `fixup!` prefix in the commit message tells `--autosquash` where the commit belongs. The end result: `abc1234` includes the review fix, the noisy `fixup!` commit is gone, history is clean.

Pair with config:

```bash
git config --global rebase.autosquash true   # default to autosquash on -i
git config --global commit.verbose true      # show diff in commit message editor
```

`git commit --squash=<sha>` works similarly but opens the message editor; `--fixup` keeps the original message of `<sha>` unchanged.

## Squash merge vs. preserved history

The PR's merge strategy on the platform side:

| Strategy | What ends up on `main` | When to choose |
|---|---|---|
| **Merge commit** | All PR commits + a merge commit | When the individual commits are valuable history |
| **Squash and merge** | One commit containing the whole PR's diff | When PR commits are noisy WIP |
| **Rebase and merge** | PR commits replayed on top of `main`, no merge commit | Linear history with full granularity |

If your team uses **squash and merge**, you don't need to worry about cleaning up commits in the PR — they all collapse into one. Just make sure the PR title is the squashed commit's message.

If your team uses **merge** or **rebase and merge**, do the cleanup work via interactive rebase before requesting final review.

## Commit message conventions

The Pro Git book and most style guides converge on this format:

```
Short subject line, imperative mood (50 chars or less)

A blank line, then a body that explains *why*, not what.
Wrap at 72 characters. Multiple paragraphs OK.

Bullet points work too:
- Like this
- And this

Refs: #123
```

Key rules a senior follows:

- **Subject in imperative mood**: "Add validation" / "Fix race condition" — NOT "Added", "Fixes", or "Adding".
- **Subject under 50 chars** (sometimes 72) so it fits in `git log --oneline`, GitHub's interface, etc.
- **Blank line between subject and body.**
- **Body explains why, not what.** The diff already shows what.
- **Reference issues/tickets** in trailers like `Refs: #123` or `Fixes: #456` (some platforms auto-close issues on merge with the right keyword).

### Conventional Commits (optional)

Many teams adopt the [Conventional Commits](https://www.conventionalcommits.org/) format for tooling (semantic-release, changelog generation):

```
feat(orders): add discount validation
fix(payments): handle Stripe webhook timeout
docs: expand README with worktree examples
chore: bump dependencies
```

Worth adopting when you generate changelogs or version bumps from commit history. Otherwise it's just noise.

## Signed commits

Git supports **GPG-signed** and **SSH-signed** commits to prove authorship. Without signing, any developer can `git config user.name "Linus Torvalds"` and commit; verification is trust-on-a-username.

```bash
# Set up signing
$ git config --global user.signingkey <YOUR-KEY>
$ git config --global commit.gpgsign true

# Now every commit is signed
$ git commit -m "Add validation"

# Verify
$ git log --show-signature
commit abc1234
gpg: Good signature from "Your Name <your@email>"
```

Platforms (GitHub, GitLab) display a "Verified" badge next to signed commits whose signing key is registered to your account. For protected branches, you can require all merged commits to be signed.

### SSH signing (.NET / GitHub-friendly)

Since Git 2.34+, you can sign commits with your SSH key (no GPG needed):

```bash
$ git config --global gpg.format ssh
$ git config --global user.signingkey ~/.ssh/id_ed25519.pub
$ git config --global commit.gpgsign true
```

This is the easiest path on Windows + WSL + corporate machines that already use SSH keys.

### When signing matters

- **Open-source projects** — provenance for security and licensing.
- **Regulated industries** — finance, healthcare, defense often require it.
- **Repositories with bots/automation** that commit on your behalf — signed commits distinguish "you" from "your CI".

For most teams, signing is optional but increasingly expected.

## The rebase-update-rebase loop

Standard PR workflow:

```bash
# Day 1: start work
$ git checkout -b feat/payment main
# ... commits ...
$ git push -u origin feat/payment

# Day 3: main has moved, sync up
$ git fetch origin
$ git rebase origin/main
$ git push --force-with-lease

# Day 4: review feedback
$ # ... fixup commits ...
$ git rebase -i --autosquash origin/main
$ git push --force-with-lease

# Final: clean linear history, ready to merge
```

`--force-with-lease` instead of `--force` is the safety belt — it refuses if someone else pushed to your branch in the meantime, preventing accidental overwrites if you forgot to fetch.

## Co-authored commits

For pairing or attribution to multiple authors:

```
Add Order validation

Co-authored-by: Alice Doe <alice@example.com>
Co-authored-by: Bob Smith <bob@example.com>
```

GitHub renders both authors on the commit page and counts the contribution toward both.

## Reviewing PR diffs locally

To review a PR locally (useful for big diffs that don't render well in a browser):

```bash
$ git fetch origin pull/123/head:pr-123
$ git switch pr-123
$ git diff main..HEAD
```

Or for a specific commit range:

```bash
$ git log main..HEAD --oneline
$ git show <sha>
```

`gh pr checkout 123` (with the GitHub CLI) does the same in one command.

## Senior-interview gotchas

- **`git commit --amend`** rewrites the last commit. Never amend a pushed commit unless you own the branch.
- **Fixup commits + `--autosquash`** — the standard idiom for review iterations on a clean-history workflow.
- **`--force-with-lease`** is safer than `--force` — refuses if remote diverged.
- **Subject line in imperative mood**, under 50 chars; body explains *why*.
- **Squash-merge vs merge vs rebase-merge** is a team policy choice — none is universally correct.
- **Signed commits** prove authorship; set up SSH signing on Git 2.34+ if your team requires it.
- **`git config rebase.autosquash true`** turns on autosquash by default — every interactive rebase respects fixup markers.

## Useful Links

- [`git-rebase --autosquash` — git-scm.com](https://git-scm.com/docs/git-rebase#Documentation/git-rebase.txt---autosquash) — fixup/squash auto-positioning
- [`git-commit --fixup` — git-scm.com](https://git-scm.com/docs/git-commit#Documentation/git-commit.txt---fixupamendrewordltcommitgt) — fixup commits
- [Pro Git book — Commit Guidelines](https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project) — message conventions
- [Conventional Commits spec](https://www.conventionalcommits.org/) — for tooling-driven changelogs
- [GitHub — About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification) — GPG and SSH signing setup

---

[← Previous: Stash, Cherry-pick, Reflog](05-stash-cherry-pick-reflog.md) | [Back to index](README.md) | [Next: Hooks →](07-hooks.md)
