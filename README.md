# faster-gh-cli-skill

GitHub's [`gh` CLI](https://cli.github.com/) is usually the fastest and safest way for an agent to work with GitHub. This skill exists to make those `gh` interactions more reliable: fewer browser clicks, fewer malformed PR bodies, fewer guessed JSON fields, and fewer dead-end retries after permission errors.

## Install

```sh
npx skills add zeke/faster-gh-cli-skill
```

## What it covers

- Prefer `gh` and `gh api` over browser automation for GitHub work.

- Always establish auth, repo, and branch context before reading or changing GitHub state.

- Use `--repo owner/name` whenever the target repo is not unambiguous from the current working directory.

- Use `--json` and `--jq` for structured reads, plus a safe way to discover valid JSON fields when the agent is unsure.

- Known JSON field gotchas from real sessions: `defaultBranch` should be `defaultBranchRef`, `stargazersCount` should be `stargazerCount`, `topics` is not a `gh repo view` field, and some search results do not expose fields like `mergedAt`.

- Safe Markdown bodies for PRs, issues, comments, and gists using single-quoted heredocs so shell backticks in Markdown do not execute as commands.

- PR workflows: inspect branch state before `gh pr create`, use `--head user:branch` for forks, avoid retrying branch errors blindly, and use supported diff/check commands.

- Actions workflows: use `gh pr checks`, `gh run list`, `gh run view --log-failed`, and avoid unattended `gh run watch` unless waiting is explicitly part of the task.

- `gh api` type handling: use `--field enabled:=true` for booleans, `--raw-field` for strings, and `--input` for complex JSON. Avoid sending string booleans to APIs that require real booleans.

- Secret handling: list secret names safely and set secret values from environment variables, never hardcode them in commands or files.

- Gist gotchas: `gh gist create --private` is not a valid flag; omit `--public` for a secret gist.

## How it was made

This skill was built by mining OpenCode's local session history: a SQLite database of tool calls and responses across many months of real GitHub CLI work.

The analysis covered:

- 1,228 bash tool invocations beginning with `gh`
- The most common successful commands, including `gh pr create`, `gh pr view`, `gh run view`, `gh issue view`, `gh issue create`, `gh repo view`, `gh pr list`, `gh pr checks`, and `gh api`
- Failure clusters for invalid JSON fields, invalid flags, shell quoting bugs, GraphQL permission errors, HTTP 422 request-shape errors, network failures, clone timeouts, and unattended auth refresh flows
- Real command bodies that succeeded, especially the heredoc pattern for multiline PR, issue, and comment text
- Current `gh` behavior on this machine with `gh version 2.92.0`
- The Agent Skills specification and authoring guidance from [agentskills.io](https://agentskills.io/)

The raw session logs are not included. The skill contains only distilled command patterns and aggregate findings.

## License

MIT
