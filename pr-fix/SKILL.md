---
name: pr-fix
description: This skill should be used when the user asks to "fix PR comments", "address PR feedback", "resolve PR comments", provides a GitHub PR URL with intent to fix, or says "pr-fix". It reads all review comments on a PR, decides which need code fixes vs. replies, makes the fixes, and posts replies — all with explicit user approval before any GitHub action.
argument-hint: "<PR URL or number>"
---

# PR Fix Skill

Address review comments on a GitHub pull request: fix code where needed, reply to discussion comments, and push changes — all with user approval at every step.

## Usage

```
/pr-fix <PR URL or number>
```

- Full URL: `/pr-fix https://github.com/org/repo/pull/123`
- PR number (uses current repo): `/pr-fix 123`

## Context

This project is a **Swift / SwiftUI / TCA (The Composable Architecture)** iOS codebase. Keep this in mind when reasoning about code changes — understand Reducer patterns, ViewStore usage, dependency injection, and the unidirectional data flow TCA enforces.

## Instructions

When this skill is invoked, follow these steps **in order**.

### Step 1: Parse Input and Fetch PR Data

Extract owner, repo, and PR number from the argument.

- If a full URL is provided, parse `owner/repo` and PR number.
- If only a number is given, use the current repository context.

Run these commands **in parallel** to gather everything:

1. **PR metadata:**
   ```
   gh pr view <number> --repo <owner/repo> --json title,body,author,baseRefName,headRefName,number,url
   ```

2. **PR diff:**
   ```
   gh pr diff <number> --repo <owner/repo>
   ```

3. **Review comments (line-level):**
   ```
   gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
   ```

4. **Review-level comments and verdicts:**
   ```
   gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate
   ```

5. **Issue-level (conversation) comments:**
   ```
   gh pr view <number> --repo <owner/repo> --json comments
   ```

6. **Current branch check:**
   ```
   git branch --show-current
   ```

### Step 2: Build a Comment Map

For every review comment and conversation comment:

1. **Group by thread** — connect replies to their parent so you see the full conversation.
2. **Record metadata** for each thread:
   - Comment ID, author, body, file path, line number, `in_reply_to_id`, created/updated timestamps.
   - Whether the thread is resolved or still open.
3. **Skip** comments authored by the PR author (the user) — those are self-replies, not reviewer feedback.
4. **Skip** already-resolved threads unless explicitly asked not to.
5. **Skip** bot comments (GitHub Actions, CI bots, etc.).

### Step 2.5: Explore Local Repository for Context

Before deciding on actions, build deeper understanding by reading the actual source files — the diff alone is not enough.

1. **Read every file touched by the PR** in full (not just the diff hunks). Use the Read tool to load each changed file so you understand the surrounding code, class hierarchy, and how the changed code fits into the broader module.
2. **Follow references.** If a comment mentions a type, protocol, or function defined elsewhere, use Grep or Glob to find its definition and read it. Understand how it's used across the codebase.
3. **Check related files.** For each changed file, look for:
   - The parent Reducer/Feature if this is a child feature (search for `Scope` or `ifLet` referencing this feature's state/action).
   - The View that consumes this Reducer's store (or vice versa).
   - Tests that cover the changed code.
   - Shared models or dependencies that the changed code relies on.
4. **Understand patterns in use.** If you're unsure whether a reviewer's suggestion matches the project's conventions, search the codebase for similar patterns. For example, if they suggest a different naming convention, check how other files in the same module handle it.

This local exploration is critical — it prevents you from making fixes that break callers, miss context, or contradict established patterns elsewhere in the project.

### Step 3: Analyze Each Comment — Decide Action

For **each remaining comment thread**, decide one of:

| Decision | When to use |
|----------|-------------|
| **FIX** | The reviewer points out a concrete code issue, requests a change, or suggests an improvement that is correct and reasonable. |
| **REPLY** | The comment is a question, a discussion point, a misunderstanding, a style preference you disagree with, or something already handled. |
| **ACK** | The reviewer left a simple acknowledgment, praise, or informational note — no action needed, but a brief "thanks" or acknowledgment reply is polite. |

**Decision guidelines — be critical but fair:**

- **Assume the reviewer has a point.** Read the comment charitably. If their suggestion improves correctness, safety, readability, or aligns better with Swift/TCA idioms — it's a FIX.
- **Push back when warranted.** If the suggestion would make the code worse, contradicts TCA patterns, introduces unnecessary complexity, or is based on a misunderstanding of the context — it's a REPLY with a clear, respectful explanation.
- **Don't blindly agree.** A "suggestion" syntax block from GitHub doesn't automatically mean "apply this." Evaluate whether the suggested change is actually better.
- **Consider the full picture.** Some comments may reference earlier conversation. Read the whole thread before deciding.

### Step 4: Present the Plan to the User

Display a clear, structured summary of what you intend to do:

```markdown
# PR Fix Plan: [PR Title]
**PR #[number]** — [url]

---

## Code Fixes ([count])

| # | File | Comment | Planned Change |
|---|------|---------|----------------|
| 1 | `path/to/File.swift:42` | [Reviewer]: "[short quote]" | [Brief description of the fix] |
| 2 | ... | ... | ... |

## Replies ([count])

| # | Location | Comment | Planned Reply |
|---|----------|---------|---------------|
| 1 | `path/to/File.swift:10` | [Reviewer]: "[short quote]" | [Your draft reply] |
| 2 | General | [Reviewer]: "[short quote]" | [Your draft reply] |

## Acknowledgments ([count])

| # | Location | Comment | Reply |
|---|----------|---------|-------|
| 1 | General | [Reviewer]: "[short quote]" | [Brief ack] |
```

Then ask:

> Please review the plan above. You can:
> 1. **Approve all** — proceed with all fixes and replies
> 2. **Edit** — tell me which items to change (e.g., "change #2 from FIX to REPLY because…", "update reply #3 to say…")
> 3. **Skip some** — tell me which items to skip entirely
> 4. **Cancel** — abort, no changes

**Do NOT proceed until the user explicitly approves.**

### Step 5: Verify Branch

Before making any code changes:

1. Check that the current branch matches the PR's `headRefName`.
2. If it does NOT match:
   - Tell the user: "You're on branch `X` but the PR is on branch `Y`. Please switch to the correct branch or confirm you want me to check it out."
   - If user confirms, run `git checkout <headRefName>` and `git pull`.
3. Ensure the working tree is clean (`git status`). If there are uncommitted changes, warn the user and ask how to proceed.

### Step 6: Apply Code Fixes

For each approved FIX item:

1. Read the file to understand surrounding context.
2. Make the change using the Edit tool.
3. Keep changes minimal and focused — fix exactly what the reviewer asked for, nothing more.
4. If a fix touches TCA Reducer logic, verify that:
   - State mutations happen in the reducer, not in the view.
   - Effects return appropriate actions.
   - Dependency injection patterns are preserved.

After all fixes are applied:

1. Run a build check if possible (`xcodebuild build` or the project's build command) to verify nothing is broken.
2. If the build fails, diagnose and fix. If you can't fix it, revert and inform the user.

### Step 7: Prepare Commit

Stage all changed files and present the commit for approval:

```markdown
## Commit Preview

**Files changed:**
- `path/to/File1.swift`
- `path/to/File2.swift`

**Commit message:**
> [Title: concise summary of what was fixed]
>
> [Body: list which reviewer comments were addressed]
> - Fixed [description] (comment by @reviewer)
> - Updated [description] (comment by @reviewer)

**Diff:**
[Show `git diff --staged` output]
```

Ask:

> Does this commit look good? Should I push it to `<branch>`?

**Do NOT commit or push until the user explicitly approves.**

### Step 8: Push Changes

Once approved:

```
git add <specific files>
git commit -m "<approved message>"
git push origin <headRefName>
```

**Do NOT** include any `Co-Authored-By` lines or AI/Claude attribution in the commit message.

### Step 9: Post Replies on GitHub

After the push (or immediately if there are no code fixes), post the approved replies.

**Do NOT** mention Claude, AI, or any automated tooling in reply text.

**Shell quoting rule:** Always use **single quotes** for the `-f body=` parameter to prevent shell interpretation of backticks, dollar signs, and other special characters in the reply text. Double quotes will cause backtick-enclosed text (e.g., \`MyType\`) to be interpreted as command substitution.

**For line-level review comments**, reply in-thread using the **pull request review comments** endpoint. The `in_reply_to` field requires the numeric comment ID from the original review comment:
```
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -f body='<reply text>' \
  -F in_reply_to=<parent_comment_id>
```

**Important:** The PR number **must** be included in the URL path (`/pulls/{number}/comments`). Omitting it (e.g., `/pulls/comments`) will return a 404 error.

**For issue-level (conversation) comments:**
```
gh pr comment <number> --repo <owner/repo> --body '<reply text>'
```

For FIX items that were pushed, the reply should reference the fix:
> Fixed in the latest push. [Brief description of what was changed.]

For REPLY items, use the approved reply text as-is.

For ACK items, post the brief acknowledgment.

### Step 10: Summary

After everything is done, present a final summary:

```markdown
# Done!

- **Pushed:** [N] code fixes to `<branch>`
- **Replied:** [N] comments
- **Acknowledged:** [N] comments

All reviewer feedback on PR #[number] has been addressed.
```

## Tone Guidelines for Replies

- Be respectful and professional. The reviewer took time to review your code.
- When disagreeing, explain **why** with concrete reasoning — reference TCA patterns, Swift semantics, or the specific context of the change.
- Don't be defensive. If the reviewer is right, say so directly: "Good catch, fixed."
- Keep replies concise. One to three sentences is usually enough.
- If a reviewer's suggestion is partially right, acknowledge the valid part and explain what you'd do differently.
- Never be dismissive. Even if a comment seems trivial, respond politely.

## Error Handling

- If `gh` commands fail (auth issues, rate limits), inform the user with the exact error and suggest fixes.
- If a file referenced by a comment no longer exists or the line numbers don't match, skip that comment and note it in the plan.
- If the PR has no open review comments, inform the user: "No open review comments found on this PR."
