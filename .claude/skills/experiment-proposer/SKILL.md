---
name: experiment-proposer
description: >
  Propose missing experiment designs based on reviewer requests in
  author_todo.md. Reads the paper to ensure proposals are consistent
  with existing methodology. Never fabricates results, never generates
  code, never invents expected outcomes.
  Use when the user says "提实验方案", "propose experiments",
  "what experiments should I add", or after /writing-reviser has
  generated author_todo.md.
argument-hint: [author_todo.md + paper/main_revised.tex]
allowed-tools: Bash(*), Read, Grep, Glob, Write
---

# Experiment Proposer

## Core Constraint
Propose designs only. Never write code, never fabricate expected
results, never invent numbers. If you cannot propose a valid design
without inventing data, say so explicitly.

## Inputs
- `author_todo.md` — missing experiments flagged by /writing-reviser
- `paper/main_revised.tex` — existing methodology and experiments

## Step 1 · Understand existing experiments

Read `paper/main_revised.tex` and extract:
- The proposed method's components
- Current baselines and datasets
- Current evaluation metrics
- What RQs are already answered

## Step 2 · Parse author_todo.md

For each missing experiment item, classify:
- **Ablation** — removing/replacing a component
- **Baseline** — adding a missing comparison method
- **Robustness** — testing under different conditions
- **Analysis** — deeper investigation of existing results

## Step 3 · Propose designs

For each item, output a structured proposal:

```markdown
### EXP-001 · [Experiment Name]
**Type**: Ablation / Baseline / Robustness / Analysis
**Addresses**: [reviewer comment from author_todo.md]
**Design**:
- Remove/modify: [specific component]
- Dataset: [which datasets, why]
- Metric: [which metrics to report]
- Baseline to compare against: [what]
**Why this design**: [1-2 sentences connecting to the paper's claims]
**Expected table format**:
| Method | Metric1 | Metric2 |
|--------|---------|---------|
| Ours (full) | - | - |
| Ours w/o [component] | - | - |
⚠️ Values to be filled by author after running experiments.
```

## Step 4 · Write experiment_proposals.md

Structure:
```markdown
# Experiment Proposals

## Summary
N experiments proposed:
- Ablations: N
- New baselines: N  
- Robustness tests: N

---
[one block per experiment as above]

---
## Implementation Notes
- All proposals are designs only — no code provided
- Run experiments and fill in table values before resubmission
- Estimated effort per experiment: [Low/Medium/High]
```

After writing, print:
```
=== Experiment Proposer Complete ===
📋 Proposals: experiment_proposals.md (N experiments)
⚠️  All values marked "-" must be filled by author
```