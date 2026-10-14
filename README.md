# Claude Code Delegate

I built this to make use of the **$100 monthly API credit that comes with the Claude Max 5x plan** (Max 20x gets $200).

That credit can't be spent in Claude Code itself, but it does cover the Claude Agent SDK. So this setup hands self-contained coding tasks from Claude Code to a background worker running on the Agent SDK, billed to the monthly credit instead of your Claude Code plan limits. When the credit runs out, Claude Code automatically carries on using the plan as normal.

It also adds global rules for Claude Code: create a `CLAUDE.md` in every new project, delegate as much as is sensible, and never add AI attribution to commits or code.

## Requirements

- A Claude Max 5x or 20x plan, with the monthly API credit claimed into a Console org (on claude.ai: Settings → Billing → API credits)
- Windows with Claude Code installed and signed in to that Max plan
- Node.js 18 or later, and Git for Windows
- An API key from the Console org that holds the credit

## Repo layout

The `claude` folder mirrors `%USERPROFILE%\.claude`.

```
claude-delegate/
├── README.md
├── .gitignore
└── claude/
    ├── CLAUDE.md                  global rules for every project
    ├── delegate/
    │   ├── delegate.mjs           runs the worker on the Agent SDK
    │   └── package.json
    └── skills/
        └── delegate/
            └── SKILL.md           tells Claude Code when and how to delegate (/delegate)
```

## Where each file goes

| Repo file | Install location | Purpose |
| --- | --- | --- |
| `claude/CLAUDE.md` | `%USERPROFILE%\.claude\CLAUDE.md` | Global rules loaded in every Claude Code session |
| `claude/skills/delegate/SKILL.md` | `%USERPROFILE%\.claude\skills\delegate\SKILL.md` | The `delegate` skill and `/delegate` command |
| `claude/delegate/delegate.mjs` | `%USERPROFILE%\.claude\delegate\delegate.mjs` | Worker script |
| `claude/delegate/package.json` | `%USERPROFILE%\.claude\delegate\package.json` | Installs the Agent SDK |

Created during setup and never committed: `api-key.txt`, `node_modules\` and `logs\` in `%USERPROFILE%\.claude\delegate`.

## Install

Run in PowerShell.

**1. Clone the repo**

```powershell
git clone https://github.com/<your-account>/claude-delegate.git
cd claude-delegate
```

**2. Copy the files into place**

```powershell
$claude = "$env:USERPROFILE\.claude"
New-Item -ItemType Directory -Force "$claude\delegate", "$claude\skills\delegate" | Out-Null
Copy-Item .\claude\delegate\delegate.mjs, .\claude\delegate\package.json "$claude\delegate\" -Force
Copy-Item .\claude\skills\delegate\SKILL.md "$claude\skills\delegate\" -Force
if (Test-Path "$claude\CLAUDE.md") {
    Write-Host "You already have a global CLAUDE.md. Paste the contents of claude\CLAUDE.md into it."
} else {
    Copy-Item .\claude\CLAUDE.md "$claude\"
}
```

**3. Install the Agent SDK**

```powershell
cd "$env:USERPROFILE\.claude\delegate"
npm install
```

**4. Save your API key**

Create a key in the Console (API keys page) in the org that holds the credit. This prompt keeps the key out of your PowerShell history:

```powershell
$k = Read-Host "Paste your Console API key"; Set-Content api-key.txt $k.Trim() -NoNewline; Remove-Variable k
```

Never set the key as an `ANTHROPIC_API_KEY` environment variable. Claude Code would pick it up and stop using your plan.

**5. Test it**

```powershell
node delegate.mjs --check
```

`DELEGATE_OK` means it works. The test costs about $0.05 of credit. To confirm the spend comes from the credit, check that the promotional balance has dropped in the Console under Settings → Billing.

**6. Restart Claude Code** so it loads the new rules and skill.

## Using it

Claude Code delegates on its own, following the rules in `CLAUDE.md`. You can also call it directly in any project:

```
/delegate write unit tests for src/utils/dates.ts
```

The worker edits files but never commits, pushes or switches branches. Claude Code reviews the report and the diff afterwards.

**What gets delegated:** tests, well-defined features, mechanical refactors, lint and type fixes, docs, boilerplate and read-only codebase investigations.

**What stays in the main session:** tiny edits, anything needing your input or a design decision, git operations, secrets and deployment.

### Script options

| Option | Default | Use |
| --- | --- | --- |
| `--cwd <dir>` | current folder | Project to work in |
| `--task <text>` | | Brief as one argument |
| `--task-file <path>` | | Brief from a file; `-` reads stdin (heredoc) |
| `--read-only` | off | Investigate and report, no changes |
| `--model <name>` | `sonnet` | Model alias or ID; `opus` for hard tasks only |
| `--budget <usd>` | `3` | Spend cap per run (max 20) |
| `--max-turns <n>` | `80` | Turn cap per run |
| `--check` | | Tiny test call |

### Output status

The first line of output is always one of these:

| Status | Meaning | What Claude Code does |
| --- | --- | --- |
| `DELEGATE_OK` | Finished | Reviews the report and diff |
| `DELEGATE_PARTIAL` | Hit the budget or turn cap | Finishes it or re-delegates the rest |
| `DELEGATE_UNAVAILABLE` | Credit used up or key not usable | Does the work itself for the rest of the session |
| `DELEGATE_FAILED` | Something else went wrong | Retries once or does the work itself |

Full logs for every run are in `%USERPROFILE%\.claude\delegate\logs`.

## Updating

```powershell
cd <path-to>\claude-delegate
git pull
```

Re-run step 2 (it won't touch your key or an existing `CLAUDE.md`), then:

```powershell
cd "$env:USERPROFILE\.claude\delegate"
npm install
```

## Replacing the API key

```powershell
cd "$env:USERPROFILE\.claude\delegate"
$k = Read-Host "Paste your Console API key"; Set-Content api-key.txt $k.Trim() -NoNewline; Remove-Variable k
node delegate.mjs --check
```

## Uninstall

```powershell
Remove-Item "$env:USERPROFILE\.claude\delegate" -Recurse -Force
Remove-Item "$env:USERPROFILE\.claude\skills\delegate" -Recurse -Force
```

Then remove the delegation section from `%USERPROFILE%\.claude\CLAUDE.md`.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `--check` returns `DELEGATE_UNAVAILABLE` | Key is wrong, from a different org, or the credit is used up. Check the Console billing page. |
| `Native CLI binary ... not found` | Run `npm install` again in `%USERPROFILE%\.claude\delegate`. |
| Claude Code never delegates | Restart Claude Code, then check `/delegate` appears in the slash command list. |
| Claude Code asks permission every time it delegates | Choose "don't ask again" on the first prompt. |
| Claude Code itself says "Credit balance too low" | `ANTHROPIC_API_KEY` is set somewhere. Remove it from your environment variables. |

## Notes

- The monthly Max API credit covers the Agent SDK but not Claude Code itself. This setup only works because the worker runs on the Agent SDK.
- Each run carries a few cents of fixed overhead, so tiny tasks cost more to delegate than to do directly.
- The credit expires at the end of each billing cycle and doesn't roll over.
