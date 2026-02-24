# MEMORY.md — Long-Term Memory

This is your agent's curated memory. Keep it **minimal** — every byte is tokens loaded on every session start.

## Operating Rules
- Keep replies short by default
- Execute specs exactly — if you can't, say so
- All instruction sets must be zero-context-proof (executable cold)

## Project Directory

Each entry is a minimal index. Full context lives in the project's README.md.

### Example Project
- **Path:** `projects/example/`
- **Key docs:** `README.md`, `docs/spec.md`
- **State:** Not started. Sprint plan queued.

<!-- Add more projects as you build them:

### Another Project
- **Path:** `projects/another/`
- **Key docs:** `README.md`
- **State:** Sprint 1 complete. Waiting for next phase.

-->
