# pi-subagent

A [pi](https://github.com/earendil-works/pi) extension that lets you delegate tasks to isolated subagents — each running in a separate `pi` process with its own context window.

> **Heavily inspired by the [official pi subagent example](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/examples/extensions/subagent/).**  
> Big thanks to the pi team for the original design.

## What's different from the official example?

The key improvement: **dynamic model selection**.

| Original example | This fork |
|---|---|
| Model is **hardcoded** in each agent's frontmatter (`model: claude-sonnet-4-5`) | Model is **optional** — no model field means "ask me" |
| You need to edit `.md` files to change models | You pick models through pi's interactive picker, just like `/model` |
| Can't reuse the same agent definition with different models | You can assign different models per-agent, remembered for the session |

When an agent is invoked without an explicit model, pi prompts you with a **scrollable model picker** (SelectList with 10 visible items, overlay). The picker shows models in this order:

1. **★ Last used** for this agent (persisted across sessions)
2. **Current** model in the main conversation
3. All other available models

You can **type to filter** with substring matching — typing `sonnet` matches `openrouter/anthropic/claude-sonnet-4-5`. Arrow keys navigate, Enter selects, Esc cancels.

The picker appears **once per user prompt** (turn). Subsequent calls to the same agent within the same turn reuse the model without prompting. The selection is remembered for the next turn as the "last used" default.

**Nested subagents** (e.g., chain mode) automatically inherit the parent's model — no picker shown in subprocesses.

## Features

- **Isolated context** — Each subagent runs in a separate `pi` process
- **Three execution modes:**
  - **Single:** `{ agent: "scout", task: "find auth code" }`
  - **Parallel:** `{ tasks: [...] }` — up to 8 tasks, 4 concurrent
  - **Chain:** `{ chain: [...] }` — sequential with `{previous}` placeholder
- **Streaming output** — See tool calls and progress as they happen
- **Dynamic model picker** — Scrollable overlay with type-to-filter, per-turn reset, nested inheritance
- **Per-agent tool restrictions** — Limit which tools each agent can use
- **Usage tracking** — Shows turns, tokens, cost per agent
- **Abort support** — Ctrl+C propagates to kill subagent processes

## Installation

```bash
# 1. Install the extension
pi install git:github.com/yonesuke/pi-subagent

# 2. Install agent definitions (manual — copy to agents dir)
mkdir -p ~/.pi/agent/agents
cp ~/.pi/agent/git/github.com/yonesuke/pi-subagent/agents/*.md ~/.pi/agent/agents/

# 3. /reload in pi, done!
```

To update: `pi update --extensions`  (pulls latest commit).
To uninstall: `pi remove git:github.com/yonesuke/pi-subagent`.

## Usage

### Single agent
```
Use scout to find all authentication code in this project
```
Or more explicitly: `subagent(agent: "scout", task: "find auth code")`

### Parallel execution
```
Run scouts in parallel: one to find model definitions, one to find provider configs
```

### Chained workflow
```
Chain: scout finds auth code → planner creates implementation plan → worker implements the plan
```

### Work### Workflow prompts (if installed)
```
/implement add Redis caching to the session store
/scout-and-plan refactor auth to support OAuth
/implement-and-review add input validation to API endpoints
```

## Agent definitions

Agents are markdown files with YAML frontmatter. Place them in `~/.pi/agent/agents/` (user-level) or `.pi/agents/` (project-level).

```markdown
---
name: my-agent
description: What this agent does
tools: read, grep, find, ls    # optional — restricts available tools
model: claude-sonnet-4-5       # optional — omit to pick at runtime
---

System prompt for the agent goes here.
```

### Pre-built agents

| Agent | Purpose | Tools |
|-------|---------|-------|
| `scout` | Fast codebase recon for handoff | read, grep, find, ls, bash |
| `planner` | Creates implementation plans | read, grep, find, ls |
| `reviewer` | Code quality & security review | read, grep, find, ls, bash |
| `worker` | General-purpose (full capabilities) | (all default) |

None of them have a hardcoded model — you'll be prompted to pick one the first time each is used.

### How model selection works

1. **Agent has `model` field** → Uses that model directly (no prompt).
2. **Nested subagent** (subagent spawned by another subagent) → Inherits the parent's model automatically.
3. **Already picked this turn** → Reuses the model (no prompt). Pickers reset each user prompt.
4. **First time this turn, no UI** → Falls back to the current model.
5. **First time this turn, interactive** → Shows the picker overlay with:
   - **★ Last used** model for this agent at the top
   - **Current** model second
   - All others below

**Persistence:** The "last used" mapping lives in `~/.pi/agent/subagent-models.json` (survives restarts). You can `cat` or edit it directly. To force a model for an agent, add `model: provider/id` to its `.md` file.

**Filter:** Type any substring — it matches against both the model ID and display name. `sonnet` catches all Claude Sonnet variants regardless of provider prefix. Backspace clears the filter.

## Tool modes

| Mode | Parameter | Description |
|------|-----------|-------------|
| Single | `{ agent, task }` | One agent, one task |
| Parallel | `{ tasks: [...] }` | Multiple agents run concurrently (max 8, 4 concurrent) |
| Chain | `{ chain: [...] }` | Sequential with `{previous}` placeholder |

## Security

- **User-level agents** (`~/.pi/agent/agents/`) are always loaded.
- **Project-level agents** (`.pi/agents/`) require `agentScope: "both"` (or `"project"`) and show a confirmation prompt in interactive mode.
- Subagents inherit the tools you specify in the agent definition — restrict tight for untrusted codebases.

## Structure

```
pi-subagent/
├── package.json                # pi package manifest
├── extensions/
│   └── subagent/
│       ├── index.ts            # Extension entry point
│       └── agents.ts           # Agent discovery logic
├── agents/                     # Agent definitions (copy to ~/.pi/agent/agents/)
│   ├── scout.md
│   ├── planner.md
│   ├── reviewer.md
│   └── worker.md
├── prompts/                    # Workflow prompts (auto-loaded)
│   ├── implement.md
│   ├── scout-and-plan.md
│   └── implement-and-review.md
└── README.md

Runtime files (created automatically):
~/.pi/agent/subagent-models.json  # Per-agent model history
```

## Acknowledgements

This project is a fork/adaptation of the [official pi subagent example](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions/subagent). Most of the rendering logic, output formatting, and subprocess management code comes directly from that example. The main addition is the dynamic model selection system described above.

## License

MIT — see [LICENSE](./LICENSE)