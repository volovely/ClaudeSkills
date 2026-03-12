---
name: pr-commit-squash-message
description: Use when the user wants to squash commits and needs a commit title and message, says "squash commit message", "pr-commit-squash-message", "generate squash commit", "squash this PR", or provides a PR URL/number with intent to squash. Reads the PR and related Jira ticket to generate a clean, structured commit message ready to copy-paste.
argument-hint: "[PR number or URL]"
model: sonnet
---

# PR Commit Squash Message

Generate a single squash-commit title and bullet-point message from a PR and its Jira ticket.

## Usage

```
/pr-commit-squash-message
/pr-commit-squash-message 21014
/pr-commit-squash-message https://github.com/org/repo/pull/21014
```

- No argument: uses the open PR for the current branch.
- PR number or URL: fetches that specific PR.

## Instructions

### Step 1: Resolve the PR

**If a PR number or URL was provided**, extract the number and run:
```bash
gh pr view <number> --repo <repo> --json title,body,commits,headRefName
```

**If no argument was given**, resolve the PR from the current branch:
```bash
gh pr view --json title,body,commits,headRefName
```

Save the output to `/tmp/pr_squash.json`.

### Step 2: Compress PR data with toon

```bash
toon /tmp/pr_squash.json -o /tmp/pr_squash.toon
```

Read `/tmp/pr_squash.toon` — use **only** this for PR context. Do not hold the raw JSON in context.

### Step 3: Extract the Jira ticket key

Look in this order:
1. PR body: find a Jira URL like `https://snapfish-llc.atlassian.net/browse/IOS-XXXXX`
2. Branch name (`headRefName`): match the leading pattern `[A-Z]+-\d+` (e.g., `IOS-17216`)

If no ticket key is found, skip Steps 4–5 and go straight to Step 6 using only the PR data.

### Step 4: Fetch the Jira ticket

Use `mcp__atlassian__getJiraIssue` with:
- `cloudId`: `snapfish-llc.atlassian.net`
- `issueIdOrKey`: the extracted key
- `fields`: `["summary", "description", "parent", "status", "issuetype"]`
- `responseContentFormat`: `"markdown"`

From the response, extract only these fields and build a minimal JSON object:
```json
{
  "key": "IOS-17216",
  "summary": "...",
  "type": "Story",
  "status": "In Dev",
  "parent": { "key": "FTR-8752", "summary": "...", "type": "Epic" }
}
```

Save to `/tmp/jira_squash.json`.

### Step 5: Compress Jira data with toon

```bash
toon /tmp/jira_squash.json -o /tmp/jira_squash.toon
```

Read `/tmp/jira_squash.toon` — use **only** this for Jira context.

### Step 6: Generate the squash commit

Using the toon data from Steps 2 and 5, produce:

**Title** — one line, max ~72 characters:
```
<TICKET-KEY> <concise imperative verb phrase>
```
- Start with the Jira ticket key (e.g., `IOS-17216`).
- The verb phrase should describe the work in plain English (e.g., `Implement equal spacing alignment snapping`).
- Do not repeat the Jira summary verbatim if it's vague — distill what was actually done from the PR commits and body.

**Message** — bullet list of what changed, one bullet per logical change:
- Use `-` bullets, no sub-bullets.
- Each bullet is specific and technical: name the type/function/struct/file affected.
- Group by theme if natural (added, changed, fixed, removed), but don't add section headers — just order them logically.
- No filler phrases like "various improvements" or "code clean up".
- Typically 4–8 bullets; more is fine if the PR is large.

### Step 7: Present the result

Output the title and message as a single code block the user can copy directly into their squash editor:

```
<title line>

<bullet 1>
<bullet 2>
...
```

No extra commentary needed — the user just wants to copy-paste.
