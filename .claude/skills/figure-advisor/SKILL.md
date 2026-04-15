---
name: figure-advisor
description: >
  Analyze and improve figures in an academic paper based on reviewer
  comments in figure_todo.md. Gives improvement suggestions and generates
  new matplotlib code for data-driven plots. Invokes /paper-illustration
  for architecture diagrams.
  Use when figure_todo.md exists, or when the user says "improve figures",
  "修改图表", "analyze figures", or after /writing-reviser completes.
argument-hint: [figure_todo.md + paper.pdf]
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit, Skill
---

# Figure Advisor

## Core Constraint
Never fabricate data. Generated plots must use existing numbers
from the paper only. If new experimental data is needed, add to
`author_todo.md` and skip.

## Inputs
- `figure_todo.md` — figure issues from /writing-reviser
- `paper/main_revised.tex` — locate figure positions
- `paper/*.pdf` — rasterize and visually inspect figures

## Step 1 · Read inputs

Read `figure_todo.md` for the list of figure issues.
Rasterize PDF pages containing figures:
```bash
python3 -c "
import fitz
doc = fitz.open('paper/main_revised.pdf')
for i, page in enumerate(doc):
    pix = page.get_pixmap(dpi=150)
    pix.save(f'figures/page_{i:03d}.png')
"
```

## Step 2 · Analyze each figure

For each figure in figure_todo.md, examine the rasterized image and check:

- Axis labels present and readable
- Legend present if multiple series
- Color-blind safe palette (avoid red+green only)
- Line width sufficient (≥1.5pt)
- Caption explains the takeaway, not just "Results"
- No JPEG artifacts on line plots (use PDF/vector format)

## Step 3 · For each figure, choose action

**Data-driven plots** (training curves, bar charts, tables, heatmaps):
Generate new matplotlib code and save to `figures/fig_N_revised.py`:
```python
# Example structure
import matplotlib.pyplot as plt
import numpy as np

# Use ONLY numbers already in the paper
# Never invent data points

plt.rcParams['font.size'] = 11
# ... plot code using existing paper data ...
plt.savefig('figures/fig_N_revised.pdf', bbox_inches='tight')
```
Run the script: `python3 figures/fig_N_revised.py`
Verify output exists: `ls figures/fig_N_revised.pdf`

**Architecture/pipeline diagrams**:
Invoke /paper-illustration for this figure:
Use Skill: /paper-illustration "[figure description from figure_todo.md]"

**Minor fixes** (axis labels, captions, colors):
Write specific LaTeX/matplotlib instructions in `figure_report.md`.



## Step 4 · Write figure_report.md

```markdown
# Figure Revision Report

## Figure 1 · [caption excerpt]
**Issues**: color-only distinction, no axis label on Y
**Action taken**: generated figures/fig_1_revised.py
**Run**: `python3 figures/fig_1_revised.py`
**Suggestion**: unified tSNE panel combining Figures 1 and 3

## Figure 2 · [caption excerpt]
**Issues**: routed to /paper-illustration
**Action taken**: invoked /paper-illustration for architecture diagram

## Author Todo
- Fig 3 requires new ablation data → added to author_todo.md
```

## Step 5 · Output files

- `figure_report.md` — per-figure analysis and actions
- `figures/fig_N_revised.py` — new plot scripts (data-driven only)
- `author_todo.md` — updated with any figure items needing new data

Print when done:
```
=== Figure Advisor Complete ===
✅ Generated: N plot scripts → figures/fig_N_revised.py
🎨 Routed to /paper-illustration: N diagrams
⚠️  Author action needed: author_todo.md (N items)
📋 Report: figure_report.md
```