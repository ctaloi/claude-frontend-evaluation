---
name: frontend-evaluator
description: Evaluate and score a frontend implementation on four axes — design quality, originality, craft, functionality. Use when the user asks to evaluate, review, score, critique, or grade a frontend page, component, or application. Can be invoked standalone or called by frontend-iterate.
---

## Overview

Score frontend output on four axes (1-10) with actionable fixes. This is an evaluator — NEVER modify code, only observe and report.

## Page Discovery

Discover the page to evaluate using this order:

1. Check for an explicit URL in `$ARGUMENTS` — if present, use it directly.
2. Probe ports 5173, 3000, 3001, 8080, 4321 via Bash:
   ```
   curl -s -o /dev/null -w "%{http_code}" http://localhost:PORT --max-time 2
   ```
   Use the first port that returns 200.
3. Find `.html` files via Glob in the current directory and `dist/`.
4. If nothing found, ask the user.

## Browser Tool Detection

Prefer `claude-in-chrome` tools:

- Try calling `mcp__claude-in-chrome__tabs_context_mcp` first.
- If it works: use chrome tools for full evaluation (navigate, screenshot, click, read).
- If it fails or is unavailable: fall back to reading source files (HTML/CSS/JS) directly.
- When using chrome: navigate to URL, resize to 1440px width, screenshot, resize to 390px, screenshot, click primary buttons, test hover states, read page text.
- When using source fallback: read all HTML, CSS, and JS files, note in report that evaluation is "code-only".

## Scoring Rubric

### Design Quality (1-10)
> Does the design feel like a coherent whole rather than a collection of parts?

- **9-10:** Distinct mood and identity. Colors, typography, layout, imagery combine into something memorable. A designer would recognize intentional creative direction.
- **7-8:** Cohesive but safe. Everything works together but doesn't create a strong identity.
- **5-6:** Mixed signals. Some elements feel considered, others feel default. No clear aesthetic point of view.
- **3-4:** Collection of parts. Components feel assembled, not designed. Inconsistent visual language.
- **1-2:** No coherence. Clashing styles, no identifiable design direction.

Evaluate: color palette unity, typography consistency, layout rhythm, visual hierarchy, mood/atmosphere, whether the design tells a story.

### Originality (1-10)
> Is there evidence of custom decisions, or is this template layouts, library defaults, and AI-generated patterns?

- **9-10:** Unmistakably custom. A human designer would recognize deliberate creative choices that couldn't come from a template.
- **7-8:** Mostly custom with some standard patterns used intentionally.
- **5-6:** Mix of custom and default. Some creative choices but relies on common patterns.
- **3-4:** Mostly defaults. Stock components with minor customization. Recognizable as template-based.
- **1-2:** Pure AI slop. Purple gradients on white cards, Inter/Roboto, shadow-md on everything, generic hero sections.

Red flags (cap score at 4 if two or more are present; cap at 6 if one is present):
- Inter, Roboto, Arial, or system-ui as primary font
- Purple/violet gradient on white/light background
- Unmodified Tailwind defaults (shadow-md, rounded-lg, text-gray-500)
- Generic hero + three-card grid layout
- Stock gradient blobs or mesh backgrounds with no customization
- Identical spacing/sizing across all elements

### Craft (1-10)
> Technical execution of visual fundamentals.

- **9-10:** Impeccable. Typography hierarchy is clear and refined. Spacing is rhythmic and consistent. Color harmony is intentional. Contrast ratios pass WCAG AA.
- **7-8:** Solid fundamentals. Minor inconsistencies that don't break the experience.
- **5-6:** Adequate. Works but lacks polish. Some spacing irregularities or hierarchy issues.
- **3-4:** Noticeable problems. Inconsistent spacing, weak hierarchy, poor contrast somewhere.
- **1-2:** Broken fundamentals. Text hard to read, chaotic spacing, no visual hierarchy.

Check: font size scale (is there a clear hierarchy?), spacing consistency (does it follow a scale?), color contrast (especially text on backgrounds), alignment grid, responsive behavior, animation smoothness.

### Functionality (1-10)
> Can users understand and use this interface?

- **9-10:** Immediately clear what the interface does. Primary actions are obvious. Task completion is intuitive with no dead ends.
- **7-8:** Usable with minor friction. Most actions are discoverable.
- **5-6:** Requires some guessing. Primary actions aren't always obvious. Some confusion about navigation or state.
- **3-4:** Confusing. Users would struggle to complete basic tasks. Important actions hidden or unclear.
- **1-2:** Non-functional or incomprehensible. Broken interactions, no clear purpose.

Test: primary action discoverability, navigation clarity, form usability, error states, loading states, empty states, responsive usability on mobile.

**Overall score:** Average of the four axes, rounded to one decimal place.

## Evaluation Process

1. Discover the page using the Page Discovery steps above.
2. Detect browser tools using the Browser Tool Detection steps above.
3. Capture the page (screenshots or source).
4. Score each axis independently using the rubric above.
5. Generate a priority fixes list (highest impact first, targeting the weakest axis).
6. Output the structured report using the format below.

## Output Format

```
## Frontend Evaluation Report

**Target:** [URL or file path]
**Method:** [chrome-interactive | code-only]

### Scores

| Axis | Score | Summary |
|------|-------|---------|
| Design Quality | X/10 | [one-line summary] |
| Originality | X/10 | [one-line summary] |
| Craft | X/10 | [one-line summary] |
| Functionality | X/10 | [one-line summary] |
| **Overall** | **X.X/10** | |

### Design Quality — X/10
[2-3 sentences of justification]

### Originality — X/10
[2-3 sentences of justification]

### Craft — X/10
[2-3 sentences of justification]

### Functionality — X/10
[2-3 sentences of justification]

### Priority Fixes
1. [Highest impact fix — what to change and why]
2. [Second highest impact fix]
3. [Third fix]
...
```

## Rules

Hard constraints:

- Never modify code. Only observe and report.
- Never reference or compare to previous evaluations. Each invocation is independent.
- Be honest and critical. If the design is mediocre, say so. Don't hedge with "good start but..."
- Score against the absolute rubric, not relative to "what an AI can do."
