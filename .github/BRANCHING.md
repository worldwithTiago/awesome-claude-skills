# How we branch & ship

Three permanent branches, flowing **one direction only**. Every repo in the GGI portfolio uses this same model so you learn it once.

```text
  feature/* ──PR──►  development  ──promote──►  preview  ──promote──►  main
  (your work)        (integration,             (staging)              (production)
                      default branch)
```

This is a standard environment-branch pipeline (GitLab Flow). It works cleanly **only** if you follow the rules below — they're what keep the "you need to rebase" wall from ever coming back.

## The one rule that prevents "needs rebase"

> **Branch from `development`, and open your PR back *into* `development`.**
> Never open a feature PR into `preview` or `main`.

The rebase wall comes from branching off `development` but targeting `preview`. Because `preview` moves on every release, those PRs are constantly "behind" their target and GitHub forces a rebase. Target `development` (the branch you cut from) and that never happens.

## Day-to-day (everyone)

```bash
git checkout development
git pull
git checkout -b fix/<short-slug>     # prefixes: feature/ fix/ chore/ content/ seo/ task/
# work, commit, push
git push -u origin HEAD
gh pr create --base development --fill
```

Before it merges into `development`:

- **CI must pass** (see "What CI checks" below).
- If the repo has Vercel previews, **a preview URL is posted on your PR** — open it and confirm your change live.
- **At least one approving review** (more on Tier-1 repos). Then merge. **Squash-merge is fine for feature → development.**

## Promotions / releases (release owner only)

Promotions move **forward only** and use **merge commits — never squash**:

```bash
# Stage: development → preview
PR=$(gh pr create --base preview --head development --title "promote: $(date +%Y-%m-%d) → preview")
gh pr merge "$PR" --merge  # MERGE COMMIT, never --squash (explicit PR — not current branch)

# Go live: preview → main
PR=$(gh pr create --base main --head preview --title "release: $(date +%Y-%m-%d)")
gh pr merge "$PR" --merge  # MERGE COMMIT, never --squash (explicit PR — not current branch)
```

**Pass the PR explicitly.** A bare `gh pr merge` resolves the PR from whatever branch you happen to be standing on. During a promotion that is rarely the branch you mean — you are typically on `development` while merging a `development → preview` PR, and a moment later merging a `preview → main` PR from the same shell. Capturing the URL that `gh pr create` returns and passing it removes the ambiguity, so the second command cannot re-target the first PR or fail confusingly.

Hard rules for promotions:

- **Never squash a promotion.** Squash creates a new commit on the target with a different SHA, which forks history and brings the rebase pain right back. Merge commits preserve ancestry.
- **Never commit directly to `preview` or `main`.** They receive forward promotions only — except scoped `hotfix/*` PRs into `main`, see [Hotfix](#hotfix-production-emergency) below. Even then it is a PR, never a direct commit.
- **No back-merge.** Because `development` is always the most-advanced branch, nothing on `preview`/`main` is ever missing from `development` — there is nothing to merge back. (An earlier version of this template shipped a `main → development` back-merge bot; it existed only to paper over squash-promotes and is gone.) For a production emergency, use **[Hotfix](#hotfix-production-emergency)** below — do *not* fast-track a full `development → preview → main` promotion.
- **Announce every promotion in chat** so the team knows what's in staging and what just went live.
- **"Automatically delete head branches" is OFF, and merge commits are allowed.** `delete_branch_on_merge` deletes the promotion PR's head — which is `development` or `preview` — the moment it merges; `allow_merge_commit=false` forces a squash, which forks history. Check before promoting: `gh api repos/<owner>/<repo> --jq '{delete_branch_on_merge, allow_merge_commit}'` must read `false` / `true`. `scripts/standardize-pipeline.sh` sets both. (Fired for real on 2026-09-02: a `preview → main` merge deleted `preview`; 384 repos carried the same setting and were fixed the same day.)
- **After a promotion, check the branches on the remote.** `git ls-remote --heads origin development preview main` — three lines or something is wrong. Do not use `git branch -r`: without `--prune` it shows remote-tracking refs that no longer exist on the remote, and reports a deleted branch as alive.

## Retired: back-merge automation (DO NOT re-enable)

**Policy (2026-05-28):** forward-only. Reverse PRs (`main → development`, `preview → development`) are forbidden.

| Do not | Do instead |
|---|---|
| Add or fix `back-merge-main-to-dev.yml` / `back-merge-preview-to-dev.yml` | Delete them; run fleet sync if needed |
| Squash-merge a promotion PR | `gh pr merge --merge` (merge commit) |
| File tasks to "enable back-merge" (e.g. P1-20260525-001) | That task is **cancelled** — see `graygroupintl-ops/docs/BRANCH-PIPELINE.md` |
| Treat drift after squash-promote as "needs back-merge bot" | Heal once, then enforce merge-commit promotions |

Full rationale, FAQ, and superseded-task list: **`graygroupintl-ops/docs/BRANCH-PIPELINE.md`** (canonical; not duplicated here).

## Hotfix (production emergency)

A hotfix is a fix that goes to `main` **before** it goes through `development → preview`. Use it only when production is broken **now** and `development` holds unpromoted work that must not ship yet. For anything else, do an **expedited promotion**: fix on `development`, then promote `development → preview → main` the same day, so nothing skips staging.

**1. Cut the hotfix from `main`, not from `development`.** A `hotfix/*` branch that carries commits from `development` which aren't on `main` would ship unreleased work. Where `.github/workflows/enforce-pipeline.yml` is installed, the Pipeline Gate rejects such a PR; elsewhere it is on you to check.

```bash
git fetch origin
git switch -c hotfix/<short-slug> origin/main
# fix, commit, push
PR=$(gh pr create --base main --fill)
gh pr merge "$PR" --merge  # MERGE COMMIT, never --squash (explicit PR — not current branch)
```

**2. Back-port the fix to `development`.** Check whether this repo has `.github/workflows/hotfix-backport.yml`. The standard docs install does not add workflows, so most repos do not have it.
- **Without it (the default):** back-port by hand, right after the hotfix merges, using the commands below.
- **With it:** when the hotfix PR merges, the workflow cherry-picks it onto a `backport/<short-slug>` branch and opens a PR into `development`.
  - **CI:** the checks don't start on their own, because `GITHUB_TOKEN` opens the PR. Close and reopen it to run them, then merge.
  - **Already on `development`:** nothing is opened.
  - **Conflict:** an issue is opened naming the commit; back-port it by hand with the commands below.

```bash
git switch -c backport/<short-slug> origin/development
git cherry-pick -x <hotfix-merge-commit-sha>   # add -m 1 for a merge commit
gh pr create --base development --fill
```

**3. Drift is reported, not blocked (yet), in repos that have `.github/workflows/main-drift.yml`.** It runs weekly and on PRs into `main`. It lists every change on `main` that `development` lacks, as a warning plus a job-summary table. Cherry-picked back-ports count as present. It never fails a run. Promoting it to a blocking check needs a measured false-positive rate first.

If `preview` must carry the fix before the next full promotion, cherry-pick the same commit onto a branch from `preview` and PR it into `preview` the same way.

**Override (repos with the Pipeline Gate):** the label `pipeline-override` on the PR bypasses the hotfix ancestry check. Use it only when the check is wrong; the bypass is recorded in the gate's log.

## Which branch is actually deployed? (fill this in per repo)

The pipeline above says how code *moves*. It does not say what is *running*, and those
are different questions — a branch with no consumer is ceremony, and a branch running
something you did not expect is an outage waiting to be discovered. **Every repo states
its deploy map here.** If this table is missing or stale, assume nothing.

| Branch | Environment | Deployed by | URL |
|---|---|---|---|
| `development` | none (integration only) | — | — |
| `preview` | staging | Vercel git integration | _fill in_ |
| `main` | production | Vercel git integration | _fill in_ |

The default above is the common case. Two patterns exist in this portfolio and they
behave very differently:

**Deploy-from-branch (almost every repo).** A hosted platform — Vercel, and anything
else that builds from a git ref — watches `preview` and `main` and builds them. Nothing
is checked out on a machine you own, so there is no clone to drift and no restart to
forget. `main` is production. This needs no extra machinery.

**Deploy-from-checkout (rare — needs a dedicated serving clone).** A long-running local
process (a daemon, a `next start`, a worker) serves files out of a working tree on a
real machine. The trigger condition is exact:

> **A process whose working directory is inside a repo that people or agents also edit.**

That is unsafe, and the failure is not hypothetical: whatever branch someone left the
tree on is what production serves, and a `build` run by one person in that tree deletes
the artifacts the live server is reading. When a repo meets that condition it gets a
**second, dedicated clone** that only ever fast-forwards from `origin` and that nobody
works in. Rules for such a clone:

- It is a **full clone, not a git worktree.** Worktrees vanish; service managers
  (launchd, systemd) do not notice and keep pointing at a path that no longer exists.
- It has **its own venv / `node_modules` / env file.** Sharing them with the working
  checkout re-creates the coupling the clone exists to break.
- **Nothing lands in it except `origin`.** No editing, no committing, no branch
  switching by hand. Deploying is one scripted command.
- The branch it tracks is **pinned and asserted**, and the deploy script refuses any
  other branch rather than trusting whatever is checked out.
- **Every serving process for that repo uses it.** Isolating one process and leaving a
  second one serving from the working checkout keeps the original failure mode alive
  on the surface you forgot.

A serving clone is **not** a release branch and should not be named as if it were. It
answers "which files is this machine reading?", not "which code is approved to ship."
Those are independent choices: a serving clone may legitimately track `development`
(fast iteration, single operator) or `main` (staged, multi-user) — but whichever it
tracks must be written in the table above and, if it is not `main`, registered in
`graygroupintl-ops/docs/BRANCH-EXCEPTIONS.md` so nobody "fixes" it later.

## What CI checks (it's one check, many gates)

The single **`CI`** status check runs every gate in one job — if `CI` is green, they all passed:

| Step | Gate |
|---|---|
| `lint` | informational by default; flip to hard once the repo is clean |
| `typecheck` | hard — blocks merge |
| `build` | hard — blocks merge (for Next.js, `next build` also runs `tsc`) |
| `test` / content audits (where present) | hard — blocks merge |
| `npm audit` (security) | informational |

It's **one job** (not separate lint/typecheck/build checks) so the self-hosted runner does one `npm ci` instead of several — cheaper and faster, and one required status check to manage portfolio-wide. CI runs on the **self-hosted `zenith-ci` runners** (luminara), so it burns no GitHub-hosted minutes.

Two hard rules baked into the canonical workflow:

- **Never add `cache: 'npm'` to `setup-node`.** On self-hosted runners `~/.npm` already persists; the GitHub cache round-trip pulled a ~3 GB cache at ~4.5 MB/s (~11 min/job) and was the single biggest CI cost we found.
- **One job, `--if-present` steps** so the same workflow fits every Node repo.

## Branch protection settings (for reference)

- Required status check on all three branches: **CI** (plus code-owner review on `development`/`main` where configured).
- **"Require branches up to date before merging"** is **OFF on all three branches.** That setting is what forces the "needs rebase" prompt whenever a target branch moves — including on forward promotions, where the merge commit makes the source look "behind." With it off, both feature PRs and promotions merge without a forced rebase/update. CI still runs on every push to `development`/`preview`/`main`, so a post-merge break surfaces immediately — that's the safety net that makes turning it off safe.
- Applied by `scripts/standardize-pipeline.sh` (or the GitHub API). On free-plan **private** repos where server-side protection is unavailable, the committed `.githooks/pre-push` guard enforces "no direct push" client-side — activate per clone with `git config core.hooksPath .githooks` (standardize-pipeline and `/claim-task` do this automatically). Belt-and-suspenders: `graygroupintl-ops/hooks/git-safety.js` also blocks protected-branch writes at the agent tool-call layer.

  > **Caveat — repos that also use Husky.** `core.hooksPath` is not additive: it tells Git to resolve *every* hook from that one directory. Pointing it at `.githooks` in a repo that installs Husky hooks into `.husky` silently stops `pre-commit`/`pre-push` from running — no error, no output, just typecheck/lint/tests quietly not running on push. If a repo has a `.husky/` directory, do **not** set `core.hooksPath` to `.githooks`; wire the pre-push guard into the Husky hook instead, or keep `core.hooksPath=.husky`. (TCC's Dantotsu gate treats `core.hooksPath != .husky` as a critical blocker for exactly this reason — pattern-root: task 108.)

## Why this design never forces a rebase

The forced "rebase" prompt comes from one setting — "require branches up to date" — firing when a PR's **target** branch moves. We removed both triggers: feature PRs now target `development` (which they were cut from, so they're not chasing a moving release branch), and that setting is off everywhere. Real merge *conflicts* (two people editing the same lines) can still happen — that's inherent to collaboration, not this model — and you resolve those by pulling `development` into your branch, which is painless because your branch is short-lived.

## Naming conventions

| Pattern | Use |
|---|---|
| `feature/<slug>` | New functionality |
| `fix/<slug>` | Bug fix |
| `chore/<slug>` | Tooling, deps, refactor |
| `content/<slug>` / `seo/<slug>` | Content / SEO work |
| `task/<id>-<slug>` | Tied to a `.tasks/` queue item |
| `hotfix/<slug>` | Emergency production fix (branch from `main`; PR to `main`, then cherry-pick to `development`) |
| `promote/<from>-<to>-<date>` | Promotion PRs |

## For AI agents

- Never push directly to `development`, `preview`, or `main` — always cut a feature/task branch and PR into `development`.
- The git-safety hook in `graygroupintl-ops/hooks/git-safety.js` blocks direct writes to protected branches at the tool-call layer.
- Use `/claim-task` to set up an isolated worktree on a `task/<id>` branch from `development`.
- Use `/precision` to enforce scope discipline.
- Before opening a PR, pull latest `development` into your branch to clear any real conflicts.

## How we work (the part that matters most)

This pipeline is a tool, not a religion. We keep it healthy by:

- **Iterating** — if something causes friction, we find the root cause and fix it; we don't live with it.
- **Communicating constantly** — say what you're working on, and announce when you promote. The model only works if we're never guessing what each other is doing.
- **Committing to "right" over "done"** — small, clean, reviewed changes; verify on the preview URL; tie off loose ends.

<!--
Repo-specific content: wrap any section this repo must keep across template
syncs in `repo-specific:start` / `repo-specific:end` HTML comment markers.
The getuptodate template sync preserves marked blocks; unmarked custom edits
are overwritten by the next sync.
-->
