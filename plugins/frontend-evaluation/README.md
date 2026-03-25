# Frontend Evaluation Plugin

Complements Anthropic's `frontend-design` plugin with structured evaluation and automated iteration.

## Skills

### `/frontend-evaluate`
Score a frontend implementation on four axes (1-10 each):
- **Design Quality** — coherence, mood, identity
- **Originality** — custom decisions vs template defaults and AI slop
- **Craft** — typography, spacing, color harmony, contrast
- **Functionality** — usability, discoverability, task completion

Uses chrome browser tools when available, falls back to source code analysis.

### `/frontend-iterate`
Automated generate→evaluate→fix loop:
1. Generate with `frontend-design`
2. Evaluate with `frontend-evaluator`
3. Fix weakest axis
4. Repeat until target score (default 8/10), plateau, or max iterations (default 10)

Captures screenshots per iteration and commits checkpoints to git.

## Usage

```
/frontend-evaluate                          # Evaluate current page (auto-discovers dev server)
/frontend-evaluate http://localhost:5173    # Evaluate specific URL

/frontend-iterate                           # Full loop with defaults
/frontend-iterate target=9 max=5            # Custom thresholds
```

## References

- [Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) — Anthropic
