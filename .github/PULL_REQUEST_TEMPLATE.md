<!--
PR target rules:
  feature/* → development   (this is where ~all PRs go)
  development → preview      (promotion — MERGE COMMIT, never squash)
  preview → main             (release — MERGE COMMIT, never squash)

Never open a feature PR into preview or main. No back-merge.
See .github/BRANCHING.md for the full flow.
-->

## What changed

<!-- One or two sentences. What does this PR do and why? -->

## Type

- [ ] Feature
- [ ] Fix
- [ ] Chore / refactor / deps
- [ ] Docs
- [ ] Promotion (`development → preview` or `preview → main` — merge commit, never squash)

## Checklist

- [ ] Targets `development` (feature work) — not `preview`/`main`
- [ ] Type-check / lint / tests pass locally
- [ ] No secrets, `.env` files, or large binaries committed
- [ ] Docs updated if behavior changed (README, BRANCHING, project docs)
- [ ] Commit messages follow [CONTRIBUTING.md](./CONTRIBUTING.md) — no AI attribution

## Test plan

<!-- How did you verify this? What should reviewers reproduce? -->

- [ ]

## Verification

<!--
  REQUIRED by the Verify Gate. "Tests pass" is NOT verification — observe the
  system do the thing. Paste a real evidence line (see
  graygroupintl-ops/templates/VERIFICATION_STANDARDS.md, L0–L3). Examples:
    verified live: clicked Save on app.example.com, row in dashboard
    verified live: curl POST /api/lead returned 200, payload matched
    inserted row: UI submission, observed leads.id=4471 within 90s
  OR run `/verify-deploy` and add the `verified` label.
  Pure docs/chore? Add `verify-skip` label AND `verify-skip: <reason>` below.
  Replace this comment with your evidence line (or use the `verified` label).
-->

## Risk

<!-- What could break? What's the blast radius? Tier-1 repos need this filled in. -->

## Related

<!-- Task IDs, issue links, related PRs -->
