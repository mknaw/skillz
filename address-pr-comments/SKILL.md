---
name: address-pr-comments
description: Pull and review unresolved PR comments for the current branch. Use when user wants to see or address PR review feedback.
allowed-tools: Bash(gh *)
---

# Address PR Comments

## Fetching the PR

Use `gh pr view` to resolve the PR for the current branch — do NOT use `gh pr list`.

```bash
gh pr view --json number,title,url
```

## Fetching Unresolved Comments

Pull inline review comments via the API:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments
```

To get the owner/repo, use:

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

## Output

For each unresolved comment, display:
1. The original comment (file, line range, and body)
2. Your assessment of whether the comment is worth addressing, with brief reasoning
