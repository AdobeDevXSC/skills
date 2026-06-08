---
name: generate-pptx
description: Generates a PowerPoint (PPTX) presentation from a customer POV document for AEM Edge Delivery Services. Downloads the official Adobe EDS POV slide template from SharePoint, introspects its layout, writes a Python script to populate slides with customer-specific content, and saves the output PPTX.
license: Apache-2.0
metadata:
  version: "1.0.0"
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

## Step 2: Fetch the Official PPTX Template

The approved slide template lives in the `AEMNAMExpertSCs` SharePoint site.

Search for the template file using the Microsoft 365 SharePoint search tool:

```
sharepoint_search query: "IQBBJ6nG6C1wQYK1Hvo-oinoAQe3nVoBFjfRK91rzoiloLY" fileType: "pptx" site: "AEMNAMExpertSCs"
```

If that returns no results, fall back to these searches (run in order, stop at first hit):

```
sharepoint_search query: "POV template" fileType: "pptx" site: "AEMNAMExpertSCs"
sharepoint_search query: "EDS POV" fileType: "pptx" site: "AEMNAMExpertSCs"
```

---

## Step 3: Download the Template

Use the following download protocol to retrieve the template file locally.

### Download Protocol

1. **Check for a pre-authenticated download URL** — look for the `downloadUrl` field (sometimes `@microsoft.graph.downloadUrl`) in the search result. **Do NOT use `webUrl`** — that is an authenticated SharePoint browser URL that cannot be fetched without credentials.

2. **If `downloadUrl` is present**, download to local storage:
   ```bash
   mkdir -p /tmp/generate-pptx
   curl -L -o "/tmp/generate-pptx/pov-template.pptx" "<downloadUrl>"
   ```

3. **If `downloadUrl` is null or absent**, apply the download recovery flow:

   a. Re-query SharePoint by the exact filename to obtain a fresh result that may include a `downloadUrl`:
      ```
      sharepoint_search query: "<exact_filename>"
      ```

   b. If the fresh result includes a non-null `downloadUrl`, download using the command above.

   c. If the fresh result still has `downloadUrl: null`, ask the user to download the template manually:

      > I found the template file but can't download it automatically — SharePoint didn't provide a pre-authenticated URL. Please open the link below in your browser, download the file, and tell me the local path where you saved it.
      >
      > **Open in browser:** `https://adobe.sharepoint.com/:p:/s/AEMNAMExpertSCs/IQBBJ6nG6C1wQYK1Hvo-oinoAQe3nVoBFjfRK91rzoiloLY?e=eVPfwJ`

      Wait for the user to respond with a local path, then copy it to `/tmp/generate-pptx/pov-template.pptx` and continue.

4. **If the template cannot be retrieved** after exhausting all recovery options, generate the PPTX programmatically using Adobe brand colors and layout conventions, and inform the user.

### Verify the Download

```bash
ls -lh /tmp/generate-pptx/pov-template.pptx
python3 -c "from pptx import Presentation; p = Presentation('/tmp/generate-pptx/pov-template.pptx'); print(f'Slides: {len(p.slides)}, Layout: {p.slide_width} x {p.slide_height}')"
```

---

## Step 4: Inspect the Template Structure

Before populating slides, introspect the downloaded template to understand its slide layouts and placeholder positions:

```python
from pptx import Presentation
from pptx.util import Inches, Pt

prs = Presentation('/tmp/generate-pptx/pov-template.pptx')

# Print all slide layouts
for i, layout in enumerate(prs.slide_layouts):
    print(f"Layout {i}: {layout.name}")
    for ph in layout.placeholders:
        print(f"  Placeholder {ph.placeholder_format.idx}: '{ph.name}' — type={ph.placeholder_format.type}")

# Print all slides and their content
for i, slide in enumerate(prs.slides):
    print(f"\n--- Slide {i+1} ---")
    for shape in slide.shapes:
        if shape.has_text_frame:
            print(f"  [{shape.name}]: {shape.text_frame.text[:120]!r}")
```

Record:
- The layout indices for: title slide, section divider, content slide, table slide, and any "internal use only" layout.
- Which placeholders (by index) map to title, subtitle, body, and footer.
- The slide dimensions (width × height in EMUs) to preserve the template's aspect ratio.

---

## Step 5: Generate the Customer PPTX

First, resolve the skill's absolute output paths:

```bash
SKILL_DIR="$(git rev-parse --show-toplevel)/plugins/aem/edge-delivery-services/skills/generate-pptx"
mkdir -p "$SKILL_DIR/output" "$SKILL_DIR/scripts"
echo "SKILL_DIR=$SKILL_DIR"
```

Write a Python script at `$SKILL_DIR/scripts/build_<customer-slug>_pptx.py` that:

1. **Opens the downloaded template** — `Presentation('/tmp/generate-pptx/pov-template.pptx')` — to inherit slide masters, fonts, color themes, and layout geometry exactly as designed.
2. **Reuses existing slide layouts** from the template rather than adding new blank slides. Use `prs.slide_layouts[<layout_index>]` for each new slide.
3. **Populates placeholders** by index (not by name) to match the template's layout, overriding only the text content.
4. **Inserts a section divider slide before every major section** — use the section divider layout identified in Step 4. Every section in the POV must be preceded by its own dedicated section slide. The required sections and their order are:
   1. Cover (title slide — no section divider needed before this)
   2. **Executive Summary** ← section divider slide, then content slide(s)
   3. **Customer Snapshot** ← section divider slide, then content slide(s)
   4. **Business Context & Strategic Priorities** ← section divider slide, then content slide(s)
   5. **Current State Assessment** ← section divider slide, then content slide(s)
   6. **The Opportunity: AEM EDS + Document Authoring** ← section divider slide, then content slide(s)
   7. **Recommended Migration Approach** ← section divider slide, then content slide(s)
   8. **Benefits Summary** ← section divider slide, then content slide(s)
   9. **Risks & Mitigations** ← section divider slide, then content slide(s)
   10. **Recommended Next Steps** ← section divider slide, then content slide(s)
   11. **Account Status & Financial Considerations** ← section divider slide (marked INTERNAL), then content slide(s)
   12. **Objection Handling** ← section divider slide (marked INTERNAL), then content slide(s)

   For each section divider slide, set the section title text to the section name listed above. If the template's section divider layout has a subtitle placeholder, populate it with a one-line summary of that section's content.
5. **Marks internal-only slides** (account financials, objection handling, and their section divider slides) with a visible "INTERNAL USE ONLY" badge using the template's designated internal layout if available, or a red-bordered text box if not.
6. **Saves the output** to `$SKILL_DIR/output/<customer-slug>-pov-<YYYY-MM-DD>.pptx` (use the absolute path resolved above).

Run the script to produce the PPTX:

```bash
python3 "$SKILL_DIR/scripts/build_<customer-slug>_pptx.py"
```

Verify the output file was created:
```bash
ls -lh "$SKILL_DIR/output/<customer-slug>-pov-<YYYY-MM-DD>.pptx"
python3 -c "from pptx import Presentation; p = Presentation('$SKILL_DIR/output/<customer-slug>-pov-<YYYY-MM-DD>.pptx'); print(f'{len(p.slides)} slides generated')"
```

---

## Step 6: Confirm and Clean Up

Confirm the saved path to the user:

> PPTX saved to `plugins/aem/edge-delivery-services/skills/generate-pptx/output/<filename>.pptx`
> Template used: `<template_filename>` (from AEMNAMExpertSCs SharePoint)

Clean up the downloaded template and any temporary files:
```bash
rm -rf /tmp/generate-pptx
```
