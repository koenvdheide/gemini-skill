# gemini-skill

A [Claude Code](https://claude.ai/code) plugin that invokes the local [Gemini CLI](https://github.com/google-gemini/gemini-cli) as an independent analysis partner from a different model family.

## What it does

Gives Claude Code a structured way to delegate analysis tasks to Gemini — brainstorming, red-teaming, diff review, or any task that benefits from a non-Claude perspective. Useful for cross-model validation and avoiding single-model blind spots.

## Convergence mode (iterative review)

For reviewing artifacts that evolve across multiple revisions — specs, plans, designs — Gemini can run in a convergence loop: review → fix → re-review until the reviewer gives an affirmative verdict or you stop. Claude orchestrates the loop with user gates after each round, cites prior findings on each pass so the reviewer can detect drift, and watches for the scope-drift failure mode where each round's "real" findings pull the artifact into a design the user never asked for.

See the Convergence Mode section in `skills/gemini/SKILL.md` for the loop shape (Gate 1 / Gate 2), per-round prompt construction, and the anti-pattern guidance that tells Claude when to stop and re-confirm scope instead of continuing.

## Prerequisites

- [Claude Code](https://claude.ai/code)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed and on PATH

## Optional: `reviewer` subagent

The skill runs a mandatory QA pass using a `reviewer` subagent after summarizing high-stakes modes (`red-team`, `diff-review`, `exhausted-hypotheses`, `attack-surface`). It expects the subagent from [koenvdheide/claude-reviewer](https://github.com/koenvdheide/claude-reviewer). Without it, the skill falls back to self-review against the same fidelity rules — workflow still works, just less rigorous.

## Installation

Via the `review-plugins` marketplace:

```text
/plugin marketplace add koenvdheide/review-plugins
/plugin install gemini@review-plugins
```

Or add this repo directly as a single-plugin marketplace:

```text
/plugin marketplace add koenvdheide/gemini-skill
/plugin install gemini@gemini-skill
```

## Usage

Claude invokes the skill automatically when a task matches, or you can invoke it directly:

```text
/gemini:gemini red-team my API design before I start implementing
```

## License

MIT
