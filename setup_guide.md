# Setup Guide: Obsidian Knowledge Graph Agent

This guide walks you through setting up a Claude Code agent (running on a local Ollama qwen model) to process notes from your Obsidian `Staging` folder and integrate them into your knowledge graph on a schedule.

## What you're building

A scheduled agent that, every time it runs:

1. Looks at files in `Staging/`
2. Cleans, restructures, and reformats each note for Obsidian
3. Extracts actions, ideas, abbreviations, definitions, people, processes, projects, systems
4. Routes each piece of information to the correct folder
5. Updates `INDEX.md` and any backlogs
6. Moves the processed note to `Archive/` with a `YYYYMMDD_` datestamp prefix
7. Logs anything it was unsure about to `_REVIEW.md` for you to resolve

## Prerequisites

You need these installed and working before going further:

- **Node.js 18+** — required by Claude Code
- **Claude Code CLI** — install with `npm install -g @anthropic-ai/claude-code`
- **Ollama** — running locally, with your qwen model pulled (`ollama pull qwen3:8b` or whichever variant you have)
- **A router that lets Claude Code talk to Ollama** — the standard choice is [`claude-code-router`](https://github.com/musistudio/claude-code-router) (npm: `@musistudio/claude-code-router`). LiteLLM also works.
- **Your Obsidian vault** — with the folder structure defined in `CLAUDE.md`

## Step 1 — Verify Ollama is reachable

```bash
ollama list
ollama run qwen3:8b "Reply with exactly: ok"
```

If that returns "ok", Ollama is healthy. Note the exact model name — you'll need it.

## Step 2 — Install and configure the router

```bash
npm install -g @musistudio/claude-code-router
```

Create or edit `~/.claude-code-router/config.json`:

```json
{
  "Providers": [
    {
      "name": "ollama",
      "api_base_url": "http://localhost:11434/v1/chat/completions",
      "api_key": "ollama",
      "models": ["qwen3:8b"]
    }
  ],
  "Router": {
    "default": "ollama,qwen3:8b"
  }
}
```

Replace `qwen3:8b` with whatever `ollama list` shows. The `api_key` value doesn't matter for local Ollama but the field has to be present.

Start the router:

```bash
ccr start
```

It runs as a background service.

## Step 3 — Drop the agent instructions into your vault

In the **root of your Obsidian vault** (the same folder that contains `Staging/`, `Actions/`, etc.), save the file `CLAUDE.md` provided alongside this guide.

Claude Code automatically picks up `CLAUDE.md` from the working directory it's launched in, so this is what teaches the agent what to do. You don't need a separate `AGENTS.md` — `CLAUDE.md` is the one Claude Code looks for.

Your vault root should look like:

```
<vault-root>/
├── CLAUDE.md              <- the instructions
├── INDEX.md               <- created by the agent on first run
├── _REVIEW.md             <- created by the agent when it has questions
├── Staging/               <- you drop notes here
├── Archive/               <- agent moves processed notes here
├── Actions/
│   ├── Action backlog.md
│   ├── Action items/
│   └── Action archive/
├── Definitions/
│   ├── Abbreviations.md
│   └── Definitions.md
├── Ideas/
│   ├── Ideas backlog.md
│   ├── Idea items/
│   └── Idea archive/
├── Induction/
├── People/
├── Processes/
├── Projects/
└── Systems/
```

You don't have to create the empty subfolders yourself — the agent will create what it needs on first run.

## Step 4 — Test it manually before scheduling

Open a terminal in the vault root:

```bash
cd "/path/to/your/vault"
ccr code
```

`ccr code` is the router's wrapper that launches Claude Code routed through Ollama. Once Claude Code is open, give it this single instruction:

```
Run the staging intake workflow described in CLAUDE.md.
```

Watch what it does. The first run should:

- Create `INDEX.md` (since it doesn't exist yet)
- Process whatever is currently in `Staging/`
- Create `_REVIEW.md` if anything was ambiguous

Read `_REVIEW.md` and the changes it made. If it did something you didn't want, that's the moment to refine `CLAUDE.md` — local models will need a couple of iterations to behave the way you want.

## Step 5 — Wrap it as a non-interactive script

Claude Code supports a non-interactive `-p` (print) mode that's designed for scripting. Create a script in your vault root.

**On Linux / macOS** — `run-agent.sh`:

```bash
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")"
ccr code -p "Run the staging intake workflow described in CLAUDE.md. When finished, print a one-line summary." \
  --output-format text \
  >> ".agent-log/$(date +%Y%m%d).log" 2>&1
```

```bash
chmod +x run-agent.sh
mkdir -p .agent-log
```

**On Windows** — `run-agent.ps1`:

```powershell
Set-Location -Path $PSScriptRoot
$logDir = Join-Path $PSScriptRoot ".agent-log"
New-Item -ItemType Directory -Force -Path $logDir | Out-Null
$logFile = Join-Path $logDir ("{0}.log" -f (Get-Date -Format "yyyyMMdd"))
ccr code -p "Run the staging intake workflow described in CLAUDE.md. When finished, print a one-line summary." --output-format text *>> $logFile
```

Test the script once before scheduling — drop a test note in `Staging/`, run the script, and confirm it processes correctly without your involvement.

## Step 6 — Schedule it

**On Linux / macOS** — edit your crontab with `crontab -e` and add a line. To run every weekday at 6pm:

```
0 18 * * 1-5 /full/path/to/your/vault/run-agent.sh
```

**On Windows** — use Task Scheduler:

1. Open Task Scheduler → Create Basic Task
2. Name: `Obsidian Knowledge Graph Agent`
3. Trigger: Daily, set the time you want
4. Action: Start a program
5. Program: `powershell.exe`
6. Arguments: `-NoProfile -ExecutionPolicy Bypass -File "C:\path\to\vault\run-agent.ps1"`
7. Finish, then open the task's Properties and tick "Run with highest privileges" if you hit permission issues

## Step 7 — The review loop

Because the agent runs unattended, it can't pause to ask you questions. Instead, it writes anything ambiguous to `_REVIEW.md` at the vault root. The expected rhythm is:

- **You**: drop notes into `Staging/` during the day
- **Agent**: processes them on schedule, asks questions in `_REVIEW.md`
- **You**: open `_REVIEW.md` when convenient, answer the questions inline, save
- **Agent**: on the next run, reads your answers and applies them, then clears those entries

The review file format is defined in `CLAUDE.md` so the agent knows how to read your responses.

## Practical notes about running this on a local model

Qwen3:8b is much smaller than the cloud models Claude Code is designed around, so:

- **Expect it to be slower** — a staging folder with 10 notes might take 15–30 minutes
- **Expect occasional poor judgment** — wrong folder choice, over-aggressive linking, missed actions. The `CLAUDE.md` is written to be conservative for this reason.
- **Watch the first 5–10 runs closely**. Read the logs in `.agent-log/`. Refine `CLAUDE.md` based on the patterns of mistakes you see.
- **If quality is unacceptable**, the realistic options in order of effort are: (a) try a larger qwen variant if your hardware allows, (b) try a different local model like llama3.1:70b on a beefier machine, or (c) accept that this workflow benefits from Anthropic's models and budget for API usage on this one task.

## Troubleshooting

**"Command not found: ccr"** — the router didn't install globally. Re-run with `sudo` (Linux/macOS) or as Administrator (Windows), or use `npx @musistudio/claude-code-router`.

**Agent doesn't see CLAUDE.md** — confirm you're in the vault root when launching. `pwd` (or `cd` on Windows) should show the vault folder.

**Agent processes the same file twice** — it failed to move the file to `Archive/`, probably a path or permissions issue. Check the log.

**Notes go to the wrong folder** — refine the routing rules in `CLAUDE.md`. Be specific about what belongs where; small models are literal.

**Agent makes up information** — this is the most dangerous failure mode. If you see invented content, tighten the "never do" rules in `CLAUDE.md` and add the specific example as a negative case.
