# TriFold Agent Setup Assistant

Private repository for TriFold's client-facing personal-productivity-agent deployments, and the Claude Cowork Skill that runs the setup.

## What this is

TriFold deploys a personal productivity agent to executive clients, built from a four-file template package (`CLAUDE.md`, `PROFILE.md`, `BOUNDARIES.md`, `STATE.md`) plus a seven-question intake (`INTAKE.md`). The **Agent Setup Assistant** is a Claude Cowork Skill that runs the entire deployment as a structured joint session with the client: precondition check, guided Hebrew intake, file generation with validation, a live smoke test, training, and a setup report — then keeps the client's agent under version control for reconfiguration and restore after go-live.

Full requirements: [`PRD-Agent-Setup-Assistant-EN.md`](PRD-Agent-Setup-Assistant-EN.md) ([Hebrew version](PRD-Agent-Setup-Assistant-HE.md)). The PRD is the source of truth for behavior; this README is an orientation layer on top of it.

## Repository layout

```
template/                        master template (never edited per-client)
  CLAUDE.md                      fixed policy — boot sequence, routines, approval line
  PROFILE.md                     mission, voice, hard floor — filled per client
  BOUNDARIES.md                  approved connectors, data classification, hard stops
  STATE.md                       live working memory — ships as a skeleton, filled by the deployed agent
  INTAKE.md                      the seven setup questions

skill/agent-setup-assistant/
  SKILL.md                       the Skill itself — every flow, validation, and self-test lives here

clients/<client-slug>/           one directory per client, created on first setup
  CLAUDE.md, PROFILE.md,
  BOUNDARIES.md, STATE.md        filled instance for that client
  setup-report.md                session summary, TriFold + client only
  .snapshots/                    local-only backup, never committed — the basis for restore

registry.md                      client ↔ template-version table
```

Git is the version-control backbone: `template/` is tagged `template-vX.Y`, and every validated per-client change is committed and tagged `known-good-<slug>-<short-hash>`. `STATE.md` is committed once, as a skeleton, and never again — its real operational content only ever lives in the client's local copy and `.snapshots/`, per the PRD's confidentiality rules.

## How to use the Skill

Everything runs inside Claude Cowork, in a project scoped to the client. Invoke by typing one of the trigger phrases below — see [`Setup-Guide-HE.md`](Setup-Guide-HE.md) for a full walkthrough in Hebrew, including what to have ready before you start and what to expect at each step.

**Full setup session** (the main entry point):
```
start setup for client [name]
```
Runs precondition check → intake → file generation → smoke test → training/close in one continuous session, ending with a single push-to-origin prompt.

**Standalone flow entry points** (resume a session at a specific phase, e.g. after an interruption or a scheduling gap):
- `check preconditions for [name]` / `precondition check for [name]`
- `send precondition email for [name]` — generate the IT approval email ahead of the session
- `run intake for [name]`
- `generate files for [name]`
- `run smoke test for [name]`
- `train and close for [name]`

**Post-go-live operations** (outside the 90-minute session, no timer):
- `reconfigure [name]` / `update [name]'s agent` — edit `PROFILE.md`/`BOUNDARIES.md` through the same validation and approval flow, logged to the client's `STATE.md`
- `restore [name] to known-good` — diff against the last local snapshot and restore on confirmation; never touches git
- `bump template version to [X.Y]` / `check for stale clients` — tag a new master template version and list clients that need it (never applies automatically)

If a session is interrupted, just invoke the same start command again — the Skill reads `clients/<slug>/.session-progress.md` and resumes from the last confirmed checkpoint instead of restarting.

## Status

All functional requirements in PRD §6 (FR-1 through FR-14) are implemented in `SKILL.md`. Pilot use (Itzik on himself, then two real clients, per PRD §11) has not yet run — `clients/` and `registry.md` are currently empty scaffolding.
