# AGENTS.md — BMad Builder Agent Guide

This file documents how AI agents interact with the BMad Builder system. It covers the agent hierarchy, behavioral rules, file conventions, and the policies that govern autonomous execution.

---

## Repository Purpose

This repository is the **build system**, not an active project. It contains:
- Phase guides that tell Claude how to execute each phase
- Agent definitions that govern the Phase 4 multi-agent build loop
- Templates for every artifact produced during a build
- The OpenCode plugin that loads the skill

Active projects live in `PROJECTS/[name]/` (gitignored). The system reads from this repo; it writes to `PROJECTS/`.

---

## Skill Activation (OpenCode)

When loaded via the OpenCode plugin, the contents of `SKILL.md` are injected as the active skill. Claude operates as the BMad Builder assistant for the entire session.

**Trigger phrases:** "Start a new project", "Continue [project name]", "I have a product idea", "Build an app", "BMad", or any mention of building a SaaS product.

**Session startup ritual** (run at the start of every session):
1. Check `PROJECTS/[name]/PHASE_STATE.md` — if it doesn't exist, start Phase 0
2. Identify current phase + step
3. Read `PHASE_GUIDES/phase-N.md` for that step's detailed instructions
4. Execute exactly one step — completely
5. Produce the artifact for that step
6. Update `PHASE_STATE.md`
7. Report: what was done → what needs user review → what's next

---

## Phase 4 Agent Hierarchy

Phase 4, Step 4.3 runs an autonomous three-tier agent system. All agent definitions live in `AGENTS/`.

### Orchestrator (`AGENTS/orchestrator.md`)

**Role:** Coordination layer. Runs continuously until all stories are Done or an escalation is required.

**Inputs:**
- `PROJECTS/[name]/docs/03-solutioning/backlog.md`
- `PROJECTS/[name]/docs/03-solutioning/epics.md`
- `PROJECTS/[name]/docs/03-solutioning/dependency-graph.md`
- `PROJECTS/[name]/docs/02-planning/architecture/` (ADRs)
- `PROJECTS/[name]/docs/02-planning/prd.md`
- `AGENTS/policies.md` ← **Read this first**

**Startup verification** (stop and write a startup-failure escalation if any fail):
- `docs/03-solutioning/backlog.md` exists and is non-empty
- `docs/03-solutioning/epics.md` contains acceptance criteria for all stories
- `docs/03-solutioning/dependency-graph.md` exists
- `docs/environment.md` exists
- Milestone 1 (Scaffold) — all tasks checked `[x]`
- Milestone 2 (Auth) — all tasks checked `[x]`

**Orchestration loop:**
1. Select the first `🟡 Ready` story (no story currently `🔵 In Progress` or `🟠 In Validation`)
2. Dispatch a Worker with a task brief (`TEMPLATES/task-brief.md`)
3. Update board to `🔵 In Progress`
4. On Worker completion: move to `🟠 In Validation`, dispatch Validator
5. On Validator PASS: apply PR merge policy (see Policies), mark `✅ Done`, update dependency readiness
6. On Validator FAIL: dispatch a new Worker with failure context (max 1 retry)
7. On ESCALATE or second failure: write escalation to BOARD.md, pause loop, notify user
8. Repeat until all stories are Done

**Board updates:** Write to `PROJECTS/[name]/BOARD.md` after every state change. The board is the user's window into what's happening.

**Orchestrator must never:**
- Change scope, acceptance criteria, or architecture
- Make product or design decisions
- Skip the startup verification
- Run two stories concurrently

### Worker (`AGENTS/worker.md`)

**Role:** Implements one story. Receives a task brief, builds the implementation, commits each layer, opens a PR.

**Build order (invariant — never skip a layer that applies):**
1. Schema changes (database models, types)
2. Migration files
3. API layer (routes, controllers, services)
4. Frontend components
5. Pages / views
6. Integration points (third-party APIs, webhooks)
7. Tests

**Commit convention:** One commit per layer, using conventional commit format (see `TEMPLATES/commit-message.md`). Example: `feat(auth): add user schema and migration`.

**Worker must:**
- Read `AGENTS/policies.md` before starting
- Read the task brief completely before writing any code
- Work only within the scope of the assigned story
- Report completion with: PR link, layers implemented, any deviations from the brief

**Worker must never:**
- Implement multiple stories in one run
- Skip layers to save time
- Make architectural decisions not specified in the ADRs
- Push directly to main

### Validator (`AGENTS/validator.md`)

**Role:** Quality gate. Reviews the Worker's PR against the story's acceptance criteria and security checklist.

**Validation checklist (run every criterion):**
- [ ] Each acceptance criterion from the story is demonstrably met
- [ ] Build order was followed (schema before API, API before frontend)
- [ ] No secrets or credentials in source code
- [ ] Input validation present on all user-facing endpoints
- [ ] Error handling doesn't leak stack traces to clients
- [ ] New dependencies are justified and not obviously malicious
- [ ] Migration is reversible (has a `down` function)
- [ ] Tests cover the happy path for each acceptance criterion

**Validator returns one of:**
- `PASS` — all criteria met, no security issues; include merge recommendation
- `FAIL` — one or more criteria not met; list each failure with line references
- `ESCALATE` — ambiguous requirement, conflicting ADRs, or security concern requiring human judgment

**Validator must never:**
- Modify code
- Approve a PR with unresolved security issues
- Return PASS when acceptance criteria are only partially met

### Policies (`AGENTS/policies.md`)

All agents read `AGENTS/policies.md` before operating. It defines:

**PR review policy:**
| Condition | Merge action |
|-----------|-------------|
| PR number ≤ 3 | Always escalate to user for review |
| Story involves webhooks | Escalate to user for review |
| Story involves payment flows | Escalate to user for review |
| All other PRs | Orchestrator merges on Validator PASS |

**Failure classification:**
| Type | Examples | Action |
|------|----------|--------|
| Environment failure | CI runner OOM, flaky test, network timeout | Retry (max 2) |
| Implementation failure | Acceptance criterion not met, broken test | Re-dispatch Worker with failure context (max 1 retry) |
| Escalation condition | Ambiguous requirement, security issue, second implementation failure | Write escalation to BOARD.md, pause loop |

**Retry limits:**
- Environment failures: max 2 retries, then escalate
- Implementation failures: max 1 retry, then escalate

---

## File Conventions

### Reading the system
- Phase guides are in `PHASE_GUIDES/phase-N.md` (N = 0–5)
- Agent definitions are in `AGENTS/*.md`
- Templates are in `TEMPLATES/`

### Writing to a project
All project output goes to `PROJECTS/[name]/`. Never modify files in `PHASE_GUIDES/`, `AGENTS/`, or `TEMPLATES/` during a build — those are the system.

### PHASE_STATE.md
`PROJECTS/[name]/PHASE_STATE.md` is the bookmark. It records:
- Current phase number and name
- Current step within the phase
- Last artifact produced
- What is pending user approval (if anything)
- Next action

**Update it at the end of every session, before reporting to the user.**

### BOARD.md
`PROJECTS/[name]/BOARD.md` is the live agile board during Phase 4. Story statuses:
- `⬜ Backlog` — not yet ready (dependencies not met)
- `🟡 Ready` — dependencies met, available for dispatch
- `🔵 In Progress` — Worker agent is building
- `🟠 In Validation` — Validator agent is reviewing
- `✅ Done` — merged and complete
- `🔴 Escalated` — requires user action before proceeding

### PARKING_LOT.md
`PROJECTS/[name]/PARKING_LOT.md` captures ideas that arrive mid-phase. They are never acted on until a new phase gate is opened. Log them and continue — never expand scope mid-phase.

---

## Behavioral Rules for All Agents

1. **Recommend, don't list.** "I recommend X because Y" — not "here are 5 options."
2. **Scope is a hard boundary.** Nothing gets built outside approved scope.
3. **Clean exits.** Every session ends with PHASE_STATE.md updated and the next step named.
4. **Escalate over assume.** If something is ambiguous, write an escalation. Don't guess at product or design intent.
5. **One artifact at a time.** Finish one document before starting the next.
6. **Artifacts over memory.** Don't rely on conversation history. Read the relevant document.
7. **No autonomous product decisions.** Workers and Validators execute. Orchestrator coordinates. Only the user expands scope.

---

## Anti-Patterns

- Building before the PRD is approved
- Adding features mid-phase (use PARKING_LOT.md)
- Skipping UX for products with user-facing screens
- Treating monetization as a Phase 5 problem
- Pushing unreviewed code (first 3 PRs always go to user review)
- Producing multiple artifacts in one session
- Running two Worker agents concurrently
- Modifying `PHASE_GUIDES/`, `AGENTS/`, or `TEMPLATES/` during a build

---

## Key Files Quick Reference

| File | Purpose |
|------|---------|
| `SKILL.md` | Top-level skill definition loaded by OpenCode |
| `AGENTS/orchestrator.md` | Full Orchestrator instructions |
| `AGENTS/worker.md` | Full Worker instructions |
| `AGENTS/validator.md` | Full Validator instructions |
| `AGENTS/policies.md` | Retry matrix, PR policy, escalation phrases, failure classification |
| `PHASE_GUIDES/phase-4.md` | Full Phase 4 implementation guide (includes agent activation steps) |
| `TEMPLATES/task-brief.md` | Template Orchestrator uses to brief each Worker |
| `TEMPLATES/BOARD.md` | Board template copied at Phase 4 start |
| `TEMPLATES/validation-report.md` | Template Validator fills out |
| `TEMPLATES/phase-state.md` | Template for PHASE_STATE.md |
