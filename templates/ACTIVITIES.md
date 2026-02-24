# ACTIVITIES.md — Current Sprint

## PRIME DIRECTIVE
Every instruction in this sprint must be executable by a cold-start agent with **zero prior context**. No step may rely on chat memory, implied intent, or ambiguous references. Spell out every file path, command, and completion condition.

---

## Project Queue (portfolio)

The **only** authoritative queue. The controller does not scan folders.

Queue states: `BACKLOG | ACTIVE | COMPLETE | BLOCKED`

Promotion rule (runs when current sprint becomes `COMPLETE`):
1. Find first `BACKLOG` row (top-to-bottom)
2. Promote to `ACTIVE`
3. Import its sprint plan into Sprint Steps below
4. If no BACKLOG rows → portfolio complete, disable automation

| Order | Project | Queue State | Plan Path | Notes |
|---:|---|---|---|---|
| 1 | Example Project | BACKLOG | `projects/example/sprint-plan.md` | Your first project — replace this row |

---

## Sprint Steps (manager-led execution)

### ACTIVE Sprint
Sprint state: **PAUSED**

No active sprint. Add a BACKLOG project above, then arm the controller.

---

## Worker Assignment Table

| Worker Session | Assigned Step # | State | Started | Retry Count | Next Action |
|---|---:|---|---|---:|---|
| (none) | - | STOPPED | - | - | Waiting for sprint activation |

---

## BLOCKED

| Item | Blocker | Unblock Condition | Owner |
|------|---------|-------------------|-------|
| (none) | - | - | - |

---

## Rules

1. One ACTIVE sprint at a time.
2. Manager may run up to 2 concurrent subagents (configurable).
3. Every dispatched task has an explicit completion condition.
4. On worker completion/failure: immediately assign next READY step.
5. Sprint ends when all steps are DONE, or all remaining are BLOCKED.
6. Disable automation only after confirming no READY work remains.
