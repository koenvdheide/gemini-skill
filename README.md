# gemini-skill

A [Claude Code](https://claude.ai/code) skill that invokes the local [Gemini CLI](https://github.com/google-gemini/gemini-cli) as an independent analysis partner from a different model family.

## What it does

Gives Claude Code a structured way to delegate analysis tasks to Gemini — brainstorming, red-teaming, diff review, or any task that benefits from a non-Claude perspective. Useful for cross-model validation and avoiding single-model blind spots.

## Prerequisites

- [Claude Code](https://claude.ai/code)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli) installed and on PATH

## Installation

Copy or clone into your Claude Code skills directory:

```bash
git clone https://github.com/koenvdheide/gemini-skill.git ~/.claude/skills/gemini
```

## Usage

Claude invokes the skill automatically when a task matches, or you can invoke it directly:

```
/gemini red-team my API design before I start implementing
```

## License

MIT
