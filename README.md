# Deterministic Controller — Starter Kit for OpenClaw

A sprint-based orchestration system for OpenClaw agents. Instead of free-form chat, your agent executes structured sprint plans with evidence-gated steps, automatic project promotion, and subagent workers.

**This is how you stop burning tokens on aimless agent loops.**

---

## What's in the Box

| File | Purpose |
|------|---------|
| `templates/CONTROLLER.md` | The control loop spec — your agent's operating system |
| `templates/ACTIVITIES.md` | Sprint tracker + project queue |
| `templates/SPRINT_TEMPLATE.md` | Reusable sprint plan structure |
| `templates/MEMORY.md` | Minimal long-term memory template |
| `docs/token-efficiency.md` | How tokens actually work and how to not waste them |
| `docs/poll-cron-setup.md` | Setting up the autonomous poll loop |

---

## Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) installed and running
- An LLM provider configured (see [Connecting Claude](#connecting-claude-opus-46) below)

---

## Install OpenClaw

```bash
# Install via npm (Node 18+ required)
npm install -g openclaw

# Start the gateway
openclaw gateway start

# Verify it's running
openclaw status
```

The gateway runs as a background daemon. It manages your agent sessions, cron jobs, and message routing.

---

## Connecting Claude Opus 4.6

You can use your Claude Pro/Max subscription directly — no API key needed. This uses the Claude Code CLI setup-token method.

### Step 1 — Install Claude Code CLI

Install on the **same machine** running OpenClaw (if you SSH in, do this over SSH):

```bash
npm install -g @anthropic-ai/claude-cli
```

### Step 2 — Authenticate

```bash
claude login
```

This opens a browser flow. Log in with your Pro/Max account.

### Step 3 — Generate a setup token

```bash
claude setup-token
```

This prints a long token like `claude_st_xxxxxxxxxxxxxxxxx`. Copy it.

### Step 4 — Feed it to OpenClaw

```bash
openclaw models auth setup-token --provider anthropic
```

It will prompt you to paste the token. Paste it, press Enter.

### Step 5 — Set your default model

```bash
openclaw models set anthropic/claude-opus-4-6
```

### Step 6 — Verify

```bash
openclaw models status
```

You should see `anthropic/claude-opus-4-6` as your default with auth confirmed.

### Caveats

- The setup-token can expire periodically — regenerate with `claude setup-token` if you hit 401 errors
- This isn't OAuth — it's a subscription token workaround. Anthropic doesn't have a formal OAuth flow for Pro/Max yet
- Install Claude CLI on the OpenClaw machine, not your laptop (unless OpenClaw also runs on your laptop)

---

## Quick Start

### 1. Copy templates to your workspace

```bash
WORKSPACE=$(openclaw status --json | jq -r '.workspace')
cp templates/CONTROLLER.md "$WORKSPACE/CONTROLLER.md"
cp templates/ACTIVITIES.md "$WORKSPACE/ACTIVITIES.md"
cp templates/SPRINT_TEMPLATE.md "$WORKSPACE/SPRINT_TEMPLATE.md"
cp templates/MEMORY.md "$WORKSPACE/MEMORY.md"
```

### 2. Create your first sprint plan

Copy `SPRINT_TEMPLATE.md` to `projects/<your-project>/sprint-plan.md` and fill it in. Every step needs:
- A concrete action
- A completion condition
- An evidence path (where to find proof it's done)

### 3. Add it to the queue

Edit `ACTIVITIES.md` and add a row to the Project Queue table:

```markdown
| 1 | My First Project | BACKLOG | `projects/my-project/sprint-plan.md` | - |
```

### 4. Set up the poll cron (optional — for autonomous execution)

See `docs/poll-cron-setup.md` for full instructions. This creates a cron that sends `POLL_TICK` to your agent every N minutes, driving the control loop.

### 5. Arm and run

Tell your agent: *"Read CONTROLLER.md and arm yourself"*

Or manually:
1. Enable the poll cron
2. The agent reads ACTIVITIES.md, promotes the first BACKLOG project to ACTIVE, imports the sprint plan, and dispatches Step 1

---

## Token Efficiency — The Most Important Thing

Read `docs/token-efficiency.md` for the full guide. Here's the summary:

### How context works

Every message you send to an LLM includes the **entire conversation history** as context. Message 1 sends just your prompt. Message 50 sends all 50 messages. This means:

- **Fewer messages with more instructions = cheaper** than many small messages
- When you hit the context window limit, the conversation gets **compacted** — the LLM loses detailed memory of earlier work
- A new session or gateway restart = fresh context = agent has no memory

### How the agent recovers context

When context is lost (compaction, new session, restart), the agent reads its workspace files:
- `MEMORY.md` — curated long-term memory (project index + key facts)
- `CONTROLLER.md` — how to operate
- `ACTIVITIES.md` — what's currently running

**These files must be as brief and efficient as possible.** Every byte is tokens. Every token is money.

### Why we don't use heartbeats

OpenClaw has a heartbeat feature — it pings your agent periodically to check emails, calendars, etc. Every heartbeat is a full context load. If your heartbeat runs every 30 minutes and your workspace context is 15K tokens, that's 15K tokens every 30 minutes just to say "nothing to do."

**The controller pattern replaces heartbeats.** Instead of constant polling with full context, the controller:
1. Only runs when there's a sprint active
2. Uses a lightweight poll cron that sends a single trigger message
3. The agent reads only `ACTIVITIES.md` per tick (not the whole workspace)
4. Disarms automatically when all work is done

### Zero-context-proof instructions

Every sprint plan, every step, every instruction must be executable by an agent with **zero prior context**. No "as discussed earlier." No "use the same approach." Spell out every file path, every command, every completion condition. Because after compaction or restart, "earlier" doesn't exist.

### The MEMORY.md pattern

Keep `MEMORY.md` as a minimal index that points to per-project READMEs:

```markdown
## Project Directory
### My Project
- **Path:** `projects/my-project/`
- **Key docs:** `docs/spec.md`, `docs/api-reference.md`
- **State:** Sprint 2 in progress. Auth system built. API integration next.
```

Each project's `README.md` contains the full context needed to work on it. The agent loads MEMORY.md first (cheap), then loads only the specific project README when needed (targeted). This keeps the base context small while still having full project context available on demand.

---

## How Sprints Work

1. **Write a sprint plan** — use `SPRINT_TEMPLATE.md`. Define steps with concrete actions and evidence paths.
2. **Queue it** — add a BACKLOG row to `ACTIVITIES.md`
3. **Arm** — enable the poll cron. The controller promotes BACKLOG → ACTIVE and imports the sprint.
4. **Execute** — each poll tick, the controller checks subagent status, reconciles completed/failed steps, and dispatches the next step.
5. **Complete** — when all steps are DONE, the controller archives the sprint, promotes the next BACKLOG project (if any), or disarms if the queue is empty.

Workers are subagents spawned for each step. The main agent (your expensive model) only orchestrates — it reads status, makes decisions, and dispatches. The actual work gets delegated to cheaper models where possible.

---

## File Structure After Setup

```
workspace/
├── CONTROLLER.md          # Control loop spec
├── ACTIVITIES.md           # Sprint tracker + queue
├── SPRINT_TEMPLATE.md      # Reusable template
├── MEMORY.md               # Long-term memory index
├── memory/
│   └── YYYY-MM-DD.md       # Daily notes (raw logs)
└── projects/
    └── my-project/
        ├── README.md        # Project context
        └── sprint-plan.md   # Current sprint
```

---

## License

MIT
