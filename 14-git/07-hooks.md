# Hooks

Git **hooks** are shell scripts that Git runs automatically at specific points in the commit/push lifecycle. They live under `.git/hooks/`. The right hooks catch entire classes of mistakes (commits with secrets, broken tests, malformed messages) before they reach the shared remote.

## How hooks work

When you run `git commit`, Git looks under `.git/hooks/` for a script named after the lifecycle event. If the script is executable and exits with status `0`, the commit proceeds. Non-zero exit blocks the operation.

Default hooks ship as `*.sample` files (e.g., `.git/hooks/pre-commit.sample`). Rename to remove the `.sample` suffix and make executable to activate.

```bash
$ ls .git/hooks/
applypatch-msg.sample       pre-rebase.sample
commit-msg.sample           pre-receive.sample
post-update.sample          prepare-commit-msg.sample
pre-applypatch.sample       update.sample
pre-commit.sample
pre-merge-commit.sample
pre-push.sample
```

## The hooks a senior should know

### Client-side (run on the developer's machine)

| Hook | Triggers when | Common use |
|---|---|---|
| `pre-commit` | Before commit message is solicited | Lint, format, fast tests, secret scans |
| `prepare-commit-msg` | Before the commit message editor opens | Auto-prefix message with branch name / ticket ID |
| `commit-msg` | After message is written | Validate message format (Conventional Commits, length) |
| `pre-push` | Before pushing to remote | Run unit tests, security scans, prevent push to protected branches |
| `pre-rebase` | Before rebase starts | Block rebase of certain branches |
| `post-checkout` | After `git checkout` / `git switch` | Repopulate `node_modules`, rebuild, etc. |
| `post-merge` | After successful merge | Re-install dependencies if `package.json` changed |

### Server-side (run on the Git server / hosting platform)

These are configured on the remote (or simulated by GitHub Actions / GitLab CI):

| Hook | Use |
|---|---|
| `pre-receive` | Reject pushes that violate policy (no force-push to main, no commits without sign-off) |
| `update` | Per-ref version of pre-receive |
| `post-receive` | Trigger CI, send notifications |

For most teams, server-side enforcement is via the platform's branch protection rules + CI workflows, not raw Git server-side hooks. Hooks remain critical client-side.

## `pre-commit` — the workhorse

The most common client hook. A typical `.git/hooks/pre-commit`:

```bash
#!/usr/bin/env bash
set -e

# Run linter on staged C# files
staged_cs=$(git diff --cached --name-only --diff-filter=ACM | grep '\.cs$' || true)
if [ -n "$staged_cs" ]; then
    dotnet format --verify-no-changes $staged_cs || {
        echo "Formatting issues. Run 'dotnet format' and re-stage." >&2
        exit 1
    }
fi

# Block secrets
if git diff --cached | grep -E 'AKIA[0-9A-Z]{16}|password\s*=\s*"' ; then
    echo "Possible secret in staged changes. Aborting." >&2
    exit 1
fi
```

Keep `pre-commit` **fast** (under a few seconds). Slow hooks make developers `--no-verify` to bypass them.

## `commit-msg` — message validation

Validate the commit message after it's written:

```bash
#!/usr/bin/env bash
msg_file=$1
first_line=$(head -1 "$msg_file")

# Reject if subject too long
if [ ${#first_line} -gt 72 ]; then
    echo "Commit subject too long (${#first_line} chars; max 72)." >&2
    exit 1
fi

# Conventional Commits prefix check
if ! grep -qE '^(feat|fix|docs|chore|refactor|test|perf|build|ci)(\([^)]+\))?!?: ' "$msg_file"; then
    echo "Subject must start with a Conventional Commit prefix (feat|fix|docs|...)." >&2
    exit 1
fi
```

This is where you enforce team conventions — Conventional Commits, ticket prefix (`PROJ-123: ...`), etc.

## `pre-push` — final gate before the world sees it

Heavier checks that are too slow for `pre-commit`:

```bash
#!/usr/bin/env bash
# Run full test suite before pushing
dotnet test --no-restore --logger "console;verbosity=minimal" || {
    echo "Tests failed. Push blocked." >&2
    exit 1
}

# Prevent push to main
remote=$1; url=$2
while read local_ref local_sha remote_ref remote_sha; do
    if [ "$remote_ref" = "refs/heads/main" ]; then
        echo "Direct push to main blocked. Use a PR." >&2
        exit 1
    fi
done
```

The "block direct push to main" guard is a backstop — branch protection on the remote is the real defense, but `pre-push` catches accidents earlier.

## The problem with `.git/hooks/`

Hooks live **inside** `.git/`, which is **not tracked**. This means:

- New team members don't get them automatically.
- Hooks vary across developers' machines.
- You can't share/version hooks via the repo.

Three solutions:

### 1. Commit a `hooks/` directory and `core.hooksPath`

```bash
# In repo root
$ mkdir hooks
$ # ... add your hooks here, commit them ...

# Each developer (or post-clone script) does:
$ git config core.hooksPath hooks
```

Now `hooks/pre-commit` is the active hook. Tracked in the repo, shared across the team.

### 2. Husky (Node ecosystem)

[`husky`](https://typicode.github.io/husky/) is the de-facto hook manager for JS / Node projects. It auto-installs hooks via a `prepare` script in `package.json`:

```json
{
  "scripts": { "prepare": "husky install" },
  "devDependencies": { "husky": "^9.0.0" }
}
```

Hook scripts live in `.husky/pre-commit`, etc. Ideal when your repo already pulls in npm.

### 3. `pre-commit` framework (Python ecosystem, language-agnostic)

[`pre-commit`](https://pre-commit.com/) is a multi-language hook framework configured via `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
  - repo: https://github.com/dotnet/format
    rev: v8.0.0
    hooks:
      - id: dotnet-format
```

Each developer runs `pre-commit install` once after cloning. The framework caches hook environments and runs them in parallel.

Despite the name, it's language-agnostic — it has hooks for .NET, Go, Python, JS, etc. Worth knowing for cross-stack repos.

## When NOT to use hooks

- **Hard policy requirements** (no commits without sign-off, no force-push to main) belong on the **server / branch protection** side, not in client hooks. Client hooks can be bypassed with `--no-verify`.
- **Slow operations** (full test suites, integration tests, build) belong in CI, not in `pre-commit`. Devs will `--no-verify` to escape, defeating the hook.
- **Operations requiring credentials** (API calls, deploys) shouldn't run from a hook — surprising side effects on routine commands.

Rule of thumb: client hooks are **convenience**, not **security**. Use them to catch mistakes before they leave the developer's machine. Use platform branch protection + CI for non-bypassable enforcement.

## Bypassing hooks

Sometimes you legitimately need to skip:

```bash
$ git commit --no-verify -m "WIP, will fix lint later"
$ git push --no-verify
```

`--no-verify` skips `pre-commit`, `commit-msg`, `pre-push`, and `pre-rebase` hooks. Use sparingly — it's the equivalent of "I know what I'm doing".

The project's CLAUDE.md is explicit:

> "Never skip hooks (--no-verify) or bypass signing (--no-gpg-sign, -c commit.gpgsign=false) unless the user has explicitly asked for it. If a hook fails, investigate and fix the underlying issue."

That's the right default — fix the failure, don't bypass it.

## Senior-interview gotchas

- **Hooks live in `.git/hooks/`** (not tracked). Use `core.hooksPath` to share.
- **Client hooks are convenience, not security.** Anyone can `--no-verify`.
- **`pre-commit` runs before message editor**, `commit-msg` runs after. Different responsibilities.
- **Keep `pre-commit` fast** (under a few seconds). Heavy stuff goes in `pre-push` or CI.
- **`husky`** for Node/JS, **`pre-commit` framework** for language-agnostic.
- **Server-side enforcement** (branch protection, signed commits, CI gates) is what you trust. Client hooks help, but don't gate.
- **Don't `--no-verify`** as a habit. It's an emergency tool.

## Useful Links

- [`githooks(5)` — git-scm.com](https://git-scm.com/docs/githooks) — full list of hooks and triggers
- [Pro Git book — Customizing Git: Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
- [husky — typicode.github.io/husky](https://typicode.github.io/husky/) — Node-ecosystem hook manager
- [pre-commit — pre-commit.com](https://pre-commit.com/) — language-agnostic hook framework

---

[← Previous: Code Review Workflows](06-code-review-workflows.md) | [Back to index](README.md) | [Next: Dangerous Operations →](08-dangerous-operations.md)
