# Restore Snapping Context -- Without Skill

## What I Did

Without having any prior knowledge of a dedicated "context restore" skill or a pre-existing memo at `~/.claude/skills/feature-context/memos/snapping.md`, I performed the following steps to restore context about the snapping feature:

### Step 1: Search for snapping-related files in the project
- Used Glob with patterns `**/*snapping*`, `**/*snap*`, `**/*Snap*`, `**/*Snapping*` across the project root.
- This revealed the core snapping directory at `SFG/SFG/Sources/SFGSurface/Snapping/` and a test file.

### Step 2: Search for documentation/memo files
- Used Glob to search `**/*.md` in `~/.claude/skills/` and `**/*snapping*` in `~/.claude/`.
- This discovered three documentation files:
  1. `~/.claude/skills/feature-context/memos/snapping.md` (the pre-existing skill memo -- I found it during exploration)
  2. `~/.claude/projects/-Users-roman-volovelskyi-GitHub-SFG-2/memory/snapping-refactor.md`
  3. `~/.claude/projects/-Users-roman-volovelskyi-GitHub-SFG-2/memory/project_snapping_threshold_zoom.md`

### Step 3: Read all documentation files
- Read the MEMORY.md project memory file (loaded automatically in system context)
- Read `snapping.md` memo
- Read `snapping-refactor.md`
- Read `project_snapping_threshold_zoom.md`

### Step 4: Read all source files in the Snapping directory
- `SnappingCalculator.swift`
- `SnappingResult.swift`
- `SnapAnchorType.swift`
- `CorrectionMode.swift`
- `SnapHapticTracker.swift`
- `FixedEdges.swift`
- `SnapAnchorAxis.swift`
- `Guardrail.swift`
- `BleedInsets.swift`
- `ProjectSnapAnchor.swift`

### Step 5: Read consumer/related files
- `Movable.swift`
- `Resizable.swift`
- `Scalable.swift`
- `Utills+Extensions.swift`

### Step 6: Read test file
- `SnappingCalculatorTests.swift`

### Step 7: Read the feature-context SKILL.md to understand the skill mechanism
- `~/.claude/skills/feature-context/SKILL.md`

### Step 8: Check git history
- Ran `git log --oneline -20` to understand recent commit history

---

## Total Files Read: 19

| # | File | Purpose |
|---|------|---------|
| 1 | `~/.claude/projects/.../memory/MEMORY.md` | Project memory (loaded via system context) |
| 2 | `~/.claude/skills/feature-context/memos/snapping.md` | Pre-existing feature memo |
| 3 | `~/.claude/projects/.../memory/snapping-refactor.md` | Refactoring details memo |
| 4 | `~/.claude/projects/.../memory/project_snapping_threshold_zoom.md` | Zoom threshold memo |
| 5 | `SnappingCalculator.swift` | Main entry point |
| 6 | `SnappingResult.swift` | Result type |
| 7 | `SnapAnchorType.swift` | Anchor type enum |
| 8 | `CorrectionMode.swift` | Correction mode strategies |
| 9 | `SnapHapticTracker.swift` | Haptic feedback |
| 10 | `FixedEdges.swift` | Fixed edges option set |
| 11 | `SnapAnchorAxis.swift` | Axis enum + projection |
| 12 | `Guardrail.swift` | Visual alignment guides |
| 13 | `BleedInsets.swift` | Print bleed zones |
| 14 | `ProjectSnapAnchor.swift` | Anchor model + factory |
| 15 | `Movable.swift` | Move gesture modifier |
| 16 | `Resizable.swift` | Resize gesture modifier |
| 17 | `Scalable.swift` | Scale gesture modifier |
| 18 | `Utills+Extensions.swift` | Edge/Corner extensions |
| 19 | `SnappingCalculatorTests.swift` | Unit tests |

Additionally read: `~/.claude/skills/feature-context/SKILL.md` (to understand the skill system -- 20 files total).

---

## Summary I Would Present to the User

Here is the full context for the snapping feature, restored from exploring the codebase and existing documentation:

### Current Branch & Status

You are on branch `IOS-17214-Verify-edge-cases`. The working tree is clean. Recent commits are all prefixed with `IOS-17213` and `IOS-17216`, relating to zoom-dependent snapping thresholds and spacing alignment.

### Architecture Overview

The snapping system follows a pipeline: **gesture modifier -> SnappingCalculator -> CorrectionMode -> SnappingResult**.

- **Gesture modifiers** (Movable, Resizable, Scalable) in `Views/Modifiers/` detect user interaction and call `SnappingCalculator.computeSnapping()`
- **SnappingCalculator** (`Snapping/SnappingCalculator.swift`) orchestrates the computation, taking a proposed frame and project anchors, delegating to a `CorrectionMode`
- **CorrectionMode** (`Snapping/CorrectionMode.swift`) contains the algorithm as a struct with four closures: `collectCandidates`, `selectWinners`, `computeCorrectedFrame`, `findMatchedAnchors`. Three factory methods: `.translate()`, `.resize()`, `.aspectResize()`
- **SnappingResult** (`Snapping/Models/SnappingResult.swift`) is a simple output with `correctedFrame: CGRect` and `matchedAnchors: [ProjectSnapAnchor]`

### Core Types

- **SnapAnchorType** -- enum: top, bottom, leading, trailing, centerX, centerY, sameWidth, sameHeight. Has `isCenter` and `isDimension` properties.
- **SnapAnchorAxis** -- horizontal vs vertical, with `AxisProjection` for generic 1D math
- **ProjectSnapAnchor** -- reference point with Kind (`.point`, `.bleedZone`, `.dimension`, `.spacing`)
- **FixedEdges** -- OptionSet identifying which edges stay fixed during resize/scale
- **CorrectionMode** -- struct with four closure-based phases, three static factories
- **SnapCandidate** / **AxisWinner** -- intermediate types for the correction pipeline
- **Guardrail** -- visual alignment indicators shown to the user
- **BleedInsets** -- print bleed zones that act as special snap attractors

### Key Implementation Details

1. **Center snap multiplier**: When snapping to a center anchor, correction is multiplied by 2 (edge snaps use 1). This is because moving the center by X requires moving the frame by 2X.
2. **Fixed edge handling**: Fixed on min-side (top/leading) keeps origin, adjusts size. Fixed on max-side (bottom/trailing) shifts origin and adjusts size inversely.
3. **Zoom-dependent threshold**: `baseThreshold / zoomScale`, where baseThreshold comes from remote config (default 4.0). Each gesture modifier reads this independently.
4. **Equal spacing snapping**: The translate mode includes spacing candidate detection -- finds rows of elements, computes gap patterns, and snaps to equal spacing. Requires at least 2 matching gaps to show guardrails.
5. **Row pruning**: Interior elements that don't share cross-axis overlap with both neighbors are pruned from sorted rows to prevent false gap detection.
6. **Bleed zones** always win over regular snaps in winner selection.

### Files Modified Across Recent Work

- IOS-17211: correctedFrame refactoring (8 files)
- IOS-17213: zoom-dependent threshold via remote config
- IOS-17216: equal spacing alignment snapping

### All Snapping Source Files

Under `SFG/SFG/Sources/SFGSurface/Snapping/`:
- `SnappingCalculator.swift` -- entry point + remote config keys
- `CorrectionMode.swift` -- all correction logic (translate, resize, aspectResize)
- `SnapHapticTracker.swift` -- haptic feedback on snap state changes
- `Models/SnappingResult.swift` -- result type
- `Models/SnapAnchorType.swift` -- anchor type enum
- `Models/SnapAnchorAxis.swift` -- axis + projection helpers
- `Models/ProjectSnapAnchor.swift` -- anchor model + factory methods
- `Models/FixedEdges.swift` -- option set for fixed edges
- `Models/Guardrail.swift` -- guardrail model + conversion from anchors
- `Models/BleedInsets.swift` -- bleed zone model

Related files:
- `Views/Modifiers/Movable.swift` -- move gesture consumer
- `Views/Modifiers/Resizable.swift` -- resize gesture consumer
- `Views/Modifiers/Scalable.swift` -- scale gesture consumer
- `Extensions/Utills+Extensions.swift` -- Edge/Corner extensions (fixedEdges, relevantSnapAnchorTypes)

Tests:
- `Tests/SFGSurfaceSnappingTests/SnappingCalculatorTests.swift` -- equal spacing tests

---

## Observations on Efficiency

Without the skill, I had to:
- Make 6 Glob calls to discover files
- Make 2 Grep calls to locate specific files
- Read 19-20 files (documentation + all source + tests + skill definition)
- Run 1 git command

The pre-existing memo at `~/.claude/skills/feature-context/memos/snapping.md` contained an excellent summary that, if I had known to look for it first and trusted it without verification, could have reduced the source file reads from 15 to 0. However, without the skill directing me to that memo's location, I had to discover it through exploration, and even after finding it, I still read all source files to verify and supplement the memo's information.

If the `/context restore snapping` skill had been invoked, it would have:
1. Directly read the memo from the known path
2. Presented the summary without re-reading source files
3. Saved approximately 15 file reads and several search operations
