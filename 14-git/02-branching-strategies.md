# Branching Strategies

A branching strategy is the team agreement on **how branches relate to environments, releases, and reviews**. Three patterns dominate: feature branching, GitFlow, and trunk-based development. There's no universal best — the right one depends on team size, deploy cadence, and how much pre-merge gating you need.

## Feature branching (the baseline)

Every change happens on a short-lived branch off `main`, opens a PR, gets reviewed, and merges back.

```
main:        ──A──B──C──────────G──H──
                    \           /
feature:             D──E──F────
```

- Branch lives **hours to days**, not weeks.
- One PR per branch; rebase or squash before merge to keep `main` history clean.
- `main` is always deployable.

This is the default for most modern teams. It's flexible, scales to many parallel features, and pairs naturally with CI gates on the PR.

**When it works:** any team size with code review and CI on PRs.
**When it breaks:** if branches live too long they drift, become merge nightmares, and start "code review fatigue" cycles.

## Trunk-based development

Everything lives on `main` (the "trunk"). Branches are extremely short-lived (often **hours**). Incomplete features are kept dark via **feature flags** rather than branches.

```
main:  ──A──B──C──D──E──F──G──H── (deploys multiple times a day)
```

- Daily integration to `main`.
- Feature flags decouple release from deploy.
- CI/CD pushes every commit to staging or even production.

This is what high-velocity orgs (Google, Meta, modern fintech) practice. It demands strong CI, fast tests, and a feature-flag culture, but eliminates the merge debt that long-lived branches create.

**When it works:** mature CI, comfort with feature flags, multiple deploys per day.
**When it breaks:** weak test suites + trunk-based = production breakage; you need to back this with discipline.

## GitFlow (legacy, mostly)

A more elaborate scheme with multiple long-lived branches:

| Branch | Purpose |
|---|---|
| `main` (or `master`) | Production-ready code, tagged with release versions |
| `develop` | Integration branch where features merge first |
| `feature/*` | Branched off `develop`, merged back to `develop` |
| `release/*` | Branched off `develop` for stabilization, merged to both `main` and `develop` |
| `hotfix/*` | Branched off `main` for urgent prod fixes, merged to both `main` and `develop` |

```
main:    ──────A───────────G─────────H─── (tags v1.0, v1.1)
                \         /  \      /
release:         R1──R2──R3   H1──H2
                /              \  /
develop: ──B──C────D────E──F────I──J──K──
              \   /\    /
feature:       D1──     E1──F1
```

GitFlow was published in 2010 and was popular when releases meant "ship a version every quarter". It maps poorly to continuous deployment because:

- Every change visits `develop` then `release` then `main` — high coordination overhead.
- Hotfix dual-merges (to both `main` and `develop`) are error-prone.
- The model assumes "release windows" — alien to deploy-on-merge teams.

> The original author of GitFlow (Vincent Driessen) published a [retrospective note](https://nvie.com/posts/a-successful-git-branching-model/) saying the model is no longer his recommendation for web/SaaS work and trunk-based is usually better. Still relevant if you ship versioned software (libraries, mobile apps with app-store review cycles) where there isn't a single "production" but multiple supported releases.

**When it works:** versioned products with multiple supported releases (mobile apps, SDKs, on-prem software).
**When it breaks:** SaaS / web with continuous deploy — too much ceremony.

## GitHub Flow / GitLab Flow

Lightweight variants of feature-branching that the major hosting platforms recommend:

- **GitHub Flow:** `main` is always deployable; branch, PR, review, merge, deploy. No `develop`.
- **GitLab Flow:** like GitHub Flow but with optional environment branches (`staging`, `production`) for promotions, useful when you can't deploy on every merge.

In practice, "feature branching with PRs" + "trunk-based-ish discipline" is the common modern blend. Most senior interviewers care less about the name than the trade-offs you can articulate.

## Branch naming conventions

The project's CLAUDE.md uses `docs/<short-description>` (e.g., `docs/expand-concurrency-section`). Common patterns industry-wide:

| Prefix | Use |
|---|---|
| `feature/` or `feat/` | New feature |
| `fix/` | Bugfix |
| `docs/` | Documentation only |
| `chore/` | Maintenance (deps, config) |
| `refactor/` | No behavior change |
| `hotfix/` | Urgent prod fix |

Pair with a Jira/Linear ticket if your team uses one: `feature/PROJ-123-add-payment`. The prefix is for fast `git branch` filtering and CI rules.

## How long should branches live?

The single most important branching question. Rough buckets:

| Branch lifespan | Risk |
|---|---|
| < 1 day | Almost no merge conflicts. Encourages small, reviewable changes. |
| 1-3 days | Manageable conflicts. The sweet spot for feature branches. |
| 1 week+ | Significant rebasing/conflicts. Review fatigue. Reviewers lose context. |
| 1 month+ | Pathological. Either feature-flag and merge to `main`, or break it up. |

Long-lived branches create their own gravity — the longer they live, the harder they are to merge, the longer they live. Discipline-wise, this is where trunk-based with feature flags shines.

## Protected branches

Most platforms (GitHub, GitLab, Azure DevOps) support **branch protection rules** on `main` (and `develop` if you use GitFlow):

- Require PR + N reviewers before merge.
- Require CI green before merge.
- Disallow force-push.
- Disallow direct push (PR-only).
- Require signed commits.
- Auto-delete head branch on merge.

These prevent the most dangerous git operations from corrupting shared history. A senior should set these up on day one of any new repo.

## Senior-interview gotchas

- **Trunk-based is the modern default for SaaS** — feature flags decouple release from deploy.
- **GitFlow is largely deprecated** for continuous deploy; still relevant for versioned/on-prem products.
- **Long-lived branches are an anti-pattern** — they accumulate conflicts and reviewer fatigue.
- **Branch names follow conventions** (`feat/`, `fix/`, `docs/`, etc.) for tooling and CI filtering.
- **Protect `main`** — require PR, CI green, no force push. Set this up day one.
- **A "branch" is just a label.** Strategies are about WHEN labels move and WHO can move them.

## Useful Links

- [GitHub Flow — official guide](https://docs.github.com/en/get-started/quickstart/github-flow) — minimal feature-branching model
- [Trunk-Based Development — trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) — the canonical reference for trunk-based
- ["A successful Git branching model" — nvie (2010, with 2020 retrospective)](https://nvie.com/posts/a-successful-git-branching-model/) — the original GitFlow post + author's note that it's no longer his default
- [Git documentation: gitworkflows](https://git-scm.com/docs/gitworkflows) — Linux-kernel-style integration workflow

---

[← Previous: Internals](01-internals.md) | [Back to index](README.md) | [Next: Rebase vs Merge →](03-rebase-vs-merge.md)
