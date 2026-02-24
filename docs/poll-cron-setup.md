# Poll Cron Setup

The poll cron drives the controller's autonomous execution loop. It sends a `POLL_TICK` trigger to your main agent session every N minutes.

## How It Works

1. OpenClaw cron fires every N minutes
2. It spawns an isolated agent turn (not in your main session)
3. That agent sends `POLL_TICK` to your main session via `sessions_send`
4. Your main agent receives it, reads `ACTIVITIES.md`, reconciles workers, dispatches next step

## Create the Cron

### Option 1: Ask your agent

Tell your agent:

> Create a cron job called `subagent-poll` that runs every 10 minutes. It should be an isolated agentTurn that sends a POLL_TICK trigger to the main session via sessions_send. Create it disabled — I'll enable it when ready.

### Option 2: Manual via CLI

```bash
openclaw cron create \
  --name "subagent-poll" \
  --schedule "*/10 * * * *" \
  --kind agentTurn \
  --disabled \
  --payload '{
    "text": "TRIGGER=POLL_TICK\nExecute CONTROLLER.md exactly.\n\nMandatory every tick:\n1. Run subagents list to check active/recent subagents\n2. If any completed but not reconciled — reconcile NOW\n3. Dispatch next work if capacity exists\n4. Report state to control channel"
  }'
```

The cron payload should instruct the relay agent to send the trigger text to your main session. The exact mechanism depends on your OpenClaw version — the agent will figure out the routing.

## Cron Configuration

| Setting | Recommended | Why |
|---------|-------------|-----|
| Frequency | Every 10 minutes | Balances responsiveness vs token cost |
| Default state | Disabled | Only enable when you have active work |
| Session target | Main agent session | The controller runs in your main session |

## Arm / Disarm

**Arm** (start working):
1. Enable the poll cron: `openclaw cron enable <job-id>`
2. The next tick will pick up the ACTIVE sprint and start dispatching

**Disarm** (stop working):
1. Disable the poll cron: `openclaw cron disable <job-id>`
2. No more ticks = no more token burn

The controller disarms automatically when the project queue is empty (all COMPLETE or BLOCKED).

## Token Cost

Each poll tick costs roughly:
- Context load: ~10-15K tokens (ACTIVITIES.md + system prompt)
- If idle (no work): ~500 tokens response
- If dispatching: ~2-3K tokens (spawn subagent + update tracker)

At 10-minute intervals with active work: ~100-200K tokens/hour for orchestration. When idle/disarmed: zero.
