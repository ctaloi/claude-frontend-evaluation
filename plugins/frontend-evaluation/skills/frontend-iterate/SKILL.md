---
name: frontend-iterate
description: Automated generate-evaluate-fix loop for frontend design. Use when the user asks to iterate on, refine, or polish a frontend, or wants the full design generation loop with evaluation. Invokes frontend-design to generate and frontend-evaluator to score, repeating until quality targets are met.
---

## Overview

This skill runs the generator/evaluator loop. It generates with `frontend-design`, evaluates with `frontend-evaluator`, applies fixes, and repeats. Stops on target score, plateau, or iteration cap.

The loop is designed around an important principle: evaluation always happens in a fresh subagent with no knowledge of previous scores or iteration context. This prevents anchoring bias and ensures each score reflects only the current state of the UI.

## Configuration

Parse parameters from `$ARGUMENTS`. Support both explicit key=value syntax and natural language.

**Defaults:**
- `target=8` — stop when all four axes score >= this value
- `max=10` — hard cap on total iterations regardless of score
- `plateau=0.5` — stop when no axis improved more than this threshold across two consecutive comparisons

**Explicit syntax:**
```
/frontend-iterate target=9 max=5 plateau=1.0
```

**Natural language examples (parse these too):**
- "target of 9" → `target=9`
- "max 5 rounds" → `max=5`
- "stop if progress stalls" → `plateau=0.5` (default)
- "be strict about plateaus" → `plateau=0.25`

If `$ARGUMENTS` is empty or no parameters are found, use all defaults.

Echo the resolved configuration at the start:
```
Configuration: target=8, max=10, plateau=0.5
```

## Loop Flow

### Subagent Dispatch Mechanism

CRITICAL: The evaluator must always run as an independent subagent via the `Agent` tool. This is non-negotiable. The subagent prompt must contain ONLY:
1. The instruction to use the `frontend-evaluator` skill
2. The URL or path to evaluate

It must NOT include: score history, iteration count, previous scores, delta information, or any context from the loop. Contaminating the subagent with history will bias the scores.

**Agent tool call format:**
```
Agent tool call:
  description: "Evaluate frontend iteration N"
  prompt: "You are a frontend evaluator. Use the frontend-evaluator skill to evaluate the page at [URL]. Return the structured evaluation report with scores and priority fixes."
```

Replace `[URL]` with the actual URL (e.g., `http://localhost:3000`) or file path. Replace `N` with the current iteration number.

After the Agent tool returns, parse the four axis scores from its output:
- Design Quality (0–10)
- Originality (0–10)
- Craft & Polish (0–10)
- Functionality (0–10)

Compute overall as the mean of the four axes.

### Iteration 1

1. Invoke the `frontend-design` skill with the user's description (pass through `$ARGUMENTS` minus iteration-control params, or the full original request)
2. Wait for generation to complete
3. Ensure the dev server is running:
   - Check `package.json` for a `dev` or `start` script
   - If not running, start it and wait for it to be ready (look for "ready" / "listening" / port output)
   - Note the URL (default: `http://localhost:3000`)
4. Create the screenshot directory in the PROJECT repo root: `mkdir -p frontend-iterate-screenshots`
5. Capture screenshot → `frontend-iterate-screenshots/iteration-1.png` (see Section 5)
6. Git commit in PROJECT repo: `git add -A && git commit -m "frontend-iterate: iteration 1 checkpoint (pre-eval)"`
7. Spawn evaluator subagent via Agent tool (fresh context, no history) — see subagent dispatch above
8. Parse the four axis scores from the subagent's returned output
9. Compute overall score (mean of four axes)
10. Git commit in PROJECT repo: `git add -A && git commit -m "frontend-iterate: iteration 1 — overall X.X/10"`
    (replace X.X with the actual overall score)
11. Check exit conditions (Section 4) — if met, go to Output (Section 6)
12. Identify the weakest axis and extract the top 2–3 priority fixes from the evaluator output
13. Proceed to iteration 2

### Iterations 2+

1. Apply targeted code edits addressing the priority fixes from the previous evaluation
   - Do NOT regenerate from scratch with `frontend-design`
   - Focus edits on the weakest axis identified in the previous iteration
   - Make surgical changes; avoid refactoring unrelated code
2. Capture screenshot → `frontend-iterate-screenshots/iteration-N.png`
3. Git commit in PROJECT repo (pre-eval checkpoint): `git add -A && git commit -m "frontend-iterate: iteration N checkpoint (pre-eval)"`
4. Spawn evaluator subagent via Agent tool (fresh context, no history)
5. Parse the four axis scores from the subagent output
6. Check for regression: if ANY axis dropped 2 or more points compared to the previous iteration's scores:
   a. Revert in PROJECT repo: `git checkout HEAD~1 -- .`
   b. Discard the regression commit: `git reset --soft HEAD~1`
   c. Identify a different fix approach for the same weakest axis
   d. Repeat steps 1–5 for this retry
   e. Maximum 2 revert-and-retry attempts per iteration slot; if still regressing after 2 retries, commit the least-bad version and move on
7. Compute overall score (mean of four axes)
8. Git commit in PROJECT repo: `git add -A && git commit -m "frontend-iterate: iteration N — overall X.X/10"`
9. Check exit conditions (Section 4) — if met, go to Output (Section 6)
10. Identify the weakest axis and top 2–3 priority fixes from the latest evaluator output
11. Go to step 1

**On iteration 1, use `frontend-design`. On iterations 2+, apply targeted code edits only. Never regenerate from scratch after the first iteration.**

## Exit Conditions

Check these after each evaluation, in order:

**SUCCESS** — stop and report success when ALL FOUR axes score >= `target`. Example: if `target=8`, all of Design, Originality, Craft, and Functionality must be >= 8.

**PLATEAU** — stop when BOTH of the following are true (requires at least 3 completed iterations):
- Comparing iteration N to iteration N-1: no single axis improved by more than `plateau` points
- Comparing iteration N-1 to iteration N-2: no single axis improved by more than `plateau` points

Both comparisons must show no improvement exceeding the threshold. A single comparison is not enough; two consecutive stalled comparisons are required.

**CAP** — stop when the completed iteration count equals or exceeds `max`. This is a hard cap regardless of scores or progress.

If multiple conditions are met simultaneously, report all that apply.

## Screenshot Capture

Preferred method: use Chrome tools if available.

1. **Chrome MCP tools (preferred):** Use `mcp__claude-in-chrome__navigate` to load the URL, then `mcp__claude-in-chrome__computer` to take a screenshot. Resize to 1440px width before capturing desktop screenshots.
2. **Desktop screenshot (1440px wide):** Save to `frontend-iterate-screenshots/iteration-{N}.png`
3. **Optional mobile screenshot (390px wide):** Save to `frontend-iterate-screenshots/iteration-{N}-mobile.png`. Resize browser window to 390px, capture, then restore to 1440px.
4. **Fallback:** If no browser tools are available, skip the screenshot and note "Screenshot unavailable (no browser tools)" in the iteration log.

Always save screenshots in the PROJECT repo root under `frontend-iterate-screenshots/`, not in the plugin directory.

## Output Format

When the loop exits for any reason, print the full iteration summary table followed by the final evaluation report.

```
## Iteration Summary

| # | Design | Originality | Craft | Functionality | Overall | Delta |
|---|--------|-------------|-------|---------------|---------|-------|
| 1 | 4 | 3 | 6 | 5 | 4.5 | — |
| 2 | 6 | 5 | 7 | 6 | 6.0 | +1.5 |
| 3 | 7 | 7 | 8 | 7 | 7.3 | +1.3 |
| 4 | 8 | 8 | 8 | 8 | 8.0 | +0.7 |

**Stopped:** [Target score (8) reached | Plateau detected | Max iterations (10) reached] after N iterations.
```

Fill in actual scores. Delta is the change in overall score from the previous iteration (first row is always `—`).

After the table:
1. Print the final evaluation report returned by the last evaluator subagent run (full text, not summarized)
2. Print the screenshot directory path: `Screenshots saved to: /absolute/path/to/project/frontend-iterate-screenshots/`

If stopped by plateau, add a note explaining which two iteration comparisons triggered it and the plateau threshold used.

## Error Handling

**Dev server crash:**
- Attempt to restart using the `dev` or `start` script from `package.json`
- Wait up to 30 seconds for the server to become ready
- If still failing after a second restart attempt, stop the loop and ask the user to investigate the server issue
- Do not continue iterating without a running dev server

**Browser tool failure:**
- If screenshot capture fails, fall back to code-only evaluation for that iteration
- Note "Screenshot unavailable (browser tool error)" in the iteration log
- Continue the loop normally — screenshots are supplementary, not required

**Syntax errors in generated code:**
- If the dev server fails to start or the page throws JavaScript errors, treat this as a scoring baseline:
  - Functionality: 1/10
  - Craft: 1/10
  - Design and Originality: carry over previous scores or use 1/10 if iteration 1
- Feed the full error output as the top priority fix for the next iteration
- Do not spawn the evaluator subagent for a broken build — assign scores directly and proceed

**Unparseable evaluator output:**
- If the evaluator subagent returns output that does not contain parseable scores, retry the evaluator once (spawn a new Agent tool call with the same prompt)
- If the second attempt is also unparseable, stop the loop and surface the raw output to the user for manual inspection
- Do not guess or fabricate scores

## Rules

Hard constraints that must never be violated:

- **Never evaluate quality yourself.** Only the evaluator subagent produces scores. The iterator reads and acts on scores but never assigns them.
- **The evaluator must run as an independent subagent with no score history.** Do not pass iteration count, previous scores, delta, or any loop context to the evaluator prompt.
- **Always commit before evaluating** so there is a clean revert point if regression is detected.
- **Focus each iteration on the weakest axis**, not on all axes equally. Spreading effort across all axes produces slower improvement than concentrating on the lowest score.
- **Git commits happen in the PROJECT repo** (wherever the user is working), NOT in the plugin directory (`~/.claude/plugins/`). Never commit to the plugin repo during a loop run.
- **Iterations 2+ use targeted edits only.** Never invoke `frontend-design` after the first iteration. Regenerating from scratch discards the evaluation history and restarts the score baseline.
- **Respect the revert limit.** Maximum 2 revert-and-retry attempts per iteration slot. If the third attempt still regresses, commit the best available version and move on.
