---
name: jira
description: Manage Jira tickets - search, view, create, edit, transition, and comment on issues. Use when user asks about Jira tickets, wants to create issues, check ticket status, or update work items.
allowed-tools: Bash(acli *)
---

# Jira Ticket Management

Use `acli` to manage Jira tickets. Requires prior authentication via `acli auth login`.

## Reading Tickets

View a ticket:
```bash
acli jira workitem view KEY-123
acli jira workitem view KEY-123 --json
acli jira workitem view KEY-123 --fields "summary,description,comment,status"
```

Search with JQL:
```bash
acli jira workitem search --jql "project = PROJ"
acli jira workitem search --jql "assignee = currentUser() AND status != Done"
acli jira workitem search --jql "project = PROJ AND type = Bug" --limit 20
```

List projects:
```bash
acli jira project list --recent
```

## Creating Tickets

```bash
acli jira workitem create \
  --project "PROJ" \
  --type "Task" \
  --summary "Ticket title" \
  --description "Description here" \
  --assignee "@me"
```

To create a child task under an epic (or any parent), use `--parent`:
```bash
acli jira workitem create \
  --project "PROJ" \
  --type "Task" \
  --parent "PROJ-100" \
  --summary "Child task title" \
  --description "Description here" \
  --assignee "@me"
```

**Note:** `--parent` is only available on `create`, not `edit`. To reparent an existing ticket, delete and recreate it with `--parent`.

Types: Task, Bug, Story, Epic

### Guidelines for Writing Ticket Descriptions

When creating tickets, follow these principles:

1. **Begin with a problem statement** - Explain the issue, gap, or need being addressed. Focus on the "what" and "why", not the "how".

2. **Avoid implementation details** - Descriptions should not prescribe specific technical solutions, code changes, or implementation approaches. Leave those decisions to the implementer.

3. **End with Acceptance Criteria** - Every ticket must include an "Acceptance Criteria" section with one or more concise bullet points describing what "done" looks like.

**Before creating a ticket**: If the problem statement or acceptance criteria are unclear from the user's request, ask clarifying questions rather than guessing.

Example structure:
```
[Problem statement explaining the issue or need]

Context or additional details if needed...

## Acceptance Criteria
- Criterion 1
- Criterion 2
```

## Editing Tickets

```bash
acli jira workitem edit --key "KEY-123" --summary "New title"
acli jira workitem edit --key "KEY-123" --assignee "user@example.com"
acli jira workitem edit --key "KEY-123" --description "Updated description"
```

## Transitioning Status

```bash
acli jira workitem transition --key "KEY-123" --status "In Progress"
acli jira workitem transition --key "KEY-123" --status "Done"
```

## Comments

```bash
acli jira workitem comment create --key "KEY-123" --body "Comment text"
acli jira workitem comment list --key "KEY-123"
```

## Tips
- Use `--json` for structured output when parsing is needed
- Use `--yes` to skip confirmation prompts for bulk operations
- JQL supports: project, assignee, status, type, priority, created, updated
