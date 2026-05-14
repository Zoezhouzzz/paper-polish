# Paper Auto-Review

An automated paper-review-and-revision pipeline built as a set of Claude Code skills. Drop in a `.tex` / `.pdf` / `.zip` and get back reviewer scores, a revised paper, and a clean change log — without leaving the editor.

[中文版本](./README_CN.md)

---

## What it does

Given an academic CS/AI paper, it:

1. Runs three independent reviewer agents in parallel (Motivation, Experiments, Writing) plus an Area Chair to aggregate scores.
2. Revises the paper based on those reviews — text, figures, and missing experiments are each handled by a dedicated skill.
3. Loops review → revise → re-review until the overall score crosses a threshold or rounds run out, with automatic rollback if a revision lowers the score.
4. Emits **three** clean files at the end. All intermediate artifacts are tucked into a hidden `.auto-review/` folder.

## Pipeline at a glance

```
                    paper.tex / .pdf
                           │
                           ▼
            ┌─────────────────────────────┐
            │       /review-loop          │ ◄─────────────┐
            │       orchestrator          │               │
            └──────────────┬──────────────┘               │
                           │                              │
   ① REVIEW phase ─────────┴────── (parallel agents)      │
                                                          │
   ┌────────────┐    ┌────────────┐    ┌────────────┐     │
   │ Motivation │    │ Experiment │    │  Writing   │     │
   │  Reviewer  │    │  Reviewer  │    │  Reviewer  │     │
   └─────┬──────┘    └─────┬──────┘    └─────┬──────┘     │
         └─────────────────┼─────────────────┘            │
                           ▼                              │
                   ┌───────────────┐                      │
                   │  Area Chair   │  ← aggregate scores  │
                   │  (sequential) │                      │
                   └───────┬───────┘                      │
                           │                              │
   ② REVISE phase ─────────┴────── (parallel skills)      │
                                                          │
   ┌────────────┐    ┌────────────┐    ┌──────────────┐   │
   │  writing-  │    │  figure-   │    │ experiment-  │   │
   │  reviser   │    │  advisor   │    │  proposer    │   │
   └─────┬──────┘    └─────┬──────┘    └──────┬───────┘   │
    text edits        redraw figs       expt designs      │
         └─────────────────┼─────────────────┘            │
                           ▼                              │
              ┌────────────────────────────┐              │
              │ score ≥ 7  or  rounds = 3? │── no ────────┘
              │ (rollback if score drops)  │   next round
              └─────────────┬──────────────┘
                            │ yes
                            ▼
              ┌────────────────────────────┐
              │  REVIEW.md                 │
              │  CHANGES.md                │
              │  paper/main_revised.tex    │
              └────────────────────────────┘
```

Both phases are **fan-out parallel**: the three reviewers run concurrently, the three revision skills run concurrently. The Area Chair fires only after all three reviewers return (it needs to aggregate). `/review-loop` wraps the whole thing into a multi-round loop with rollback.

## Skills

| Skill | Role | Invocation |
|---|---|---|
| `paper-reviewer` | 3 reviewers + AC, produces scores and revision priorities | `/paper-reviewer paper.pdf` |
| `writing-reviser` | Text revision; flags items needing author action | (called by `review-loop`) |
| `figure-advisor` | Improves figures, regenerates matplotlib plots, routes architecture diagrams | (called by `review-loop`) |
| `experiment-proposer` | Designs missing ablations / baselines / robustness tests | (called by `review-loop`) |
| `review-loop` | Orchestrates everything, manages rollback, emits final summary | `/review-loop paper.tex` |
| `paper-illustration` | AI-generated architecture / pipeline diagrams | (called by `figure-advisor`) |
| `research-lit` | Fetches real citations for related-work expansion | (called by `writing-reviser`) |

## Inputs

Put your paper in the project root (or any path you'll pass as an argument):

- **`.pdf`** — submitted PDF
- **`.tex`** — LaTeX source (preferred — only `.tex` can be auto-revised)
- **`.zip`** — full LaTeX bundle (figures, bib, style files)

You also need:

- `paper/` folder containing the source file(s), or
- a direct path passed via the slash command.

If only the PDF is available, the reviewer will still score and comment, but `review-loop` cannot apply text edits (no `.tex` to edit).

## How to use

### 1. Just get reviewer feedback (read-only)

```
/paper-reviewer paper/main.pdf
```

Produces `review_output.md` containing per-agent scores, AC decision, and a prioritized revision list.

### 2. Full review + revision loop

```
/review-loop paper/main.tex
```

This runs review → revise → re-review until either:
- overall score ≥ `SCORE_THRESHOLD` (default **7**), or
- `MAX_ROUNDS` reached (default **3**), or
- two consecutive rollbacks occur.

### 3. Use external reviewer comments

Drop your reviewer comments into `external_reviews.txt` at the project root, then:

```
/review-loop paper/main.tex
```

The loop will skip its own reviewer for the first round and use your comments instead, but will still run `paper-reviewer` after each revision to track progress.

## Outputs

After `review-loop` finishes, your working directory contains exactly three files:

| File | What's in it |
|---|---|
| `REVIEW.md` | Overall score, decision, per-reviewer highlights, score trajectory across rounds, remaining revision priorities, items still needing author action, proposed experiments |
| `CHANGES.md` | Every text edit (before → after with section/line) and every figure change the agent made |
| `paper/main_revised.tex` | The revised paper, latest accepted version |

Everything else lands under `.auto-review/` for forensics:

```
.auto-review/
├── review_output.md           # last round's raw reviewer output
├── writing_report.md          # text-change log (merged into CHANGES.md)
├── figure_report.md           # figure-change log (merged into CHANGES.md)
├── figure_todo.md             # pipe: writing-reviser → figure-advisor
├── author_todo.md             # items the loop couldn't auto-fix
├── experiment_proposals.md    # full experiment designs (summarised in REVIEW.md)
├── loop_state.json            # last round's score / decision / rollback state
└── snapshots/                 # per-round main.tex + review.md + _FAILED.tex
```

## How reviews are produced

`paper-reviewer` launches four agents:

1. **Motivation Reviewer** — novelty (1–10) + logical completeness (1–10)
2. **Experiment Reviewer** — completeness (1–10) + support for claims (1–10)
3. **Writing Reviewer** — writing quality (1–10) + figures/tables (1–10)
4. **Area Chair** — aggregates the three, emits overall score + Accept / Weak Accept / Borderline / Reject + revision priorities

Reviewers 1–3 run in parallel, AC runs after they all return. Each reviewer is forced to give a numeric score, a 2–3 sentence justification, and specific weaknesses with locations (section / figure / table).

## How revisions are produced

Inside each loop iteration, the orchestrator runs three revision skills:

- **writing-reviser** classifies each revision priority as **SAFE** (auto-fix), **MARK** (flag for author with a `% [NEEDS_AUTHOR]` LaTeX comment), or **SKIP** (route to figure-advisor / experiment-proposer). SAFE items are edited directly in `main.tex` with `% [REV-NNN]` tags.
- **figure-advisor** reads `figure_todo.md`, rasterizes the PDF, regenerates matplotlib plots for data-driven figures (using **only** numbers already in the paper), and routes architecture diagrams to `paper-illustration`.
- **experiment-proposer** reads `author_todo.md` and writes structured proposals (type, dataset, metric, baseline, expected table format) — designs only, never invented results.

## Rollback

If round N scores lower than round N-1:

1. The failed revision is renamed `.auto-review/snapshots/main_round_N_FAILED.tex`.
2. `paper/main.tex` is restored from the previous snapshot.
3. The reviser retries with the failed diff + low review as extra context, instructed to be more conservative.
4. Two consecutive rollbacks halt the loop — manual intervention is needed.

This prevents "revision degradation" where well-meaning edits break coherence faster than they fix issues.

## Anti-hallucination constraints

These are hard-coded into the skills' system prompts:

- **No invented experiment results.** If a fix requires new numbers, the item is marked `NEEDS_AUTHOR` and routed to `author_todo.md`.
- **No hallucinated citations.** New references must come from `research-lit` (real-paper search) or be left as `\cite{PLACEHOLDER_REV_NNN}` for author verification.
- **No fabricated plot data.** `figure-advisor` may only re-render with numbers already present in the paper.
- **No code generation in `experiment-proposer`.** It outputs designs, not implementations.

## Configuration

Constants live at the top of `.claude/skills/review-loop/SKILL.md`:

```
MAX_ROUNDS = 3
SCORE_THRESHOLD = 7
```

Edit them there. There's no separate config file.

## Project layout

```
paper-auto-review/
├── .claude/skills/             # the 7 skills
│   ├── paper-reviewer/
│   ├── writing-reviser/
│   ├── figure-advisor/
│   ├── experiment-proposer/
│   ├── review-loop/
│   ├── paper-illustration/
│   └── research-lit/
├── paper/                      # put your .tex / .pdf here
│   └── main.tex
└── (after running review-loop)
    ├── REVIEW.md
    ├── CHANGES.md
    ├── paper/main_revised.tex
    └── .auto-review/
```

## Requirements

- Claude Code CLI
- `python3` with `PyMuPDF` (`fitz`) for PDF rasterization and text extraction
- LaTeX toolchain if you want to compile the revised `.tex`
