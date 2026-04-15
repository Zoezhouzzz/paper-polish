---
name: review-loop
description: >
  Orchestrate the full paper revision loop: review → revise → re-review,
  until score threshold is reached or MAX_ROUNDS exceeded.
  Supports two entry points: auto-review (run /paper-reviewer first) or
  external reviews (skip to revision using provided comments).
  Includes automatic rollback: if a revision lowers the score, revert to
  the previous version and log the failed attempt.
  Use when the user says "开始修改循环", "run review loop",
  "iterate until accepted", or "根据意见修改到通过".
argument-hint: [paper.tex — or — paper.tex + external_reviews.txt]
allowed-tools: Bash(*), Read, Grep, Glob, Write, Edit, Skill
---

# Review Loop

## Constants
- MAX_ROUNDS = 3
- SCORE_THRESHOLD = 7
- ENTRY_POINT = auto — set to `external` if reviewer comments provided

## Entry Points

**Entry A · Auto review** (default):
Start from scratch — run /paper-reviewer first, then revise.

**Entry B · External reviews**:
Skip /paper-reviewer. Read provided reviewer comments directly.
Still run /paper-reviewer after each revision round to track progress.

Detect entry point:
- If `external_reviews.txt` exists → Entry B
- Otherwise → Entry A

## Loop Structure

```
prev_score = 0

Round 1..MAX_ROUNDS:

  1. SNAPSHOT
     Before any changes, save a recoverable snapshot:
     bash: cp paper/main.tex paper/snapshots/main_round_N.tex
     bash: cp review_output.md paper/snapshots/review_round_N.md
     This ensures we can always roll back to the last known-good state.

  2. GET REVIEWS
     Entry A, Round 1:  invoke /paper-reviewer
     Entry B, Round 1:  read external_reviews.txt, parse into
                        same format as review_output.md
     Round 2+:          always invoke /paper-reviewer on latest
                        revised paper

  3. CHECK SCORE
     Read overall score from review_output.md → current_score

     If current_score >= SCORE_THRESHOLD → STOP, go to Final Report

     If round > 1 AND current_score < prev_score → ROLLBACK:
       a. Log the regression in loop_state.json:
          "rollback": {
            "round": N,
            "prev_score": prev_score,
            "regressed_score": current_score,
            "reason": "Score dropped from X to Y after revision"
          }
       b. Restore previous version:
          bash: cp paper/snapshots/main_round_{N-1}.tex paper/main.tex
       c. Keep the failed revision for reference:
          bash: mv paper/main_revised.tex paper/snapshots/main_round_N_FAILED.tex
       d. Print warning:
          ⚠️  Score dropped {prev_score} → {current_score}. Rolled back to Round {N-1} version.
       e. Decrement effective round (do NOT count a rolled-back round)
       f. If this is the 2nd consecutive rollback → STOP, go to Final Report
          with stop reason "Consecutive rollbacks — further auto-revision
          is unlikely to help. Manual intervention recommended."
       g. Otherwise → retry step 3 (REVISE) with a modified strategy:
          - Pass the failed review + diff to /writing-reviser as context
          - Instruct it to attempt a more conservative revision
          - Continue loop from step 3

     If round == MAX_ROUNDS → STOP, go to Final Report

     Set prev_score = current_score

  4. REVISE
     Invoke /writing-reviser   → paper/main_revised.tex
                                  writing_report.md
                                  figure_todo.md
                                  author_todo.md

     Invoke /figure-advisor    → figure_report.md
                                  figures/fig_N_revised.py

     Invoke /experiment-proposer → experiment_proposals.md

  5. UPDATE PAPER
     Copy revised paper for next round:
     bash: cp paper/main_revised.tex paper/main.tex

  6. SAVE ROUND STATE
     Write loop_state.json:
     {
       "round": N,
       "score": current_score,
       "prev_score": prev_score,
       "decision": "Borderline",
       "rolled_back": false,
       "reports": ["writing_report.md", "figure_report.md"]
     }
```

## Rollback Strategy

The rollback mechanism prevents "revision degradation" — where well-meaning
edits accidentally break coherence, remove important content, or introduce
new problems that outweigh the fixes.

**When rollback triggers:**
- Round N score < Round N-1 score (any drop, even 0.5)

**What happens:**
1. The paper reverts to the previous round's snapshot
2. The failed revision is saved for forensic review
3. The reviser retries with extra context: "previous attempt scored lower;
   here is what went wrong" + the diff of the failed attempt
4. If two consecutive rollbacks occur, the loop halts — this signals that
   the remaining issues likely require human judgment or new experiments

**Files created during rollback:**
- `paper/snapshots/main_round_N_FAILED.tex` — the revision that scored lower
- `paper/snapshots/review_round_N.md` — the review that triggered rollback

## Final Report

Write `LOOP_SUMMARY.md`:

```markdown
# Revision Loop Summary

## Rounds Completed: N / MAX_ROUNDS
## Stop Reason: [Score reached threshold / Max rounds exceeded / Consecutive rollbacks]

| Round | Score | Delta | Decision | Rolled Back? | Key Changes |
|-------|-------|-------|----------|--------------|-------------|
| 1 | X/10 | — | Reject | No | Fixed venue placeholder, flagged LLM identifiers |
| 2 | Y/10 | +1.5 | Borderline | No | Expanded related work, unified figures |
| 2→3 | Z/10 | -0.5 | — | ⚠️ Yes | Attempted restructure broke flow; reverted |
| 3 | W/10 | +1.0 | Weak Accept | No | Conservative fix after rollback |

## Score Trajectory
Round 1: ██████░░░░ 6.0
Round 2: ███████░░░ 7.5
Round 3: ████████░░ 8.0  ✓ threshold met

## Rollback Log
- Round 3 (attempt 1): Score dropped 7.5 → 7.0. Cause: over-aggressive
  restructuring of Section 4 removed key ablation discussion. Reverted,
  retried with conservative strategy. Second attempt scored 8.0.

## Remaining Author Actions
See `author_todo.md` — these require new experiments before resubmission.

## Files
- `paper/main_revised.tex` — latest revised paper (best scoring version)
- `paper/snapshots/` — all round snapshots and failed revisions
- `experiment_proposals.md` — proposed experiments for author
- `author_todo.md` — items requiring author action
```

Print when done:
```
=== Review Loop Complete ===
Rounds: N / 3   (M rollbacks)
Final score: X/10
Best score:  X/10  (Round N)
Decision: [Accept / Weak Accept / Borderline / Reject]

Next steps:
1. Run experiments in experiment_proposals.md
2. Fill author_todo.md items
3. Review rollback log in LOOP_SUMMARY.md if any regressions occurred
4. Resubmit
```
