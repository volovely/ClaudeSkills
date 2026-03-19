---
name: pr-review
description: This skill should be used when the user asks to "review a PR", "review a pull request", provides a GitHub PR URL, or says "pr-review". It performs a focused code review using GitHub CLI concentrating on modeling correctness and code readability.
argument-hint: "<PR URL or number>"
---

# PR Review Skill

Review a GitHub pull request with focus on **proper modeling** and **code readability**.

## Usage

```
/pr-review <PR URL or number>
```

- Full URL: `/pr-review https://github.com/org/repo/pull/123`
- PR number (uses current repo): `/pr-review 123`

## Instructions

When this skill is invoked, follow these steps:

### Step 1: Parse Input and Fetch PR Metadata

Extract the owner, repo, and PR number from the argument.

- If a full URL is provided, parse `owner/repo` and PR number from it.
- If only a number is given, use the current repository context.

Run these commands in parallel to gather PR context:

1. **PR details:**
   ```
   gh pr view <number> --repo <owner/repo> --json title,body,author,baseRefName,headRefName,files,additions,deletions,changedFiles
   ```

2. **PR diff:**
   ```
   gh pr diff <number> --repo <owner/repo>
   ```

3. **PR comments (existing review context):**
   ```
   gh pr view <number> --repo <owner/repo> --json comments,reviews
   ```

### JSON → TOON Conversion

Whenever a tool returns JSON data (especially `gh` CLI responses), convert it to TOON format before processing to reduce token usage. Use a temporary file:

```bash
echo '<json_response>' > /tmp/pr-review-tmp.json
toon /tmp/pr-review-tmp.json -o /tmp/pr-review-tmp.toon
```

Then read `/tmp/pr-review-tmp.toon` instead of using the raw JSON. Apply this to all JSON responses from `gh` commands (e.g., `gh pr view --json ...`).

### Step 2: Understand the Change Scope

Before reviewing line-by-line, build a mental model of the change:

- **What is the purpose?** Read the PR title and description.
- **Which files changed?** Categorize them (models, views, features, tests, configs).
- **How big is the change?** Note additions/deletions to calibrate review depth.

If the diff is very large (>2000 lines), focus on model types, public API surfaces, and architectural files first. Summarize utility/boilerplate changes at a higher level.

### Step 2.5: Explore Local Repository for Context

The diff alone is not enough for a quality review. Use the local codebase to build deeper understanding.

1. **Read every file touched by the PR** in full (not just the diff hunks). Use the Read tool to load each changed file so you understand the surrounding code, class hierarchy, and how the changed code fits into the broader module.
2. **Follow references.** If the diff introduces or modifies a type, protocol, or function, use Grep or Glob to find where it's used or defined elsewhere. Understand the call sites and consumers.
3. **Check related files.** For each changed file, look for:
   - The parent Reducer/Feature if this is a child feature (search for `Scope` or `ifLet` referencing this feature's state/action).
   - The View that consumes this Reducer's store (or vice versa).
   - Tests that cover the changed code.
   - Shared models or dependencies that the changed code relies on.
4. **Understand existing patterns.** Before flagging something as an issue, search the codebase for similar patterns. If the code follows an established convention used elsewhere in the project, it's not an issue — even if you'd personally do it differently.

This local exploration is critical — it prevents false positives from missing context and makes your review comments grounded in how the project actually works.

### Step 3: Review the Diff

Read the full diff carefully. For each changed file, evaluate against the review criteria below. Track every finding with its file path and line context.

#### Review Criteria

Focus exclusively on these two areas:

**A. Proper Modeling**
- Are types (structs, enums, classes, protocols) modeled correctly for the domain?
- Do type names accurately reflect what they represent?
- Are value types vs reference types chosen appropriately?
- Is the data model normalized or does it have unnecessary duplication?
- Are optionals used correctly — does `nil` have clear semantic meaning?
- Are enums used where a fixed set of cases exists, instead of stringly-typed values?
- Are protocol conformances appropriate and not overly broad?
- Are generics used where they simplify, not where they complicate?
- Is state modeled in a way that makes invalid states unrepresentable?
- Are identifiers strongly typed (e.g., `UserID` wrapper) vs raw `String`/`Int`?

**B. Code Readability**
- Can a new team member understand this code without extra explanation?
- Are names (variables, functions, types) self-documenting and intention-revealing?
- Is nesting depth reasonable (≤3 levels preferred)?
- Are functions short and focused on a single responsibility?
- Is control flow straightforward — minimal use of complex ternaries or nested closures?
- Are guard/early-return patterns used to reduce indentation?
- Is related logic grouped together, not scattered?
- Are magic numbers/strings extracted into named constants?
- Is there dead code, commented-out code, or TODOs that should be addressed?
- Are complex expressions broken into well-named intermediate values?

#### What NOT to Review

Do not comment on:
- Formatting or whitespace (handled by linters)
- Import ordering
- Minor style preferences that don't affect clarity
- Test coverage quantity

### Step 4: Produce the Review Summary

Present findings in this structure:

```markdown
# PR Review: [PR Title]
**PR #[number]** | [additions]+ / [deletions]- | [changedFiles] files

---

## Overview
[1-3 sentences: what this PR does and the overall impression]

## Modeling Findings
### Issues
- **[file:context]** — [Description of the modeling concern and suggested improvement]

### Suggestions
- **[file:context]** — [Non-blocking suggestion for better modeling]

## Readability Findings
### Issues
- **[file:context]** — [Description of the readability concern and suggested improvement]

### Suggestions
- **[file:context]** — [Non-blocking suggestion for better readability]

## Verdict
[APPROVE / REQUEST_CHANGES / COMMENT — with a 1-sentence rationale]
```

**Categorization rules:**
- **Issues** — Problems that should be fixed before merge (incorrect modeling, confusing logic, potential bugs from bad modeling).
- **Suggestions** — Non-blocking improvements that would make the code better but are not required.

If a section has no findings, omit it entirely. Do not pad the review with trivial observations.

### Step 5: Ask About Leaving Comments

After presenting the review summary, ask the user:

> Would you like me to leave these comments on the PR?

Provide these options:
1. **Leave all comments** — Post every finding as individual review comments on the relevant lines
2. **Leave only issues** — Post only the items marked as "Issues", skip suggestions
3. **Submit review summary only** — Post the summary as a single PR review comment
4. **Don't post anything** — Keep the review local, no GitHub activity

### Step 6: Post Comments (if requested)

**Do NOT** mention Claude, AI, or any automated tooling in review comments.

#### Step 6a: Check for an existing pending review

Before creating anything, check if the authenticated user already has a pending review on this PR:

```bash
gh api graphql -f query='
{
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <number>) {
      reviews(states: PENDING, first: 1) {
        nodes { id databaseId }
      }
    }
  }
}' --jq '.data.repository.pullRequest.reviews.nodes[0].id'
```

- **If a pending review exists**, inform the user and ask:
  1. **Append to it** — Add new comments to the existing pending review (your existing draft comments are preserved)
  2. **Submit it first, then create a new review** — Submit the pending review as-is, then create a fresh review with the new comments
  3. **Cancel** — Don't post anything

- **If no pending review exists**, proceed directly to posting.

#### Step 6b: Post comments

**Do NOT** mention Claude, AI, or any automated tooling in review comments.

Based on the user's choice and whether a pending review exists:

##### Creating a new review (no pending review exists)

**For individual line comments**, create a new review and add comments:
```
gh api repos/{owner}/{repo}/pulls/{number}/reviews \
  --method POST \
  -f event="COMMENT" \
  -f body="Review summary" \
  --jq '.id'
```

Then for each comment:
```
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -f body="[comment text]" \
  -f path="[file path]" \
  -f line=[line number] \
  -f side="RIGHT"
```

##### Appending to an existing pending review

GitHub's REST API cannot add comments to an existing pending review. Use the **GraphQL API** with `addPullRequestReviewThread` instead.

For each comment, write a JSON file and call the GraphQL API:

```bash
cat > /tmp/pr-review-comment.json << 'ENDJSON'
{
  "query": "mutation($reviewId: ID!, $path: String!, $line: Int!, $body: String!) { addPullRequestReviewThread(input: { pullRequestReviewId: $reviewId, path: $path, line: $line, side: RIGHT, body: $body }) { thread { id } } }",
  "variables": {
    "reviewId": "<graphql-review-node-id>",
    "path": "<file path>",
    "line": <line number>,
    "body": "<comment text>"
  }
}
ENDJSON
gh api graphql --input /tmp/pr-review-comment.json
```

**IMPORTANT:** Always use JSON file input (`--input`) for GraphQL mutations with comment bodies. Inline `-f query='...'` breaks on quotes, apostrophes, and special characters in the comment text.

After appending, remind the user that the comments are part of their pending review and need to be submitted from the PR page (or they can ask you to submit it).

##### Submitting a pending review

If the user chose to submit the existing pending review first:
```bash
gh api graphql -f query='
mutation {
  submitPullRequestReview(input: {
    pullRequestReviewId: "<graphql-review-node-id>",
    event: COMMENT
  }) { pullRequestReview { id } }
}'
```

Then create a new review as described above.

**For a summary-only review** (no pending review conflict), use:
```
gh pr review <number> --repo <owner/repo> --comment --body "<review summary>"
```

Match the `event` field to the verdict:
- APPROVE → `--approve`
- REQUEST_CHANGES → `--request-changes`
- COMMENT → `--comment`

## Review Tone Guidelines

- Be direct and specific. State what the problem is and what to do instead.
- Reference concrete code — never give vague advice like "consider improving readability."
- Respect the author's intent — suggest alternatives, don't rewrite their approach.
- One concern per comment. Don't bundle unrelated issues.
- If overall quality is good, say so briefly and move on.
