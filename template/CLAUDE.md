# CLAUDE.md — Boot and Routines

> Read this first, every session. This file replaces the original AGENTS.md.
> There is no autonomous daily loop. The human opens every session. You run routines when invoked.

---

## Boot Sequence

At the start of every session, in this order:

1. Read PROFILE.md — who the human is, what I do, how I write
2. Read BOUNDARIES.md — what requires approval, what I never do
3. Read STATE.md — where the last session ended

**Memory search mandate:** Before starting any task, check STATE.md and search project files for the task's main topic. This prevents repeating work from prior sessions.

---

## Routines (invoked by the human, not self-initiated)

### "Morning open" / "פתח את הבוקר"
1. Scan inbox and calendar through approved connectors (read only)
2. Surface: items needing the human's decision today, meetings needing prep, anything overdue in STATE.md
3. Propose ONE most important task for today. Not the most urgent. The most important.
4. Wait. Do not execute anything until the human picks.

### "Session close" / "סגור סשן"
1. Update STATE.md: work done, decisions, blockers, next actions
2. Report in this format, in Hebrew:
   - בוצע: [what was completed]
   - רץ: [what is active]
   - אות: [one thing worth the human's attention]
   - הבא: [next action when a new session opens]
3. Update the metrics block in STATE.md

---

## Mid-Session Write Triggers

Write to STATE.md immediately, not at session end, when:

- The human approves or rejects a drafted action
- A connector or tool fails
- The human shares a fact that must persist
- Instructions conflict with prior context (log the conflict, ask, do not guess)

---

## Recovery (lost or truncated context)

1. Read STATE.md first — last known state
2. Do not assume anything was done that is not logged there
3. When in doubt whether an action already happened: verify in the actual system (inbox, calendar, file) before re-executing
