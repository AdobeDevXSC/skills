---
name: generate-pptx
description: Generates a PowerPoint (PPTX) presentation from a customer POV document for AEM Edge Delivery Services. Builds slides programmatically using Adobe brand colors and layout conventions — no SharePoint template required.
license: Apache-2.0
metadata:
  version: "2.0.0"
---

# Generate PPTX — AEM Edge Delivery Services Customer POV

You are generating a PowerPoint presentation based on a customer POV document for AEM Edge Delivery Services + Document Authoring.

---

## Step 1: Collect Inputs

You need the following before proceeding:

1. **Customer name** — the full customer name (e.g., "Acme Corporation")
2. **Customer slug** — lowercase, hyphens instead of spaces/special chars (e.g., `acme-corporation`)
3. **POV content** — the full text of the POV markdown document (paste directly or provide the file path)
4. **Date** — today's date in `YYYY-MM-DD` format

If invoked from the `/customer-pov` skill, these values are already available in context — use them directly without asking again.

---

## Step 2: Generate the Customer PPTX

First, resolve the skill's absolute output paths:

```bash
SKILL_DIR="$(git rev-parse --show-toplevel)/plugins/aem/edge-delivery-services/skills/generate-pptx"
mkdir -p "$SKILL_DIR/output" "$SKILL_DIR/scripts"
echo "SKILL_DIR=$SKILL_DIR"
```

Write a Python script at `$SKILL_DIR/scripts/build_<customer-slug>_pptx.py` that builds the presentation **from scratch** — no template file, no SharePoint dependency.

### Adobe Brand Palette

Always use these exact color constants at the top of the script:

```python
from pptx.dml.color import RGBColor

RED       = RGBColor(0xFA, 0x0F, 0x00)   # Adobe Red — stripes, header bars, logo box
DARK      = RGBColor(0x1A, 0x1A, 0x1A)   # Near-black — cover/divider backgrounds, footer
WHITE     = RGBColor(0xFF, 0xFF, 0xFF)   # Text on dark slides
LIGHT_BG  = RGBColor(0xF5, 0xF5, 0xF5)  # Content slide background
MID_GREY  = RGBColor(0x6E, 0x6E, 0x6E)  # Body text on light slides
INTERNAL  = RGBColor(0xB8, 0x00, 0x00)  # Internal badge red (darker than RED)
FONT_FACE = "Adobe Clean"               # Use for all text runs
```

### Slide Dimensions

```python
from pptx.util import Inches

W = Inches(13.333)   # 16:9 widescreen width
H = Inches(7.5)      # 16:9 widescreen height

prs = Presentation()
prs.slide_width  = W
prs.slide_height = H

BLANK = prs.slide_layouts[6]   # blank layout — used for every slide
```

### Layout Conventions

Every slide type is built from the same BLANK layout using helper functions. Follow these exact layout rules:

#### Cover Slide
- Full DARK background (`13.333" × 7.5"`)
- RED left accent stripe (`0.25" wide × full height`)
- Customer name: 44pt bold WHITE, left edge at `x=0.6"`, top at `y=1.6"`
- Subtitle line (e.g. "Point of View — AEM Edge Delivery Services"): 26pt, `RGBColor(0xCC,0xCC,0xCC)`, `y=2.8"`
- Thin RED horizontal rule at `y=3.65"`
- Migration score / scope summary: 16pt, `RGBColor(0xCC,0xCC,0xCC)`, `y=3.75"`
- Prepared-by block: 13pt, `RGBColor(0x88,0x88,0x88)`, `y=5.5"`
- Adobe logo box: RED rectangle `2.0" × 0.6"` at bottom-right (`x=11.0"`, `y=6.6"`), "adobe" in 22pt bold WHITE centered

#### Section Divider Slide (external sections)
- Full DARK background
- RED left stripe (`0.18" wide × full height`)
- Section title: 40pt bold WHITE at `(x=0.55", y=2.8")`
- Optional one-line subtitle: 18pt italic `RGBColor(0xCC,0xCC,0xCC)` at `y=4.1"`
- Breadcrumb footer: 10pt `RGBColor(0x99,0x99,0x99)` at `y=6.9"` — e.g. `"<Customer Name> — AEM Edge Delivery Services POV"`

#### Section Divider Slide (internal — INTERNAL USE ONLY)
- Same as external section divider
- Add INTERNAL badge: RED rectangle (`2.5" × 0.35"`) at `(x=10.5", y=0.15")`, "INTERNAL USE ONLY" in 8pt bold WHITE centered

#### Content Slide (standard)
- WHITE (`#FFFFFF`) full background
- Thin RED top bar: `13.333" × 0.12"` at top
- Slide title: 22pt bold DARK at `(x=0.5", y=0.22")`
- Body content area: starts at `y=0.95"`, use `add_multiline` or bullet-style text boxes
- Footer bar: DARK rectangle `13.333" × 0.28"` at `y=7.22"`
  - Left label: 8pt WHITE — `"Adobe Confidential | AEM Edge Delivery Services"`
  - Right label: 8pt MID_GREY — `"© <YEAR> Adobe. All Rights Reserved."` right-aligned

#### Content Slide (internal)
- Same as standard content slide
- Add INTERNAL badge identical to the internal section divider badge

#### Two-Column Content Slide
- Same background, title, and footer as standard content slide
- Left column: `x=0.5"`, `w=6.0"`, `y=0.95"`, `h=6.0"`
- Right column: `x=6.9"`, `w=6.0"`, `y=0.95"`, `h=6.0"`
- Vertical divider: MID_GREY rectangle `0.02" wide × 5.5" tall` at `x=6.65"`, `y=1.1"`

### Helper Functions to Include

```python
from pptx.util import Inches, Pt
from pptx.enum.text import PP_ALIGN
import lxml.etree as etree

def add_rect(slide, x, y, w, h, fill_color, line_color=None, line_width_pt=0):
    shape = slide.shapes.add_shape(1, Inches(x), Inches(y), Inches(w), Inches(h))
    shape.fill.solid()
    shape.fill.fore_color.rgb = fill_color
    if line_color:
        shape.line.color.rgb = line_color
        shape.line.width = Pt(line_width_pt)
    else:
        shape.line.fill.background()
    return shape

def add_text(slide, text, x, y, w, h,
             font_size=18, bold=False, color=DARK,
             align=PP_ALIGN.LEFT, wrap=True, italic=False):
    txb = slide.shapes.add_textbox(Inches(x), Inches(y), Inches(w), Inches(h))
    tf = txb.text_frame
    tf.word_wrap = wrap
    p = tf.paragraphs[0]
    p.alignment = align
    run = p.add_run()
    run.text = text
    run.font.size = Pt(font_size)
    run.font.bold = bold
    run.font.italic = italic
    run.font.color.rgb = color
    run.font.name = FONT_FACE
    return txb

def add_multiline(slide, lines, x, y, w, h,
                  font_size=14, bold=False, color=DARK,
                  align=PP_ALIGN.LEFT):
    txb = slide.shapes.add_textbox(Inches(x), Inches(y), Inches(w), Inches(h))
    tf = txb.text_frame
    tf.word_wrap = True
    for i, line in enumerate(lines):
        p = tf.paragraphs[0] if i == 0 else tf.add_paragraph()
        p.alignment = align
        run = p.add_run()
        run.text = line
        run.font.size = Pt(font_size)
        run.font.bold = bold
        run.font.color.rgb = color
        run.font.name = FONT_FACE
    return txb

def hr(slide, y, color=RED, thickness_pt=2):
    line = slide.shapes.add_shape(1, Inches(0.5), Inches(y), Inches(12.333), Inches(0.02))
    line.fill.solid()
    line.fill.fore_color.rgb = color
    line.line.fill.background()

def footer(slide, label="Adobe Confidential | AEM Edge Delivery Services"):
    add_rect(slide, 0, 7.22, 13.333, 0.28, DARK)
    add_text(slide, label, 0.3, 7.24, 10, 0.24,
             font_size=8, color=WHITE, align=PP_ALIGN.LEFT)
    add_text(slide, "© 2026 Adobe. All Rights Reserved.", 10.5, 7.24, 2.6, 0.24,
             font_size=8, color=MID_GREY, align=PP_ALIGN.RIGHT)

def internal_badge(slide):
    add_rect(slide, 10.5, 0.15, 2.5, 0.35, INTERNAL)
    add_text(slide, "INTERNAL USE ONLY", 10.52, 0.16, 2.46, 0.32,
             font_size=8, bold=True, color=WHITE, align=PP_ALIGN.CENTER)

def section_divider(slide, title, subtitle="", internal=False, customer_label=""):
    add_rect(slide, 0, 0, 13.333, 7.5, DARK)
    add_rect(slide, 0, 0, 0.18, 7.5, RED)
    add_text(slide, title, 0.55, 2.8, 10, 1.2,
             font_size=40, bold=True, color=WHITE)
    if subtitle:
        add_text(slide, subtitle, 0.55, 4.1, 10, 0.6,
                 font_size=18, color=RGBColor(0xCC, 0xCC, 0xCC), italic=True)
    add_text(slide, customer_label or "AEM Edge Delivery Services POV",
             0.55, 6.9, 11, 0.4, font_size=10,
             color=RGBColor(0x99, 0x99, 0x99))
    if internal:
        internal_badge(slide)

def content_bg(slide):
    """Apply standard content slide background (white + red top bar + footer)."""
    add_rect(slide, 0, 0, 13.333, 7.5, WHITE)
    add_rect(slide, 0, 0, 13.333, 0.12, RED)
    footer(slide)
```

### Required Slide Sequence

Build slides in this exact order. Insert a section divider before every major section:

1. **Cover** — title slide
2. **Section divider** — "Executive Summary"
3. **Content** — Executive Summary
4. **Section divider** — "Customer Snapshot"
5. **Content (two-col)** — Customer Snapshot
6. **Section divider** — "Business Context & Strategic Priorities"
7. **Content** — Business Context & Strategic Priorities
8. **Section divider** — "Current State Assessment"
9. **Content** — Current State Assessment
10. **Section divider** — "The Opportunity: AEM EDS + Document Authoring"
11. **Content** — Why EDS
12. **Section divider** — "Recommended Migration Approach"
13. **Content** — Migration Score & Effort Estimate
14. **Content** — Migration Guiding Principles
15. **Section divider** — "Benefits Summary"
16. **Content (two-col)** — Benefits Summary
17. **Section divider** — "Risks & Mitigations"
18. **Content** — Risks & Mitigations
19. **Section divider** — "Recommended Next Steps"
20. **Content** — Recommended Next Steps
21. **Section divider (internal)** — "Account Status & Financial Considerations"
22. **Content (internal)** — Account Status & Financial Considerations
23. **Section divider (internal)** — "Objection Handling"
24. **Content (internal)** — Objection Handling

Slides 21–24 must include the INTERNAL badge. Their section dividers use the internal variant.

### Save the Output

```python
import os

OUTPUT = (
    f"{SKILL_DIR}/output/{customer_slug}-pov-{date}.pptx"
)

os.makedirs(os.path.dirname(OUTPUT), exist_ok=True)
prs.save(OUTPUT)
print(f"Saved: {OUTPUT}  ({len(prs.slides)} slides)")
```

Run the script:

```bash
python3 "$SKILL_DIR/scripts/build_<customer-slug>_pptx.py"
```

Verify the output:

```bash
ls -lh "$SKILL_DIR/output/<customer-slug>-pov-<YYYY-MM-DD>.pptx"
python3 -c "from pptx import Presentation; p = Presentation('$SKILL_DIR/output/<customer-slug>-pov-<YYYY-MM-DD>.pptx'); print(f'{len(p.slides)} slides generated')"
```

---

## Step 3: Confirm

Report the saved path to the user:

> PPTX saved to `plugins/aem/edge-delivery-services/skills/generate-pptx/output/<filename>.pptx`
> Built programmatically using Adobe brand colors — no external template required.
