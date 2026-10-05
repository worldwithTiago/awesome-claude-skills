# Contributing

Thanks for working on this repo. This is part of the Gray Group International portfolio; the contribution flow is consistent across every repo in the org so you only learn it once.

## Quick start

```bash
# 1. clone (or pull latest)
git checkout development
git pull

# 2. branch
git checkout -b feature/<short-slug>

# 3. work
# ...

# 4. push and open PR into development
git push -u origin HEAD
gh pr create --base development --fill
```

## The flow

Read [`BRANCHING.md`](./BRANCHING.md) for the full pipeline. The short version:

```text
feature/* ─► development ─► preview ─► main
```

You PR into `development`. Releases promote `development → preview → main`. Never push directly to any of the three permanent branches.

## Before you open a PR

- [ ] PR targets `development` (never `preview`/`main`) — pull latest `development` in only to clear real conflicts
- [ ] Code compiles / type-checks / lints clean
- [ ] Tests added or updated for the change
- [ ] No secrets, no `.env` files, no large binaries committed
- [ ] Commit messages follow the convention below
- [ ] PR description fills out the template (auto-loads)

## Commit messages

Use imperative present tense, lower-case first word, no trailing period:

```text
fix login redirect loop on session expiry
add manual-J calculation for two-stage condensers
refactor invoice version reducer to pure function
chore: bump next to 16.0.3
```

Conventional prefixes (`fix:`, `feat:`, `chore:`, `docs:`, `refactor:`, `test:`) are encouraged but not required. What matters is the message tells you *what changed and why*.

### Attribution rule

**Do not attribute commits, PRs, or any artifact in this repo to AI tools.** No `Co-Authored-By: Claude`, no "Generated with…" footers, no model/vendor references. The work belongs to the contributors.

## Code style

- Existing files set the standard — match what's already there
- New files follow the language/framework defaults
- Type-check passes (`tsc --noEmit` or equivalent) before pushing
- No silent dead code, no commented-out blocks, no TODOs without an owner

## Reviews

- One review required to merge to `development`
- Two reviews required to merge to `preview` (where applicable) — or one + green CI on revenue-critical repos
- Tier-1 repos (gateway/TCC platform, thecoolingco-website, chamomile, petunia, portfolio, graygroupintl) require at least one human review even when an agent did the work

## Working with AI agents

This repo is set up for agent-assisted development:

- `/claim-task` creates an isolated worktree on a `task/<id>` branch
- `/precision` enforces scope discipline on edits
- `/safe-worktree` is the git-safety cage (blocks pushes to protected branches)
- `/dantotsu` is the completion gate before marking a task done
- `/rebase` syncs your branch with latest `development` before PR

Agents may open PRs; humans approve them.

## Questions

- Project-level questions: see `PROJECT.md` (in `-ops`) or `CLAUDE.md` (at repo root)
- Portfolio-level context: `graygroupintl-ops/CLAUDE.md` in the [graygroupintl-ops](https://github.com/graygroupintl/graygroupintl-ops) repo
- Branch pipeline questions: [`BRANCHING.md`](./BRANCHING.md)

<!--
Repo-specific content: wrap any section this repo must keep across template
syncs in `repo-specific:start` / `repo-specific:end` HTML comment markers.
The getuptodate template sync preserves marked blocks; unmarked custom edits
are overwritten by the next sync.
-->
