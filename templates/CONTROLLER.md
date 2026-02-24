# CONTROLLER.md — Deterministic Sprint Controller

## Scope
This file defines the only valid control loop for sprint execution.
Execute exactly as written. Do not use prior chat context.

## Trigger Set
- `POLL_TICK` — from poll cron (every N minutes)
- `WORKER_EVENT` — from completed/failed subagent
- `MANUAL_RECONCILE` — from operator instruction

Every trigger runs `controller_tick(trigger)`.

## Required Reads Per Tick
- `POLL_TICK`: read `ACTIVITIES.md` only
- `WORKER_EVENT`: read `ACTIVITIES.md` only
- `MANUAL_RECONCILE`: read `ACTIVITIES.md` (+ full workspace if operator requests refresh)

Exception: on queue promotion, also read the promoted row's `Plan Path` to import the sprint.

## State Model

**Step states:** `TODO | DOING | DONE | BLOCKED`

**Sprint states:** `ACTIVE | BLOCKED | COMPLETE | PAUSED`

**Queue states:** `BACKLOG | ACTIVE | COMPLETE | BLOCKED`

## Invariants
- No step may be `DONE` without evidence.
- No step may remain `DOING` after its completion condition is met.
- Sprint cannot be `COMPLETE` if any step is `TODO` or `DOING`.
- If a READY step exists and worker capacity exists → dispatch in the same tick.

## Same-Cycle Dispatch Rule
After reconciliation, before persist:

```
IF sprint_state == ACTIVE:
  IF ready_steps > 0 AND worker_capacity_available:
    assign next READY step
    mark DOING
    dispatch subagent
```

Never defer dispatch to a later tick when capacity exists now.

## Worker Reconciliation

**MAX_RETRIES = 2** per step.

| Worker Event | Condition | Action |
|---|---|---|
| COMPLETED | Evidence exists | Step → DONE, worker → STOPPED |
| COMPLETED | No evidence | Do NOT mark DONE — investigate |
| FAILED/TIMED_OUT | Retries remaining | Increment retry, reassign same step |
| FAILED/TIMED_OUT | Retries exhausted | Step → BLOCKED with concrete unblock condition |

## Queue Promotion (on sprint COMPLETE)

1. Archive completed sprint: write full step table (with evidence) to the sprint's `Plan Path`
2. Update finished queue row Notes with 1-2 sentence outcome summary
3. Find first `BACKLOG` row in queue
4. Promote to `ACTIVE`, import its sprint plan into `## Sprint Steps`
5. If no BACKLOG rows: emit `CTRL_PORTFOLIO_COMPLETE` and disable automation

## Logging

All control-plane logs go to your configured channel (e.g. Telegram group).

```
CTRL_START trigger=POLL_TICK
PROJECT_TASK_STARTED task="Build auth module"
PROJECT_TASK_COMPLETE task="Build auth module"
CTRL_COMPLETE trigger=POLL_TICK state=WORKING doing=1 todo=3 blocked=0
```

Log types:
- `CTRL_START` / `CTRL_COMPLETE` — cycle boundaries
- `PROJECT_TASK_STARTED` / `COMPLETE` / `FAILED` / `BLOCKED` — step transitions
- `PROJECT_PROMOTED` — queue advancement
- `CTRL_PORTFOLIO_COMPLETE` — all work done
- `CTRL_AUTOMATION_STOPPED` — poll disabled

## Control Function (execute this sequence every tick)

1. Emit `CTRL_START trigger=<trigger>`
2. Read required files
3. If `ACTIVITIES.md` unreadable → abort tick
4. Snapshot current state
5. Collect and apply worker events (reconciliation protocol)
6. Reconcile step states and sprint state
7. Emit transition logs (diff prev vs current)
8. Execute same-cycle dispatch if capacity exists
9. If sprint COMPLETE → archive + promote next (or disable automation)
10. Persist updated `ACTIVITIES.md`
11. Emit `CTRL_COMPLETE trigger=<trigger> state=<STATE> doing=<n> todo=<n> blocked=<n>`

## Arming Protocol
To start execution:
1. Disable heartbeat (`heartbeat.every=""`) — the controller replaces it
2. Enable the poll cron job
3. Run one immediate controller cycle

## Disarming Protocol
To stop execution:
1. Disable the poll cron job
2. Leave heartbeat disabled

## Subagent Spawn Hygiene
- Always set `cleanup=delete` unless transcript retention is needed
- Route work to appropriate models (cheap models for implementation, expensive for architecture/audit)

## Determinism Rule
For identical input state + identical worker events + identical capacity → identical state transitions and logs.
