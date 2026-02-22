# Implementation Plan: Pomodoro Timer with Work/Break Cycles and Session Tracking

**Branch**: `001-pomodoro-timer` | **Date**: 2026-02-22 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-pomodoro-timer/spec.md`

---

## Summary

Implement the core Pomodoro timer as a service-first Python application exposing the classic 25/5/15 cycle (4 work intervals → long break) via both MCP tools (for Claude Desktop) and a REST API (for web/Postman clients). Timer state is persisted atomically to a local JSON file on every lifecycle event; session history is appended to a local JSON Lines file. All business logic lives in `TimerService`; MCP and REST layers are thin adapters. No background threads — remaining time is computed on-demand using wall-clock math from stored UTC timestamps.

---

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**:
- `fastmcp>=3.0` — MCP server framework (stdio + Streamable HTTP transport; v3.0.1 current)
- `fastapi>=0.110` — REST API framework
- `uvicorn[standard]` — ASGI server (single-process combined mode)
- `pydantic>=2.0` — data validation and serialisation
- `tzlocal>=4.0` — local IANA timezone detection (no network, MIT licensed)
- `tzdata>=2024.1` — IANA tz database for Linux environments lacking `/usr/share/zoneinfo`
- `pytest`, `pytest-asyncio`, `httpx` — test stack
**Storage**: Local filesystem only
- `timer_state.json` — atomic JSON, one record, rewritten per lifecycle event
- `session_history.jsonl` — append-only JSON Lines, one record per terminal session
**Testing**: `pytest` (single command from repo root)
**Target Platform**: macOS and Linux (single-user local machine)
**Project Type**: MCP server + REST API service (dual-transport, shared service layer)
**Performance Goals**:
- Session start: < 2 s (SC-001)
- Status query: < 1 s (SC-004)
- Timer accuracy: ±1 s (SC-002)
- State restore on restart: < 3 s (SC-003)
**Constraints**: All data stored locally; no paid external services; no background tick threads
**Scale/Scope**: Single user, single process; up to ~12 sessions/day; history retained indefinitely

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-checked after Phase 1 design — all gates still pass.*

### I. Service-First Architecture ✅

All business logic (timer math, break-type determination, set counter management, midnight reset, history queries) lives exclusively in `TimerService` and `HistoryService`. MCP tool definitions (`mcp_server.py`) and REST route handlers (`rest_server.py`) contain zero business logic — they validate input and delegate. Both transports are first-class production interfaces. Breaking changes to either require a MAJOR version bump.

### II. Timer Correctness ✅ (NON-NEGOTIABLE)

Remaining time is computed using wall-clock math: `configured_duration_s - elapsed_at_pause_s - (now_utc - started_at_utc).total_seconds()`. This requires no periodic writes and is accurate to Python `float` precision (~microseconds). State is written atomically (`os.replace()`) after every lifecycle event — no event returns a response before state is persisted. Expired-session detection on restart is a single subtraction; interrupted sessions are logged and discarded, not silently completed.

### III. Simplicity & YAGNI ✅

Classic Pomodoro cycle only (25/5/15). No custom durations. No background threads. JSON file for state (not SQLite). JSON Lines for history (not SQLite). Two thin adapter modules (not a plugin system). No configuration file for this feature.

### IV. Local-First & Privacy ✅

`timer_state.json` and `session_history.jsonl` written to platform-specific local directory (`~/Library/Application Support/pomodoro-mcp/` on macOS, `~/.local/share/pomodoro-mcp/` on Linux). Zero network calls at runtime. `tzlocal` reads `/etc/localtime` — filesystem only.

### V. Observability ✅

Every lifecycle event (`start`, `pause`, `resume`, `complete`, `cancel`, `skip`, `interrupt`) emits a structured log entry to **stderr** containing: UTC timestamp, session ID, event type, remaining/elapsed duration. MCP resources `pomodoro://status` and `pomodoro://history/{date}` expose current state and queryable history. No stdout pollution.

**Complexity Tracking**: No violations — table omitted per template instruction.

---

## Project Structure

### Documentation (this feature)

```text
specs/001-pomodoro-timer/
├── spec.md              # Feature specification
├── plan.md              # This file
├── research.md          # Phase 0 research findings
├── data-model.md        # Phase 1 entity definitions
├── quickstart.md        # Phase 1 developer runbook
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── contracts/
    ├── mcp-tools.md     # MCP tool + resource schemas
    └── rest-api.md      # REST endpoint contracts
```

### Source Code (repository root)

```text
pomodoro-mcp-app/
├── pyproject.toml               # Project config, dependencies, entry points
├── uv.lock                      # Dependency lockfile
│
└── src/
    └── pomodoro/
        ├── __init__.py
        │
        ├── models/
        │   └── session.py       # Pydantic models: Session, TimerState, SessionRecord,
        │                        #   StatusResponse, HistoryResponse
        │
        ├── services/
        │   ├── timer_service.py # Core business logic: start/pause/resume/cancel/skip,
        │   │                    #   break-type determination, midnight reset, restart recovery
        │   ├── history_service.py # Session history: append, query by date range & type
        │   └── timezone_utils.py  # get_local_tz(), get_utc_start_of_today(),
        │                           #   get_today_local_date(), has_day_changed()
        │
        ├── persistence/
        │   ├── state_store.py   # Atomic JSON read/write for TimerState
        │   └── history_store.py # JSONL append + startup recovery for SessionRecord
        │
        ├── mcp_server.py        # FastMCP instance; @mcp.tool() and @mcp.resource()
        │                        #   definitions; delegates to TimerService/HistoryService
        ├── rest_server.py       # FastAPI app; route definitions; delegates to same services
        └── asgi.py              # Combined ASGI app: REST routes + MCP mounted at /mcp

tests/
├── unit/
│   ├── test_timer_service.py    # Business logic (no I/O — uses in-memory stubs)
│   ├── test_history_service.py
│   ├── test_state_store.py      # Atomic write, corrupt-file recovery
│   ├── test_history_store.py    # Append, malformed-line recovery
│   └── test_timezone_utils.py  # Midnight boundary, DST edge cases
├── integration/
│   ├── test_mcp_tools.py        # FastMCP test client; full lifecycle flows
│   └── test_rest_endpoints.py  # httpx + FastAPI test client; full lifecycle flows
└── contract/
    └── test_schema_parity.py    # Verify MCP tool output shapes == REST response shapes
```

**Structure Decision**: Single project (`src` layout), Option 1. No monorepo split — MCP server and REST server are co-located and share the same service layer in-process. The `asgi.py` entry point mounts both for HTTP deployment; `mcp_server.py` is the entry point for stdio (Claude Desktop).

---

## Phase 0: Research Summary

See [`research.md`](./research.md) for full findings. Key decisions:

| Decision | Chosen | Rationale |
|---|---|---|
| FastMCP version | `>=2.0` (latest stable) | v3.0 not released; 2.x fully capable; architecture is forward-compatible |
| Dual-transport pattern | Shared `TimerService`; separate MCP + REST adapters; single ASGI mount for HTTP | Service-first per Principle I; no logic duplication |
| Timer state format | Atomic JSON file (`os.replace()`) | Single record; human-readable; stdlib-only; crash-safe |
| Remaining time calculation | Wall-clock math from `started_at_utc` + `elapsed_at_pause_s` | No periodic writes; accurate across pauses; expired-session check is trivial |
| History format | JSON Lines (`.jsonl`), `fsync` on each append | Crash-safe; O(n) scan within SC-006 limit at projected scale; YAGNI vs SQLite |
| Data directory | XDG/macOS-native per platform | Standard convention; survives upgrades |
| Timezone | `tzlocal>=4.0` + `tzdata` | Named IANA zone; DST-aware; no network; no paid services |
| Python version | 3.11+ | FastMCP requirement; `datetime.UTC`; best stdlib support |

---

## Phase 1: Design Artifacts

See:
- [`data-model.md`](./data-model.md) — entity definitions, state transitions, persistence schemas
- [`contracts/mcp-tools.md`](./contracts/mcp-tools.md) — MCP tool and resource schemas
- [`contracts/rest-api.md`](./contracts/rest-api.md) — REST endpoint contracts
- [`quickstart.md`](./quickstart.md) — install, run, test, integrate with Claude Desktop

### Key Design Decisions

**No background timer thread**: The server does not tick in the background. `remaining_s` is computed on every read using `(now_utc - started_at_utc).total_seconds()`. This is accurate to ±1 second for all practical purposes and eliminates threading complexity entirely (Principle III).

**Pending-break state**: After a work interval completes, the system enters `status = "pending-break"`. The user must explicitly call `start_break` or `start_work` (which implicitly skips the break). This state is persisted so it survives a restart.

**Implicit break skip**: Calling `start_work` while in `pending-break` state automatically records a `skipped` `SessionRecord` with `actual_duration_s = 0`, then starts the new work session. No separate `skip_break` call is required (FR-007).

**Midnight reset**: The `set_position` counter is reset to 0 whenever `today_local_date > last_reset_date`. This check occurs on every lifecycle event (not on a timer). The reset is atomic — it is written to `timer_state.json` alongside the next state update.

**MCP + REST response parity**: Tool output dicts and REST response JSON are produced by the same Pydantic model `.model_dump()`. Contract tests (`tests/contract/test_schema_parity.py`) enforce this parity automatically.

**Structured logging to stderr**: Every lifecycle event calls a `log_event()` helper that writes a JSON line to `sys.stderr`. Format: `{"ts": "...", "session_id": "...", "event": "start", "remaining_s": 1500.0}`.
