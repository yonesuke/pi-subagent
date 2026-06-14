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

When an agent is invoked without an explicit model, pi prompts you with a model picker (listing all available models with configured auth). The selection is remembered for that agent for the rest of the session.

## Features

- **Isolated context** — Each subagent runs in a separate `pi` process
- **Three execution modes:**
  - **Single:** `{ agent: "scout", task: "find auth code" }`
  - **Parallel:** `{ tasks: [...] }` — up to 8 tasks, 4 concurrent
  - **Chain:** `{ chain: [...] }` — sequential with `{previous}` placeholder
- **Streaming output** — See tool calls and progress as they happen
- **Dynamic model picker** — Pick a model per-agent at invocation time
- **Per-agent tool restrictions** — Limit which tools each agent can use
- **Usage tracking** — Shows turns, tokens, cost per agent
- **Abort support** — Ctrl+C propagates to kill subagent processes

## Installation

```bash
# 1. Clone this repo
git clone https://github.com/yonesuke/pi-subagent.git
cd pi-subagent

# 2. Symlink the extension into pi's global extensions directory
mkdir -p ~/.pi/agent/extensions/subagent
# On macOS / Linux:
ln -sf "$(pwd)/index.ts" ~/.pi/agent/extensions/subagent/index.ts
ln -sf "$(pwd)/agents.ts" ~/.pi/agent/extensions/subagent/agents.ts

# On Windows (PowerShell as admin if needed):
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.pi\agent\extensions\subagent\index.ts" -Target "$(pwd)\index.ts"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.pi\agent\extensions\subagent\agents.ts" -Target "$(pwd)\agents.ts"

# 3. Install agent definitions
mkdir -p ~/.pi/agent/agents
cp agents/*.md ~/.pi/agent/agents/

# 4. (Optional) Install workflow prompts
mkdir -p ~/.pi/agent/prompts
cp prompts/*.md ~/.pi/agent/prompts/

# 5. Restart pi or run /reload inside pi
```

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

### Workflow prompts (if installed)
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
2. **No `model` field, but previously picked** → Reuses the remembered model (no prompt).
3. **No `model` field, first time this session** → Shows you a model picker with all available models. The currently active model is pre-selected as default.

The mapping is stored in the session and survives `/reload`. To change an agent's model mid-session, simply edit the agent `.md` file to add a `model` field temporarily, or add the `model` field permanently.

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
├── README.md
├── index.ts             # Extension entry point
├── agents.ts            # Agent discovery logic
├── agents/              # Sample agent definitions
│   ├── scout.md
│   ├── planner.md
│   ├── reviewer.md
│   └── worker.md
└── prompts/             # Workflow prompt templates
    ├── implement.md
    ├── scout-and-plan.md
    └── implement-and-review.md
```

## Acknowledgements

This project is a fork/adaptation of the [official pi subagent example](https://github.com/earendil-works/pi/tree/main/packages/coding-agent/examples/extensions/subagent). Most of the rendering logic, output formatting, and subprocess management code comes directly from that example. The main addition is the dynamic model selection system described above.

## License

MIT — see [LICENSE](./LICENSE)