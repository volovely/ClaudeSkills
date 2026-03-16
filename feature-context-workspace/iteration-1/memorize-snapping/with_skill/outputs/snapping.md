# Snapping

> Snap-to-guide system that aligns surface elements to each other's edges, centers, dimensions, bleed zones, and equal-spacing positions during move, resize, and scale gestures.

## Location

- **Primary directory:** `SFG/SFG/Sources/SFGSurface/Snapping/`
- **Related files:**
  - `SFG/SFG/Sources/SFGSurface/Views/Modifiers/Movable.swift` — consumes snapping for drag/move gestures (translate mode)
  - `SFG/SFG/Sources/SFGSurface/Views/Modifiers/Resizable.swift` — consumes snapping for edge-handle resize gestures (resize mode)
  - `SFG/SFG/Sources/SFGSurface/Views/Modifiers/Scalable.swift` — consumes snapping for corner-handle scale gestures (aspectResize mode)
  - `SFG/SFG/Sources/SFGSurface/Extensions/Utills+Extensions.swift` — defines `Edge`, `Corner`, their `fixedEdges`, and `relevantSnapAnchorTypes`
  - `SFG/SFG/Sources/SFGSurface/Views/Modifiers/ObjectEditing.swift` — orchestrates all three modifiers, builds `projectAnchors` array, owns `activeGuardrails` state
  - `SFG/SFG/Sources/SFGSurface/Views/OverlayAndBackground/GuardrailsOverlayModifier.swift` — renders visual guide lines/arrows from `[Guardrail]` via SwiftUI preference key

## Architecture

The snapping system follows a pipeline pattern:

```
Gesture input (proposed frame)
    -> SnappingCalculator.computeSnapping(...)
        -> CorrectionMode (strategy object with 4 closures)
            1. collectCandidates  -- find all anchor pairs within threshold
            2. selectWinners      -- pick best match per axis (horizontal & vertical)
            3. computeCorrectedFrame -- apply correction to proposed frame
            4. findMatchedAnchors -- identify which anchors aligned (for visual guides)
        <- SnappingResult { correctedFrame, matchedAnchors }
    -> Modifier applies correctedFrame to element
    -> matchedAnchors converted to [Guardrail] for visual overlay
```

**Key design choice:** `CorrectionMode` is a struct of closures (strategy pattern), not a protocol. Three factory methods produce different strategies: `.translate(threshold:)`, `.resize(fixedEdges:threshold:)`, `.aspectResize(fixedEdges:aspectRatio:threshold:)`.

**Coordinate spaces:** Snapping operates in "composite layout" coordinates (the coordinate space of the full multi-surface canvas). Modifiers convert between surface-local element frames and composite frames using `surfaceToCompositeOffset`.

## Core Concepts

- **SnapAnchorType** -- Which geometric feature of a rect to snap: `.top`, `.bottom`, `.leading`, `.trailing`, `.centerX`, `.centerY`, `.sameWidth`, `.sameHeight`. Each has an `axis` (`.horizontal` for Y-dimension types, `.vertical` for X-dimension types). `isCenter` and `isDimension` computed properties distinguish categories.
- **SnapAnchorAxis** -- `.horizontal` (Y axis: top/bottom/centerY/sameHeight) or `.vertical` (X axis: leading/trailing/centerX/sameWidth). The naming follows "which axis the guide line runs along" not "which coordinate changes."
- **ProjectSnapAnchor** -- A concrete anchor instance with `type`, `axis`, `kind` (`.point`, `.bleedZone`, `.dimension`, `.spacing`), and `ownerRect`. The `snapTarget` property returns the effective position regardless of kind.
- **SnapCandidate** -- A pairing of a moving anchor and a project anchor with the computed `delta` (correction needed to align them).
- **AxisWinner** -- The winning candidate per axis: stores `delta`, `anchorType`, and `isBleed` (bleed zones always win over regular anchors).
- **FixedEdges** -- `OptionSet` marking which edges stay pinned during resize/scale (e.g., dragging `.top` edge fixes `.bottom`).
- **CorrectionMode** -- Strategy object (struct of 4 closures) that encapsulates the snapping pipeline for a specific gesture type.
- **SnappingResult** -- Output of the pipeline: `correctedFrame: CGRect` (the frame after snapping) and `matchedAnchors: [ProjectSnapAnchor]` (anchors that aligned, used for visual guides).
- **Guardrail** -- Visual indicator model: `.alignment(position, lineRange)` for guide lines, `.dimension(elementRect, side)` for same-width/height arrows, `.spacing(gapStart, gapEnd, crossPosition)` for equal-spacing indicators.
- **BleedInsets** -- Product-specific bleed zones (top/bottom/leading/trailing in screen coordinates) that create `.bleedZone` anchors near surface edges. Elements snap to the surface border when entering a bleed zone. The bleed zone range is 1.5x the actual bleed value (buffered).
- **AxisProjection** -- 1D projection of a CGRect onto a single axis (min, max, mid, size), used to generalize axis-specific logic without `switch` statements.

## Key Files

### SnappingCalculator.swift
Entry point. `computeSnapping(proposedFrame:anchorTypes:borderWidth:projectAnchors:mode:)` builds moving anchors from the proposed frame, then delegates to `CorrectionMode`'s four closures. Registered as a `DependencyKey` for TCA dependency injection. Also defines the remote config keys: feature flag `"snapping"` and variable `"threshold"` (default `4.0`).

### CorrectionMode.swift
The core algorithm file. Contains the three factory methods (`.translate`, `.resize`, `.aspectResize`) and all shared implementations:

- **`defaultCollectCandidates`** -- Iterates all moving x project anchor pairs. Filters by `isComparable` (same axis, same category). Skips center anchors against bleed zones. Uses `matchDelta` to check if within threshold.
- **`translateCollectCandidates`** -- Extends default with spacing candidates. Reconstructs the proposed frame from moving anchors, extracts static element frames, and calls `computeSpacingCandidates` for each axis.
- **`computeSpacingCandidates`** -- Builds a sorted row of static elements that overlap the moving element on the cross-axis. Tries three patterns: extend-left (snap moving.min to neighbor.max + referenceGap), extend-right (snap moving.max to neighbor.min - referenceGap), and middle centering (equalize gaps to both neighbors).
- **`buildSortedRow`** -- Filters static frames to those with cross-axis overlap, sorts along the axis, prunes interior elements that don't have cross-axis overlap with both neighbors (to avoid corrupting gap detection).
- **`defaultSelectWinners`** -- Picks smallest absolute delta per axis. Bleed anchors always win over non-bleed.
- **`aspectResize.selectWinners`** -- After default selection, keeps only ONE axis winner (the one with smallest delta). Bleed always wins. This ensures aspect ratio is preserved by correcting one axis and deriving the other.
- **`applySizeCorrection`** -- For dimension anchors, delta is direct size difference. For position anchors, uses multiplier of **2 for center snaps** and **1 for edge snaps**. Determines sign based on whether the moving edge is on the min side (positive delta shrinks element) or max side.
- **`applyDimensionDelta`** -- Applies a size delta to a frame respecting fixed edges. If fixed on min-side (top/leading): size grows, origin unchanged. If fixed on max-side (bottom/trailing): origin shifts back, size grows. If no fixed edge on that axis: pure translation.
- **`translateFindMatchedAnchors`** -- Phase 1: standard alignment matching. Phase 2: spacing guardrails. Expands corrected frame by borderWidth to match visual space, then calls `computeSpacingGuardrails`.
- **`computeSpacingGuardrails`** -- Inserts corrected frame into sorted row, computes target gap from neighbor, scans all consecutive pairs for matching gaps. Requires at least 2 matching gaps to display (1 gap is trivial).

### Models/ProjectSnapAnchor.swift
The anchor model. `Kind` enum has four cases: `.point(CGFloat)`, `.bleedZone(range:snapTo:)`, `.dimension(value:)`, `.spacing(snapTo:ownerEdge:crossRange:)`. The `isComparable(to:)` method gates which anchor pairs can be compared (same axis, same dimension/position category, cross-range overlap for spacing anchors, no center-to-spacing comparisons). Also has `Array<ProjectSnapAnchor>.from(...)` factory that builds all project anchors from resolved element positions, surface centers, and bleed insets.

### Models/SnapAnchorType.swift
Enum of 8 cases. `axis` maps each type to horizontal/vertical. Convenience sets: `.all`, `.center`, `.centersAndSides`.

### Models/SnapAnchorAxis.swift
Two-case enum with `perpendicular` property. Contains `AxisProjection` struct and `CGRect.projection(on:)` extension.

### Models/FixedEdges.swift
`OptionSet` with `.top`, `.bottom`, `.leading`, `.trailing`.

### Models/BleedInsets.swift
Converts product bleed (inches) to screen coordinates via `surfaceDPI * surfaceScale`. Creates `.bleedZone` anchors with range of 1.5x the bleed value.

### Models/Guardrail.swift
Visual guide model with three kinds: `.alignment`, `.dimension`, `.spacing`. The `toGuardrails(elementRect:borderWidth:isBeingTransformed:visibleRect:)` extension on `[ProjectSnapAnchor]` converts matched anchors to guardrails. Includes smart dimension-arrow placement that scores 4 combinations of sides to avoid overlap and off-screen placement.

### SnapHapticTracker.swift
`@Observable` class that detects when snapped anchor positions change and toggles a `trigger` bool. Compares current vs previous snapped positions per axis with epsilon of `0.1`. Used to fire haptic feedback.

## Implementation Details

### Threshold and zoom scaling
The base snapping threshold defaults to `4.0` points and is configurable via remote config (`SFGRemoteFeatureClient.VariableKey.snappingThreshold`). Before each snapping computation, the threshold is divided by `zoomScale` so that snapping feels consistent regardless of zoom level: `snapThreshold = baseThreshold / zoomScale`.

### Center snap multiplier
When snapping a center anchor during resize/scale, the correction applied to the frame size is **doubled** (multiplier = 2). This is because moving the center by `delta` requires the edge to move by `2 * delta` to keep the opposite edge fixed. Edge snaps use multiplier = 1.

### Aspect-ratio scaling (aspectResize mode)
Only one axis winner is kept (the one with the smallest delta; bleed always wins). After correcting that axis's size, the other axis is derived: `width = height * aspect` or `height = width / aspect`. This preserves the element's aspect ratio.

### Translate mode: spacing snapping
In addition to standard alignment snapping, translate mode detects equal-spacing patterns. It:
1. Builds a sorted row of static elements with cross-axis overlap to the moving element
2. Prunes interior elements that break the overlap chain
3. Tries three patterns: extend-left, extend-right, and middle-centering
4. For guardrails, requires at least 2 matching gaps to display (1 gap is trivial)

### Bleed zone behavior
Bleed zones use range-based matching (`.bleedZone(range:snapTo:)`): if the moving anchor falls within the range, it snaps to the surface border. Range is 1.5x the actual bleed value. Bleed anchors always win over regular anchors in winner selection. Center anchors cannot snap to bleed zones.

### Rotated elements
When an element is rotated (rotation != 0), only center anchors are used for snapping (`.center` set for project anchors, `.centerX`/`.centerY` for snap types). Non-rotated elements use all anchor types. Modifiers compensate for rotation when adjusting the element's origin after resizing.

### Coordinate space reconciliation (Movable)
During cross-surface drag transfers (e.g., Calendar top/bottom swap), `reconcileCompositeFrameIfNeeded` adjusts `initialCompositeFrame` by the delta between the tracked surface's current and previous positions. This keeps the proposed frame in the correct composite coordinate space.

### Border width handling
Element border widths expand the visual rect. `CGRect.snapAnchors(including:borderWidth:)` offsets anchor positions by `borderWidth` and uses `insetBy(dx: -borderWidth, ...)` for `ownerRect`. In translate mode's guardrail phase, the corrected frame is expanded by `borderWidth` before gap comparison to match visual space.

### Edge/Corner anchor type selection
- **Edge resize:** The moving edge anchor + center on that axis + sameWidth/sameHeight. E.g., dragging `.top` uses `[.top, .centerY, .sameHeight]`.
- **Corner scale:** Both edge anchors for the active corner + both centers + sameWidth + sameHeight. E.g., `.topLeading` uses `[.top, .leading, .centerX, .centerY, .sameWidth, .sameHeight]`.

## Dependencies & Integration

**Imports:**
- `Foundation` / `SwiftUI` -- geometry types, view system
- `Dependencies` (TCA) -- `SnappingCalculator` registered as dependency key
- `SFGRemoteFeatureClient` -- remote config for threshold
- `SFGProject` -- `SFGSurface`, `SFGSurfaceElement`, `Product` types
- `ComposableArchitecture` / `SFGCreationViewState` -- shared state for surface selection and scale factors

**Integration flow:**
1. `ObjectEditing.swift` builds `projectAnchors` via `[ProjectSnapAnchor].from(resolvedElementsPositioning:resolvedSurfacesFrames:excludingElementId:visibleRect:bleedInsets:)` when an element starts being transformed
2. Passes `projectAnchors` and `activeGuardrails` binding to `Movable`, `Resizable`, `Scalable` modifiers
3. Each modifier calls `snappingCalculator.computeSnapping(...)` with the appropriate `CorrectionMode`
4. Writes matched anchors back to `activeGuardrails` as `[Guardrail]`
5. `GuardrailPreferenceKey` propagates guardrails up to `GuardrailsOverlayModifier` which draws alignment lines, dimension arrows, and spacing indicators
6. `SnapHapticTracker` fires haptic feedback when snapped positions change

**What's NOT here:**
- Element selection, tap handling, and overall editing orchestration live in `ObjectEditing.swift`
- Surface layout and positioning logic lives elsewhere in `SFGSurface`
- The actual element model (`SFGSurfaceElement`) is in `SFGProject`

## Notes

- The `correctedFrame` refactoring (IOS-17211) replaced the old `snapCorrection: CGSize` approach. The new API returns a fully corrected frame instead of a correction vector.
- The `CorrectionMode` struct-of-closures approach was introduced to consolidate three different snapping behaviors into a single pipeline with swappable steps (previously was `SnapAdjustmentStrategy`).
- Spacing snapping is only available in translate mode (not resize/scale).
- The `alignmentEpsilon` for post-correction matching is `0.001`, while `SnapHapticTracker` uses a coarser epsilon of `0.1`.
- Remote config feature flag key is `"snapping"`, variable key is `"threshold"`.
- `Edge.fixedEdgeOnResizeAxis`, `Edge.relevantSnapAnchorTypes`, `Corner.fixedEdges`, `Corner.relevantSnapAnchorTypes` extensions live in `Utills+Extensions.swift`, not in the Snapping directory.
- Zoom-dependent threshold (IOS-17213) added remote config integration for the snap distance threshold.

---
*Last updated: 2026-03-16*
