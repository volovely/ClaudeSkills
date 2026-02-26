---
name: pr-create
description: Use when the user asks to "create a PR", "open a pull request", or says "pr-create". Gathers diff context, fetches Jira ticket details, generates title and description, then opens the browser for the user to review and submit.
argument-hint: "[base branch]"
---

# PR Create Skill

Create a GitHub pull request with context from the code diff and Jira, then open the browser for final review.

## Usage

```
/pr-create
/pr-create develop
/pr-create main
```

- If no base branch is provided, defaults to `develop`.

## Instructions

When this skill is invoked, follow these steps:

### Step 1: Parse Input and Determine Context

1. **Base branch:** Use the argument if provided, otherwise default to `develop`.
2. **Current branch:** Run `git rev-parse --abbrev-ref HEAD` to get the current branch name.
3. **Extract Jira ticket key:** Parse the Jira ticket number from the beginning of the branch name. Branch names follow the pattern `<TICKET-KEY>-<description>` (e.g., `IOS-17211-Implement-Snapping` → ticket key is `IOS-17211`). The ticket key is the first segment matching the pattern of uppercase letters, a hyphen, and digits (e.g., `IOS-17211`, `PROJ-456`).

If the current branch is the same as the base branch (e.g., both are `develop`), stop and inform the user that they need to be on a feature branch.

### Step 2: Gather Diff and Jira Context

Run these operations **in parallel**:

1. **Diff between current branch and base branch:**
   ```
   git diff <base-branch>...HEAD
   ```

2. **Commit log since divergence from base branch:**
   ```
   git log <base-branch>..HEAD --oneline
   ```

3. **Check if branch is pushed to remote:**
   ```
   git rev-parse --abbrev-ref --symbolic-full-name @{upstream}
   ```
   If this fails, the branch has no upstream — it will need to be pushed.

4. **Fetch Jira ticket details:** Use `mcp__atlassian__getAccessibleAtlassianResources` to get the cloud ID, then use `mcp__atlassian__getJiraIssue` with:
   - `issueIdOrKey`: the extracted ticket key
   - `fields`: `["summary", "description", "status"]`

### Step 3: Understand the Changes

Before generating the PR description, build a solid understanding of what changed:

1. **Categorize changed files** from the diff (models, views, features, tests, configs, assets).
2. **Read key changed files** in full using the Read tool — not just the diff hunks. Focus on files with significant logic changes, new types, or API modifications.
3. **Follow references** if needed — use Grep or Glob to understand how new or modified types/functions are used elsewhere in the codebase.
4. **Summarize the change** in your own words: what was added, modified, or fixed, and why (using the Jira ticket description as context for the "why").

### Step 4: Compose the PR Title and Body

**Title:** `<TICKET-KEY> <Jira ticket summary>` — start with the Jira ticket key followed by the issue summary (title) exactly as it appears in Jira (e.g., `IOS-17211 Implement Snapping`).

**Body:** Follow the project's PR template format:

```markdown
## SUMMARY

<A clear, concise description of the changes based on your analysis from Step 3.
Include:
- What was changed and why
- Key implementation decisions
- Notable new types, patterns, or dependencies introduced>

## JIRA

https://snapfish-llc.atlassian.net/browse/<TICKET-KEY>
```

Keep the summary factual and focused. Use bullet points for multiple distinct changes. Do not pad with obvious or redundant information.

**Do NOT** include any `Co-Authored-By` lines, Claude attribution, or AI mentions in the PR title or body.

### Step 5: Push Branch if Needed

If Step 2 determined the branch has no upstream:

```
git push -u origin <current-branch>
```

If the branch already has an upstream, check if there are unpushed commits:

```
git status -sb
```

If there are unpushed commits, push them:

```
git push
```

### Step 6: Present the PR Preview and Open Browser

Show the user the composed PR details:

```markdown
# PR Preview

**Title:** <title>
**Base:** <base-branch> ← **Head:** <current-branch>
**Jira:** https://snapfish-llc.atlassian.net/browse/<TICKET-KEY>

## Body
<the composed body>
```

Then open the browser for the user to review and finalize:

```
gh pr create --base <base-branch> --title "<title>" --body "<body>" --web
```

**Important shell quoting:** The title and body must be properly escaped. Use a HEREDOC or single-quotes to prevent shell interpretation of special characters. If the title or body contains single quotes, escape them appropriately.

Tell the user that the browser has been opened and they can review, edit, and submit the PR from there.

**Do NOT** create the PR without `--web`. The user must always have the opportunity to review and modify before submission.
