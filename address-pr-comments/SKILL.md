---
name: address-pr-comments
description: >-
  Addresses GitHub pull request comments using the gh CLI. Use when the user
  provides a PR link/URL and wants review comments addressed, answered, or
  implemented. Understands the PR first, investigates each comment, reports
  answers and proposed fixes in chat, and waits for the user to confirm before
  changing code. Never posts on GitHub.
---

# Address PR comments

The user invokes this skill with a GitHub PR link. Use `gh` for all GitHub reads. Never post on GitHub.

## Rules

- **Never** reply, resolve, or otherwise write to the PR (`gh pr comment`, review replies, GraphQL mutations, emoji reactions). Code and docs in the repo are fine; leave the GitHub conversation to the user.
- Do not edit code until the user confirms which fixes to make. The first pass is read-only.
- Do not commit or push unless the user asks.
- If the user did not give a PR URL, check for PRs against the current branch.

## Workflow

1. Fetch PR metadata, diff, and comments with gh
2. Check out the PR branch if needed
3. Read the diff and codebase until the change is understood
4. Read the PR comments, classify them, and investigate each one (read-only)
5. Report in chat with proposed fixes, then stop
6. After the user confirms, implement the approved fixes

### 1. Fetch with gh

`gh` needs network access and GitHub auth.

```bash
gh pr view "$PR" --json number,title,body,baseRefName,headRefName,url,author
gh pr diff "$PR"
```

Parse `OWNER`, `REPO`, and `NUMBER` from the URL (`https://github.com/OWNER/REPO/pull/NUMBER`).

Fetch review threads (preferred: includes resolved state), conversation comments, and reviews. Filter with `--jq` so you do not dump raw payloads into context:

```bash
gh api graphql -F owner=OWNER -F name=REPO -F number=NUMBER -f query='
query($owner:String!, $name:String!, $number:Int!) {
  repository(owner:$owner, name:$name) {
    pullRequest(number:$number) {
      reviews(first:100) { nodes { author { login } body state } }
      reviewThreads(first:100) {
        nodes {
          isResolved
          isOutdated
          path
          comments(first:50) {
            nodes { author { login } body url createdAt }
          }
        }
      }
    }
  }
}'

gh api "repos/OWNER/REPO/issues/NUMBER/comments" --paginate \
  --jq '.[] | {user: .user.login, body, html_url, created_at}'
```

Skip resolved threads, outdated threads that are already fixed, and pure acknowledgements (`LGTM`, "fixed"). Handle unresolved review threads and top-level conversation comments.

### 2. Check out the PR branch

Investigation and later fixes must happen on the PR head. If the working tree is not already that branch:

```bash
gh pr checkout "$PR"
```

Do not discard uncommitted local work. If checkout would clobber changes, stop and ask the user.

### 3. Understand the change first

Read the diff, then the surrounding codebase, until you understand what the PR does and why. **Do not read or act on comments until this is done.** Comments are easy to misread without that context.

### 4. Read comments, classify, and investigate

For each remaining comment, choose one:

| Kind          | What it is                                                 | Action                                                                     |
| ------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| ❓ Question   | Asks why/how, requests clarification, or seeks information | Research (code, docs, tests, diff). Answer **in this chat**, not on GitHub. |
| 💡 Suggestion | Proposes a code/doc change, a fix, a nit, or a bot finding | Check whether it is correct and needed; work out the fix that follows existing repo patterns. Do not apply it. |
| ☯️ Mixed      | Question plus a clear actionable change                    | Answer the question and work out the fix. Do not apply it.                 |
| 🤷 Unclear    | Ambiguous, architectural, or beyond this PR                | Lay out the options and ask the user. Do not guess.                        |

Always write the kind with its emoji (e.g. `❓ Question`) in the report.

Investigation is read-only: read code, docs, tests, and history, and run read-only checks if useful, but do not edit files.

### 5. Report in chat, then stop

One numbered section per comment, so the user can refer to them by number:

```markdown
## 1. [path:line or "conversation"] @author

**Kind:** ❓ Question | 💡 Suggestion | ☯️ Mixed | 🤷 Unclear
**Comment:** (one-line paraphrase)
**Assessment:** answer and/or whether the comment is right, with brief evidence
**Proposed fix:** (one short line; omit if no change is needed)
```

End the report by asking which fixes to make or change, then **stop and wait** for the user. Do not start implementing in the same turn.

### 6. Implement confirmed fixes

Implement only the fixes the user approved, with any changes they asked for. Follow existing repo patterns.

You may use subagents for **non-overlapping** fixes (different files, no shared types/state/behavior). Keep overlapping fixes and coordination in the parent.

- Give each subagent the PR context it needs (paths, comment body, relevant diff, approved fix). Tell it not to post on GitHub and not to commit.
- Do not run two subagents that would edit the same file or the same behavior.

Afterward, report per comment number what changed, the files touched, and any checks you ran. If you skipped or could not do something, say what you need from the user.
