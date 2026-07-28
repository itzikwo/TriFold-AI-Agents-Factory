---
name: agent-setup-assistant
description: Runs TriFold's Agent Setup Assistant inside Claude Cowork — the precondition Go/No-Go check, the guided Hebrew intake, file generation, the live smoke test, training/close, post-go-live reconfiguration, and restore-to-known-good for a client's personal-productivity-agent deployment (PRD: PRD-Agent-Setup-Assistant-EN.md). Use when the user (Itzik, the implementer) says "start setup for client [name]", "check preconditions for [name]" / "precondition check for [name]", "send precondition email for [name]", "run intake for [name]", "generate files for [name]", "run smoke test for [name]", "train and close for [name]", "reconfigure [name]" / "update [name]'s agent", "restore [name] to known-good", or "bump template version to [X.Y]" / "check for stale clients". Current scope: Flow 5.1 (precondition check + No-Go email), Flow 5.2 (guided intake), Flow 5.3 (file generation + validation), Flow 5.4 (smoke test), Flow 5.5 (training and close), and Flow 5.6 (reconfiguration), all with an implementer-only phase timer (FR-11) and resume-on-interruption running throughout the 90-minute setup session — plus git-backed version tagging (FR-10, FR-13) and restore-to-known-good (FR-14). This closes every functional requirement in PRD §6.
---

# Agent Setup Assistant

Implements the TriFold Agent Setup Assistant described in `PRD-Agent-Setup-Assistant-EN.md`. That PRD is the source of truth — read it before changing this file.

## Scope of this file

This file implements **Flow 5.1 (Precondition Check)**, **Flow 5.2 (Guided Intake)**, **Flow 5.3 (File Generation and Validation)**, **Flow 5.4 (Smoke Test)**, **Flow 5.5 (Training and Close)**, and **Flow 5.6 (Reconfiguration, post go-live)** — the full PRD §11 v0 build order plus the v0.1 items from §12 (Flow 5.6, FR-14, and the FR-10/FR-13 git work that was blocked pending a git repository and is blocked no longer). This closes every FR in PRD §6.

| Flow | PRD § | Status |
|---|---|---|
| 5.1 Precondition Check (Go/No-Go) | §5.1 | **Implemented (this file)** |
| 5.2 Guided Intake | §5.2 | **Implemented (this file)** |
| 5.3 File Generation and Validation | §5.3 | **Implemented (this file)** |
| 5.4 Smoke Test | §5.4 | **Implemented (this file)** |
| 5.5 Training and Close | §5.5 | **Implemented (this file)** |
| 5.6 Reconfiguration (post go-live) | §5.6 | **Implemented (this file)** |

**Cross-cutting, runs through all of 5.1–5.5**: an implementer-only phase timer against the §2 schedule (FR-11) and resume-on-interruption (NFR). Both are mechanisms, not separate flows — see the Boot context step below for resume, and each flow's steps for where a timer note or a progress-file checkpoint gets written. Neither is duplicated per-row in the table above. **Flow 5.6 runs outside the 90-minute session** (it's post-go-live and asynchronous), so it carries no timer note and writes no `.session-progress.md` checkpoint — see "What this phase does NOT do."

**Also cross-cutting, git-backed (FR-10, FR-13, FR-14)**: every validated write (Flow 5.3's write, a Flow 5.4 rerun that edits Voice, Flow 5.5's report, Flow 5.6's update) is committed locally and tagged `known-good-<slug>-<short-hash>`; pushing to `origin` is always a separate explicit prompt, never automatic. A standalone template-version-bump operation (FR-10) is documented after Flow 5.6. FR-14 (restore) is documented after that, and never touches git — see each section for detail.

If invoked as "start setup for client [name]" (the full-session entry point, FR-1): run Flow 5.1. On **NO-GO**, stop with the gap list and the IT email offer, as before. On **GO**, continue straight into Flow 5.2; once all seven questions are confirmed, continue straight into Flow 5.3. On a successful write, continue straight into Flow 5.4, then Flow 5.5, ending the session with the setup report and the single end-of-session push-to-origin prompt. On a blocked Flow 5.3 validation, stop with the gap list — do not offer the approval prompt, and do not continue into 5.4/5.5.

Flow 5.6 (reconfiguration), the template-version-bump operation (FR-10), and restore-to-known-good (FR-14) are **not** part of this chain — they're invoked standalone, post-go-live, per their own trigger phrases below.

## Boot context

Before running anything, read:
1. `PRD-Agent-Setup-Assistant-EN.md` — full requirements.
2. `registry.md` — existing client ↔ template version records.
3. `clients/<slug>/` if it already exists for this client — do not assume a first-time setup.
4. `clients/<slug>/.session-progress.md` if it exists (resume-on-interruption, NFR). If present and its `Current flow` is not `complete`, this is a resumed session: report the last confirmed checkpoint to the implementer and continue from there by default, reusing everything under `Confirmed so far` instead of re-collecting it — do not restart a flow from its beginning unless the implementer explicitly asks to.

## Invocation

- **"start setup for client [name]"** — full-session entry point (FR-1). Begins at the precondition check.
- **"check preconditions for [name]"** / **"precondition check for [name]"** — standalone check, same flow, no session framing.
- **"send precondition email for [name]"** — standalone, ahead-of-time (PRD §5.1: "the tool supports generating it standalone, ahead of time"). Skip the live connector probe (there is usually no connected session yet) and go straight to the No-Go email using the connector list the implementer supplies.
- **"run intake for [name]"** — standalone entry into Flow 5.2 for when Flow 5.1 already resolved GO in an earlier session. The implementer confirms GO status verbally (and restates the connector list/rationale and classification statement from that earlier check, for Q6/Q7 reuse); the check itself is not re-run.
- **"generate files for [name]"** — standalone entry into Flow 5.3 for when the Intake Record was already confirmed in an earlier session. The implementer restates the confirmed 7-item Intake Record (and the required-connector list from the original precondition check, for the connector-match validation); Flow 5.2 is not re-run.
- **"run smoke test for [name]"** — standalone entry into Flow 5.4 for when the four files already exist at `clients/<slug>/` from an earlier session. The implementer restates (or this file re-derives from the sensitivity guard capture if it exists) the smoke test's input scope; Flow 5.3 is not re-run.
- **"train and close for [name]"** — standalone entry into Flow 5.5 for when a Smoke Test Log already exists from an earlier session. The implementer restates the smoke test outcome; Flow 5.4 is not re-run.
- **"reconfigure [name]"** / **"update [name]'s agent"** — Flow 5.6, post-go-live. Requires the client's four files to already exist at `clients/<slug>/`.
- **"restore [name] to known-good"** — FR-14, post-go-live. Requires `clients/<slug>/.snapshots/` to already contain a saved snapshot.
- **"bump template version to [X.Y]"** / **"check for stale clients"** — FR-10, standalone, not tied to any one client.

In every case, no further prompting is needed to *start* (FR-1) — but Flow 5.1 itself needs three inputs from the implementer before it can produce a result, since there is no persisted intake record to supply them from on a first run. Ask for these up front, together, one message:

1. **Client slug/name.**
2. **Required connectors for this client**, each with a one-line rationale (what the agent will use it for) — e.g. "Outlook — read inbox, draft replies", "SharePoint — read shared planning docs". This would normally come from Itzik's own pre-session scoping; for now he states it directly.
3. **Who determines data classification** — usually the client, live; if this is a dry run ahead of the actual session, Itzik may state it provisionally and flag it as unconfirmed.
4. **The current time** — starts the phase timer (FR-11, see below). Only needed for a full "start setup for client [name]" session; standalone entries only need the current time at their own phase boundary, not the whole session's history.

### Phase timer (FR-11, implementer-only)

Ask the implementer to state the current time when a phase starts, and again at that phase's own output moment (the end of 5.1's Decision rule, the end of 5.2's Intake Record, the end of 5.3's write, the end of 5.4's Smoke Test Log, the end of 5.5's report write). Compute elapsed time and compare against the §2 target for that phase (Open+precondition 10 min, Guided intake 20 min, Generation+validation 5 min, Smoke test 15 min, Training+close 35 min). Report the result as a one-line aside labeled `[Implementer only]`, appended after that flow's own output contract — never inside the Hebrew text directed at the client. This is a cooperative timer: the tool has no autonomous clock access in a plain-instructions skill, so it relies on the implementer stating the time rather than polling one.

### Session progress checkpoints (resume-on-interruption, NFR)

At every confirmed checkpoint across all five flows (each Flow 5.1 step resolved, each Flow 5.2 question confirmed, Flow 5.3's approval and write, each Flow 5.4 test, each Flow 5.5 hand-off and the final report write), write or update `clients/<slug>/.session-progress.md`:

```
Session Progress — <client>
Last updated: <implementer-stated time>

Current flow: <5.1 | 5.2 | 5.3 | 5.4 | 5.5 | complete>
Last confirmed checkpoint: <e.g. "Intake Q3 confirmed">
Confirmed so far: <running list of confirmed distilled answers / resolved steps, enough to reconstruct output without re-asking>

Phase timing (implementer-only):
- Open + precondition: started <t>, target 10 min
- Guided intake: started <t>, target 20 min
- File generation + validation: started <t>, target 5 min
- Smoke test: started <t>, target 15 min
- Training + close: started <t>, target 35 min
```

On Flow 5.5's Step 4 write, append `Session complete: total <elapsed> vs 90 min target` and set `Current flow: complete`. The file is never deleted — it's a durable audit trail of actual session durations. Boot context Step 4 above is what reads this file back in on a resumed session.

## Flow 5.1 — Precondition Check (Go/No-Go)

### Step 1 — Cowork project check

Confirm the current Cowork session/project is scoped to this client (name/slug matches). If this is visibly the wrong project, or no client project exists yet, treat that as a gap: `Cowork project — not found/mismatched`.

### Step 2 — Connector probe (read-only)

For each required connector from the invocation inputs:
- If no tool/connector for it is attached to this session at all, that is an automatic **fail** — do not attempt anything, do not guess.
- If attached, perform exactly one lightweight, read-only action against it (e.g. list the most recent item, list calendars, list recent files — whatever that connector's cheapest read is) and record the outcome.
- Record each connector as `ok` or `fail: <short reason>` (not attached / auth error / probe call failed / etc).

### Step 3 — Data classification

Ask (or confirm, if supplied at invocation) for one explicit sentence stating what data classification is permitted into the agent. Do not proceed past this step without an explicit answer — a missing or hedged answer ("probably fine") is a gap, not a pass: `Data classification — not confirmed`.

### Step 4 — Sensitivity guard (smoke test warning)

State plainly to the client that during the smoke test (Flow 5.4), their real inbox and calendar content will be visible on screen to the implementer. Ask whether they want to restrict the smoke test to a specific date range, or run it against everything; record the answer ("no restriction" or the specific range) for Flow 5.4 to use. This step is informational capture only — it does **not** factor into the Decision rule below, which stays scoped to project + connectors + classification.

After each of Steps 1–4 resolves, update `clients/<slug>/.session-progress.md` (Current flow: 5.1; checkpoint: which step; Confirmed so far: gap list / connector results / classification / sensitivity answer as each lands) — see Invocation's "Session progress checkpoints."

### Decision rule

- **Go**: Cowork project confirmed, every required connector `ok`, classification explicitly stated.
- **No-Go**: any of the above fails. List every failing item — do not stop at the first one; the implementer needs the full gap list in one pass (FR-2).

Update the progress checkpoint once more with the Decision rule's result.

### Output contract

Report in this shape:

```
Precondition Check — <client>
Status: GO | NO-GO
Checked: <timestamp>

Gaps (if NO-GO):
- <item>: <reason>
- <item>: <reason>

Next step: <intake can proceed | fix gaps and re-run | send IT email>
```

`[Implementer only]` If this is a full "start setup" session, append the phase timer result for Open + precondition (elapsed vs. 10 min target) — see Invocation's "Phase timer." Never part of the block above.

## No-Go path — Hebrew IT email (FR-3)

Triggered automatically whenever the Decision rule above resolves **NO-GO**, or directly via "send precondition email for [name]".

Draft (never send — this is always a draft handed to the implementer, matching the template package's fixed approval-line policy in BOUNDARIES.md) a Hebrew email to IT that:

- Is addressed generically to IT ("צוות IT") unless a named contact was given.
- Lists **every** gapped/missing connector from the gap list, each with a short rationale tied to what the agent will actually use it for (draw the rationale from the connector's stated purpose in the invocation inputs).
- States this is a precondition for a scheduled client setup session and asks for approval ahead of it (PRD Risks: send this roughly a week before the session).
- Requests confirmation of the permitted data classification if that was also a gap.
- Reads as native Hebrew: no em-dashes in Hebrew text (same rule as `template/CLAUDE.md` Language Rules).

Present the drafted email text to the implementer; do not send it through any connector.

## Flow 5.2 — Guided Intake

Runs only after Flow 5.1 resolves **GO** in this session, or when invoked standalone ("run intake for [name]") with the implementer confirming GO already happened elsewhere. Never run intake against a NO-GO client.

The seven questions come from `template/INTAKE.md`, asked in Hebrew, **one at a time**, in order:

| # | Topic | Fills |
|---|---|---|
| 1 | Who you are: role, company, recurring weekly problem, preferred info format | PROFILE.md / The Human |
| 2 | Mission sentence — "exists to help ___ do ___ so that ___" | PROFILE.md / Mission |
| 3 | Recurring actions (up to 5, from the default list: inbox triage, meeting prep, drafts, decision tracking, close-out report) | PROFILE.md / What I Actually Do |
| 4 | Voice — three words + banned phrases | PROFILE.md / Voice |
| 5 | Hard floor — 1–2 things the agent must never do, beyond the two built-ins | PROFILE.md / Hard Floor |
| 6 | IT-approved connectors | BOUNDARIES.md / Approved Environment |
| 7 | Data classification | BOUNDARIES.md / Approved Environment |

### Q1–Q5 loop

For each question, in order:

1. **Ask** — the question in Hebrew, following `template/INTAKE.md`'s wording.
2. **Vagueness check (FR-5)** — do not accept an answer that has no concrete instance behind it (no named example, nothing anchored to something that actually happened). Generic answers ("תעזור לי באופן כללי", "תהיה יעיל", "מקצועי" alone for Q4) are vague. Ask for a specific example from the past week that illustrates the answer, and hold on this question — repeat the request until the answer is concrete. Do not advance.
3. **Reflect and confirm (FR-4)** — once concrete, reflect back a distilled one- or two-line Hebrew phrasing of what was understood. Ask for explicit confirmation (e.g. "זה נכון?"). If the client corrects it, redo the reflection with the correction and ask again. Only an explicit yes advances to the next question; silence or an ambiguous reply does not.
4. **Checkpoint** — once confirmed, update `clients/<slug>/.session-progress.md` (Current flow: 5.2; checkpoint: "Intake Q<n> confirmed"; append this question's confirmed distilled answer to Confirmed so far).

**25-minute proposal (PRD Risks)**: `[Implementer only]` if the phase timer (see Invocation) shows intake elapsed past 25 minutes at any question boundary, propose to the implementer — do not apply automatically — locking the remaining unanswered questions to their default option (e.g. Q3's default recurring-actions list) and finishing intake now, flagged for refinement in week one. This does not relax the Vagueness check or Reflect-and-confirm above: any question still answered now still needs a concrete answer and explicit confirmation, just without further open-ended discussion. If the implementer declines, continue the Q1–Q7 loop normally.

### Q6/Q7 — confirm, don't re-ask

Per PRD §5.2: "Questions 6 and 7 ... were already answered in the precondition check; present them for final confirmation only." Reuse the required-connector list + rationale and the classification statement gathered during Flow 5.1 Step 2/3 (or restated at standalone invocation). Present them back as the distilled phrasing directly — do not ask the client to answer these from scratch — and run the same explicit-confirm step as Q1–Q5, including the same checkpoint update.

### Output — Intake Record

Once all seven are confirmed, emit a structured record:

```
Intake Record — <client>
1. The Human: <confirmed distilled answer>
2. Mission: <confirmed distilled answer>
3. What I Actually Do: <confirmed distilled answer>
4. Voice: <confirmed distilled answer>
5. Hard Floor: <confirmed distilled answer>
6. Approved Connectors: <confirmed distilled answer>
7. Data Classification: <confirmed distilled answer>

Next step: file generation (Flow 5.3).
```

`[Implementer only]` If this is a full "start setup" session, append the phase timer result for Guided intake (elapsed vs. 20 min target) — see Invocation's "Phase timer." Never part of the block above.

This record is the handoff Flow 5.3 consumes directly, in this same session or restated at standalone invocation.

## Flow 5.3 — File Generation and Validation

Runs only after Flow 5.2 produces a confirmed 7-item Intake Record in this session, or when invoked standalone ("generate files for [name]") with the implementer restating that confirmed record and the Flow 5.1 required-connector list.

### Step 1 — Fill

Build the four template files in English from the confirmed answers:

- **CLAUDE.md** — no brackets exist in the master template. Copy it verbatim. The entire file is fixed policy (boot sequence, routines, mid-session write triggers, recovery) and is never customized per client.
- **PROFILE.md** — fill:
  - Title `[Agent Name]` — not tied to any of the 7 questions; auto-derive as `<Client name> Agent` unless the implementer states an override.
  - The Human — Intake Q1's confirmed distilled answer.
  - Mission — Intake Q2's three blanks (`[who]`, `[what]`, `[result]`).
  - What I Actually Do — replace/adjust the default bullet list per Intake Q3's confirmed answer (kept / dropped / added).
  - Voice — Intake Q4: the three words and the banned-phrases list; leave "Sentence length" as `short / varied` unless Q4's answer specified one.
  - Hard Floor — **items 1–2 ("never present a draft as sent", "never invent facts") are fixed policy, not bracketed — leave them verbatim.** Append Intake Q5's confirmed answer as item 3 (and item 4 if two things were given).
  - Language Rules section has no brackets — copy verbatim, fixed policy.
- **BOUNDARIES.md** — fill:
  - Approved Environment → Approved connectors — Intake Q6's confirmed answer.
  - Approved Environment → Maximum data classification — Intake Q7's confirmed answer.
  - Attention Budget `[3]` — not an intake field; resolve to the literal default `3`.
  - Everything else (Core Principle, Symmetry Test, both Trust Levels subsections, Hard Stops, Enforcement Map, Known Attack Vectors) is fixed policy — copy verbatim. The "Requires Human Approval — always" list under Trust Levels is "the approval line" referenced in FR-7 below.
- **STATE.md** — per `clients/README.md` this ships as **skeleton only**, not a filled instance. Copy the master template verbatim, brackets and all. It is populated by the live agent starting with its first real session, not by setup.

### Step 2 — Diff display (FR-6)

Before validation or any approval prompt, show the implementer a per-file, per-section breakdown against the master template: what was filled (with the actual text used), and an explicit "unchanged — fixed policy" marker on every section that must stay verbatim. Show this diff regardless of whether validation will pass — the implementer sees the same view either way.

### Step 3 — Blocking validations (FR-7)

Run after the diff, before any approval prompt. Check all three, independently, and list **every** violation found in one pass — do not stop at the first (same pattern as the Flow 5.1 decision rule):

1. **No empty brackets** — scan CLAUDE.md, PROFILE.md, and BOUNDARIES.md (STATE.md is excluded by design, see Step 1) for any leftover `[...]` placeholder. Each one found is a violation: `<file> — empty bracket: <placeholder text>`.
2. **Connector match** — BOUNDARIES.md's filled connector list must equal, exactly (same set, no extras, no omissions), the required-connector list gathered in Flow 5.1 Step 2. Any mismatch is a violation: `BOUNDARIES.md — connector mismatch: <detail>`.
3. **Fixed-policy sections untouched** — CLAUDE.md must be byte-identical to the master template; PROFILE.md's Hard Floor items 1–2 and Language Rules section must be byte-identical to the master; BOUNDARIES.md must be identical to the master everywhere except Approved Environment and the Attention Budget number. Any deviation is a violation: `<file> — fixed-policy section modified: <detail>`.

### Step 4 — Approval gate

- If validations pass: ask the implementer for explicit written approval to write the files (English, implementer-facing — this is not client-facing Hebrew). Silence or an ambiguous reply does not authorize a write.
- If validations fail: do not offer the approval prompt at all. Show the full violation list and stop — the implementer must fix the source data (re-run part of intake, correct a fixed-policy edit, etc.) and re-run Flow 5.3.

On explicit approval, update `clients/<slug>/.session-progress.md` (Current flow: 5.3; checkpoint: "approval granted").

### Step 5 — Write

Only after explicit approval:

1. Create `clients/<slug>/` if it does not already exist.
2. Write the four files (CLAUDE.md, PROFILE.md, BOUNDARIES.md, STATE.md) into it.
3. Add or update the client's row in `registry.md`: Client, Slug, Template Version, Setup Date, Last Updated, Status. Record Template Version as the current `template-vX.Y` tag (`git tag -l "template-v*" --sort=-v:refname | head -1`, or ask the implementer if git is unavailable).
4. Save a snapshot copy of the four just-written files into `clients/<slug>/.snapshots/` (FR-13, local half — this is one of the two "validated change" checkpoints that get a snapshot; Flow 5.5's close is the other).
5. Commit CLAUDE.md, PROFILE.md, BOUNDARIES.md, and STATE.md (this one time only, while it's still skeleton — see the git-backed cross-cutting note above and "What this phase does NOT do" for why STATE.md is never re-committed after this) under `clients/<slug>/` to git, with a message stating this is the initial setup for `<client>` against `template-vX.Y`. Tag that commit `known-good-<slug>-<short-hash>` (`git rev-parse --short HEAD` right after the commit). Do not push yet — pushing happens once, at the end of Flow 5.5.
6. Update `clients/<slug>/.session-progress.md` (checkpoint: "files written, committed, and snapshotted").

No write, of any file, happens before Step 4's explicit approval.

`[Implementer only]` If this is a full "start setup" session, note the phase timer result for File generation + validation (elapsed vs. 5 min target) — see Invocation's "Phase timer."

## Flow 5.4 — Smoke Test

Runs only after Flow 5.3's approved write in this session, or when invoked standalone ("run smoke test for [name]") once the four files already exist at `clients/<slug>/`.

This flow is where the session stops setting up the agent and starts **acting as** it, for one live test: once the files exist, this same conversation re-boots against them and runs the client's actual routines on the client's actual data. This is scoped to the smoke test inside this session only — it is not ongoing autonomous operation, which stays out of scope (PRD Non-Goals).

### Step 1 — Sensitivity guard restatement

Restate the warning and date-range answer captured in Flow 5.1 Step 4. If this is a standalone invocation with no Flow 5.1 record available, capture it now — do not touch any real inbox or calendar content before this is settled.

### Step 2 — Boot as the deployed agent

Read, in order, `clients/<slug>/CLAUDE.md` → `PROFILE.md` → `BOUNDARIES.md` → `STATE.md`, following that CLAUDE.md's own Boot Sequence. From this point the session operates as the client's deployed agent for the remainder of Flow 5.4, not as the Setup Assistant.

### Step 3 — Test 1: Morning open

Actually run "Morning open" / "פתח את הבוקר" (`CLAUDE.md` → Routines), scoped to the connectors approved in BOUNDARIES.md and any date restriction from Step 1: scan inbox/calendar (read-only), surface items needing a decision and meetings needing prep, propose one most-important task, then wait. Show the triage output to the client and ask them to confirm it makes sense. Record `pass` or `fail` plus a one-line client comment.

### Step 4 — Test 2: Draft rating

Produce one email draft (e.g. a reply to something Step 3 surfaced). Show it to the client and ask them to rate it against the three Voice words from Intake Q4 — does it read the way those three words describe.

- **Pass**: record the rating and move to Step 5. If a Voice edit landed on the way to this pass (see Fail below), commit `clients/<slug>/PROFILE.md` to git now (message: what the smoke test surfaced and what changed) and tag `known-good-<slug>-<short-hash>` — this is the "approved smoke test" validated-change checkpoint from PRD 5.6. Do not push yet.
- **Fail**: edit the client's `PROFILE.md` Voice section to address the specific gap the client named, then rerun the draft. Allow up to two reruns (three attempts total).
- **Still failing after the second rerun**: do not block — log it as an unresolved finding to carry into Flow 5.5's setup report as a candidate "first action for next week," and move to Step 5 anyway. Any Voice edit already made stays in the file but is uncommitted — it's not a passed validation, so it does not get a `known-good` tag; Flow 5.5's report will flag it for follow-up instead.

After each of Steps 1–4, update `clients/<slug>/.session-progress.md` (Current flow: 5.4; checkpoint: which step, e.g. "Test 2 attempt 2 failed" / "Test 1 passed").

### Step 5 — Smoke Test Log (FR-8)

Emit a structured record (same output-contract pattern as the earlier flows):

```
Smoke Test Log — <client>
Input scope: <connectors used>; date range: <no restriction | YYYY-MM-DD to YYYY-MM-DD>

Test 1 — Morning open: <pass | fail> — <one-line client feedback>
Test 2 — Draft rating: <pass on attempt N | unresolved after 3 attempts>
  Attempt 1: <rating vs. the three voice words> — <feedback>
  [Attempt 2, 3 if needed]

Unresolved finding (if any): <what still doesn't land>

Next step: training and close (Flow 5.5)
```

`[Implementer only]` If this is a full "start setup" session, append the phase timer result for Smoke test (elapsed vs. 15 min target) — see Invocation's "Phase timer." Never part of the block above. Update the progress checkpoint (log emitted).

## Flow 5.5 — Training and Close

Runs only after Flow 5.4 produces a Smoke Test Log in this session, or when invoked standalone ("train and close for [name]") once a Smoke Test Log already exists from an earlier session.

### Step 1 — Hand off "Morning open"

The implementer has the **client**, not himself, invoke "Morning open" / "פתח את הבוקר" once, unaided. Confirm this happened via the client's own words — the implementer relaying it on the client's behalf does not count.

Update `clients/<slug>/.session-progress.md` (Current flow: 5.5; checkpoint: "Morning open client-run confirmed").

### Step 2 — Hand off "Session close"

Same, client-run, once: "Session close" / "סגור סשן". Per `CLAUDE.md`'s own Routines section, this is the first time `clients/<slug>/STATE.md` is legitimately written with real content (Work done, Decisions, Blockers, Next Actions, Metrics) — this is expected here, not a violation of Flow 5.3's "STATE.md ships as skeleton only" rule, which scoped to generation, not live use.

Update the progress checkpoint again ("Session close client-run confirmed").

### Step 3 — Setup report (FR-9)

Do not compose the report until both Step 1 and Step 2 are confirmed client-run. Compose:

```
Setup Report — <client>
Template Version: <same value as the registry.md row from Flow 5.3>
Session date: <date>

Customizations:
- The Human: <from Flow 5.3>
- Mission: <from Flow 5.3>
- What I Actually Do: <from Flow 5.3>
- Voice: <from Flow 5.3, noting any Step 4 edit made during the smoke test>
- Hard Floor (client items): <from Flow 5.3>
- Approved Connectors: <from Flow 5.3>
- Data Classification: <from Flow 5.3>

Smoke Test Results: <Flow 5.4's Smoke Test Log>

First action for the coming week: <one item — from Morning open's proposed task, or an unresolved Flow 5.4 finding>

This report is shared between TriFold and <client> only. Do not send it to the client's IT.
```

### Step 4 — Write and snapshot

1. Write the report to `clients/<slug>/setup-report.md`.
2. Update the client's `registry.md` row (Last Updated, Status).
3. Save a snapshot of the four files — now including STATE.md's real content — into `clients/<slug>/.snapshots/` (FR-13, local half; the other checkpoint is Flow 5.3 Step 5). This snapshot is local only, per FR-13/FR-14's dual-layer design (see FR-14 below) — it is never committed to git, and it's the one place STATE.md's real content is allowed to live, since the git-tracked copy stays frozen at its Step 5 skeleton (see the git-backed cross-cutting note above).
4. Commit `clients/<slug>/setup-report.md` to git (message: session summary — smoke test outcome, first action) and tag `known-good-<slug>-<short-hash>`.
5. Ask the implementer, once, whether to push everything committed this session to `origin` (Flow 5.3's commit, any Flow 5.4 Voice-edit commit, and this report commit). Push only on explicit yes.
6. Update `clients/<slug>/.session-progress.md`: append `Session complete: total <elapsed since the session-start time> vs 90 min target` and set `Current flow: complete`.

`[Implementer only]` If this is a full "start setup" session, note the phase timer result for Training + close (elapsed vs. 35 min combined target) alongside the total. See Invocation's "Phase timer" and "Session progress checkpoints."

## Flow 5.6 — Reconfiguration (post go-live)

Runs standalone, post-go-live, whenever "drafts approved without edits" drops or the client requests a change. Requires `clients/<slug>/` to already exist with all four files. Not part of the 90-minute session chain — no phase timer note, no `.session-progress.md` checkpoint (see "What this phase does NOT do").

### Step 1 — Scope the change

Ask what's changing and why. Restrict edits to **PROFILE.md or BOUNDARIES.md only** — same restriction as Flow 5.3 Step 1: CLAUDE.md is fixed policy (never customized), STATE.md is live-managed by the deployed agent itself, not by this tool. A request to change either of those two is out of scope for this flow; say so and stop.

### Step 2 — Diff and validate

Apply the requested edit, then run Flow 5.3's **Step 2 (Diff display)** and **Step 3 (Blocking validations)** exactly as written there — same three checks (empty brackets, connector match if BOUNDARIES.md changed, fixed-policy sections untouched), same "list every violation in one pass" behavior. Do not restate that logic here; point back to Flow 5.3.

### Step 3 — Approval gate

Same pattern as Flow 5.3 Step 4: if validations pass, ask the implementer for explicit written approval before writing. If they fail, show the violation list and stop — no approval prompt offered.

### Step 4 — Write, log, commit

On explicit approval:

1. Write the changed file into `clients/<slug>/`.
2. Save a snapshot of all four current files into `clients/<slug>/.snapshots/` (FR-13, local half).
3. Append a Lessons Log entry to the client's **live** `STATE.md` — what changed and why (PRD 5.6: "record what changed and why in the STATE.md Lessons Log"). This is local only; STATE.md is never committed to git (see the git-backed cross-cutting note above).
4. Commit only the file(s) actually changed (PROFILE.md and/or BOUNDARIES.md — never STATE.md) to git, message stating what changed and why. Tag `known-good-<slug>-<short-hash>`.
5. Update the client's `registry.md` row (Last Updated, Status).
6. Ask the implementer, once, whether to push this commit to `origin`. Push only on explicit yes.

### Output contract

```
Reconfiguration — <client>
Changed: <PROFILE.md | BOUNDARIES.md>
What changed: <summary>
Why: <client request | drafts-approved-without-edits drop>

Validation: passed
Committed: <short-hash>, tagged known-good-<slug>-<short-hash>
Pushed to origin: <yes | not yet>
```

## FR-10 — Template version bump and stale-client list

Standalone operation, not tied to any one client. Triggered by "bump template version to [X.Y]" or "check for stale clients."

### Step 1 — Confirm the change

Show `git diff template-v<current> -- template/` (find `<current>` via `git tag -l "template-v*" --sort=-v:refname | head -1`) so the implementer confirms what's actually different before tagging anything.

### Step 2 — Commit and tag

Commit the `template/` change (message: what changed and why) and tag the new version `template-v<X.Y>` (implementer states the number — no auto-computed semver logic here).

### Step 3 — Stale-client list

Read every row in `registry.md`. Any row whose Template Version is older than the new tag is stale. For each stale client, show `git diff <their-tag> <new-tag> -- template/` — same per-section diff shape Flow 5.3's Step 2 already uses, so the implementer sees exactly what a later Flow 5.6 update to that client would need to apply.

### Step 4 — Present, do not apply

Present the stale-client list with each diff. **Do not touch any client's files here** — PRD 5.6: "Applying to a client is always manual, never automatic." Applying a template update to a specific client is a separate, later Flow 5.6 invocation for that client.

### Step 5 — Push

Ask the implementer, once, whether to push the new tag/commit to `origin`. Push only on explicit yes.

## FR-14 — Restore to known-good

Standalone, post-go-live. Triggered by "restore [name] to known-good." Requires `clients/<slug>/.snapshots/` to already contain a saved snapshot (written by Flow 5.3 Step 5, a passed Flow 5.4 rerun, Flow 5.5 Step 4, or Flow 5.6 Step 4).

This path **never touches git** — PRD 5.6: "the client must never need credentials to TriFold's repo, so restore is dual-layer... the local snapshot is the client's safety net." Restore reads and writes only `clients/<slug>/` and its `.snapshots/` subfolder.

### Step 1 — Diff against the snapshot

Compare the current `clients/<slug>/{CLAUDE,PROFILE,BOUNDARIES,STATE}.md` against the corresponding files in `.snapshots/`.

### Step 2 — Show the difference in Hebrew

Present what's different in plain Hebrew (FR-12: client-facing) — describe the change in business terms (e.g. "הטון בתשובות שונה מהגרסה השמורה"), not a raw text diff.

### Step 3 — Restore on confirmation

Restore only after the client's explicit confirmation — same reflect-and-confirm pattern used throughout. No restore on silence or an ambiguous reply.

### Step 4 — Log, no new snapshot

On a confirmed restore only (if the client declined in Step 3, stop there — no log entry, no further action): append a short Lessons Log line to the live `STATE.md`: what broke, what was restored, when. Local only, same rule as Flow 5.6 Step 4. No new snapshot is needed — the restored state already equals the existing snapshot. No git operation happens at any point in this flow.

### Output contract

```
Restore to known-good — <client>
Differences found: <file>: <plain-Hebrew description>
Restored: <yes, on client confirmation | no, client declined>
```

## What this phase does NOT do

- Does not run Flow 5.6, the FR-10 template-version-bump operation, or FR-14 restore inside the 90-minute session chain — all three are standalone, post-go-live operations with their own trigger phrases, and none of them write a phase timer note or a `.session-progress.md` checkpoint (those are scoped to the initial setup session in §2/FR-11).
- Does not auto-apply a template update to any client — FR-10's stale-client list is informational; applying is always a separate, manual, per-client Flow 5.6 invocation (PRD 5.6).
- Does not commit `STATE.md` to git more than once (at Flow 5.3's write, while still skeleton) — its real content, from Flow 5.5 onward, lives only in the local Cowork copy and `.snapshots/`, never in a git commit (PRD §7 NFR: no client STATE.md content in the repo).
- Does not push to `origin` automatically anywhere — every flow that commits locally ends with one explicit push prompt, never a silent push.
- Does not use git in the FR-14 restore path at all — restore is local-snapshot-only by design, so the client never needs TriFold repo credentials.
- Does not build v1's self-serve flow (client runs intake alone, async implementer approval) — PRD §11 places that after five pilot clients, as a separate decision.
- Does not poll a real system clock — the phase timer (FR-11) is implementer-cooperative: it relies on the implementer stating the current time at phase boundaries, since a plain-instructions skill has no autonomous clock access.
- Does not simulate an actual dropped connection for resume-on-interruption — resume is verified by pre-seeding `.session-progress.md` mid-flow and confirming the next invocation reads it correctly, not by an unrecoverable session crash.

## Self-test walkthrough (FR-2 / FR-3 acceptance — precondition check)

Run this after any edit to this file:

1. Invoke "check preconditions for client Acme" with required connectors `Outlook` (rationale: read inbox, draft replies) and `Teams` (rationale: read channel context) — but only actually attach/allow `Outlook` in the session, leaving `Teams` unattached.
2. Expect: `Status: NO-GO`, gap list containing exactly `Teams: not attached` (Outlook must not appear in the gap list). This is the FR-2 acceptance check.
3. Expect the No-Go email offer to follow, drafted in Hebrew, naming Teams with its rationale, addressed to IT, presented as a draft only. This is the FR-3 acceptance check.
4. Re-run with both connectors attached and reachable and an explicit classification statement supplied — expect `Status: GO` and no gaps.

## Self-test walkthrough (FR-4 / FR-5 acceptance — guided intake)

Run this after any edit to Flow 5.2:

1. "Run intake for client Acme" with GO already confirmed. Answer Q1 with "תעזור לי באופן כללי" — expect the flow to hold, ask for a concrete example from the past week, and NOT advance to Q2. This is the FR-5 acceptance check (mirrors the PRD's own test case: "just help me generally").
2. Give a concrete Q1 answer — expect a distilled Hebrew reflection and an explicit confirmation request; only after an explicit yes should the flow advance to Q2. This is the FR-4 acceptance check.
3. Walk through Q2–Q5 the same way: concrete answer → reflect → explicit confirm → advance.
4. At Q6/Q7, verify the flow presents back the connector list and classification statement already gathered in Flow 5.1 rather than asking fresh, and still requires explicit confirmation before advancing.
5. Confirm the final output is a complete 7-item Intake Record in the contracted shape.

## Self-test walkthrough (FR-6 / FR-7 acceptance — file generation and validation)

Run this after any edit to Flow 5.3:

1. Run Flow 5.3 with a complete, valid confirmed Intake Record for a fake client. Expect the diff to display in full, validations to pass with no violations, and an explicit approval prompt to follow. Withhold approval — confirm no files are written and `registry.md` gains no row. This is the FR-6 acceptance check.
2. Re-run and approve — confirm the four files are written to `clients/<slug>/` with the correct content, and `registry.md` gains a row for the client.
3. Re-run with one bracket deliberately left unfilled (e.g. PROFILE.md Voice banned phrases still `[list...]`) — expect a block naming that exact gap, and no approval prompt offered. This is FR-7 violation-type check #1 (empty brackets).
4. Re-run with BOUNDARIES.md's connector list missing one connector that was in the Flow 5.1 approved list — expect a block naming the mismatch. FR-7 check #2 (connector mismatch).
5. Re-run with PROFILE.md Hard Floor item 1 reworded from the master's fixed text — expect a block naming the fixed-policy violation. FR-7 check #3 (modified fixed-policy section).
6. Re-run with all three violations present at once — expect all three named in a single violation list, not just the first.

## Self-test walkthrough (FR-8 acceptance — smoke test log)

Run this after any edit to Flow 5.4:

1. Run Flow 5.4 against a fake client's already-written files. Confirm the sensitivity guard from Flow 5.1 Step 4 is restated before anything else happens. Confirm Step 2 boots against the *client's* CLAUDE.md/PROFILE.md/BOUNDARIES.md/STATE.md, not the master template. Confirm Test 1 produces a triage summary and asks for client confirmation before moving on.
2. Confirm Test 2 produces a draft and asks for a rating against the three voice words.
3. Force a Test 2 failure — confirm the edit lands in PROFILE.md's Voice section specifically (not Mission, not Hard Floor), and the draft reruns. Force two more failures — confirm the flow stops rerunning after the second retry (three attempts total), logs an unresolved finding, and does not block moving to Step 5.
4. Confirm the final Smoke Test Log names the input scope (connectors + date range), both test results, and the unresolved finding if one exists. This is the FR-8 acceptance check ("log exists after test run" with input scope, output, and client rating all present).

## Self-test walkthrough (FR-9 acceptance — setup report)

Run this after any edit to Flow 5.5:

1. Run Flow 5.5 immediately after a passing Flow 5.4 for the same fake client. Confirm the flow instructs the implementer to hand off "Morning open" and "Session close" to the *client* — not run them himself — and won't proceed past Steps 1–2 without confirmation that the client ran them.
2. Confirm `clients/<slug>/setup-report.md` is written only after both routines are confirmed client-run, and that it contains: Template Version, the full Customizations list, the Smoke Test Results, and exactly one first action for the coming week.
3. Confirm the report's closing line states it stays between TriFold and the client and is never sent to the client's IT. This is the FR-9 acceptance check.
4. Confirm `registry.md`'s existing row for the client is updated (not duplicated) and that `clients/<slug>/.snapshots/` contains a copy of all four files, now including STATE.md's real post-training content.

## Self-test walkthrough (FR-13 git-tag half acceptance)

Run this after any edit to the git-backed steps in Flow 5.3 Step 5, Flow 5.4 Step 4, or Flow 5.5 Step 4:

1. Run Flow 5.3's write for a fake client. Confirm `git log` shows a new commit touching exactly `clients/<slug>/CLAUDE.md`, `PROFILE.md`, `BOUNDARIES.md`, `STATE.md`, and `git tag -l "known-good-<slug>-*"` shows exactly one new tag pointing at that commit.
2. Confirm `registry.md`'s Template Version for this client matches the current `template-vX.Y` tag, not a placeholder string.
3. Continue into Flow 5.4 and force a Voice-edit-then-pass on Test 2. Confirm a second commit touching only `PROFILE.md` exists, with its own `known-good-<slug>-*` tag.
4. Continue into Flow 5.5. Confirm a third commit touching only `setup-report.md`, its own tag, and exactly **one** push prompt at the end covering all commits made this session — not one prompt per commit.
5. Confirm at every commit that `STATE.md` is untouched after Flow 5.3's initial commit — Flow 5.4 and Flow 5.5's commits must not include it, even though the live copy on disk has real content by Flow 5.5.

## Self-test walkthrough (FR-10 acceptance — template bump and stale-client list)

Run this after any edit to the FR-10 section:

1. Seed `registry.md` with a fake client row on `template-v1.0`. Make a deliberate change under `template/` (e.g. edit a Language Rules line). Invoke "bump template version to 1.1."
2. Confirm Step 1 shows the actual `git diff` before anything is tagged, Step 2 creates commit + `template-v1.1` tag, and Step 3's stale-client list names the fake client (still on `v1.0`) with a diff scoped to `template/`.
3. Confirm no file under `clients/<slug>/` for that client is modified by this operation — this is the FR-10 acceptance check ("bump template tag → tool lists affected clients with diffs," without applying anything).
4. Confirm exactly one push prompt at the end.

## Self-test walkthrough (Flow 5.6 acceptance — reconfiguration)

Run this after any edit to Flow 5.6:

1. Invoke "reconfigure Acme" for a fake client with existing files, requesting a Voice-section change. Confirm Step 2 reuses Flow 5.3's diff/validation exactly (test with one deliberate violation, e.g. an empty bracket — confirm it blocks with no approval prompt, same as FR-7).
2. Re-run cleanly and approve. Confirm only `PROFILE.md` is rewritten, a Lessons Log entry appears in the live `STATE.md`, a git commit exists touching only `PROFILE.md` (not `STATE.md`), and it's tagged `known-good-<slug>-*`.
3. Attempt "reconfigure Acme" requesting a CLAUDE.md change — confirm the flow refuses and stops at Step 1, per the fixed-policy restriction.
4. Confirm no phase-timer note and no `.session-progress.md` update happen anywhere in this flow.

## Self-test walkthrough (FR-14 acceptance — restore to known-good)

Run this after any edit to the FR-14 section:

1. For a fake client with an existing snapshot, manually edit `clients/<slug>/PROFILE.md` outside the tool (simulating breakage). Invoke "restore Acme to known-good."
2. Confirm Step 2 shows the difference in Hebrew, in plain business language — not a raw diff.
3. Decline the restore — confirm the file is left as-is and no `STATE.md` Lessons Log entry is added.
4. Re-run and confirm the restore — confirm `PROFILE.md` now matches the snapshot exactly, a Lessons Log line is appended to `STATE.md`, and — checked directly — no `git` command of any kind ran during this flow (`git status` before/after shows no new commits, tags, or staged changes from this operation). This is the FR-14 acceptance check ("restores on confirmation, no credentials involved").

## Self-test walkthrough (FR-11 acceptance — phase timer)

Run this after any edit to the Phase timer mechanism in Invocation:

1. Run "start setup for client Acme," supplying a stated start time. Walk Flow 5.1 to a GO, supplying a later stated time at the Decision rule. Confirm a `[Implementer only]` timer note appears after the Output contract, comparing elapsed time against the 10-minute target — and confirm this note never appears inside the Hebrew-facing precondition text itself.
2. Continue into Flow 5.2. Supply stated times such that elapsed intake time crosses 25 minutes partway through the Q1–Q7 loop. Confirm the tool proposes — does not automatically apply — locking the remaining unanswered questions to their defaults, and that declining the proposal continues the loop with the Vagueness check and Reflect-and-confirm steps fully intact (no relaxation of FR-4/FR-5). This is the FR-11 acceptance check ("timer output not shown in client-facing turns").

## Self-test walkthrough (resume-on-interruption acceptance)

Run this after any edit to Boot context Step 4 or the per-flow checkpoint updates:

1. Pre-seed `clients/acme/.session-progress.md` by hand with `Current flow: 5.2`, `Last confirmed checkpoint: Intake Q3 confirmed`, and Q1–Q3's confirmed distilled answers recorded under `Confirmed so far`. Invoke "start setup for client Acme" fresh (no other context given). Confirm Boot context Step 4 detects the file, tells the implementer the prior session was interrupted at Q3, and resumes directly at Q4 — reusing the recorded Q1–Q3 answers rather than re-asking them or restarting Flow 5.1.
2. Repeat, but have the implementer explicitly decline the resume offer. Confirm the flow restarts from Flow 5.1 Step 1 instead.
3. Confirm that once Flow 5.5 Step 4 completes for a fresh client, `.session-progress.md` ends with `Current flow: complete` and a `Session complete: total <elapsed> vs 90 min target` line, and that invoking "start setup for client [name]" again for that same client does not offer a resume (nothing left to resume from).

## Full dry run — end-to-end acceptance walkthrough

Run this after any edit that touches more than one flow, and before treating v0 as functionally complete:

Run one continuous "start setup for client [name]" session from Flow 5.1 through Flow 5.5's close, for a single fake client, and confirm across the whole transcript:

- **FR-1**: the single invocation starts Flow 5.1 with no further prompting beyond the four up-front inputs.
- **FR-2/FR-3**: a deliberately induced gap (e.g. one unattached connector) produces NO-GO, the full gap list, and a drafted Hebrew IT email naming it; a clean re-run produces GO.
- **FR-4/FR-5**: at least one intake question is answered vaguely first, held, then answered concretely, reflected, and explicitly confirmed before advancing.
- **FR-6/FR-7**: the generation diff is shown; one deliberate violation (e.g. an empty bracket) blocks with no approval prompt; a clean re-run shows the diff, passes validation, and only writes after explicit approval.
- **FR-8**: a Smoke Test Log is emitted with input scope, both test results, and any unresolved finding.
- **FR-9**: `setup-report.md` is written only after both routines are confirmed client-run, contains all required elements, and closes with the TriFold/client-only line.
- **FR-11**: an `[Implementer only]` timer note appears at every phase boundary (end of 5.1, 5.2, 5.3, 5.4, 5.5), and none of them appear inside client-facing Hebrew text.
- **FR-12**: every file written to `clients/<slug>/` is in English; every turn addressed to the client is in Hebrew — spot-check at least one turn from each flow.
- **Resume bookkeeping**: `clients/<slug>/.session-progress.md` shows a checkpoint update after each flow and ends at `Current flow: complete`.
- **FR-13**: the Flow 5.3 write produces a local git commit and a `known-good-<slug>-*` tag, and the session ends with exactly one push prompt covering every commit made.

Flow 5.6, FR-10, and FR-14 are intentionally out of scope for this dry run — they're standalone, post-go-live operations outside the 90-minute session chain (see "What this phase does NOT do") and have their own dedicated self-test walkthroughs above.
