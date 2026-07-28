# PRD — Agent Setup Assistant

**Version:** 0.2-en | **Date:** 2026-07-28 | **Owner:** Itzik Woda, TriFold Technologies
**Status:** Decisions resolved. Ready for v0 build.
**Audience:** Claude Code. This is the working document for implementation. Read it fully before writing any code or skill files.

---

## 1. Context and Problem

TriFold deploys a personal productivity agent for executive clients. The agent is defined by a four-file template package (CLAUDE.md, PROFILE.md, BOUNDARIES.md, STATE.md) plus a seven-question intake questionnaire (INTAKE.md). All five exist and are the source of truth for what gets generated. Do not redesign them; this tool fills and manages them.

Today the deployment is manual: Itzik runs the intake, fills the bracket placeholders, uploads to the client's Cowork project. This breaks in three places: it does not scale past a few clients a month, validation depends on the implementer's memory (missed connector approvals surface only after go-live), and there is no record of which template version each client received, which makes template-wide updates impossible.

The Agent Setup Assistant runs inside Claude Cowork and manages the setup as a structured joint session: Itzik and the client together.

## 2. Organizing Constraint: The 90-Minute Session

One product constraint drives every design decision: full setup in a single joint session. Setup itself within 30 minutes, the whole session within 90 including tests and training. When evaluating any feature, ask: does this fit the clock?

| Phase | Target | Output |
|-------|--------|--------|
| Open + precondition check | 10 min | Go / No-Go |
| Guided intake | 20 min | 7 confirmed answers |
| File generation + validation | 5 min | 4 files in client project |
| Smoke test | 15 min | Inbox triage + one approved draft |
| Training | 25 min | Client runs both routines alone |
| Close + documentation | 10 min | Setup report, version committed |

## 3. Goals and Non-Goals

**Goals**

1. Full working-agent setup in one joint session: setup ≤ 30 min, session ≤ 90 min.
2. Zero validation misses: handoff is impossible without confirmed connector approvals and data classification.
3. Version record per client: which master template, which customizations, when.
4. The client leaves the session able to run both routines without Itzik.

**Non-Goals — explicitly out of scope. Do not build these.**

- Client self-serve. v0 is consultant-led only. Self-serve is a v1 decision.
- Microsoft Copilot support. Environment is Claude Cowork / Claude Code only.
- Any change to the approval line. "Every action with external or internal effect requires human approval" is template policy, not a questionnaire option.
- Autonomous actions by the deployed agent. This tool sets up; it does not operate.
- Payments or licensing.

## 4. Personas

**Itzik, the implementer.** Leads the session. Needs the tool to own the clock and the validations so he can focus on the client, not a checklist. Success: leaves with a working agent and a committed record, no follow-up work.

**The executive, the client.** VP/C-level in a 500–5,000 employee Israeli organization. Not technical, busy. Answers the intake in Hebrew, never touches the English files. Success: sees a first draft in their own voice inside the session.

**IT / security, the gatekeeper.** Not in the room but sets its boundaries: approved connectors and permitted data classification. The tool treats their approvals as a precondition, never as bureaucracy to route around.

## 5. Core Flows

### 5.1 Precondition Check (Go / No-Go)

Before the intake, verify: a Cowork project exists for the client, the required connectors are attached and working (read-only probe), and the client can state the permitted data classification. If IT approvals are missing, the session stops here: the tool generates a ready-to-send Hebrew email to IT listing the required connectors and the rationale, and the session is rescheduled. Burning 10 minutes beats burning 90.

Process requirement (not a feature): the precondition email goes to the client a week before the session. The tool supports generating it standalone, ahead of time.

### 5.2 Guided Intake

Run the seven INTAKE.md questions as a Hebrew conversation, one question at a time. After each answer, reflect a distilled phrasing back to the client and get confirmation before moving on. Vague answers ("the agent should generally help me") are not accepted: ask for a concrete example from the past week and do not proceed without one. Questions 6 and 7 (connectors, classification) were already answered in the precondition check; present them for final confirmation only.

### 5.3 File Generation and Validation

Fill the four template files in English from the confirmed answers. Show Itzik a diff against the master template (what was filled, what deviates from defaults). Write to the client project only after his approval. Blocking validations: no empty brackets, the connector list in BOUNDARIES.md matches the precondition-approved list exactly, and the fixed policy sections are untouched (the approval line; the two built-in Hard Floor items).

### 5.4 Smoke Test

Two tests on the client's real data. First: run the "morning open" routine on the real inbox; the client confirms the triage makes sense. Second: the agent produces one email draft; the client rates it against the three voice words from question 4. If the draft fails, fix the Voice section in PROFILE.md and rerun, up to twice. A third failure is logged as a finding and does not block handoff, but goes into first-week follow-up.

Sensitivity guard: the precondition phase includes an explicit warning that real inbox content will appear on screen, and lets the client restrict the smoke test to a date range they choose.

### 5.5 Training and Close

The client, not Itzik, runs "morning open" and "session close" once each, alone. The tool ends with a setup report: template version, customizations, smoke test results, and one first action for the coming week. The report is saved to the client project and a copy to TriFold. The report stays between TriFold and the client; it is never sent to the client's IT. If IT wants documentation, the client forwards it themselves.

### 5.6 Reconfiguration (post go-live)

When the "drafts approved without edits" metric drops or the client requests a change, allow a controlled update: edit PROFILE.md or BOUNDARIES.md through the same validations, and record what changed and why in the STATE.md Lessons Log. A master-template update produces the list of clients on older versions, with a per-client diff. Applying to a client is always manual, never automatic.

**Version registry.** One private Git repository on TriFold's GitHub: a `template/` directory for the master, one directory per client for their instance. Template versions are tagged (`template-v1.0`, ...); every client change is a commit whose message explains what and why. After every passed validation (setup, approved smoke test, or update), tag the commit `known-good`.

**Restore.** The client must never need credentials to TriFold's repo, so restore is dual-layer: after every validated change, the tool saves a snapshot copy of the four files inside the client's own Cowork project. When a client breaks a file (manual edit, accidental deletion), the "restore to known-good" command diffs current state against the snapshot, shows the differences in Hebrew, and restores on the client's confirmation. Git at TriFold remains the implementer's source of truth; the local snapshot is the client's safety net.

## 6. Functional Requirements

Each FR is testable. Do not mark an FR done without demonstrating its acceptance check.

| ID | Requirement | Acceptance check |
|----|-------------|------------------|
| FR-1 | Packaged as a Skill in TriFold's Cowork project, invoked with one command ("start setup for client [name]") | Invocation starts flow 5.1 with no further prompting |
| FR-2 | Precondition check runs before intake, returns explicit Go / No-Go with a gap list | Simulated missing connector produces No-Go + named gap |
| FR-3 | On No-Go, generate a Hebrew draft email to IT listing required connectors | Email includes every missing connector and a rationale |
| FR-4 | Intake runs in Hebrew, one question at a time, reflect-and-confirm per answer | No question advances without explicit confirmation |
| FR-5 | Vague answers trigger a request for a concrete example; flow does not proceed without it | Test with "just help me generally" — flow holds |
| FR-6 | Generation shows a diff against master template; write requires implementer approval | No write occurs before approval in test run |
| FR-7 | Blocking validation: empty brackets, connector mismatch, or modified fixed-policy sections prevent writing | Each of the three violations blocks in test |
| FR-8 | Smoke test runs on real client data and is logged: input scope, output, client rating | Log exists after test run |
| FR-9 | Setup report generated automatically at session end, includes template version + customizations | Report present in client project and TriFold copy |
| FR-10 | Version registry in a private GitHub repo: tagged `template/` + per-client directories; master update produces stale-client list with diffs | Bump template tag → tool lists affected clients with diffs |
| FR-11 | Phase timer visible to implementer only, against the Section 2 schedule | Timer output not shown in client-facing turns |
| FR-12 | All generated files in English; all client interaction and reports in native Hebrew | Review of one full session transcript |
| FR-13 | After every validated change: local snapshot in client project + `known-good` tagged commit in TriFold repo | Both artifacts exist after a test update |
| FR-14 | "Restore to known-good" diffs against local snapshot, shows differences in Hebrew, restores on client confirmation, requires no client GitHub access | Break a file, restore it, verify no credentials involved |

## 7. Non-Functional Requirements

- Runs entirely inside Cowork: no external server, no database, no dependency outside the project and the GitHub repo.
- Data classification is respected: the smoke test uses only data the client approved in question 7; TriFold's side stores setup metadata only, never client email content.
- Client instances in TriFold's Git repo contain business information (mission, voice, boundaries) but never operational content: no emails, no client STATE.md, no smoke test outputs. This holding is anchored in the engagement agreement.
- Interruption-resilient: if the session drops, resume from the last confirmed question, not from the start.
- Hebrew output reads as native Hebrew. No em-dashes in Hebrew text.

## 8. Repository and Artifact Layout

```
trifold-agent-clients/            # private GitHub repo
├── template/                     # master template, tagged template-vX.Y
│   ├── CLAUDE.md
│   ├── PROFILE.md
│   ├── BOUNDARIES.md
│   ├── STATE.md
│   └── INTAKE.md
├── clients/
│   └── <client-slug>/
│       ├── CLAUDE.md            # filled instance
│       ├── PROFILE.md
│       ├── BOUNDARIES.md
│       ├── STATE.md             # skeleton only — never synced after go-live
│       └── setup-report.md
└── registry.md                   # client ↔ template version table
```

Inside each client's Cowork project, the tool additionally maintains `.snapshots/` with the last known-good copy of the four files (FR-13).

## 9. Success Metrics

| Metric | v0 target |
|--------|-----------|
| Setup duration (open → files written) | ≤ 30 min |
| Total session duration | ≤ 90 min |
| Smoke test passes on first or second run | 80% of sessions |
| Sessions stopped at No-Go rather than failing mid-session | 100% of IT-gap cases |
| Client runs a routine alone by session end | 100% |
| "Drafts approved without edits" measured from week one | 100% of clients |

The long-term health metric belongs to the deployed agent, not this tool: drafts approved without edits in weeks 1–4. The tool is measured on making sure that measurement exists.

## 10. Risks

**IT as bottleneck.** Biggest risk: sessions that never launch because connectors weren't approved. Mitigation: precondition email a week ahead (process rule), standalone email generation (FR-3).

**Intake overruns.** Executives like to talk. Mitigation: reflect-and-confirm per answer plus the implementer timer. If the intake phase crosses 25 minutes, the tool proposes locking defaults and refining during week one.

**Smoke test on a sensitive inbox.** A sensitive email may surface in front of the consultant. Mitigation: explicit warning in preconditions and client-chosen date range (5.4).

**Template drift.** Per-client customizations accumulate until there is no template. Mitigation: FR-7 locks the policy sections, FR-10 keeps every diff visible.

## 11. Release Plan → Build Phases

**v0 (target: September 2026).** Consultant-led Skill. Flows 5.1–5.5, the Git registry, and local snapshots (FR-13). Pilot: Itzik on himself, then two real clients.

Suggested build order for Claude Code:
1. Repo scaffold + registry (Section 8) and template import
2. Skill skeleton + flow 5.1 with FR-2/FR-3
3. Flow 5.2 intake engine with FR-4/FR-5
4. Flow 5.3 generation + blocking validations FR-6/FR-7
5. Flows 5.4–5.5 + report FR-8/FR-9 + snapshot FR-13
6. Timer FR-11, resume-on-interruption, full dry run against acceptance checks

**v0.1.** Reconfiguration and restore (flow 5.6, FR-14), shaped by pilot findings.

**v1 (decide after five clients).** Self-serve evaluation: client runs intake alone, Itzik approves asynchronously. The decision hinges on one v0 datum: how many implementer interventions were necessary versus merely nice.

## 12. Resolved Decisions (were open questions in v0.1)

1. **Version registry:** private GitHub repo, tagged master + per-client directories (5.6, FR-10).
2. **Setup report:** stays between TriFold and the client; client forwards to their IT if required (5.5).
3. **Restore after breakage:** yes — "restore to known-good" command, local-snapshot based, no client repo access (5.6, FR-13, FR-14). Snapshot ships in v0 (nothing to restore without it); the restore command ships in v0.1.
