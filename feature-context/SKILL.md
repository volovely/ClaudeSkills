---
name: feature-context
description: Save and restore feature context to avoid re-exploring the same codebase areas across conversations. Use when the user says "context memorize", "context restore", "memorize feature", "restore feature", "save context", "load context", "remember this feature", "feature-context", or wants to pick up where they left off on a feature. Also trigger when the user starts working on a feature they've previously memorized — proactively suggest restoring context if a memo exists.
argument-hint: "<memorize|restore> <feature-name>"
---

# Feature Context Skill

Save and restore feature knowledge so Claude can skip expensive codebase exploration in future conversations.

## Usage

```
/feature-context memorize <feature-name>
/feature-context restore <feature-name>
/feature-context restore <feature-name> --update
```

**Examples:**
```
/feature-context memorize snapping
/feature-context restore snapping
/feature-context restore snapping --update
```

## Storage

Memos are stored in: `~/.claude/skills/feature-context/memos/<feature-name>.md`

Each feature gets one file. The feature name is lowercased and hyphenated (e.g., "snapping calculator" → `snapping-calculator.md`).

---

## Mode: Memorize

When the user runs `/feature-context memorize <feature-name>`, create or update the memo file for that feature. This is the most important mode — the quality of the memo directly determines how much time and tokens are saved later.

### Step 1: Identify the feature scope

Ask the user if it's not obvious:
- Which directory or directories contain the feature?
- Are there related files outside the main directory that matter?

If you've been working on this feature in the current conversation, you likely already know the answers — use that context.

### Step 2: Explore and document

Thoroughly read the feature's source files. Focus on capturing information that is expensive to re-derive:

1. **Directory structure** — Relative paths from project root to all relevant files
2. **Architecture** — How the feature is structured, key types and their roles, the flow of data
3. **Core concepts** — Domain-specific terms, what they mean in this context
4. **Key implementation details** — Non-obvious decisions, tricky logic, important algorithms
5. **Dependencies** — What this feature depends on (other modules, external frameworks)
6. **Integration points** — How other parts of the codebase interact with this feature
7. **Current state** — Any known issues, ongoing work, or recent changes worth noting

### Step 3: Write the memo

Write the memo file using this structure:

```markdown
# <Feature Name>

> One-line summary of what this feature does.

## Location

- **Primary directory:** `<relative path from project root>`
- **Related files:**
  - `<path>` — <why it matters>

## Architecture

<How the feature is structured. Key types, protocols, and their relationships.
Include a brief data flow description if applicable.>

## Core Concepts

<Domain terms and what they mean in this codebase. Think of this as a glossary
that lets Claude immediately understand the code without guessing.>

## Key Files

<For each important file, a 1-3 line summary of what it contains and its role.
Order by importance, not alphabetically.>

### <filename.swift>
<summary>

## Implementation Details

<Non-obvious logic, algorithms, important constants, edge cases.
This section captures the "why" behind implementation choices.>

## Dependencies & Integration

<What this feature imports/uses from other modules.
How other modules call into this feature.>

## Notes

<Anything else useful: known issues, gotchas, recent refactors, planned work.>

---
*Last updated: <YYYY-MM-DD>*
```

Adapt the template to what makes sense for the specific feature — skip sections that don't apply, add sections that do. The goal is completeness without noise.

### Step 4: Confirm

After writing, tell the user what was saved and where. Show the key sections so they can spot anything missing.

---

## Mode: Restore

When the user runs `/feature-context restore <feature-name>`, load the saved context.

### Standard restore (no --update flag)

1. Read `~/.claude/skills/feature-context/memos/<feature-name>.md`
2. If the file doesn't exist, tell the user and suggest running memorize first
3. Present a brief summary of what was loaded: feature name, location, last updated date
4. You now have the feature context — proceed with whatever the user asks next

Do not re-read source files unless the user asks you to. The whole point of the memo is to avoid that.

### Restore with update (`--update` flag)

1. Read the existing memo
2. Re-explore the feature's source files to check for changes since the memo was written
3. Update the memo with any new findings
4. Proceed as with standard restore

---

## Listing available memos

If the user asks what features are memorized, or runs just `/feature-context` with no arguments, list all `.md` files in `~/.claude/skills/feature-context/memos/` with their one-line summaries and last-updated dates.

---

## Best Practices for Good Memos

- **Be specific about types and their roles.** "SnappingCalculator handles snap logic" is too vague. "SnappingCalculator takes a moving frame and reference anchors, computes the closest snap point per axis, and returns a SnapComputation with correction vectors and winning anchor types" is useful.
- **Include method signatures for key APIs.** The exact function name and parameters save the most re-exploration time.
- **Document non-obvious relationships.** If File A calls into File B in a way that's not obvious from the names, say so.
- **Capture magic numbers and constants.** If there's a threshold of 4.0 or a multiplier of 2 for center snaps, document it and explain why.
- **Note what's NOT in this feature.** If a closely related concern is handled elsewhere, say where. This prevents Claude from wasting time looking for it in the wrong place.
