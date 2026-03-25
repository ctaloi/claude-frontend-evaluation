# Claude Plugins Local

Personal Claude Code plugin marketplace. Install with:

```bash
claude plugin marketplace add https://github.com/ctaloi/claude-plugins-local
claude plugin install frontend-evaluation@claude-plugins-local
```

---

# Frontend Evaluation Plugin

A Claude Code plugin that brings structured design evaluation to AI-generated frontends. Inspired by the [generator/evaluator pattern](https://www.anthropic.com/engineering/harness-design-long-running-apps) from Anthropic's research on harness design for long-running apps — where separating the creator from the critic produces dramatically better results than self-evaluation.

This plugin complements Anthropic's official `frontend-design` plugin. Where `frontend-design` generates UIs, `frontend-evaluation` scores them honestly and drives iterative improvement.

## The Problem

AI agents "confidently praise their own mediocre work." When Claude generates a frontend and then evaluates it in the same context, scores are inflated and real issues get glossed over. The fix: a dedicated evaluator with no memory of the generation process, scoring against an absolute rubric with no room for hedging.

## Two Skills

### `frontend-evaluator` — Score any frontend on four axes

Run it against any page, component, or application to get an honest assessment:

```
/frontend-evaluator                         # Auto-discovers running dev server
/frontend-evaluator http://localhost:5173   # Score a specific URL
```

It produces a structured report with four scores (1-10), justifications, and a prioritized fix list:

| Axis | What It Measures |
|------|-----------------|
| **Design Quality** | Does the design feel like a coherent whole? Colors, typography, layout, and imagery should combine into a distinct mood and identity — not a collection of parts. |
| **Originality** | Evidence of custom creative decisions vs. template layouts, library defaults, and AI-generated patterns. A human designer should recognize deliberate choices. |
| **Craft** | Technical execution: typography hierarchy, spacing consistency, color harmony, contrast ratios. This is a competence check — failing means broken fundamentals. |
| **Functionality** | Usability independent of aesthetics. Can users understand what the interface does, find primary actions, and complete tasks without guessing? |

#### Scoring Standards

The rubric is intentionally strict. A "7" means solid but safe. An "8" means genuinely good. Scores of 9-10 are reserved for work that would impress a professional designer.

**Originality has hard caps for common AI tells:**
- One red flag present (e.g., Inter font, purple gradient on white) → score capped at 6
- Two or more red flags → score capped at 4

Red flags include: Inter/Roboto/Arial as primary font, purple/violet gradients on white backgrounds, unmodified Tailwind defaults, generic hero + three-card grid layouts, stock gradient blobs, and identical spacing throughout.

#### How It Evaluates

When `claude-in-chrome` browser tools are available:
1. Navigates to the page
2. Takes screenshots at desktop (1440px) and mobile (390px) widths
3. Clicks primary action buttons, tests hover states, checks navigation
4. Reads page text for content evaluation
5. Scores each axis against the rubric

When browser tools aren't available, it falls back to reading HTML/CSS/JS source files directly and notes the evaluation is "code-only" in the report.

#### Example Output

```
## Frontend Evaluation Report

**Target:** http://localhost:5173
**Method:** chrome-interactive

### Scores

| Axis | Score | Summary |
|------|-------|---------|
| Design Quality | 6/10 | Cohesive color palette but no strong identity |
| Originality | 4/10 | Inter font + purple gradient cap the score |
| Craft | 7/10 | Solid spacing and hierarchy, minor contrast issue in footer |
| Functionality | 8/10 | Clear navigation, all actions discoverable |
| **Overall** | **6.3/10** | |

### Design Quality — 6/10
The teal-to-purple palette is consistent across components and the card
layouts have uniform spacing. However, the design doesn't commit to a
clear aesthetic direction — it reads as "modern SaaS template" rather
than something with a distinct mood or identity.

### Originality — 4/10
Inter as the primary font combined with a violet gradient hero section
triggers two red flags, capping the score at 4. The three-card feature
grid and rounded-lg shadow-md card pattern are stock Tailwind. No
evidence of custom creative decisions.

[...]

### Priority Fixes
1. Replace Inter with a distinctive font pairing (e.g., Bricolage Grotesque + Instrument Serif)
2. Redesign the hero section — replace the gradient with a bold visual concept
3. Break the three-card grid with an asymmetric layout
```

---

### `frontend-iterate` — Automated generate→evaluate→fix loop

The full generator/evaluator loop. Give it a description and it builds, scores, fixes, and repeats:

```
/frontend-iterate                           # Build + iterate with defaults
/frontend-iterate target=9 max=5            # Custom thresholds
/frontend-iterate target=9 max=8 plateau=1.0
```

#### How It Works

```
┌─────────────────────────────────────────────────────┐
│ Iteration 1                                         │
│                                                     │
│  1. Invoke frontend-design to generate initial UI   │
│  2. Start dev server                                │
│  3. Screenshot the page                             │
│  4. Spawn evaluator (fresh subagent, no history)    │
│  5. Parse scores                                    │
│  6. Git commit checkpoint                           │
│  7. Check exit conditions                           │
│  8. Extract weakest axis + priority fixes           │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│ Iterations 2+                                       │
│                                                     │
│  1. Apply targeted code edits (NOT regenerate)      │
│  2. Screenshot the page                             │
│  3. Spawn evaluator (fresh subagent, no history)    │
│  4. Parse scores                                    │
│  5. If regression (2+ point drop): revert & retry   │
│  6. Git commit checkpoint                           │
│  7. Check exit conditions                           │
│  8. Extract weakest axis + priority fixes           │
│                                                     │
│  Loop until exit condition met                      │
└─────────────────────────────────────────────────────┘
```

#### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `target` | 8 | Stop when all four axes hit this score |
| `max` | 10 | Hard cap on iterations |
| `plateau` | 0.5 | Stop if no axis improves more than this across two consecutive iteration pairs |

#### Exit Conditions

The loop stops when any of these are met:

- **Target reached** — all four axes score >= `target`
- **Plateau detected** — comparing N to N-1 AND N-1 to N-2, no axis improved more than `plateau` in either comparison (requires 3+ iterations)
- **Cap hit** — iteration count reaches `max`

#### Key Behaviors

**Fresh evaluator context.** Every evaluation runs as an independent subagent via the Agent tool. The evaluator never sees previous scores, iteration counts, or loop context. This prevents anchoring bias — each score reflects only the current state of the UI.

**Focus on the weakest axis.** Each iteration targets the lowest-scoring dimension rather than spreading fixes across all axes. Concentrated effort produces faster improvement.

**Git checkpointing.** Every iteration creates a git commit in the project repo. If a fix causes a 2+ point regression on any axis, the iterator reverts to the previous commit and tries a different approach (max 2 retries per iteration).

**Screenshot history.** Before each evaluation, a screenshot is saved to `frontend-iterate-screenshots/iteration-{N}.png`. After the loop completes, you have a visual history of the design's progression.

**Targeted edits, not regeneration.** Only iteration 1 uses `frontend-design` to generate from scratch. Iterations 2+ apply surgical code edits based on the evaluator's priority fixes. This preserves what's working and focuses effort on what needs improvement.

#### Example Output

```
## Iteration Summary

| # | Design | Originality | Craft | Functionality | Overall | Delta |
|---|--------|-------------|-------|---------------|---------|-------|
| 1 | 4      | 3           | 6     | 5             | 4.5     | —     |
| 2 | 6      | 5           | 7     | 6             | 6.0     | +1.5  |
| 3 | 7      | 7           | 8     | 7             | 7.3     | +1.3  |
| 4 | 8      | 8           | 8     | 8             | 8.0     | +0.7  |

**Stopped:** Target score (8) reached after 4 iterations.

Screenshots saved to: /Users/you/project/frontend-iterate-screenshots/
```

#### Error Handling

| Situation | Behavior |
|-----------|----------|
| Dev server crashes | Restart twice, then stop and ask user |
| Browser tools fail | Fall back to code-only evaluation for that iteration |
| Generated code has syntax errors | Score Functionality and Craft at 1/10, feed errors as top priority fix |
| Evaluator returns unparseable output | Retry once, then stop and surface raw output |

## Installation

**From GitHub (recommended for multiple machines):**

```bash
claude plugin marketplace add https://github.com/ctaloi/claude-plugins-local
claude plugin install frontend-evaluation@claude-plugins-local
```

**Prerequisites:**
- [Claude Code](https://claude.com/claude-code) CLI
- Anthropic's `frontend-design` plugin (for the iterator's generation step)
- `claude-in-chrome` MCP tools (optional, for browser-based evaluation — falls back to source code analysis)

## How It Relates to frontend-design

This plugin does NOT replace Anthropic's `frontend-design`. They work together:

| Plugin | Role | Modifies Code? |
|--------|------|---------------|
| `frontend-design` (Anthropic) | Generator — creates UIs | Yes |
| `frontend-evaluator` (this plugin) | Evaluator — scores UIs | No, read-only |
| `frontend-iterate` (this plugin) | Orchestrator — runs the loop | Yes (applies fixes) |

You can use `frontend-evaluator` standalone to score any page — it doesn't require `frontend-design`. The `frontend-iterate` skill invokes `frontend-design` for the initial generation, then drives improvement via the evaluator.

## Background

Based on [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps) by Prithvi Rajasekaran at Anthropic. The key insight: separating generation from evaluation — inspired by GAN architecture — produces dramatically better results than self-evaluation. The research found 5-15 iterations typically show improvement before plateauing.

The four evaluation axes (Design Quality, Originality, Craft, Functionality) are adapted from the rubric used in that research to guide both the generator and evaluator agents.
