---
name: address-pr-comments
description: >-
  Addresses GitHub pull request comments using the gh CLI. Use when the user
  provides a PR link/URL and wants review comments addressed, answered, or
  implemented. Understands the PR first, implements suggestions in code, and
  answers questions in chat without posting on GitHub.
---

# Address PR comments

The user invokes this skill with a GitHub PR link. Use `gh` for all GitHub reads. Never post on GitHub.

## Rules

- **Never** reply, resolve, or otherwise write to the PR (`gh pr comment`, review replies, GraphQL mutations, emoji reactions). Code and docs in the repo are fine; leave the GitHub conversation to the user.
- Do not commit or push unless the user asks.
- If the user did not give a PR URL, check for PRs against the current branch.

## Workflow

1. Fetch PR metadata, diff, and comments with gh
2. Check out the PR branch if needed
3. Read the diff and codebase until the change is understood
4. Read the PR comments
5. Classify and act (questions in chat, suggestions in code)
6. Report in chat

### 1. Fetch with gh

Run `gh` via the Shell tool (needs network; use `required_permissions: ["all"]` when auth or git checkout requires it).

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

Implementations must land on the PR head. If the working tree is not already that branch:

```bash
gh pr checkout "$PR"
```

Do not discard uncommitted local work. If checkout would clobber changes, stop and ask the user.

### 3. Understand the change first

Read the diff, then the surrounding codebase, until you understand what the PR does and why. **Do not read or act on comments until this is done.** Comments are easy to misread without that context.

### 4. Read comments, then classify

For each remaining comment, choose one:

| Kind          | What it is                                                 | Action                                                                     |
| ------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------- |
| ❓ Question   | Asks why/how, requests clarification, or seeks information | Research (code, docs, tests, diff). Reply **in this chat**, not on GitHub. |
| 💡 Suggestion | Proposes a code/doc change, a fix, a nit, or a bot finding | Implement it, following existing repo patterns.                            |
| ☯️ Mixed      | Question plus a clear actionable change                    | Implement the change and answer the question in chat.                      |
| 🤷 Unclear    | Ambiguous, architectural, or beyond this PR                | Ask the user in chat. Do not guess.                                        |

Always write the kind with its emoji (e.g. `❓ Question`) in the report.

### 5. Optional subagents

You may use subagents for **non-overlapping** suggestion work (different files, no shared types/state/behavior). Keep overlapping comments, questions, and coordination in the parent.

- Give each subagent the PR context it needs (paths, comment body, relevant diff). Tell it not to post on GitHub and not to commit.
- Do not run two subagents that would edit the same file or the same behavior.
- Synthesize subagent results into the chat report.

### 6. Report in chat

One section per comment:

```markdown
## [path:line or "conversation"] @author

**Kind:** ❓ Question | 💡 Suggestion | ☯️ Mixed | 🤷 Unclear
**Comment:** (one-line paraphrase)
**Response / change:** answer and/or what you implemented
```

If you implemented code, list the files touched and any checks you ran. If you skipped or could not act, say what you need from the user.