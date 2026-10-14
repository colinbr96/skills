---
name: pr-review
description: Review a GitHub PR against a fixed quality checklist. Takes a PR link/number, or infers the PR from the current branch.
argument-hint: "[PR URL or number]"
disable-model-invocation: true
---

Review the pull request against the checklist below.

## 1. Find the PR

- If a PR URL or number was given, use it.
- Otherwise, infer it from the current conversation or git branch: `gh pr view --json url,number`. If no PR exists, tell me.

## 2. Gather context

- `gh pr view <pr> --json title,body,baseRefName,headRefName,files,url`
- `gh pr diff <pr>`
- Read the full surrounding code for changed files (not just the diff hunks) so you can judge correctness and consistency with existing patterns. Look at neighboring files and existing tests to learn the codebase's conventions.
- If the PR body references a ticket or design doc, use it to understand the intended problem.

## 3. Evaluate criteria

1. **Solves the problem correctly** — Does the logic actually achieve the stated goal? Look for bugs, wrong conditions, off-by-one, race conditions, incorrect error handling. This is the most important check.
2. **Privacy/security** — PII stored securely, permission checks enforced server-side, user input validated, no moderation bypass for user-generated content.
3. **Edge cases** — nil/empty inputs, boundaries, failures of dependencies/network, concurrency, permissions, feature flag on/off states, backwards compatibility. List specific unhandled cases.
4. **Rollout/rollback safety** — Is risky behavior gated behind a flag with a kill switch? Can it be rolled back safely (no irreversible data/schema changes, client/server version skew handled)?
5. **Unit tested** — Are new/changed behaviors covered by tests? Do tests cover the edge cases? Are assertions meaningful?
6. **Performance** — Scrutinize new network calls, allocations, or heavy work in hot paths (startup, feed scrolling, playback, per-frame code). Flag N+1 calls and missing caching.
7. **Accurately described** — Does the title/description match what the diff actually does? Flag undocumented changes and described changes that are missing.
8. **Complete test plan** — Does the PR description include a test plan that covers how the change was verified (automated + manual), including relevant edge cases and flag states?
9. **Observability** — Does new behavior have the metrics, logs, or error reporting needed to tell if it's broken in production?
10. **Follows established patterns** — Naming, file structure, error handling, logging, flags, and idioms consistent with the surrounding codebase. Cite the existing pattern it deviates from.
11. **Not over-engineered** — Unnecessary abstraction, premature generalization, speculative config/options, code that could be meaningfully simpler.
12. **Scope** — Does the PR do one thing? Flag unrelated changes and suggest splitting if it mixes concerns.
13. **No leftover temporary artifacts** — Debug prints/logs, commented-out code, TODO/FIXME added in this PR, hardcoded test values, stray files, local config, disabled tests, unused imports/variables.

## 4. Report

Output:

- Decision for each criterion: ✅ Pass, ⚠️ Concern, or ❌ Fail.
- For each ⚠️/❌, give `file:line` references and a suggested fix. Keep ✅ items to one line.

Do not post comments to the PR or make any changes — this review is for me only, unless I explicitly ask otherwise.
