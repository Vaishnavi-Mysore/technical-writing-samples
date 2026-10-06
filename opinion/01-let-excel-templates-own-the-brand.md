# Stop Hard-Coding Excel Layouts: Let the Template Own the Brand

> **Type:** Opinion / experience piece  
> **Stack:** Python, openpyxl  
> **Based on:** Poc_exceltemplate + lessons from Statistical Reports

---

## The claim

If your statistical or ops team already has an official Excel look, **the template file should be the source of truth for branding**—not a 400-line Python script that sets fonts cell by cell.

Automate the **data**. Version the **design** as a real `.xlsx`.

---

## What usually goes wrong

Teams automate exports, then spend months chasing:

- Column widths that “reset” every run  
- Missing freeze panes  
- Logos that drift or vanish after save  
- A director who changed the gold header once and expected every report to follow  

The instinct is to encode every style in code:

```python
cell.font = Font(name="Arial", bold=True, color="C4A35A")
cell.fill = PatternFill("solid", fgColor="C4A35A")
# …repeat for 30 columns, 3 languages, 2 themes
```

That works until design changes. Then every deploy is a branding ticket.

---

## The better split

| Owns | Artifact |
|------|----------|
| Look & feel | `TEMPLATE.xlsx` (fonts, fills, merges, widths, freeze panes, logo slots) |
| Numbers | `INPUT.xlsx` / API payload / Qlik hypercube |
| Glue | A small applier or report engine |

In the Excel Style Applier POC, that glue is deliberately boring:

1. Extract title / header / body styles from the template  
2. Copy the template as the output base (keep merges)  
3. Write input rows with copied styles  
4. Restore widths and freeze panes  
5. Optional logo  

In Statistical Reports, the same idea scales up: **anchors** in the workbook, generators that fill them, YAML theme rules—not “draw the entire FCSC style guide in Python.”

---

## Why this is an engineering decision, not a taste preference

**1. Stakeholders review Excel, not pull requests.**  
They can open a template and say “change this.” They will not review your `PatternFill` constants.

**2. Merged cells and drawings are real.**  
openpyxl will punish you if you treat a merge like a normal cell. Starting from a template copy respects structure you would otherwise re-implement badly.

**3. Number formats are part of the brand.**  
A wrong `number_format` turns percentages into decimals. Copying style includes format codes—easy to forget when you rebuild sheets from scratch.

**4. Bilingual / RTL reports multiply complexity.**  
Hard-coding Arabic and English fonts in every generator is how themes drift. Centralize theme tokens; let the sheet hold layout.

---

## What you should still put in code

Opinion, sharpened:

- **Validation** — reject templates with no anchors / required named regions  
- **Mapping** — Qlik field → column role  
- **Rules** — zebra striping, max rows, which sheets to clone  
- **Safety** — magic-byte checks on uploads, no arbitrary macros  

Code should be the **policy**. The `.xlsx` should be the **paint**.

---

## A pattern I would not repeat

Shipping “pretty Excel” by screenshotting NPrinting (or a designer’s file) and recreating it pixel-by-pixel in openpyxl “because automation.”

You will lose. Every quarter. Forever.

Instead:

```text
designer updates TEMPLATE.xlsx
     ↓
CI runs applier against a fixture INPUT
     ↓
operators get OUTPUT that still looks official
```

---

## When hard-coded layout *is* fine

- One-off internal dumps nobody brands  
- CSV is enough  
- The “template” changes faster than the data model (rare, but it happens)  

For government / statistical bulletins with a fixed visual identity: **template-first wins**.

---

## Bottom line

Your future self does not want to redeploy an API because marketing moved a logo two cells left.

Give them a template. Write a thin applier. Keep the fights in Excel—where the reviewers already live.

---

## Related reading in this repo

- Tutorial: [Qlik-to-Excel statistical reports](../tutorials/02-qlik-to-excel-statistical-reports.md)  
- Companion how-to (implementation detail): see `Poc_exceltemplate/style_applier.py` in the projects folder  
