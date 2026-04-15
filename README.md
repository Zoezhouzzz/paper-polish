# paper-polish

An AI-assisted academic paper revision toolkit built on Claude Code. Drop your paper in, run a slash command, get structured reviews, targeted revisions, and publication-ready figures — all without leaving the terminal.

---

## Quick Start

```bash
# 1. Put your paper in the paper/ directory
cp your_paper.tex paper/main.tex   # or paper.pdf

# 2. Open Claude Code in this directory
claude

# 3. Run the full review loop (recommended)
/review-loop
```

That's it. The loop will review, revise, and re-review until the score threshold is met or 3 rounds are exhausted.

---

## Skills

### `/paper-reviewer` — Multi-agent review
Simulates a three-reviewer + area chair panel from NeurIPS / ICML / ICLR.

- **3 parallel agents**: motivation & novelty · experiments · writing & figures
- **Area Chair**: aggregates scores, gives final decision, ranks revision priorities
- **Output**: `review_output.md` with per-dimension scores and prioritized action list

```
/paper-reviewer paper/main.tex
```

---

### `/writing-reviser` — Automated text revision
Reads `review_output.md` and applies safe revisions directly to the `.tex` source.

- Rewrites unclear passages, expands related work, fixes structure
- **Never fabricates** results or citations — calls `/research-lit` for real references
- Flags items that need new experiments as `NEEDS_AUTHOR` in-file comments
- Routes figure issues to `figure_todo.md`, experiment gaps to `author_todo.md`
- **Output**: `paper/main_revised.tex` · `writing_report.md` · `figure_todo.md` · `author_todo.md`

```
/writing-reviser
```

---

### `/figure-advisor` — Figure analysis and improvement
Reads `figure_todo.md` and improves each figure.

- Rasterizes PDF pages and visually inspects every figure
- Generates new `matplotlib` code for data-driven plots (training curves, bar charts, heatmaps)
- Routes architecture diagrams to `/paper-illustration`
- **Output**: `figure_report.md` · `figures/fig_N_revised.py`

```
/figure-advisor
```

---

### `/paper-illustration` — AI-generated architecture diagrams
Generates publication-quality diagrams using a multi-stage Gemini pipeline. Requires `GEMINI_API_KEY`.

- Claude plans the figure → Gemini optimizes layout → Gemini verifies CVPR/NeurIPS style → Gemini renders → Claude strictly reviews (target score ≥ 9/10)
- Up to 5 refinement iterations
- **Output**: `figures/ai_generated/figure_final.png` · `latex_include.tex`

```bash
export GEMINI_API_KEY="your-key"
/paper-illustration "encoder-decoder architecture with cross-attention fusion"
```

---

### `/experiment-proposer` — Missing experiment designs
Reads `author_todo.md` and proposes concrete experiment designs that are consistent with your existing methodology.

- Classifies each gap as ablation / new baseline / robustness / analysis
- Outputs a table template with blank values for the author to fill
- **Never invents expected results**
- **Output**: `experiment_proposals.md`

```
/experiment-proposer
```

---

### `/research-lit` — Literature search
Finds and synthesizes related work from arXiv, Zotero, local PDFs, Semantic Scholar, and more.

- Searches multiple sources in priority order; degrades gracefully when sources are unavailable
- Returns a structured literature table + narrative synthesis
- Used automatically by `/writing-reviser` when expanding related work

```
/research-lit "multimodal reasoning with chain-of-thought"
```

---

### `/review-loop` — Full revision loop (recommended entry point)
Orchestrates the complete review → revise → re-review cycle.

- Runs up to **3 rounds**, stops when overall score ≥ **7 / 10**
- Supports two entry points:
  - **Auto** (default): runs `/paper-reviewer` from scratch
  - **External reviews**: place reviewer comments in `external_reviews.txt` to skip the first review round
- **Rollback protection**: if a revision lowers the score, the previous version is restored automatically
- **Output**: `LOOP_SUMMARY.md` with score trajectory · `paper/snapshots/` with every round saved

```
/review-loop                        # auto entry
/review-loop external_reviews.txt   # external reviews entry
```

---

## Typical Workflow

```
/review-loop
     │
     ├── /paper-reviewer        → review_output.md
     ├── /writing-reviser       → main_revised.tex, figure_todo.md, author_todo.md
     ├── /figure-advisor        → figure_report.md, fig_N_revised.py
     └── /experiment-proposer   → experiment_proposals.md
          │
          └── repeat until score ≥ 7 or 3 rounds done → LOOP_SUMMARY.md
```

After the loop, check:
- `author_todo.md` — items that require your own experimental runs
- `experiment_proposals.md` — suggested designs for missing ablations / baselines
- `figure_report.md` — per-figure improvement notes and new plot scripts

---

## File Layout

```
paper-polish/
├── paper/
│   ├── main.tex              ← your paper goes here
│   ├── main_revised.tex      ← latest revision
│   └── snapshots/            ← per-round backups
├── review_output.md          ← latest review
├── writing_report.md         ← revision change log
├── figure_todo.md            ← figure issues for /figure-advisor
├── author_todo.md            ← items needing author action
├── experiment_proposals.md   ← proposed experiment designs
├── LOOP_SUMMARY.md           ← revision loop final report
└── figures/
    ├── ai_generated/         ← /paper-illustration output
    └── fig_N_revised.py      ← /figure-advisor plot scripts
```

---

## In Progress

- **`/paper-write`** — draft new sections (related work, intro) from an outline + literature
- **Zotero / Obsidian sync** — `/research-lit` already supports these; deeper integration coming
- **Semantic diff view** — side-by-side before/after for every writing revision
- **Multi-paper support** — run the loop across a folder of papers in batch mode
- **Score history chart** — auto-generate a matplotlib score trajectory after each loop

---

## Requirements

| Skill | Extra requirement |
|-------|------------------|
| All skills | Claude Code CLI |
| `/paper-illustration` | `GEMINI_API_KEY` (Google AI Studio) |
| `/research-lit` (Zotero) | Zotero MCP server configured |
| PDF extraction | `pymupdf` (`pip install pymupdf`) |
