---
name: lint-review
description: Lint your own PR review comments for factual accuracy, tone, clarity, and relevance. Checks pending (draft) comments first, falls back to submitted. Use when user wants to sanity-check their review before or after submitting.
allowed-tools: Bash(gh *), Read, Grep, Agent
---

# Lint PR Review

Audit the authenticated user's review comments on a PR for quality issues.

You are an **orchestrator**. Fetch comments, group them, then delegate evaluation to subagents in parallel. Do NOT evaluate comments yourself — that's the subagents' job.

## Phase 1: Fetch

### Identifying the PR

Accept a PR number or URL as an argument. If none provided, use `gh pr view` on the current branch. If the current branch has no PR, ask.

```bash
gh pr view --json number,url,headRefName,baseRefName
```

### Resolving the Repo

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

### Fetching Review Comments

#### Get the authenticated user's login

```bash
gh api user -q .login
```

#### Try pending (draft) reviews first

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --jq '.[] | select(.state == "PENDING" and .user.login == "{username}")'
```

If a PENDING review exists, fetch its comments:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews/{review_id}/comments
```

#### Fall back to submitted reviews

If no pending review, fetch submitted review comments by the user:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --jq '[.[] | select(.user.login == "{username}")]'
```

## Phase 2: Group

Pre-group the fetched comments **thematically by file or feature area** before delegating. Each group should contain comments that share enough context that a single subagent can evaluate them efficiently — typically comments on the same file or closely related files.

For each group, prepare a briefing that includes:
- The comment bodies with their file paths and line numbers
- The `diff_hunk` from the API response for each comment
- The list of source files the subagent will need to read

If there are very few comments (say, 3 or fewer), a single subagent is fine. Otherwise, target 3-6 comments per group.

## Phase 3: Delegate

Spawn one subagent **per group**, in parallel. Each subagent prompt must be self-contained — include the full briefing from Phase 2 and the evaluation criteria below. Tell the subagent which source files to read for verification.

### Subagent Evaluation Criteria

Each subagent evaluates its assigned comments against these criteria, **in priority order**:

#### 1. Factual Correctness (Critical)

The most important check. Verify claims about what the code does, how it behaves, or what would happen if changed.

- Read the actual code around the commented lines — don't rely only on the diff hunk
- Check that any API, language, or framework claims are accurate
- Flag comments that assert something incorrect about the code's behavior
- Severity: **ERROR** — these undermine the reviewer's credibility

#### 2. On-Topic (Important)

Is the comment relevant to the change in this PR?

- Flag tangential suggestions that belong in a separate ticket/PR
- Nit-level style comments on unchanged code are off-topic
- Severity: **WARNING**

#### 3. Tone (Moderate)

Firm and direct is fine. Flag only genuine excesses.

**Acceptable (do NOT flag):**
- "This is wrong because..." — direct and factual
- "I don't think this handles X" — clear disagreement
- "Nit: prefer Y here" — standard review shorthand
- Dry humor or light personality

**Flag these:**
- Dismissive or condescending language ("obviously", "this is clearly wrong", "did you even test this")
- Sarcasm that could sting ("I see we're just ignoring error handling now")
- Piling on — multiple comments making the same point with escalating frustration
- Imperative commands without rationale ("Just do X")
- Severity: **WARNING**

#### 4. Clarity (Moderate)

Would a competent developer who is not a native English speaker understand this?

- Flag dense idioms or slang that don't translate well ("this is a footgun", "smells off")
- Flag ambiguous pronouns or references ("this should be changed" — which 'this'?)
- Flag comments that assume context the author may not have
- Do NOT flag standard programming jargon (refactor, race condition, etc.) — that's expected vocabulary
- Severity: **INFO**

### Subagent Output Format

Each subagent must return its findings as a structured list. For each comment that has findings:

1. **File and line** — `path/to/file.rs:42`
2. **Comment body** — the review comment text
3. **Findings** — list of issues with severity tag: `[ERROR]`, `[WARNING]`, or `[INFO]`
4. **Suggested revision** — only for ERROR and WARNING level findings; a concrete rewrite of the comment

Skip comments with no findings.

## Phase 4: Synthesize

Collect results from all subagents and present a unified report.

### Per-Comment Assessment

Show findings from all subagents, ordered by severity (ERRORs first, then WARNINGs, then INFOs).

### Summary

At the end, provide:
- Total comments reviewed
- Counts by severity
- One-line overall verdict: **SUBMIT** (no errors, few warnings), **REVISE** (errors or many warnings), or **RETHINK** (multiple factual errors)
