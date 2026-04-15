---
name: paper-reviewer
description: >
  Review an academic CS/AI paper from four dimensions: motivation and
  novelty, logical completeness, experiment quality, and writing+figures.
  Three parallel reviewer agents each score and comment independently.
  Use when the user says "review my paper", "帮我review论文",
  "评审这篇文章", "给论文打分", or uploads a pdf/latex/zip file
  and asks for review or feedback.
argument-hint: [paper.pdf or paper.tex or paper.zip]
allowed-tools: Bash(*), Read, Grep, Glob, Write, Agent, Skill
---
# Paper Reviewer

## Agents

**Agent 1 · Motivation Reviewer**
Evaluates: innovation of motivation, logical completeness
(whether the paper delivers what it claims).

**Agent 2 · Experiment Reviewer**
Evaluates: completeness of experiments, whether results
support the claimed contributions.

**Agent 3 · Writing Reviewer**
Evaluates: writing quality, figure and table design.

**Agent 4 · Area Chair (AC)**
Aggregates the three reviews, gives final decision and
prioritized revision directions.

## Workflow

### Step 1 · Read the paper


**1. Find the file**
If the user provided a path via $ARGUMENTS, use that directly.
Otherwise search the `paper/` folder:
```bash
find paper/ -name "*.pdf" -o -name "*.tex" | head -5
```
If multiple files found:
- Prefer `main.tex` or .tex matching the folder name
- If only PDF exists, use that

**2. Extract text**
Run: `python3 scripts/extract_paper.py {found_file_path}`
This produces `paper_text.txt` and `figures/` folder.

**3. Read result**
Read `paper_text.txt` for the full text.
Pass this text to the three reviewer agents in Step 2.


### Step 2 · Run three reviewers in parallel

Launch the following three Agents using parallel tool calls.
Do not wait for one to finish before starting the next.

**Agent 1 prompt:**
"You are a motivation reviewer for a top CS/AI venue (NeurIPS/ICML/ICLR).
Read the following paper and evaluate:
1. Novelty of motivation (1-10): Is the problem new and important?
2. Logical completeness (1-10): Does the paper deliver what it claims?
For each dimension provide: score, 2-3 sentence justification, specific weaknesses.
Insert the full extracted paper text here directly.
Do not use a placeholder — paste the actual text into the prompt."

**Agent 2 prompt:**
"You are an experiment reviewer for a top CS/AI venue.
Read the following paper and evaluate:
1. Experiment completeness (1-10): Are baselines, ablations sufficient?
2. Support for claims (1-10): Do results actually prove the contributions?
For each dimension provide: score, 2-3 sentence justification, specific weaknesses.
Insert the full extracted paper text here directly.
Do not use a placeholder — paste the actual text into the prompt."

**Agent 3 prompt:**
"You are a writing and presentation reviewer for a top CS/AI venue.
Read the following paper and evaluate:
1. Writing quality (1-10): Clarity, structure, grammar.
2. Figures and tables (1-10): Are they clear, informative, well-designed?
For each dimension provide: score, 2-3 sentence justification, specific weaknesses.
Insert the full extracted paper text here directly.
Do not use a placeholder — paste the actual text into the prompt."


### Step 3 · AC aggregates reviews

Wait for all three agents to complete, then launch Agent 4:

**Agent 4 prompt:**
"You are an Area Chair at a top CS/AI venue (NeurIPS/ICML/ICLR).
You have received three independent reviews of the same paper.

[REVIEW 1 - Motivation & Logic]
{agent1_output}

[REVIEW 2 - Experiments & Evidence]
{agent2_output}

[REVIEW 3 - Writing & Figures]
{agent3_output}

Your job:
1. Summarize each reviewer's key points and scores in 2-3 sentences each.
2. Give an overall score (1-10) that weighs all three reviews.
3. Give a final decision: Accept / Weak Accept / Borderline / Reject.
4. List revision priorities in order of importance — most critical issues first.
   Be specific: reference the section or figure that needs fixing, not just
   the general area.

Output in this exact format:
## AC Summary
**Reviewer 1 (Motivation)**: [summary] — Score: X/10
**Reviewer 2 (Experiments)**: [summary] — Score: X/10
**Reviewer 3 (Writing)**: [summary] — Score: X/10
**Overall score**: X/10
**Decision**: [Accept / Weak Accept / Borderline / Reject]
**Revision priorities**:
1. [specific issue + location]
2. [specific issue + location]
3. [specific issue + location]"


### Step 4 · Write all output to `review_output.md` without asking for confirmation.

Use this structure:

# Paper Review

## Agent 1 · Motivation Review
**Novelty of motivation**: X/10
[justification]
**Logical completeness**: X/10
[justification]

## Agent 2 · Experiment Review
**Experiment completeness**: X/10
[justification]
**Support for claims**: X/10
[justification]

## Agent 3 · Writing Review
**Writing quality**: X/10
[justification]
**Figures and tables**: X/10
[justification]

## AC Summary
**Overall score**: X/10
**Decision**: Accept / Weak Accept / Borderline / Reject
**Revision priorities**:
1. [most critical issue]
2. [second issue]
3. [third issue]
---