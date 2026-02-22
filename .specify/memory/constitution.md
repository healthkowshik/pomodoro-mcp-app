<!--
  ============================================================================
  SYNC IMPACT REPORT
  ============================================================================
  Version change: 1.0.3 → 1.1.0 (MINOR — Principle I fundamentally
  redefined: REST API is now a first-class production interface alongside
  MCP; both are adapters over a shared service layer)

  Modified principles:
    - Principle I renamed: "MCP-Native Interface" → "Service-First
      Architecture". REST API is now a first-class production transport,
      not a dev/test-only tool. Business logic MUST live exclusively in
      the service layer; MCP and REST are thin adapters over it.

  Modified sections:
    - Technology Constraints: stripped language/runtime/SDK/file-format
      mandates (those belong in plan.md Technical Context). Retained only
      the governance-level constraint: no paid external service dependencies.
      Project default (Python + FastMCP) recorded as a note, not a mandate.

  Added sections:
    - Core Principles (I–V)
    - Technology Constraints
    - Development Workflow
    - Governance

  Removed sections:
    - N/A (first version)

  Templates reviewed:
    ✅ .specify/templates/plan-template.md — "Constitution Check" section
       present; principle names are referenced generically via
       [Gates determined based on constitution file]. No update needed;
       the generated plan.md will reference these principle names at
       plan-generation time.
    ✅ .specify/templates/spec-template.md — no direct principle
       references; template is principle-agnostic. No update needed.
    ✅ .specify/templates/tasks-template.md — task categories (Setup,
       Foundational, User Stories, Polish) align with Simplicity &
       YAGNI and Observability principles. No update needed.
    ✅ .specify/templates/checklist-template.md — generic; no
       principle-specific references. No update needed.
    ✅ .specify/templates/agent-file-template.md — generic; no
       principle-specific references. No update needed.
    ✅ .specify/templates/constitution-template.md — source template;
       not modified (this file is the filled output).

  Deferred TODOs:
    - None. All placeholders resolved.
  ============================================================================
-->

# Pomodoro MCP App Constitution

## Core Principles

### I. Service-First Architecture

All business logic MUST reside exclusively in the **service layer**. The
REST API and the MCP server are both thin transport adapters over that same
service layer — neither MUST contain business logic of its own.

Both the REST API and MCP tools are **first-class production interfaces**:

- **REST API**: serves non-Claude clients (web app, mobile, third-party
  integrations, Postman-based developer workflows). Its Postman collection
  doubles as the canonical contract documentation for all transports.
- **MCP tools**: serve Claude as a client, exposing the same underlying
  operations as natural-language-callable tools.

The service layer MAY make outbound calls to external services where
necessary, subject to Principle IV (Local-First & Privacy).

Both transport schemas (REST OpenAPI spec and MCP tool schemas) MUST be
versioned and documented. Breaking changes to either MUST trigger a MAJOR
version bump of the affected interface.

**Rationale**: The REST API is not redundant — it is the foundation that
validates and documents the service layer's contracts. MCP is a second
transport over that same foundation. Duplicating business logic across
transports is what creates maintenance burden; a shared service layer
eliminates it.

### II. Timer Correctness (NON-NEGOTIABLE)

Session durations MUST be accurate to within ±1 second of the configured
interval. Timer state (active session, elapsed time, break counts) MUST
survive server process restarts via durable local persistence. Any code
path that starts, pauses, resumes, or cancels a timer MUST update
persisted state atomically before returning a response.

**Rationale**: Reliable time-keeping is the core contract of a Pomodoro
app. Inaccurate or lost state destroys user trust and defeats the
productivity purpose entirely.

### III. Simplicity & YAGNI

The default session cycle MUST follow the classic Pomodoro Technique:
25-minute work interval, 5-minute short break, 15-minute long break after
every 4 completed work intervals. New configuration options or features
MUST be justified against the criterion: "Does this deliver daily,
observable value to a Pomodoro user?" If the answer is not clearly yes,
the feature MUST be deferred. Premature abstractions, unused configuration
knobs, and speculative generalisations are forbidden.

**Rationale**: Complexity compounds. An MCP Pomodoro server that does one
thing excellently is more valuable than one that does many things poorly.

### IV. Local-First & Privacy

All timer session data MUST be persisted exclusively on the user's local
filesystem. No session metadata, work-interval counts, or timing data
MUST be transmitted to any remote service without explicit, documented,
opt-in user configuration. Dependencies that introduce network calls MUST
be audited and approved before inclusion.

**Rationale**: Users' focus habits and work patterns are personal. The
default posture MUST protect privacy without requiring any user action.

### V. Observability

Every timer lifecycle event — `start`, `pause`, `resume`, `complete`, and
`cancel` — MUST emit a structured log entry containing a UTC timestamp,
session ID, event type, and remaining/elapsed duration. MCP resources MUST
expose both current session state and queryable session history. Log output
MUST go to stderr to avoid polluting MCP protocol stdout.

**Rationale**: Observability enables debugging, user self-reflection on
work patterns, and future analytics features — without invasive or
privacy-violating monitoring. Structured logs on stderr are the idiomatic
MCP pattern for server-side output.

## Technology Constraints

Language, runtime, SDK, and persistence choices are **implementation
decisions** made per-feature in `plan.md` (Technical Context section), not
governance mandates. They evolve as the project does.

The one governance-level constraint: no runtime dependency MUST require a
paid external service. This is a principled stance against vendor lock-in
and cost unpredictability, not a technical preference.

> **Project default** (record for plan.md authors): Python with the
> FastMCP v3.0 library. All tests MUST be runnable with a single command from
> the repository root. Deviations from the default MUST be documented in
> the relevant `plan.md`.

## Development Workflow

Feature development MUST follow the speckit workflow:

1. `speckit.specify` — write the feature spec (`spec.md`).
2. `speckit.clarify` — resolve underspecified areas before planning.
3. `speckit.plan` — produce the implementation plan (`plan.md`).
4. `speckit.tasks` — generate the ordered task list (`tasks.md`).
5. `speckit.implement` — execute tasks.
6. `speckit.analyze` — cross-artifact consistency check.

The Constitution Check gate in `plan.md` MUST be completed before any
Phase 0 research begins and re-verified after Phase 1 design. Complexity
violations (deviations from Simplicity & YAGNI) MUST be documented in the
Complexity Tracking table of `plan.md` before proceeding.

All commits MUST reference the task ID (e.g., `T014`) in the commit message
subject. Feature branches MUST follow the pattern `###-feature-name`.

## Governance

This constitution supersedes all prior verbal agreements, wiki pages, and
ad-hoc conventions. When any practice conflicts with a principle stated
here, the constitution takes precedence.

**Amendment procedure**: Amendments require (a) a written rationale
explaining why the change improves the project, (b) a version bump
following semantic versioning (MAJOR / MINOR / PATCH as defined below),
(c) propagation across all dependent templates verified via the Sync Impact
Report, and (d) a commit with the message format:
`docs: amend constitution to vX.Y.Z (<summary>)`.

**Versioning policy**:
- MAJOR — backward-incompatible governance change: principle removal,
  redefinition that invalidates prior work, or technology mandate change.
- MINOR — additive change: new principle, new mandatory section, or
  materially expanded guidance.
- PATCH — clarification, wording improvement, typo fix, or non-semantic
  refinement.

**Compliance review**: Every PR MUST include a "Constitution Check"
confirmation (pass / justified violation) in its description. The
`speckit.analyze` command MUST be run and its output attached or
summarised before merge approval.

**Version**: 1.1.0 | **Ratified**: 2026-02-22 | **Last Amended**: 2026-02-22
