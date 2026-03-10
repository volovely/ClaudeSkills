---
name: pr-cherry-pick
description: Use when the user says "pr-cherry-pick <PR> <branch>", "/pr-cherry-pick", or asks to cherry-pick a pull request onto another branch. Takes all commits from an existing PR and recreates it targeting a different base branch, preserving the original title, body, assignees, and labels. Opens the browser for the user to review before submission.
argument-hint: "<PR number or URL> <target branch>"
---

# PR Cherry-Pick Skill

Cherry-pick all commits from an existing PR onto a new base branch, then open the browser to create an equivalent PR.

## Usage

```
/pr-cherry-pick 1234 staging
/pr-cherry-pick https://github.com/org/repo/pull/1234 staging
```

## Instructions

### Step 1: Parse Arguments

Extract from the arguments:
- **PR reference**: either a full GitHub URL (e.g. `https://github.com/org/repo/pull/1234`) or a bare number (e.g. `1234`).
- **Target branch**: the branch to cherry-pick onto (e.g. `staging`).

If either is missing, ask the user before proceeding.

**Determine the repo:**
- If a full URL was given, parse `org/repo` from it.
- Otherwise, run:
  ```
  gh repo view --json nameWithOwner -q .nameWithOwner
  ```

### Step 2: Fetch PR Details

Run:
```
gh pr view <number> --repo <org/repo> \
  --json title,body,commits,headRefName,baseRefName,assignees,labels
```

Collect:
- `title`, `body` — for the new PR description
- `commits[].oid` — the commit SHAs to cherry-pick, **in order**
- `headRefName` — the original branch name, used to derive the new branch name
- `assignees[].login` — to preserve assignees
- `labels[].name` — to preserve labels

### Step 3: Prepare the New Branch

New branch name: `<headRefName>-<targetBranch>`
(e.g. original `IOS-17745-fix`, target `staging` → `IOS-17745-fix-staging`)

Run:
```
git fetch origin <targetBranch>
git checkout -b <newBranchName> origin/<targetBranch>
```

### Step 4: Cherry-Pick All Commits

Cherry-pick each commit **in order**:
```
git cherry-pick <oid>
```

If any cherry-pick produces a conflict, **stop immediately**. Do NOT continue cherry-picking. Tell the user which commit caused the conflict and what files are conflicted, and ask them to resolve it manually before continuing.

### Step 5: Push the New Branch

```
git push -u origin <newBranchName>
```

### Step 6: Open the Browser to Create the PR

Build the `gh pr create --web` command, preserving all metadata from the original PR:

- `--base <targetBranch>`
- `--title "<original title>"`
- `--body "<original body>"`
- One `--assignee <login>` per assignee (if any)
- One `--label "<name>"` per label (if any)
- `--web` — always required; never create the PR directly

Example:
```
gh pr create \
  --base staging \
  --title "IOS-17745 Deluxe mat doesn't disappear on design change" \
  --body "$(cat <<'EOF'
## SUMMARY
...
EOF
)" \
  --assignee roman-volovelskyi_sflyinc \
  --label bug \
  --web
```

**Shell quoting:** Use a HEREDOC for the body to avoid issues with special characters, quotes, and newlines.

Tell the user the browser has opened and they can review, edit, and submit the PR from there. Remind them that assignees and labels from the original PR have been pre-filled.

**Do NOT** omit `--web`. The user must always review before the PR is created.
