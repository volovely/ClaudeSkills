# Test Documentation: `/context restore payments` (nonexistent memo)

## What I Did

1. **Read the skill file** at `/Users/roman.volovelskyi/.claude/skills/feature-context/SKILL.md` to understand the restore procedure.

2. **Parsed the user request.** The command `/context restore payments` is a standard restore (no `--update` flag) for a feature named "payments".

3. **Attempted to read the memo file** at `~/.claude/skills/feature-context/memos/payments.md`. The file does not exist.

4. **Listed existing memos** in `~/.claude/skills/feature-context/memos/`. Only one memo exists: `snapping.md`.

5. **Followed the skill's "file doesn't exist" branch.** The skill instructions for standard restore say:
   > 2. If the file doesn't exist, tell the user and suggest running memorize first

## What I Would Have Told the User

---

**No saved context found for "payments".**

There is no memo file at `~/.claude/skills/feature-context/memos/payments.md`. This means the "payments" feature has not been memorized yet.

To create a context memo for this feature, run:

```
/context memorize payments
```

This will explore the relevant source files, document the feature's architecture, key files, and implementation details, and save it all so future conversations can skip the exploration step.

**Currently available memos:**
- `snapping` — (`~/.claude/skills/feature-context/memos/snapping.md`)

---

## Skill Behavior Analysis

- The skill correctly defines a clear path for this edge case (step 2 of the restore flow).
- No memo file was created — the skill does not instruct creating one on a failed restore.
- No source code was read or modified — the skill says "Do not re-read source files unless the user asks you to" and in this case there was nothing to restore in the first place.
- The response is minimal and actionable: tells the user the memo doesn't exist and suggests the exact command to fix it.
