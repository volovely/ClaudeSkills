---
name: daily-report
description: Generate a comprehensive daily activity report from Jira showing what you accomplished and how it helps the team (3PM-3PM Jerusalem time)
argument-hint: "[date]"
---

# Daily Activity Report Skill

Generate a comprehensive daily activity report from Jira showing what you accomplished and how it helps the team.

## Usage

```
/daily-report [date]
```

- Without arguments: generates report for the current workday (3PM yesterday to 3PM today, Jerusalem time)
- With date argument: generates report for that workday (3PM on date-1 to 3PM on date, Jerusalem time)

## Time Range

**Important:** The report covers activity from **3:00 PM (15:00) yesterday to 3:00 PM (15:00) today, Jerusalem time (Asia/Jerusalem timezone, UTC+2/+3).**

This aligns with the typical workday for the team. When filtering changelog entries, only include changes that fall within this time window.

## Instructions

When this skill is invoked, follow these steps:

### Step 1: Calculate Time Range
- Convert 3PM Jerusalem time to the appropriate UTC offset (Israel Standard Time UTC+2 or Israel Daylight Time UTC+3)
- Start: Yesterday at 15:00 Jerusalem time
- End: Today at 15:00 Jerusalem time
- Use these timestamps when filtering changelog entries

### Step 2: Get User and Cloud Info
1. Call `mcp__atlassian__atlassianUserInfo` to get the current user's account ID and name
2. Call `mcp__atlassian__getAccessibleAtlassianResources` to get the cloud ID

### Step 3: Search for Issues Worked On
Run these JQL searches in parallel to find all issues the user touched during the time range:

1. **Updated issues assigned to user:**
   ```
   assignee = currentUser() AND updated >= -1d ORDER BY updated DESC
   ```

2. **Status changes by user:**
   ```
   status changed BY currentUser() DURING (-1d, now()) ORDER BY updated DESC
   ```

3. **Worklog entries:**
   ```
   worklogAuthor = currentUser() AND worklogDate >= -1d
   ```

Note: JQL doesn't support hour-level precision, so fetch issues from the last day and filter by exact timestamps in the changelog.

### Step 4: Get Detailed History
For each unique issue found, fetch the issue with changelog expanded:
- Use `mcp__atlassian__getJiraIssue` with:
  - `issueIdOrKey`: the issue key
  - `fields`: `["summary", "status", "issuetype", "assignee", "priority"]`
  - `expand`: `"changelog"`
- The changelog is paginated by the Jira API. Request only the first page (default). If the changelog contains more entries than returned, rely on the `created` timestamps within the returned entries to filter to the time window — do not paginate further since we only need the last ~24h of changes.
- **Filter changelog entries to only include those between 3PM yesterday and 3PM today Jerusalem time**

### Step 5: Analyze Changes
For each issue, identify what the user specifically did within the time window:
- Status transitions (e.g., "In Dev" → "In Review")
- Assignments (assigned to others = handoff, assigned to self = took ownership)
- Field updates (effort logged, QA handoff documentation, etc.)
- Resolution changes (marking as Done/Deployment Ready)

**Only include actions where the timestamp falls within the 3PM-3PM Jerusalem time window.**

### Step 6: Generate Report

Structure the report to be **readable in under 1 minutes** - concise, scannable, no filler:

```markdown
# Activity Report - [User Name]
## [Date Range] | Shutterfly iOS

---

### Completed: [TICKET-ID] - [Summary]
- [Key detail about what was built/done - 1 line]
- [Handoff info: moved to X, assigned to Y]

### Started: [TICKET-ID] - [Summary]
- [What and why - 1 line]

### Code Reviews
- **[TICKET-ID]** ([Summary]) - [Approved/Sent back] -> [outcome in a few words]
- **[TICKET-ID]** ([Summary]) - [Approved/Sent back] -> [outcome in a few words]

---

### Impact
[1-2 sentences max. What moved forward, what's unblocked.]
```

### Writing Style Guidelines
- **Keep the report readable in under 2 minutes.** Be concise - no filler, no verbose explanations.
- Use short bullet points, not paragraphs
- One line per action, one line per impact - no multi-sentence descriptions
- Skip timestamps for individual actions unless the timing itself matters
- Group code reviews together in a compact list rather than separate sections
- Use meaningful, descriptive language
- Focus on WHAT was done and WHY it matters
- Quantify impact where possible
- Group related tickets together
- Highlight completed/resolved items prominently
- For code reviews, note how this unblocks teammates in a single line
- End with a 1-2 sentence impact summary, not a table

### Categories to Use
- **Completed and Deployed** - Tickets marked as Done/Deployment Ready
- **Completed Development** - Moved to In Review
- **Code Review Assignments** - Assigned to user for review
- **Bug Fixes** - Bug type issues worked on
- **Investigation/Research** - Investigation tickets
- **Unblocked Work** - Moved from Blocked status

## Example Invocations

- `/daily-report` - Report for current workday (3PM yesterday to 3PM today)
- `/daily-report 2026-02-18` - Report for workday ending on Feb 18 at 3PM
