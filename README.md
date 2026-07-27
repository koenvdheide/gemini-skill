# antigravity-skill

A [Claude Code](https://claude.ai/code) plugin that invokes the local [Antigravity CLI](https://antigravity.google/docs/cli/using) (`agy`) as an independent analysis partner from a different model family.

(The repository is still named `gemini-skill`, from when it wrapped the Gemini CLI. The plugin itself is now `antigravity`. See Migration below.)

## What it does

Gives Claude Code a structured way to delegate analysis to Antigravity: brainstorming, red-teaming, diff review, or anything that benefits from a non-Claude perspective. Useful for cross-model validation and avoiding single-model blind spots.

The skill runs `agy` headless (`--print`) in plan mode, and covers the parts of headless operation that Google's docs don't: how to get content in (stdin piping doesn't work), how to scope file access, and how to tell a finished review from one that stopped early.

## Why the completion contract matters

Headless `agy` can stop partway through a run and still look like it succeeded: exit code 0, empty stderr, and stdout containing narration that reads as though the work happened. The skill defends against this by asking for a sentinel line at the end of every response and discarding any output that lacks it. Checking the exit code alone will not catch it, and neither will checking stderr alone.

## Convergence mode (iterative review)

For artifacts that evolve across revisions (specs, plans, designs), the skill runs a convergence loop: review, fix, re-review, until the reviewer gives an affirmative verdict or you stop. Rounds resume a pinned conversation ID so the reviewer keeps its own prior findings in context, and each round re-supplies the current artifact so it never critiques a stale version.

See the Convergence Mode section in `skills/antigravity/SKILL.md` for the loop shape (two user decisions per round: which fixes to apply, then whether to continue) and the scope-drift guidance that tells Claude when to stop and re-confirm scope.

## Prerequisites

- [Claude Code](https://claude.ai/code)
- Antigravity CLI (`agy`) installed and on PATH. If the binary is installed but the `agy` command is not found, run the installer by its absolute path and restart the shell (on Windows that is `%LOCALAPPDATA%\agy\bin\agy.exe install`).
- A logged-in Antigravity account.

Developed against `agy` 1.1.7. Print mode is undocumented upstream, so flags may drift: the skill tells Claude to trust `agy --help` and `agy models` over its own tables when they disagree.

## Installation

Add this repo directly as a single-plugin marketplace:

```text
/plugin marketplace add koenvdheide/gemini-skill
/plugin install antigravity@gemini-skill
```

Refresh later with `/plugin marketplace update gemini-skill`.

If you consume this through the `agent-tools` marketplace, that catalog still lists the plugin under its old `gemini` name. It needs updating in its own repository before `antigravity@agent-tools` will resolve.

## Migration from the `gemini` plugin

The plugin name is its installation identity, so editing the manifest does not convert an installed copy. Uninstall the old one and install the new one:

```text
/plugin uninstall gemini
/plugin install antigravity@gemini-skill
```

Invocation changes from `/gemini:gemini` to `/antigravity:antigravity`. The old skill targeted the Gemini CLI (`gemini`), which this release no longer supports.

## Optional: `reviewer` subagent

The skill can run a QA pass over its own summaries of high-stakes modes (`red-team`, `diff-review`, `exhausted-hypotheses`, `attack-surface`) using a `reviewer` subagent, such as the one from [koenvdheide/claude-reviewer](https://github.com/koenvdheide/claude-reviewer). Without it the skill self-reviews against the same fidelity rules, so the workflow works either way.

## Usage

Claude invokes the skill automatically when a task matches, or you can invoke it directly:

```text
/antigravity:antigravity red-team my API design before I start implementing
```

## License

MIT
