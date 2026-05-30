---
name: faster-gh-cli-skill
description: >
  GitHub CLI gh usage guide for agents. Load this skill whenever you are about
  to use gh, GitHub CLI, GitHub PRs, issues, comments, reviews, Actions runs,
  repositories, secrets, gists, or gh api. Covers reliable command patterns,
  JSON field gotchas, shell quoting for Markdown bodies, non-interactive auth,
  pull request workflows, Actions log inspection, and common gh failure modes
  observed in real OpenCode sessions.
license: MIT
compatibility: Requires gh, git, jq for some examples, GitHub network access, and an authenticated gh session.
---

# GitHub CLI: faster patterns

Use `gh` for GitHub work whenever possible. It is fast, scriptable, and easy to audit.

## First checks

Before doing GitHub work, establish context:

```sh
gh auth status
gh repo view --json nameWithOwner,url,defaultBranchRef
git status
```

If the target repository is not definitely the current working directory, pass `--repo owner/name`. This avoids reading or editing the wrong repo when the agent is running from a different checkout.

For commands that change GitHub state, inspect first. Examples:

```sh
gh pr view 123 --repo owner/repo --json number,title,state,url,headRefName,baseRefName
gh issue view 456 --repo owner/repo --json number,title,state,url
gh run list --repo owner/repo --limit 10
```

## Structured output

Prefer `--json` with `--jq` for reads. It is more reliable than parsing human output.

```sh
gh pr view 123 --repo owner/repo \
  --json number,title,state,url,headRefName,baseRefName,author,commits,files \
  --jq '{number,title,state,url,head: .headRefName, base: .baseRefName}'
```

When unsure which JSON fields are valid, deliberately trigger the field list with a harmless invalid field, then retry with valid fields:

```sh
gh repo view owner/repo --json _fields 2>&1
```

Common field corrections observed in real sessions:

| Wrong field                    | Use instead                                |
| ------------------------------ | ------------------------------------------ |
| `defaultBranch`                | `defaultBranchRef`                         |
| `stargazersCount`              | `stargazerCount`                           |
| `merged`                       | `state`, `mergedAt`, or `closed`           |
| `mergedAt` on search results   | `closedAt` or use `gh pr view`             |
| `topics` on `repo view`        | Use `gh api repos/owner/repo/topics`       |

`gh search prs --state merged` is invalid. Search supports `--state open` or `--state closed`; use `--merged` to filter merged PRs.

```sh
gh search prs "query" --repo owner/repo --merged --limit 20 --json number,title,url,closedAt
```

## Markdown bodies

Never put Markdown containing backticks inside a double-quoted `--body "..."` string. The shell will execute backticked text before `gh` sees it.

Use a single-quoted heredoc:

```sh
gh pr create --repo owner/repo \
  --head user:branch-name \
  --base main \
  --title "fix: describe the change" \
  --body "$(cat <<'EOF'
This PR fixes the issue.

```ts
const example = true
```
EOF
)"
```

Use the same pattern for `gh pr edit`, `gh issue create`, `gh issue edit`, `gh pr comment`, and `gh issue comment` when the body is more than one line.

## Pull requests

Read a PR:

```sh
gh pr view 123 --repo owner/repo \
  --json number,title,body,state,url,headRefName,baseRefName,author,commits,files,additions,deletions
```

List PRs by branch:

```sh
gh pr list --repo owner/repo --state all --head branch-name --json number,title,state,url
```

Create a PR only after confirming the branch exists remotely and has commits relative to the base:

```sh
git branch --show-current
git status
git log --oneline --decorate -10
git fetch origin
git log --oneline origin/main..HEAD
git push -u origin HEAD
gh pr create --repo owner/repo --base main --head branch-name --title "fix: title" --body "$(cat <<'EOF'
This PR...
EOF
)"
```

If creating a PR from a fork, include the owner in `--head`:

```sh
gh pr create --repo upstream/repo --head your-user:branch-name --base main --title "fix: title" --body "..."
```

`GraphQL: Head sha can't be blank`, `Base sha can't be blank`, `No commits between`, or `Head ref must be a branch` means the branch/base is wrong, the branch was not pushed, or there are no commits to compare. Do not retry blindly. Inspect branch names and push state first.

Diff a PR:

```sh
gh pr diff 123 --repo owner/repo
gh pr diff 123 --repo owner/repo --name-only
```

`gh pr diff --stat` is not supported. For stats, use `gh pr view --json additions,deletions,changedFiles`.

Review or comment on a PR:

```sh
gh pr comment 123 --repo owner/repo --body "$(cat <<'EOF'
Comment body.
EOF
)"
```

For inline review comments, prefer `gh api` only after reading existing comments and the PR files. GitHub's review comment API is strict about `commit_id`, `path`, `side`, `line`, and `start_line`; invalid positioning returns HTTP 422. If responding to an existing inline comment, reply to that comment instead of creating a new top-level PR comment.

## Issues

Read issue details:

```sh
gh issue view 123 --repo owner/repo --json number,title,body,state,url,author,comments
```

List issues:

```sh
gh issue list --repo owner/repo --state open --limit 50 --json number,title,state,url,labels
```

Create an issue:

```sh
gh issue create --repo owner/repo --title "docs: add homepage" --body "$(cat <<'EOF'
Issue body.
EOF
)"
```

## Actions and checks

For PR checks:

```sh
gh pr checks 123 --repo owner/repo
gh pr checks 123 --repo owner/repo --watch
```

For workflow runs:

```sh
gh run list --repo owner/repo --branch branch-name --limit 10 --json databaseId,status,conclusion,displayTitle,workflowName,createdAt,url
gh run view <run-id> --repo owner/repo --json status,conclusion,url,jobs
gh run view <run-id> --repo owner/repo --log-failed
```

`gh run watch <run-id> --exit-status` is useful but can run until timeout. Use it only when waiting is explicitly part of the task, and set an appropriate command timeout.

If `gh workflow run` returns `HTTP 422: Workflow does not have 'workflow_dispatch' trigger`, the workflow cannot be manually dispatched. Inspect the workflow file instead of retrying.

## Repositories

Read repository metadata:

```sh
gh repo view owner/repo --json nameWithOwner,description,url,homepageUrl,defaultBranchRef,isFork,parent
```

List repositories:

```sh
gh repo list owner --limit 100 --json nameWithOwner,description,url,updatedAt,isPrivate
```

`gh repo list --sort` is not supported. Sort with `--jq` or `jq` after fetching JSON.

Create a repository:

```sh
gh repo create owner/name --private --source=. --description "Short description" --homepage "https://example.com" --push
```

Org repository creation often fails with `GraphQL: ... correct permissions to execute CreateRepository`. If that happens, stop and report the permission issue; do not keep changing flags.

For large clones that hit timeouts or sideband disconnects, retry with Git directly and consider shallow clone:

```sh
git clone --depth 1 https://github.com/owner/repo.git
```

## gh api

Use `gh api` when the dedicated `gh` command cannot do the job.

Read paginated API data:

```sh
gh api repos/owner/repo/pulls/123/files --paginate --jq '.[] | {filename,status,patch}'
```

Fetch raw file content:

```sh
gh api repos/owner/repo/contents/path/to/file -H "Accept: application/vnd.github.raw"
```

Be careful with value types:

| Need             | Pattern                                      |
| ---------------- | -------------------------------------------- |
| String field     | `--raw-field name=value`                     |
| Typed JSON value | `--field count:=10 --field enabled:=true`    |
| Complex JSON     | `--input file.json`                          |

Do not use `-f enabled=true` when the API requires a boolean. That sends the string `"true"` and often returns HTTP 422.

Permission errors are usually real. If `gh api` says the operation needs `admin:org`, `workflow`, or another scope, check `gh auth status`; do not start `gh auth refresh` unless the user can complete the interactive device flow.

## Secrets

List secret names only:

```sh
gh secret list --repo owner/repo
```

Set secrets from environment variables. Never put secret values directly in a command, PR body, issue, or file:

```sh
gh secret set SECRET_NAME --body "$SECRET_NAME" --repo owner/repo
```

If an environment variable may be empty, check first:

```sh
test -n "$SECRET_NAME" && gh secret set SECRET_NAME --body "$SECRET_NAME" --repo owner/repo
```

## Gists

Create a public gist from stdin:

```sh
gh gist create --public --filename "notes.md" --desc "Short description" - <<'EOF'
# Notes
EOF
```

`gh gist create --private` is invalid. Omit `--public` for the default secret gist behavior, or include `--public` for a public gist.

If using stdin, include `-` and provide content. A blank stdin returns `a gist file cannot be blank`.

## Anti-patterns

| Anti-pattern                                  | Instead                                                   |
| --------------------------------------------- | --------------------------------------------------------- |
| Relying on current repo from an unknown cwd    | Pass `--repo owner/name`                                  |
| Parsing human output when JSON is available   | Use `--json` and `--jq`                                   |
| Guessing JSON fields repeatedly               | Ask the command for available fields, then retry          |
| Markdown body inside double quotes            | Use `--body "$(cat <<'EOF' ... EOF )"`                   |
| Retrying permission errors with random flags   | Check auth scopes and user/org permissions                |
| `gh auth refresh` in an unattended session     | Stop and ask the user to complete auth                    |
| `gh pr diff --stat`                           | `gh pr view --json additions,deletions,changedFiles`      |
| `gh gist create --private`                    | Omit `--public` for secret gists                          |
| `-f enabled=true` for API booleans             | `--field enabled:=true` or `--input` with JSON            |

## Recovery checklist

When a `gh` command fails:

1. Read the exact error text.
2. If it is an unknown JSON field or flag, inspect the valid fields/usage and retry once.
3. If it is a branch or PR creation error, inspect `git status`, current branch, remotes, and `origin/base..HEAD`.
4. If it is HTTP 401, 403, or a GraphQL permission error, check `gh auth status` and stop if the user lacks permission.
5. If it is HTTP 422, inspect request shape, typed fields, and required positioning parameters before retrying.
6. If it is network or timeout related, retry once with a clearer timeout or narrower request.
7. Do not loop on the same command more than twice without changing the diagnosis.
