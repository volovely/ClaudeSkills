---
name: qa-handoff
description: >
  Fill the QA handoff field in a Jira ticket with risk level, change description, test guidance, and checklist
  based on the actual code diff and changed files. Use this skill when the user says "qa handoff", "fill qa handoff",
  "update qa handoff", "qa-handoff", or asks to prepare a ticket for QA. Also use when the user mentions
  transitioning a Jira ticket to QA, writing test guidance, or preparing handoff notes for testers.
argument-hint: "[TICKET-KEY] [PR-URL-or-number]"
---

# QA Handoff Skill

Analyze the current branch's code changes and fill the QA handoff field (`customfield_17288`) in the corresponding Jira ticket with specific, actionable test guidance.

## Usage

```
/qa-handoff
/qa-handoff IOS-17215
/qa-handoff IOS-17215 https://github.com/.../pull/123
/qa-handoff IOS-17215 123
```

- First argument (optional): Jira ticket key. If omitted, extract from the current branch name (e.g., `IOS-17215-Width-and-height-alignment` → `IOS-17215`).
- Second argument (optional): PR URL or PR number. If omitted, the skill will look up the open PR for the current branch automatically.

## Instructions

### Step 1: Determine Context

1. **Current branch:** Run `git rev-parse --abbrev-ref HEAD`.
2. **Jira ticket key:** Use the first argument if provided, otherwise parse from the branch name. The ticket key is the first segment matching `UPPERCASE-DIGITS` (e.g., `IOS-17215`).
3. **PR and base branch:**
   - If the user provided a PR URL or number as the second argument, run `gh pr view <number-or-url> --json number,baseRefName,url` to get PR details.
   - Otherwise, run `gh pr view --json number,baseRefName,url` to find the open PR for the current branch automatically.
   - Extract `baseRefName` as the base branch for the diff.
   - If no PR is found (command fails or returns nothing), fall back to `develop` as the base branch and note that no PR was found.

If the current branch equals the base branch, stop and inform the user.

### Step 2: Gather Diff and Ticket Context

Run these in **parallel**:

1. **Diff stats:**
   ```
   git diff <base-branch>...HEAD --stat
   ```
   (Use the base branch determined in Step 1.)

2. **Full diff:**
   ```
   git diff <base-branch>...HEAD
   ```

3. **Commit log:**
   ```
   git log <base-branch>..HEAD --oneline
   ```

4. **Fetch Jira ticket** using `mcp__atlassian__getJiraIssue`:
   - `cloudId`: `snapfish-llc.atlassian.net`
   - `issueIdOrKey`: the extracted ticket key
   - `responseContentFormat`: `"markdown"`
   - `fields`: `["summary", "description", "status", "issuetype", "priority", "labels", "parent", "customfield_10009", "customfield_12202", "customfield_17288"]`

   Field reference:
   - `priority` — P1/P2/P3/TBD: directly informs risk level assessment
   - `labels` — e.g. `RevenueTargets` warrants elevated risk
   - `parent` — parent epic key and summary
   - `customfield_10009` — epic link key (alternative to `parent`)
   - `customfield_12202` — testing type: `"Manual"` or `"Automated"` — use in the Unit Tests section
   - `customfield_17288` — existing QA handoff content (check before overwriting)

5. **Fetch parent/epic if present:** If the ticket has a `parent.key` or a non-null `customfield_10009`, fetch the parent with:
   - `issueIdOrKey`: the parent key
   - `responseContentFormat`: `"markdown"`
   - `fields`: `["summary", "description", "issuetype", "priority", "labels"]`

   Use the parent epic's description for additional "why" context when composing the handoff.

### JSON → TOON Conversion

**MANDATORY:** Whenever a tool returns JSON data (especially MCP/Jira responses), convert it to TOON format before processing to reduce token usage. Use a temporary file:

```bash
echo '<json_response>' > /tmp/qa-handoff-tmp.json
toon /tmp/qa-handoff-tmp.json -o /tmp/qa-handoff-tmp.toon
```

Then read `/tmp/qa-handoff-tmp.toon` instead of using the raw JSON. Apply this to all JSON responses from MCP tools (e.g., `mcp__atlassian__getJiraIssue`, `mcp__atlassian__editJiraIssue`).

### Step 3: Analyze the Changes

Before writing the handoff, build a thorough understanding:

1. **Categorize changed files** — models, views, business logic, tests, configs, assets.
2. **Read key files** in full using the Read tool — not just diff hunks. Focus on files with significant logic changes, new types, or API modifications.
3. **Follow references** — use Grep/Glob if needed to understand how new or modified types are used elsewhere.
4. **Determine the nature of the change:**
   - Is this a new feature, enhancement, bug fix, or refactoring?
   - What user-facing behavior changed?
   - What existing behavior could regress?
   - Are there edge cases worth calling out?
5. **Assess risk level** — use all available signals:
   - Ticket `priority` (`P1 Critical` / `P2 High` → raise risk floor)
   - Labels (e.g. `RevenueTargets` → at least Medium)
   - Scope of the diff (how many files, what layers)
   - **Low:** Cosmetic changes, copy updates, isolated additions with no effect on existing flows.
   - **Medium:** New behavior added, moderate refactoring touching existing logic, changes across multiple interaction points.
   - **High:** Core logic rewritten, changes to data models or persistence, security-sensitive areas, broad cross-cutting changes.

### Step 4: Compose the QA Handoff

Build the content for `customfield_17288` in Atlassian Document Format (ADF). The field uses a structured template — fill every section:

**Risk Level:** `Low` / `Medium` / `High` — followed by a brief justification referencing what changed and why it carries that risk level.

**Change Description:** A concise summary (2-3 sentences) of what changed and why. Reference the Jira ticket description and parent epic description for the "why". Mention key files or areas affected.

**Root Cause Analysis:** For bug fixes, explain what caused the bug and how the fix addresses it. For new features or refactoring, write `N/A — new feature` or `N/A — refactoring`.

**Unit Tests:** State whether new unit tests were added. Cross-reference `customfield_12202` (testing type): if `"Manual"`, explicitly note that no automated tests cover this and full manual regression is required. If tests exist, mention what they cover.

**Guidance to QA:** This is the most important section. Write a **numbered list** of specific test scenarios. Each item should have:
- A **bold label** summarizing the scenario (e.g., "Width/Height snapping (new):")
- A clear description of what to do and what to verify
- Cover: new behavior, regression areas, edge cases, interaction with existing features

The guidance must be **specific to the actual code changes** — not generic. Reference concrete UI interactions, specific element types, particular gestures, or exact flows that the diff touches. Think about what a QA engineer who hasn't seen the code needs to know to test thoroughly.

**Checklist items** — answer each with Yes/No, adding brief context when the answer isn't obvious:
- Test Large Accounts (Yes/No)
- Test Login Path (Yes/No)
- Test Single/Multiple Upload (Yes/No)
- Test Creation Path (Yes/No)
- Performance impact (Yes/No)

### Step 5: Update the Jira Ticket

Use `mcp__atlassian__editJiraIssue` to update the ticket:
- `cloudId`: `snapfish-llc.atlassian.net`
- `issueIdOrKey`: the ticket key
- `fields`: `{"customfield_17288": <ADF document>}`

The ADF document structure:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    // Each section is a paragraph with bold label + plain text
    // Guidance to QA uses an orderedList node
    // Each list item has bold label + description text
  ]
}
```

ADF formatting reference:
- **Bold text:** `{"type": "text", "text": "Label: ", "marks": [{"type": "strong"}]}`
- **Plain text:** `{"type": "text", "text": "content"}`
- **Paragraph:** `{"type": "paragraph", "content": [...]}`
- **Ordered list:** `{"type": "orderedList", "content": [{"type": "listItem", "content": [{"type": "paragraph", "content": [...]}]}]}`

### Step 6: Confirm

After the update succeeds, tell the user which ticket was updated and give a brief summary of what was written (risk level, number of test scenarios, any notable items). Include the Jira link:

```
https://snapfish-llc.atlassian.net/browse/<TICKET-KEY>
```
