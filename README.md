# BMad Builder — Agentic SaaS Build System

> Take any product idea from raw concept to live, revenue-generating SaaS — guided by a structured, phase-gated methodology with a built-in multi-agent implementation engine.

## What It Is

**BMad Builder** is an AI skill for [OpenCode.ai](https://opencode.ai) that acts as your co-founder and CTO rolled into one. It takes a product idea through six disciplined phases — Foundation, Discovery, Planning, Solutioning, Implementation, and Launch — producing a concrete artifact at every step, gating every transition on your approval, and auto-building your product in Phase 4 with a three-tier agent system (Orchestrator → Worker → Validator).

The system defeats two failure modes:
- **Building without planning** → scope creep, rewrites, wasted sessions
- **Planning without building** → analysis paralysis, nothing shipped

## Core Philosophy

| Principle | What it means in practice |
|-----------|--------------------------|
| **You own the vision** | Every phase gate requires your explicit approval before anything moves forward |
| **Claude executes** | Research, planning, building, deploying — all handled autonomously within approved scope |
| **Artifacts are truth** | Every decision lives in a document. Conversation memory is ephemeral; documents are permanent |
| **Revenue is first-class** | Monetization is designed in Phase 2 and built in Phase 4 — never bolted on post-launch |
| **Visual quality is non-negotiable** | Phase 2 produces a polished prototype. If you wouldn't show it to a potential customer, it fails the gate |
| **One step per session** | Prevents scope creep and context thrash; every session ends with a clear artifact and a named next action |

## The Six Phases

| # | Phase | Artifact | Gate |
|---|-------|----------|------|
| 0 | **Foundation** | `PROJECT.md` — elevator pitch, scope, revenue model, success metrics | Your approval |
| 1 | **Discovery** | `docs/01-discovery.md` — market, competitors, users, risks | Your approval |
| 2 | **Planning** | PRD + UX Design + Architecture (ADRs) + Monetization plan | Your approval |
| 3 | **Solutioning** | Epics, user stories, ordered backlog, dependency graph | Your approval |
| 4 | **Implementation** | Production code, CI/CD, security reviewed, deployed to staging | Your approval |
| 5 | **Launch** | Live product, metrics dashboard, growth loop | Ongoing |

## Installation

Add BMad Builder to the `plugin` array in your `opencode.json`:

**Global install** (`~/.config/opencode/opencode.json`):
```json
{
  "plugin": ["bmad-builder@git+https://github.com/R3dy/BMAD-builder.git"]
}
```

Restart OpenCode. The plugin loads automatically — no manual activation needed.

**Verify** by asking: `"Start a new project"` — Claude should respond in BMad Builder mode.

For Windows troubleshooting or pinning a specific version, see [`.opencode/INSTALL.md`](.opencode/INSTALL.md).

## Quick Start

| Say this | What happens |
|----------|-------------|
| `"Start a new project"` | Creates a new project workspace, begins Phase 0 |
| `"Continue [project name]"` | Reads `PHASE_STATE.md`, resumes the last step |
| `"I have a product idea: [description]"` | Triggers BMad, starts Phase 0 |
| `"Build an app"` | Triggers BMad Builder |

Every project lives in `PROJECTS/[name]/` and is **gitignored** — your product code stays in your project, the build system stays here.

## Project Workspace Layout

When you start a project, BMad Builder creates and populates this structure:

```
PROJECTS/[name]/
├── PHASE_STATE.md              # Current phase + step (the system's bookmark)
├── PARKING_LOT.md              # Future ideas that arrived mid-phase
├── PROJECT.md                  # Phase 0 artifact: vision, scope, revenue model
├── BOARD.md                    # Phase 4 live agile board (updated by agents)
├── docs/
│   ├── 01-discovery.md         # Phase 1: market research
│   ├── 02-planning/
│   │   ├── prd.md              # Product requirements
│   │   ├── ux-design.md        # Design system, components, prototypes
│   │   ├── architecture/       # ADRs — one per major technical decision
│   │   └── monetization.md     # Revenue model and pricing strategy
│   └── 03-solutioning/
│       ├── epics.md            # Epics with acceptance criteria
│       ├── backlog.md          # Ordered milestone task list
│       └── dependency-graph.md # What blocks what
└── environment.md              # Dev environment setup (required by Phase 4)
```

## Phase 4: Multi-Agent Build Loop

Phase 4, Step 4.3 runs an autonomous three-tier agent system that builds your entire backlog without you having to manage individual tasks:

```
Orchestrator
  ├── reads backlog + dependency graph
  ├── manages BOARD.md (live status for every story)
  ├── dispatches Worker agents (one story at a time)
  ├── dispatches Validator agents after each PR
  └── escalates to you only when blocked

Worker (per story)
  ├── implements in strict order: Schema → Migration → API → Frontend
  ├── commits each layer separately (one commit per layer)
  └── opens a PR when the story is complete

Validator (per PR)
  ├── checks every acceptance criterion against the implementation
  ├── runs security checklist
  └── returns PASS / FAIL / ESCALATE
```

**PR review policy:**
- PRs #1–3 always require your review
- Any PR touching webhooks or payment flows requires your review
- All other PRs: Orchestrator merges on Validator PASS

**Board visibility:** `PROJECTS/[name]/BOARD.md` is updated after every agent action. You can see every story's status, the full run log, and any escalations at a glance.

## Repository Layout

```
.opencode/
├── INSTALL.md              # Detailed installation instructions
└── plugins/bmad-builder.js # OpenCode plugin bootstrap

AGENTS/
├── orchestrator.md         # Orchestrator agent instructions
├── worker.md               # Worker agent instructions
├── validator.md            # Validator agent instructions
└── policies.md             # Retry matrix, escalation rules, failure classification

PHASE_GUIDES/
├── phase-0.md              # Foundation step-by-step guide
├── phase-1.md              # Discovery step-by-step guide
├── phase-2.md              # Planning step-by-step guide
├── phase-3.md              # Solutioning step-by-step guide
├── phase-4.md              # Implementation step-by-step guide
└── phase-5.md              # Launch step-by-step guide

TEMPLATES/
├── project.md              # Phase 0 artifact template
├── discovery.md            # Phase 1 artifact template
├── prd.md                  # Phase 2: PRD template
├── ux-design.md            # Phase 2: UX design system template
├── adr.md                  # Phase 2: Architecture decision record template
├── monetization.md         # Phase 2: Revenue model template
├── epic.md                 # Phase 3: Epic template
├── story.md                # Phase 3: User story template
├── task-brief.md           # Phase 4: Worker task spec template
├── BOARD.md                # Phase 4: Agile board template
├── validation-report.md    # Phase 4: Validator report template
├── phase-state.md          # PHASE_STATE.md template
├── launch-checklist.md     # Phase 5: Pre-launch checklist template
├── metrics-dashboard.md    # Phase 5: Metrics dashboard template
└── commit-message.md       # Conventional commit guidelines

SKILL.md                    # Main skill definition (loaded by OpenCode plugin)
package.json                # npm metadata
```

## Design Decisions

**Build order is invariant.** Workers always implement in this order: Schema → Migration → API → Component → Page → Integration → Test. Skipping layers is not allowed — it creates hidden dependencies that cause silent failures later.

**Scope is a hard boundary.** Anything that arrives mid-phase goes to `PARKING_LOT.md`. Nothing gets built outside the approved scope without a new phase gate. This prevents the most common AI-assisted dev failure: the "while I'm in here..." spiral.

**One artifact per session.** Finishing one document cleanly is worth more than starting three. The constraint forces prioritization and keeps PHASE_STATE.md trustworthy.

**Escalation over assumption.** Workers and validators execute only. If they hit something ambiguous, they escalate — they never make product or design decisions autonomously.

## License

MIT — see [package.json](package.json).

## Issues & Contributions

Report issues at [github.com/R3dy/BMAD-builder/issues](https://github.com/R3dy/BMAD-builder/issues).
