---
name: antigravity
description: >-
  Invoke the local Antigravity CLI (agy) as an independent analysis partner from
  a different model family. Use for brainstorming, red-teaming, diff review, or
  any task needing a non-Claude perspective.
  Do NOT use for trivial tasks, simple lookups, or when no concrete artifact
  or question exists yet.
---

# Antigravity as a Thinking Partner

`agy --print` runs a single prompt non-interactively against Google's Antigravity CLI and
prints the response to stdout. Use it for an independent read on an artifact you already have.

> **Prerequisites.** `agy` on PATH, and a logged-in Antigravity account. If the binary is
> installed but not resolvable, run its installer by absolute path (on Windows,
> `%LOCALAPPDATA%\agy\bin\agy.exe install`) and restart the shell.
>
> **Shell.** These recipes use `cygpath`, heredocs, and `/tmp`, so on Windows they assume Git
> Bash. Adapt paths and quoting if you run them from PowerShell.

> **Version drift.** These tables describe `agy` 1.1.7, and print mode is undocumented
> upstream. When a table here disagrees with `agy --help`, the CLI wins. For models,
> `agy models` wins.

## When to Use

- Have a Codex answer you want to cross-check, or need reasoning from a non-Anthropic model
- Content is too large for Codex to take comfortably
- Codex is unavailable: rate-limited, auth broken, CLI failing, or erroring

## When NOT to Use

- Single-file mechanical edit (typo, rename, one-import change) with no new concepts
- Answer is already in context
- Conversation is active back-and-forth, or the user signalled urgency, so a 1-5 min wait breaks flow
- Already sent *this same question* to Antigravity this session, or you are about to fire it and Codex on the same prompt in parallel. A prior Codex pass does not block one Antigravity cross-check; that cross-check is the point.
- No specific artifact or concrete question, just a topic to "think about"
- Prompt would contain secrets, credentials, or PII
- A directory you would have to grant via `--add-dir` holds secrets or private data (see Prepare)
- Question is about Claude Code internals (hooks, skills, MCP, settings): `/claude-code-docs` knows, external CLIs do not
- Answer lives in library or tool docs, where WebFetch, Context7, or `man` is cheaper
- Missing local facts. Reproduce the issue, inspect logs, run `rg`/`git`/`blame`, or ask the user first
- Decision depends on product priority, compliance, or release timing you do not have. Ask the user, who owns it
- Codex is available, the prompt fits comfortably, and nobody asked for an independent
  cross-check. Antigravity is the fallback reviewer in that case. An explicit request for a
  second, non-Anthropic opinion outranks this bullet

## Precedence

- **WNTU wins over WTU.** If any When NOT to Use bullet matches, do not fire, even if a When
  to Use bullet also matches. If unsure, ask ("I'd skip Antigravity here because X; proceed anyway?").
- **Among WTU, pick the most specific.**
- **Among WNTU, privacy beats cost.** Privacy skips are hard (never fire). Session and cost
  skips are soft (escalate to the user). A prompt containing secrets overrides everything else.

# Prepare → Run → Validate → Recover

Follow all four steps every time. Step 3 is what stops a truncated run being reported as a finding.

## 1. Prepare

### Choose an execution profile

| Profile | Use for | Flags |
|---------|---------|-------|
| **A. Context-only** (default) | Everything the reviewer needs fits in the prompt: red-team, diff review, brainstorm, post-mortem, compare | `--mode plan` and no `--add-dir` |
| **B. Workspace-reading** | Reviewer must navigate files itself (explain, attack surface, exhausted hypotheses), or the artifact is too big for a command line | `--mode plan --add-dir <smallest dir>` |

Prefer A. Reach for B only when you genuinely cannot name the relevant files up front.

### Get the content in

**Stdin does not work.** `cat file | agy --print "..."` does not prepend the file the way
some CLIs do. In an observed run the model tried to shell out to read the content instead,
and that tool call was denied.

Two supported routes:

- **Short artifact:** inline it into the prompt via command substitution (Profile A).
- **Large artifact:** Windows caps a whole command line at 32,767 characters, so a big diff
  cannot ride in `-p`. Write it to a file, grant its smallest containing directory with
  `--add-dir`, and tell `agy` the absolute path to read (Profile B). Delete the file afterwards.

### Privacy check before granting a directory

`--add-dir` takes directories, and it exposes everything beneath the one you grant to an
external service: `.env` files, credentials, private datasets, and whatever any symlinks
under it point at. Grant the smallest directory that does the job. Treat
`read_file(<whole repo>)` as a broad grant that needs justification. If the tree holds
secrets, do not fire.

### Profile B needs a read permission, not just `--add-dir`

`--add-dir` puts a directory in the workspace. It does not by itself grant the `read_file`
tool permission, and headless mode auto-denies anything that is not allowed. Before running
Profile B: pick the smallest directory, take its absolute path, propose the narrow rule
(`read_file(<that path>)`) to the user, and wait for them to apply it. Then run, validate,
and delete any temporary file you created.

## 2. Run

```bash
# Profile A: context-only red-team, artifact inlined
agy --print "$(cat <<PROMPT
Mode: red-team
Question: Find failure modes in this approach.
Context:
$(git diff --staged)
As the very last line of your response, output exactly: <<<AGY_COMPLETE>>>
PROMPT
)" --mode plan --model gemini-3.1-pro-high --print-timeout 15m \
  > /tmp/agy-redteam-auth.out 2> /tmp/agy-redteam-auth.err

# Profile B: reviewer reads a directory itself (needs a read_file allow-rule for that path)
agy --print "Explain the module at C:\\path\\to\\src\\parser.rs. Flag anything that looks like a bug.
As the very last line of your response, output exactly: <<<AGY_COMPLETE>>>" \
  --mode plan --model gemini-3.1-pro-high \
  --add-dir "$(cygpath -w /c/path/to/src)" --print-timeout 15m \
  > /tmp/agy-explain-parser.out 2> /tmp/agy-explain-parser.err
```

### Flags this skill uses

| Flag | Purpose |
|------|---------|
| `-p` / `--print` | Run one prompt non-interactively and print the response. Also aliased `--prompt`. |
| `--mode plan` | Execution mode, and the default for every mode in this skill. The alternative, `accept-edits`, is for changing files, which this skill never does. |
| `--model <id>` | Pin the model. Always set it (see Model selection). |
| `--add-dir <abs>` | Add a directory to the workspace, repeatable. **Absolute paths only**; a relative path fails with "must be an absolute path". |
| `--print-timeout <dur>` | Wait before giving up. Defaults to `5m`, which is short for a deep review. Set `15m` for anything substantial. |
| `--log-file <path>` | Redirect the CLI log. Needed to capture a conversation ID (see Sessions). |
| `--conversation <id>` | Resume a specific conversation by ID. |
| `-c` / `--continue` | Resume the most recent conversation. Racy; see Sessions. |
| `--sandbox` | Run with terminal restrictions. Platform support varies, so treat it as defence in depth on top of the permission gate rather than a guarantee, and confirm it applies on your platform before relying on it. |

**Never pass `--dangerously-skip-permissions`.** It auto-approves every tool permission
request, removing the gate that blocks writes and shell commands. Upstream
[issue #36](https://github.com/google-antigravity/antigravity-cli/issues/36) reports it can
also authorise a sandbox bypass when combined with `--sandbox`. This flag was not probed
locally, so the prohibition is skill policy rather than a measured result. If a run is
blocked, add a narrow allow-rule instead (see Recover).

### Execution rules

- Run with `run_in_background: true` so the user is not blocked.
- **Capture stdout and stderr to separate files.** Never use `2>/dev/null`. When a tool is
  denied, stderr often carries the only notice, and discarding it turns a blocked run into a
  silent empty answer. Stderr is also sometimes empty on a blocked run, which is why the
  sentinel below is the decisive check.
- Use descriptive, unique slugs (`/tmp/agy-redteam-auth.out`). On re-launch, use a *different*
  slug; two runs sharing an output path collide.
- **Wait for completion.** Never read or delete an output file before the
  `<task-notification>` confirms the background task finished. An empty file before then means
  nothing.
- Clean up output files after reading them.
- **Passing output paths to subagents:** a subagent's tool environment does not resolve Git
  Bash `/tmp/` to its Windows location. Either inline the content into the subagent prompt
  (preferred, for output up to roughly 50KB), or pass `$(cygpath -w /tmp/agy-<slug>.out)`.

## 3. Validate

Three checks, in order. A run failing any of them is unusable, whatever the exit code says.

**Exit code 0 means nothing here.** A blocked run exits 0.

### The completion contract

Append this to every prompt you send:

> As the very last line of your response, output exactly: `<<<AGY_COMPLETE>>>`

Then check the last line of stdout. **No sentinel means the run was cut short.** Discard the
output. Do not summarise it, and do not report partial findings from it as if the review
finished. A blocked tool is one cause; a timeout, dropped auth, or network failure produces
the same shape, so read stderr to find out which.

This check exists because a truncated run is otherwise indistinguishable from a complete one.
An observed failure: `agy` was asked to modify a file, emitted three lines of stdout ending
`"I will overwrite the contents of tracked.txt..."`, wrote **empty stderr**, exited **0**, and
did nothing at all. Stdout was non-empty and stderr was clean, so only the missing sentinel
revealed the run had stopped early.

### Read stderr every run

Stderr is diagnostic when populated, and it can be empty even on a blocked run (see the
observed failure above), so treat it as a source of detail rather than the denial oracle.
When a denial is reported, the notice names the tool:

```
jetski: no output produced — a tool required the "write_file" permission that headless mode
cannot prompt for, so it was auto-denied. Add an allow-rule under permissions.allow in
settings.json ...
```

A denial invalidates any conclusion that depended on that operation, even when stdout has content.

### Narration is intent, not evidence

Print-mode stdout interleaves the agent's step narration ("I will read X", "I will overwrite
Y") with its final answer. A narrated step may never have run. Treat narration as a claim
about what the model meant to do, and quote only the final answer as a finding.

## 4. Recover

| Symptom | Cause | Fix |
|---------|-------|-----|
| Empty stdout, exit 0, stderr names a permission | Headless auto-denied a tool it could not prompt for | Add the narrowest allow-rule that unblocks it, or switch to Profile A and inline the content |
| Stdout has narration but no sentinel | Run stopped early. A blocked tool is one cause; timeout, dropped auth, or network failure look the same | Discard output. Read stderr to identify the cause, then re-run after granting access, raising the timeout, or inlining the content |
| "must be an absolute path" | A relative path reached `--add-dir` or a tool | Pass absolute paths; on Git Bash use `$(cygpath -w …)` |
| "You are not logged into Antigravity" | Auth expired or absent | Log in to Antigravity again; the CLI reads a keyring-backed OAuth token |
| Run dies at five minutes | Default `--print-timeout 5m` | Raise it (`--print-timeout 15m`) |
| Allow-rule added but still denied | Permissions merge across project settings, shared Antigravity settings, and CLI settings, with **Deny > Ask > Allow** | Inspect the *effective* policy and look for a higher-precedence Deny or Ask, rather than adding another Allow |
| Model rejected | Stale model ID | Run `agy models` and pick from the live list |

### Permissions

Config lives at `~/.gemini/antigravity-cli/settings.json`:

```json
{ "permissions": { "allow": [], "deny": [], "ask": [] } }
```

Rule forms: `read_file(*)`, `write_file(/path)`, `read_url(domain)`, `execute_url(domain)`,
`command(prefix)`, `unsandboxed(prefix)`, `mcp(server/tool)`. Precedence is **Deny > Ask >
Allow**, and anything unconfigured defaults to Ask, which headless mode auto-denies.

Observed behaviour under `--mode plan` with default permissions: attempts to overwrite a
tracked file, create an untracked file, and run a shell command were all blocked, and the
working tree was unchanged. Rely on that as an observation rather than a guarantee, and keep
prompts read-only in intent.

This skill instructs; it does not edit the user's `settings.json`. Propose a rule and let the
user apply it.

## Model selection

Run `agy models` for the live list. Pin a model explicitly on every invocation.

Default to a **Gemini** model: `gemini-3.1-pro-high` for deep analysis, a flash variant for
faster turnaround. Several Gemini IDs carry a `-high` / `-medium` / `-low` effort suffix, so
use whichever exact ID `agy models` returns. A separate `--effort` flag also exists; prefer
the suffix and do not assume the two compose.

**`agy` also serves `claude-*` models.** Selecting one gives up the cross-family read that is
the usual reason to call this skill. Warn the user before launching with a `claude-*` model,
then go ahead if that is what they want: this is a warning rather than a block, so the escape
hatch survives a Gemini outage. Label such a result as same-family in your summary.

**Name the exact model in every summary you present.** The user cannot otherwise tell whether
they got an independent review.

If one model is rate-limited, try another from `agy models` and report the switch.

## Sessions

Session resume works and is the backbone of convergence mode.

`-c` / `--continue` resumes the *most recent* conversation, so two runs going at once can pick
up each other's context. Use it only for a quick one-off. For anything multi-round, pin the ID:

```bash
LOG=/tmp/agy-review.log
agy --print "<round 1 prompt, ending with the sentinel instruction>" \
  --mode plan --model gemini-3.1-pro-high --print-timeout 15m \
  --log-file "$(cygpath -w $LOG)" > /tmp/agy-r1.out 2> /tmp/agy-r1.err

CID=$(grep -oE "Created conversation [0-9a-f-]{36}" "$LOG" | tail -1 | awk '{print $3}')
[ -n "$CID" ] || echo "no conversation ID captured; fall back to a stateless round"

agy --print "<round 2 prompt, ending with the sentinel instruction>" --conversation "$CID" \
  --mode plan --model gemini-3.1-pro-high --print-timeout 15m \
  > /tmp/agy-r2.out 2> /tmp/agy-r2.err

rm -f "$LOG"
```

Rules:

- The ID-capture step reads a log line format confirmed on `agy` 1.1.7. Check `$CID` is
  non-empty before resuming, since the format may change between versions.
- Repeat `--model`, `--mode`, `--add-dir`, and `--print-timeout` on every resume. Do not assume
  they carry over.
- One live invocation per conversation ID at a time.
- If the ID cannot be recovered, fall back to a stateless round: send the full artifact plus a
  `Previously identified findings:` block.
- Delete the log when the loop finishes.

## Base Prompt Template

Fill the relevant fields and append the mode clause. Omit empty fields.

```text
Mode: {brainstorm|red-team|diff-review|explain|attack-surface|exhausted-hypotheses}
Question: {what you want decided or critiqued}
Context:
{the smallest useful artifact}
Current belief: {your hypothesis, so it can be attacked}
Constraints: {hard facts: time, risk, compatibility, scope}

Return: verdict, top risks, missing evidence, concrete next step.
Be direct. If evidence is insufficient, say exactly what is missing.

Response style: compress prose. Drop fillers, hedges, connectives unless load-bearing. Prefer
short active sentences. Keep verbatim: code blocks, diffs, file:line citations, log entries,
numbers, names, paths, quoted context, and tables. Never compress code. If compression would
obscure a finding, write normal prose.

As the very last line of your response, output exactly: <<<AGY_COMPLETE>>>
```

Every mode below builds on this template, so the response-style and sentinel clauses carry
into all of them. Spell the sentinel out inline in each command you actually run, since a
template does not propagate itself into a shell invocation.

## Modes

**Brainstorm** — include constraints (dead ends, existing hypotheses, "do not rediscover"
lists) and the specific question. Ask for 3-5 alternatives with tradeoffs.

**Red-team** — include the plan being attacked and your constraints as hard facts. Ask for
weaknesses under two headings: **Breakage** (failure modes, edge cases, wrong assumptions;
attack assumptions and give the strongest counterargument) and **Simplifications**
(over-engineering and missed reductions; for each, what to cut, why it is safe, expected
impact). Tell it not to strip defensive code at system boundaries or WHY comments. Add: "Do
not agree just to be agreeable."

**Diff Review** — include the diff and any source it references. Ask it to verify each claim,
flag assumptions stated as facts, and check for stale line numbers.

**Explain** — Profile A with the file inlined where you know which file matters, otherwise
Profile B.

**Attack Surface** — Profile B. Include the dead-end list and known patterns as constraints.
Ask for overlooked vectors, underexplored entry points, and non-obvious vulnerability classes.

**Exhausted Hypotheses** — Profile B. Include full pipeline state (scope, dead ends, coverage,
existing hypotheses). Ask for 5-10 novel hypotheses absent from the dead-end list, each with
exact `file:line` references and an attack scenario.

## Convergence Mode (iterative review)

Some reviews converge rather than conclude. When an artifact will go through several
revisions, run a loop: review → fix → re-review, until the verdict is affirmative, the user
stops, or scope drift shows up.

### Loop shape

1. Round 1: send the full artifact and the question. Capture the conversation ID.
2. **Validate before reading anything into the result**: check the sentinel and read stderr.
   A round without its sentinel is discarded and re-run, never summarised.
3. Parse findings, summarise to the user, propose fixes.
4. **Gate 1, apply fixes.** Ask `yes-all / per-finding / skip`.
5. **Gate 2, continue or stop.** Re-state the original one-sentence brief. Ask
   `continue / stop / switch-mode`.
6. Round N: resume the conversation ID **and supply the current artifact again**.
7. Stop when the verdict is affirmative and no findings remain open, or the user stops, or
   drift appears, or the current artifact cannot be supplied.

**Supply the exact current artifact every round.** A resumed conversation carries the
discussion, and it does not hold a canonical copy of the file. Sending only a delta risks the
reviewer critiquing a version that no longer exists. Use the session for prior findings and
rationale, and let each round re-read the artifact as it now stands.

### Anti-pattern: the scope-drift spiral

**The loop is excellent at deepening a design and poor at questioning its direction.** Each
round's findings are individually valid, while the cumulative effect can pull the artifact
somewhere the user never asked for. Signs:

- The artifact grows by hundreds of lines per round.
- New rounds find issues in *fixes from prior rounds*.
- The user answers "yes-all" every time with no pushback.
- Simplification findings get absorbed as refactors ("merge X and Y") instead of acting as
  stop signals ("did we need either?").

What to do: re-state the original brief at every Gate 2 and ask whether the next round still
serves it. Weight Simplifications at least as heavily as Breakage, since the default bias runs
toward addition. If a round grows the artifact by more than half, stop and re-confirm scope.
Treat routine "yes-all" as a prompt to add friction and offer remove-this options alongside
add-machinery ones.

## Handling Output

- **Extract, do not relay.** Summarise findings, disagreements, and next steps. Quote the
  reviewer's own wording where the exact phrasing carries the finding.
- If it disagrees with your approach, present both perspectives.
- **Validate every cited file path and line number against the actual codebase.** Cited
  references can be hallucinated.
- If output is generic, retry once with a narrower question. Do not retry twice.

## Summarization Fidelity

Three rules, ordered by how often they are broken.

### 1. Quote evaluative language verbatim

Model verbs are calibrated. "I disagree" is weaker than "rejects". "Too narrow" is weaker than
"misses an entire class". When compressing, quote the verb rather than reaching for a stronger
synonym.

- **Bad:** "Antigravity rejects the approach in 7 of 7 dimensions."
- **Good:** It restructures 6 of 7 items and says *"I disagree with the belief that X is
  highest-leverage"* on the 7th.

### 2. Do not add explanatory bridges absent from the source

When it makes a bare claim without an example, do not supply one from elsewhere in your
context. Connecting two true facts is fabrication if the reviewer did not connect them.

### 3. Count inline citations in prose, not just bullets

`file:line` references often sit inside an explanatory sentence. Scan the prose when counting
call sites, or you will undercount.

### QA for high-stakes modes

After summarising `red-team`, `diff-review`, `exhausted-hypotheses`, or `attack-surface`
output, re-read your summary against the three rules above before presenting it: quote every
evaluative verb verbatim, count inline citations as well as bullets, and check each cited path
against the repository.

If a reviewer subagent is available, run it on the summary as well. It is an optional
enhancement, and the skill works without it.

**Short-output exception:** output under roughly 200 words with no bullets, numbered findings,
or `file:line` citations has too little surface area for these failure modes. Skip the pass.
