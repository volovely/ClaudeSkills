---
name: sfg-presentation
description: >
  Generate a Shutterfly-branded PowerPoint presentation using python-pptx.
  Use when the user asks to "create a presentation", "build slides",
  "make a pptx", "sfg-presentation", or wants to generate a PowerPoint deck.
argument-hint: "<topic or outline>"
---

# SFG Presentation Skill

Generate a Shutterfly-branded `.pptx` presentation using `python-pptx`. Output to `~/Desktop/<Presentation_Name>.pptx`.

## Usage

```
/sfg-presentation <topic or outline>
```

The user will provide either a topic, a bullet list, a detailed plan, or a markdown file with content. Build the full deck from that input.

## Instructions

### Step 1: Ensure python-pptx is installed

Run `python3 -c "import pptx"` to check. If missing, install with `pip3 install --break-system-packages python-pptx`.

### Step 2: Gather content

- If the user provides a file path, read it
- If the user provides inline content, use that
- Clarify slide count / structure if the input is vague

### Step 3: Write a single Python script

Write a self-contained Python script to `~/Desktop/build_presentation.py` that generates the `.pptx`. The script MUST follow every convention below.

### Step 4: Run the script and open the result

```bash
python3 ~/Desktop/build_presentation.py
open ~/Desktop/<Presentation_Name>.pptx
```

---

## Shutterfly Brand Palette (MANDATORY)

Every presentation MUST use these exact colors and fonts. Define them as module-level constants.

```python
from pptx.dml.color import RGBColor
from pptx.util import Inches, Pt
from pptx.enum.text import PP_ALIGN
from pptx.enum.shapes import MSO_SHAPE

# ── Brand palette ──
DARK         = RGBColor(0x24, 0x1F, 0x16)   # Titles, body text, dark backgrounds
ACCENT       = RGBColor(0xC9, 0x3E, 0x24)   # Headers, highlights, accent bars
LIGHT_BG     = RGBColor(0xF9, 0xF8, 0xF6)   # Slide backgrounds
WHITE        = RGBColor(0xFF, 0xFF, 0xFF)    # Text on dark, alternate backgrounds
GRAY         = RGBColor(0x88, 0x88, 0x88)    # Subtitles, captions
LIGHT_GRAY   = RGBColor(0xE8, 0xE6, 0xE2)   # Card borders, dividers
CODE_BG      = RGBColor(0x2B, 0x2B, 0x2B)   # Code block background
CARD_BG      = RGBColor(0xFF, 0xFF, 0xFF)    # Card fill
ACCENT_LIGHT = RGBColor(0xFD, 0xF0, 0xED)   # Highlight boxes, callout backgrounds

# ── Fonts ──
FONT = "Calibri"       # All body/title text
MONO = "Courier New"   # Code blocks, monospace content

# ── Slide dimensions (16:9 widescreen) ──
SLIDE_W = Inches(13.333)
SLIDE_H = Inches(7.5)
```

## Required Helper Functions

The script MUST include these helper functions. Copy them exactly — they are the shared building blocks for all slides.

### `set_slide_bg(slide, color)`
Set a solid fill background on a slide.

### `add_shape(slide, left, top, width, height, fill_color, line_color=None)`
Add a `RECTANGLE` shape. If no `line_color`, hide the border with `shape.line.fill.background()`.

### `add_rounded_rect(slide, left, top, width, height, fill_color, line_color=None)`
Same as `add_shape` but uses `ROUNDED_RECTANGLE`.

### `set_text(shape, text, font_name=FONT, size=Pt(18), color=DARK, bold=False, align=PP_ALIGN.LEFT)`
Clear shape's text frame, set word_wrap, write text into the first paragraph with the given formatting.

### `add_para(tf, text, font_name=FONT, size=Pt(18), color=DARK, bold=False, align=PP_ALIGN.LEFT, space_before=Pt(4), space_after=Pt(2), level=0)`
Append a new paragraph to an existing text frame.

### `add_textbox(slide, left, top, width, height)`
Shorthand for `slide.shapes.add_textbox(...)`.

### `add_header_bar(slide)`
Draw an ACCENT-colored rectangle at the very top of the slide: `Inches(0), Inches(0), SLIDE_W, Inches(0.06)`. Call this on every content slide (not the title slide).

### `add_slide_title(slide, title, subtitle=None)`
Calls `add_header_bar`, then places a textbox at `(0.8, 0.35)` with the title at `Pt(32)` bold DARK. If subtitle is given, adds it as a `Pt(16)` GRAY paragraph below.

### `add_code_block(slide, left, top, width, height, code_text, font_size=Pt(11))`
Dark rounded rect (`CODE_BG`) with monospace white text. Margins: `0.2` left/right, `0.15` top/bottom. Split `code_text` by newlines, one paragraph per line.

### `add_bullet_list(tf, items, size=Pt(16), color=DARK, bold_prefix=False, indent_level=0)`
Append bullet items to a text frame. If `bold_prefix=True`, split on ` — ` or `: ` and bold the prefix portion.

### `add_card(slide, left, top, width, height, title, body_lines, accent_color=ACCENT)`
Rounded rect with `CARD_BG` fill and `LIGHT_GRAY` border. Thin accent bar across the top. Title in accent color bold, body lines in DARK `Pt(13)`.

## Slide Layout Patterns

Use blank slide layouts (`prs.slide_layouts[6]`) for all slides. Build everything from shapes and textboxes — never rely on placeholder content.

### Title Slide (first slide only)
- Background: `DARK`
- Centered title in `WHITE`, `Pt(48)`, bold
- Accent bar under the title (horizontal `ACCENT` rectangle)
- Subtitle below in `LIGHT_GRAY`, `Pt(24)`
- Attribution at bottom in `GRAY`, `Pt(16)`

### Content Slides (all other slides)
- Background: `LIGHT_BG`
- Always call `add_slide_title(slide, title, subtitle)` — this adds the accent header bar + title area
- Content starts at `y = Inches(1.5)` or below
- Left/right margins: `Inches(0.8)` minimum

### Common Layout Types

**Two-column**: Left content at `x=0.8`, right content at `x=6.5–7.0`. Each column ~5.5" wide.

**Comparison table**: Use `slide.shapes.add_table(rows, cols, ...)`. Header row: DARK bg, WHITE text. Alternating row fills: `ACCENT_LIGHT` and `CARD_BG`.

**Card grid**: Use `add_card()` or `add_rounded_rect()` placed in a row. Typical: 3–4 cards across at equal width with `Inches(0.25)` gap.

**Numbered steps**: ACCENT-filled circle/oval for the number, title textbox to the right, body textbox below the title.

**Screenshot slide**: Use `slide.shapes.add_picture(image_path, left, top, width, height)`. Add a GRAY caption label below each image.

**Highlight/callout box**: `add_rounded_rect` with `ACCENT_LIGHT` fill and `ACCENT` border. Used for pro tips, key insights, important notes.

## Script Structure

```python
#!/usr/bin/env python3
"""Generate <name> presentation (Shutterfly branded)."""

from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR
from pptx.enum.shapes import MSO_SHAPE

# ── Brand palette constants ──
# ... (as above)

# ── Helper functions ──
# ... (as above)

# ── Slide builders ──
# One function per slide: build_slide_N_short_name(prs)
# Each function:
#   1. Adds a blank slide
#   2. Sets background
#   3. Adds title (via add_slide_title for content slides)
#   4. Adds content using helpers

def build_slide_1_title(prs):
    ...

def build_slide_2_...(prs):
    ...

# ── Main ──
def main():
    prs = Presentation()
    prs.slide_width = SLIDE_W
    prs.slide_height = SLIDE_H

    build_slide_1_title(prs)
    build_slide_2_...(prs)
    # ...

    out = "/Users/roman.volovelskyi/Desktop/<Name>.pptx"
    prs.save(out)
    print(f"Saved -> {out}")
    print(f"Slides: {len(prs.slides)}")

if __name__ == "__main__":
    main()
```

## Typography Scale

| Element | Font | Size | Color | Bold |
|---------|------|------|-------|------|
| Slide title | Calibri | 32pt | DARK | Yes |
| Subtitle | Calibri | 16pt | GRAY | No |
| Section header | Calibri | 20-22pt | ACCENT or DARK | Yes |
| Body text | Calibri | 14-16pt | DARK | No |
| Bullet items | Calibri | 14-16pt | DARK | No |
| Code | Courier New | 10-12pt | WHITE on CODE_BG | No |
| Captions | Calibri | 12-13pt | GRAY | No |
| Card title | Calibri | 16pt | ACCENT | Yes |
| Card body | Calibri | 13pt | DARK | No |
| Title slide title | Calibri | 48pt | WHITE | Yes |
| Title slide subtitle | Calibri | 24pt | LIGHT_GRAY | No |

## Important Rules

- NEVER use slide layout placeholders — always blank layout + shapes/textboxes
- NEVER use colors outside the brand palette
- ALWAYS use `Calibri` for text and `Courier New` for code
- ALWAYS set `word_wrap = True` on text frames
- ALWAYS hide shape borders unless intentionally visible (`shape.line.fill.background()`)
- Every content slide gets the accent header bar via `add_slide_title`
- Title slide uses DARK background; all other slides use LIGHT_BG
- Code blocks always use dark rounded rects with white monospace text
- Tables get DARK header row with WHITE text, alternating ACCENT_LIGHT/CARD_BG rows
- For mixed bold/normal text in a paragraph, use multiple runs (`.add_run()`) instead of separate textboxes
- Output path is always `~/Desktop/<Descriptive_Name>.pptx`
- Script path is always `~/Desktop/build_presentation.py`
