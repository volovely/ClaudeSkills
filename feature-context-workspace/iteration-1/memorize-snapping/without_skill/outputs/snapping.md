# Snapping Feature -- Comprehensive Documentation

## Overview

The snapping feature provides interactive alignment, dimension-matching, and equal-spacing guides when users move, resize, or scale elements on the SFG design surface. When an element is dragged or resized near an alignment target, the system snaps the element to that target, displays visual guardrail lines, and triggers haptic feedback.

All source files live under `/Users/roman.volovelskyi/GitHub/SFG_2/SFG/SFG/Sources/SFGSurface/`.

---

## Architecture

```
                                   Modifier (Movable / Resizable / Scalable)
                                     |
                                     | Builds a proposed frame from the drag gesture
                                     | Reads remote config threshold, divides by zoomScale
                                     | Chooses a CorrectionMode (.translate / .resize / .aspectResize)
                                     v
                               SnappingCalculator.computeSnapping(...)
                                     |
                                     | 1. Generates moving anchors from proposedFrame
                                     | 2. Delegates to CorrectionMode closures:
                                     |      collectCandidates -> selectWinners -> computeCorrectedFrame -> findMatchedAnchors
                                     v
                               SnappingResult
                                 - correctedFrame: CGRect
                                 - matchedAnchors: [ProjectSnapAnchor]
                                     |
                    +----------------+------------------+
                    |                |                  |
            Element frame    Guardrails overlay   Haptic feedback
            is updated       via matchedAnchors   via SnapHapticTracker
                             .toGuardrails(...)
```

---

## File Inventory

### Core Snapping Engine

| File | Path (relative to SFGSurface/) | Purpose |
|------|-------------------------------|---------|
| **SnappingCalculator.swift** | `Snapping/SnappingCalculator.swift` | Entry point. Single method `computeSnapping(...)`. Also holds remote config key definitions for the snapping feature flag and threshold variable. Registered as a TCA `DependencyKey`. |
| **CorrectionMode.swift** | `Snapping/CorrectionMode.swift` | Strategy pattern struct with four closure properties. Three factory methods: `.translate`, `.resize`, `.aspectResize`. Contains all shared snapping math including candidate collection, winner selection, frame correction, and spacing detection. |
| **SnappingResult.swift** | `Snapping/Models/SnappingResult.swift` | Simple value type: `correctedFrame: CGRect` + `matchedAnchors: [ProjectSnapAnchor]`. |

### Model Types

| File | Path | Purpose |
|------|------|---------|
| **SnapAnchorType.swift** | `Snapping/Models/SnapAnchorType.swift` | Enum: `.top`, `.bottom`, `.leading`, `.trailing`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight`. Has `axis`, `isCenter`, `isDimension` computed properties. Set extensions for `.all`, `.center`, `.centersAndSides`. |
| **SnapAnchorAxis.swift** | `Snapping/Models/SnapAnchorAxis.swift` | Enum: `.horizontal` (Y axis), `.vertical` (X axis). Also defines `AxisProjection` struct for axis-generic rect projections. |
| **ProjectSnapAnchor.swift** | `Snapping/Models/ProjectSnapAnchor.swift` | The anchor type used both for moving-element anchors and static-element anchors. Has `type`, `axis`, `kind`, `ownerRect`. `Kind` is an enum: `.point`, `.bleedZone`, `.dimension`, `.spacing`. Includes `matchDelta(movingPosition:threshold:)` for threshold-based matching. Contains factory methods on `[ProjectSnapAnchor]` and `CGRect` for building anchors from element positioning data. |
| **FixedEdges.swift** | `Snapping/Models/FixedEdges.swift` | `OptionSet` with `.top`, `.bottom`, `.leading`, `.trailing`. Identifies which edges stay fixed during resize/scale operations. |
| **BleedInsets.swift** | `Snapping/Models/BleedInsets.swift` | Converts product bleed dimensions (inches) to screen coordinates. Generates bleed-zone `ProjectSnapAnchor`s that create "forbidden zones" near surface edges where elements snap to the surface border. |
| **Guardrail.swift** | `Snapping/Models/Guardrail.swift` | Visual guide model. Has three kinds: `.alignment` (line at a position), `.dimension` (arrow showing same width/height), `.spacing` (equal gap indicator). Contains logic for converting matched anchors to guardrails, including smart placement of dimension arrows to avoid overlap. |

### Haptic Feedback

| File | Path | Purpose |
|------|------|---------|
| **SnapHapticTracker.swift** | `Snapping/SnapHapticTracker.swift` | `@Observable` class that fires a haptic toggle when the set of snapped anchor positions changes. Uses approximate equality (epsilon=0.1) to avoid false triggers from floating-point drift. |

### Visual Rendering

| File | Path | Purpose |
|------|------|---------|
| **GuardrailsOverlayModifier.swift** | `Views/OverlayAndBackground/GuardrailsOverlayModifier.swift` | SwiftUI overlay that renders guardrails. Uses a `PreferenceKey` to bubble guardrails up the view hierarchy. Draws three shapes: `AlignmentLineShape`, `DimensionArrowShape`, `SpacingIndicatorShape`. Line width scales inversely with zoom. |

### Gesture Modifiers (Consumers)

| File | Path | Purpose |
|------|------|---------|
| **Movable.swift** | `Views/Modifiers/Movable.swift` | Handles drag-to-move gestures. Uses `.translate` correction mode. Supports cross-surface transfer (Calendar, Book). Derives translation correction from `correctedFrame.origin - proposedFrame.origin`. |
| **Resizable.swift** | `Views/Modifiers/Resizable.swift` | Handles edge-drag resize gestures. Uses `.resize` correction mode with `edge.fixedEdgeOnResizeAxis`. Each `Edge` provides its relevant snap anchor types. Reads snapped size directly from `correctedFrame.size`. |
| **Scalable.swift** | `Views/Modifiers/Scalable.swift` | Handles corner-drag scale gestures with aspect ratio preservation. Uses `.aspectResize` correction mode with `corner.fixedEdges`. Each `Corner` provides relevant snap anchor types. Applies aspect ratio to the uncorrected axis after snapping. |

### Extensions

| File | Path | Purpose |
|------|------|---------|
| **Utills+Extensions.swift** | `Extensions/Utills+Extensions.swift` | Defines `Corner` and `Edge` enums. `Corner.fixedEdges` and `Corner.relevantSnapAnchorTypes(isElementRotated:)` for scaling. `Edge.fixedEdgeOnResizeAxis` and `Edge.relevantSnapAnchorTypes(isElementRotated:)` for resizing. Also has geometry helpers: `rotateAround`, `anchorFor`, `center`, etc. |

### Tests

| File | Path | Purpose |
|------|------|---------|
| **SnappingCalculatorTests.swift** | `Tests/SFGSurfaceSnappingTests/SnappingCalculatorTests.swift` | Comprehensive tests for equal-spacing snapping. Covers horizontal and vertical spacing, first/middle/last element positions, multi-row interference from elements in different Y/X bands, and the cross-axis overlap pruning algorithm. |

---

## CorrectionMode -- The Strategy Pattern

`CorrectionMode` is the heart of the snapping logic. It is a struct with four closure properties, each representing a step in the snapping pipeline:

### Pipeline Steps

1. **`collectCandidates`** -- Pairs each moving anchor with each project anchor, computing snap deltas within threshold. For translate mode, also generates equal-spacing candidates.

2. **`selectWinners`** -- Picks the best candidate per axis (horizontal and vertical independently). Bleed anchors always win over non-bleed. Otherwise smallest absolute delta wins.

3. **`computeCorrectedFrame`** -- Applies winners to the proposed frame. Behavior varies by mode:
   - **Translate**: shifts origin by winner deltas.
   - **Resize**: calls `applySizeCorrection` per axis using `fixedEdges`.
   - **AspectResize**: corrects one axis, then derives the other from the aspect ratio.

4. **`findMatchedAnchors`** -- After correction, finds all project anchors that now align with the corrected frame (within epsilon=0.001). For translate mode, also discovers spacing guardrails.

### Three Modes

#### `.translate(threshold:)`
- Used by: **Movable.swift**
- Moving anchors: `.centersAndSides` (or `.center` if rotated)
- Frame correction: pure origin offset
- Special: includes equal-spacing candidate detection (extend-left, extend-right, middle-centering patterns)

#### `.resize(fixedEdges:threshold:)`
- Used by: **Resizable.swift**
- Moving anchors: the dragged edge + center + sameWidth/sameHeight (not for rotated elements)
- Frame correction: size adjustment relative to fixed edges
- Center snap multiplier: 2x (because moving the center by delta means the edge moves by 2*delta)
- Edge snap multiplier: 1x

#### `.aspectResize(fixedEdges:aspectRatio:threshold:)`
- Used by: **Scalable.swift**
- Moving anchors: the two edges at the dragged corner + centers + sameWidth/sameHeight
- Winner selection: only keeps one axis (smallest delta; bleed always wins over non-bleed)
- Frame correction: corrects the winning axis, then computes the other axis from aspectRatio

---

## Size Correction Logic

The `applySizeCorrection` and `applyDimensionDelta` private methods handle the relationship between snap deltas and frame changes during resize/scale:

### For dimension anchors (sameWidth/sameHeight):
- Delta is `targetDim - proposedDim` -- directly applied as size change.

### For position anchors (edges, centers):
- **Center snaps use multiplier=2**: moving center by `delta` means the free edge moves by `2*delta`.
- **Edge snaps use multiplier=1**: moving an edge by `delta` changes size by `delta`.
- Sign depends on which edge is fixed:
  - If the max-side edge is fixed (e.g., `.bottom` for horizontal), a positive delta shifts the min-side edge in the positive direction, which *shrinks* the element. So `sizeDelta = -delta * multiplier`.
  - If the min-side edge is fixed (e.g., `.top` for horizontal), a positive delta shifts the max-side edge in the positive direction, which *grows* the element. So `sizeDelta = +delta * multiplier`.

### `applyDimensionDelta` edge handling:
- **Fixed on min-side** (top/leading): size grows, origin stays.
- **Fixed on max-side** (bottom/trailing): size grows, origin shifts backward by delta.
- **No fixed edge** (translation only): origin shifts by delta.

---

## Anchor System

### SnapAnchorType
Eight types covering position and dimension matching:
- **Position**: `.top`, `.bottom`, `.leading`, `.trailing`, `.centerX`, `.centerY`
- **Dimension**: `.sameWidth`, `.sameHeight`

### Axis Convention
- `.horizontal` axis = Y direction (top, bottom, centerY, sameHeight)
- `.vertical` axis = X direction (leading, trailing, centerX, sameWidth)

This may seem counterintuitive but follows the convention that a "horizontal line" runs along the X axis at a Y position.

### ProjectSnapAnchor.Kind
Four kinds with different matching behavior:
- **`.point(CGFloat)`**: Standard threshold-based matching. Used for edges and centers.
- **`.bleedZone(range:snapTo:)`**: Range-based matching. If the moving position falls within the range, snaps to the border. Always wins over point anchors in winner selection.
- **`.dimension(value:CGFloat)`**: Threshold-based matching for width/height equality.
- **`.spacing(snapTo:ownerEdge:crossRange:)`**: Equal-spacing matching. Has a cross-range for perpendicular overlap validation.

### Anchor Generation
`ProjectSnapAnchor.from(...)` builds the full set of project anchors:
1. From each visible element (excluding the moving one): all 8 anchor types if not rotated, centers only if rotated.
2. From each surface frame: centerX and centerY anchors.
3. From bleed insets: bleed-zone anchors at each surface edge with bleed.

---

## Equal Spacing Detection (Translate Only)

The most complex part of the snapping system. Only active during translation (moving), not resize/scale.

### Candidate Generation (`translateCollectCandidates`)

After standard point/dimension candidates, computes spacing candidates:

1. **Reconstruct proposed frame** from moving anchors.
2. **Extract static frames** from project anchors (via `.leading` point anchors to get unique rects).
3. **Build sorted row** per axis: filter static frames by cross-axis overlap with the moving frame, sort by position, prune interior elements that break the overlap chain.
4. **Three patterns**:
   - **Extend from left**: moving element's leading edge = left neighbor's trailing edge + reference gap from any consecutive pair in the row.
   - **Extend from right**: moving element's trailing edge = right neighbor's leading edge - reference gap.
   - **Middle centering**: equalize gaps to both left and right neighbors.

### Row Pruning Algorithm

Interior elements that don't share cross-axis overlap with *both* their neighbors are removed. This prevents elements in different visual "bands" from interrupting valid spacing chains. End elements (first/last) are never removed.

### Guardrail Generation (`translateFindMatchedAnchors`)

After standard anchor matching (Phase 1), Phase 2 reconstructs the equal-spacing context:
1. Insert corrected frame into sorted static sequence.
2. Compute target gap from immediate neighbor.
3. Scan all consecutive pairs for matching gaps (within epsilon).
4. Require at least 2 matching gaps to display guardrails (1 gap is trivial).

---

## Remote Configuration

### Feature Flag
- Feature key: `"snapping"` (defined on `SFGRemoteFeatureClient.FeatureKey`)
- Variable key: `"threshold"`, type `Double`, default `4.0`
- Debug settings: title "Snapping Threshold", group `.cgdCreationPath`

### Zoom-Dependent Threshold
Each modifier computes `baseThreshold / zoomScale`:
- At 1x zoom: threshold = 4pt
- At 2x zoom: threshold = 2pt (more precise snapping when zoomed in)
- At 0.5x zoom: threshold = 8pt (more forgiving when zoomed out)

---

## Visual Feedback

### Guardrails

Three visual indicator types rendered by `GuardrailsOverlayModifier`:

1. **Alignment lines**: Thin lines showing edge/center alignment between elements. Span from one element's edge to the other's.

2. **Dimension arrows**: Double-headed arrows with extension lines showing same-width or same-height. Smart placement algorithm scores 4 possible side combinations (preferred side, flipped) considering:
   - Right-handed user preference (width arrows above, height arrows to the left)
   - Available visible space (penalty if < 20pt from screen edge)
   - Overlap between the two arrows (penalty if they intersect)

3. **Spacing indicators**: Short perpendicular extension lines connected by a line showing equal gap distances. Drawn at the midpoint of the cross-axis overlap between adjacent elements.

### Haptics

`SnapHapticTracker` fires `.snapping` sensory feedback whenever the set of snapped positions changes. Tracks horizontal and vertical snapped positions separately. Uses approximate comparison (epsilon=0.1) to prevent rapid re-triggering from sub-pixel drift.

---

## Rotation Handling

When an element is rotated (`.rotation != .zero`):
- Only center anchors are used for snapping (`.center` set = `[.centerX, .centerY]`).
- Edge/dimension anchors are excluded because the visual edges no longer align with the coordinate axes.
- Both the `relevantSnapAnchorTypes(isElementRotated:)` methods on `Edge` and `Corner` respect this.
- Static project anchors for rotated elements also only include centers.

### Rotation Compensation in Resize/Scale

Both `Resizable` and `Scalable` have `elementShiftToCompensateForRotation(...)` methods. Because SwiftUI's `.rotationEffect` rotates only the visual layer (not the frame), scaling must be done relative to the non-rotated frame's anchor corner. The compensation shift ensures the *rotated* anchor corner stays visually pinned by computing the difference between the rotated anchor positions before and after scaling.

---

## Cross-Surface Transfer (Movable Only)

When a user drags an element so its center enters a different surface (e.g., Calendar top/bottom, Book pages):
1. Element center is converted to the target surface's coordinate space.
2. Content frame is adjusted preserving the origin offset.
3. Frames are scaled back to original size using the target surface's scale factor.
4. The external system is notified via `onMoveBetweenSurfaces`.
5. Gesture state is reset to continue the drag on the new surface.
6. `reconcileCompositeFrameIfNeeded` compensates for composite layout shifts after transfer.

---

## Bleed Zone Snapping

Products with bleed (extra printable area beyond the trim line) generate special bleed-zone anchors:
- Created by `BleedInsets.bleedAnchors(for:)`.
- Use a buffered range of 1.5x the bleed value.
- When any element edge enters this range, it snaps to the surface border.
- Bleed anchors always win over regular point anchors in `selectWinners`.
- Center anchors are explicitly excluded from bleed matching.
- Bleed-only anchors can be generated separately via `bleedOnlyFrom(...)`.

---

## Key Coordinate Spaces

The system operates in two coordinate spaces:

1. **Surface coordinates**: Element frames relative to their surface origin. Used for element.frame storage and content frame updates.

2. **Composite coordinates**: Element frames in the shared layout that contains all surfaces. Used for snapping calculations (because elements on different surfaces need a common reference). The `surfaceToCompositeOffset` is computed at gesture start as `coordinatesDiff` (element.frame.origin offset to elementRect.origin).

Modifiers convert between these spaces:
- Proposed frame for snapping is in composite coordinates.
- Correction is computed in composite coordinates, then applied back to surface coordinates.

---

## Edge and Corner Enums

### Edge (used by Resizable)
Cases: `.top`, `.bottom`, `.leading`, `.trailing`

| Edge | Fixed Edge | Relevant Anchors (non-rotated) | Relevant Anchors (rotated) |
|------|-----------|-------------------------------|---------------------------|
| `.top` | `.bottom` | `.top`, `.centerY`, `.sameHeight` | `.centerY` |
| `.bottom` | `.top` | `.bottom`, `.centerY`, `.sameHeight` | `.centerY` |
| `.leading` | `.trailing` | `.leading`, `.centerX`, `.sameWidth` | `.centerX` |
| `.trailing` | `.leading` | `.trailing`, `.centerX`, `.sameWidth` | `.centerX` |

### Corner (used by Scalable)
Cases: `.topLeading`, `.topTrailing`, `.bottomLeading`, `.bottomTrailing`

| Corner | Fixed Edges | Relevant Anchors (non-rotated) | Relevant Anchors (rotated) |
|--------|------------|-------------------------------|---------------------------|
| `.topLeading` | `[.bottom, .trailing]` | `.top`, `.leading`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight` | `.centerX`, `.centerY` |
| `.topTrailing` | `[.bottom, .leading]` | `.top`, `.trailing`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight` | `.centerX`, `.centerY` |
| `.bottomLeading` | `[.top, .trailing]` | `.bottom`, `.leading`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight` | `.centerX`, `.centerY` |
| `.bottomTrailing` | `[.top, .leading]` | `.bottom`, `.trailing`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight` | `.centerX`, `.centerY` |

---

## Test Coverage

Tests are in `Tests/SFGSurfaceSnappingTests/SnappingCalculatorTests.swift`.

### Test Classes

1. **HorizontalEqualSpacingTests** (8 tests): Tests horizontal equal spacing for first, middle, and last elements; unequal gap rejection; multi-element matching; and cross-band interference (tall moving elements with disturbing elements in different Y bands).

2. **MultiRowEqualSpacingTests** (2 tests): Tests that elements in different visual rows/columns (different Y/X bands) don't break spacing detection for elements in a different band.

3. **VerticalEqualSpacingTests** (8 tests): Mirror of horizontal tests for the vertical axis.

Test helpers include `buildProjectAnchors(...)` for constructing anchor sets and `spacingGuardrails(...)` for extracting spacing-specific guardrails from results.

---

## Dependencies

- **ComposableArchitecture (TCA)**: `SnappingCalculator` is a TCA `DependencyKey`. Modifiers use `@Dependency(\.snappingCalculator)`.
- **SFGRemoteFeatureClient**: Remote config for the snapping threshold. Feature key and variable key defined in `SnappingCalculator.swift`.
- **SFGProject**: Types like `SFGSurfaceElement`, `SFGSurface`, `Product`.
- **SFGCreationViewState**: For surface selection context and scale factors.

---

## Historical Context / Refactoring Notes

### IOS-17211 (correctedFrame refactoring)
Replaced `snapCorrection: CGSize` with `correctedFrame: CGRect` in `SnappingResult`. This simplified consumer code -- modifiers now read the corrected frame directly instead of applying a correction vector. The `SnapAdjustmentStrategy` was refactored into what is now `CorrectionMode`.

### IOS-17213 (zoom-dependent threshold)
Made the snapping threshold zoom-aware and remote-configurable. The threshold is `baseThreshold / zoomScale`, computed in each modifier. Factory methods on `CorrectionMode` accept threshold as a parameter rather than reading it internally.

### IOS-17216 (spacing/alignment)
Added equal-spacing detection and the spacing guardrail visual indicators.

### IOS-17214 (edge cases)
Current branch. Addresses edge cases in spacing detection, particularly the row pruning algorithm that prevents elements in different visual bands from interfering with spacing chains.
