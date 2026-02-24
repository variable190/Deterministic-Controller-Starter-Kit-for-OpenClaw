# Token Efficiency Guide

## How LLM Context Works

Every message you send to an LLM includes the **entire conversation history**. Not just your latest message — everything.

```
Message 1:  sends [message 1]                    → ~500 tokens
Message 2:  sends [message 1 + message 2]        → ~1,000 tokens
Message 10: sends [all 10 messages]              → ~5,000 tokens
Message 50: sends [all 50 messages]              → ~25,000 tokens
```

This means **every message gets more expensive than the last**.

### Practical implications

1. **Batch instructions.** One message with 5 tasks = cheaper than 5 messages with 1 task each. The 5-message version resends the full context 5 times.

2. **Context window limit.** Every model has a maximum context size (e.g. 200K tokens for Opus). When you hit it, the conversation gets **compacted** — the LLM summarises earlier messages and loses detailed memory. After compaction, your agent may forget instructions, file paths, decisions, or context from earlier in the conversation.

3. **New session = blank slate.** A gateway restart, new session, or `/new` command means the agent starts with zero memory. It only knows what's in its workspace files.

## Why Heartbeats Burn Tokens

OpenClaw's heartbeat feature pings your agent periodically. Every ping loads:
- Your workspace context files (AGENTS.md, IDENTITY.md, etc.)
- The heartbeat prompt
- Recent conversation history

If your workspace context is 15K tokens and heartbeat runs every 30 minutes:
- **720 tokens/hour** just for "nothing to do" responses
- **~15K tokens/ping** for context loading
- Over 24 hours: **~720K tokens** doing nothing

### The controller alternative

The deterministic controller replaces heartbeats:
- Only runs when there's active work
- Poll cron sends a single lightweight trigger
- Agent reads only `ACTIVITIES.md` per tick (not the full workspace)
- Disarms automatically when work is done
- Zero token burn when idle

## Zero-Context-Proof Instructions

After compaction or restart, your agent remembers nothing from "earlier." So every instruction must be **self-contained**:

❌ Bad: *"Use the same approach as last time"*
❌ Bad: *"The file we discussed"*
❌ Bad: *"Continue where we left off"*

✅ Good: *"Read `projects/auth/spec.md` and implement the JWT validation function in `src/auth.py`"*
✅ Good: *"Run `python test.py` — done when exit code is 0 and output contains 'All tests passed'"*

This applies to sprint plans, cron payloads, MEMORY.md entries — everything your agent might read cold.

## The MEMORY.md Pattern

Your agent reads `MEMORY.md` on every session start. Keep it as a **minimal index** pointing to per-project READMEs:

```markdown
## Project Directory

### Auth Service
- **Path:** `projects/auth/`
- **Key docs:** `README.md`, `docs/api-spec.md`
- **State:** Sprint 2 done. JWT + refresh tokens working. Rate limiting next.
```

When the agent needs to work on Auth Service, it loads `projects/auth/README.md` for full context. This keeps the base context small (~2-3K tokens for MEMORY.md) while having deep project context available on demand.

**Rule of thumb:** If it's not needed every session, it doesn't belong in MEMORY.md.

## Workspace File Sizing

Keep these files lean:

| File | Target Size | Contains |
|------|-------------|----------|
| MEMORY.md | <3K tokens | Project index + key rules |
| CONTROLLER.md | <4K tokens | Control loop spec |
| ACTIVITIES.md | <2K tokens | Current sprint only |
| Project README.md | <2K tokens each | Full project context |

Total base context on cold start: ~10-15K tokens. That leaves 185K+ for actual work.

## Model Routing for Cost

Use expensive models (Opus) for orchestration and decision-making. Use cheaper models (Codex, GPT) for implementation work via subagents:

| Role | Model | Why |
|------|-------|-----|
| Manager/orchestrator | Opus | Needs judgment, reads specs, makes decisions |
| Code implementation | Codex/GPT | Follows clear instructions, writes code |
| Research/data | Cheaper model | Web search, summarisation |

The controller dispatches subagent workers with `model=<cheaper-model>` — the main agent only orchestrates.
