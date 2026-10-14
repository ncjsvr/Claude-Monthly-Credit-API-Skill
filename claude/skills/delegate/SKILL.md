---
name: delegate
description: Hand a self-contained coding task to a background worker billed to Nathan's API credit instead of his Claude Code plan limits. Use proactively for anything that can be fully briefed up front and checked from a diff - writing or updating tests, implementing a well-defined function or feature, mechanical refactors and renames, fixing lint or type errors, docs and READMEs, boilerplate and scaffolding, and read-only investigations of the codebase. Also runs as /delegate <task>.
argument-hint: [task]
allowed-tools: Bash(node ~/.claude/delegate/delegate.mjs *)
---

# Delegate

The worker is a separate Claude agent with none of this conversation's context. It edits files but never commits.

## Don't delegate
Tiny edits (one or two tool calls), anything needing Nathan's input or a design decision, git operations, secrets or deployment, or files you are editing right now.

## How
1. Write a complete brief: goal, relevant files, constraints and conventions, what "done" means (tests that should pass), and what not to touch.
2. Run it:

```bash
node ~/.claude/delegate/delegate.mjs --cwd "$PWD" --task-file - <<'BRIEF'
<brief>
BRIEF
```

Options: `--read-only` for investigations, `--budget <usd>` (default 3, max 20) for bigger jobs, `--model opus` only for genuinely hard tasks. For anything that may take over two minutes, run in the background and don't touch the same files until it finishes.

3. Act on the first line of output:
- `DELEGATE_OK`: review the report and the diff, fix small issues yourself, tell Nathan what was done and the cost.
- `DELEGATE_PARTIAL`: hit its cap. Review, then finish it yourself or re-delegate the rest with a bigger budget.
- `DELEGATE_UNAVAILABLE`: credit used up or not set up. Do the task yourself and stop delegating for the rest of this session.
- `DELEGATE_FAILED`: read the details; retry once with a clearer brief or do it yourself.

If invoked as /delegate with a task, turn it into a full brief and delegate it: $ARGUMENTS
