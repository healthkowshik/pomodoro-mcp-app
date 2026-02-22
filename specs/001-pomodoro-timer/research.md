# Research: Pomodoro Timer with Work/Break Cycles and Session Tracking

**Phase**: 0 — Research & Decision
**Date**: 2026-02-22
**Branch**: `001-pomodoro-timer`

---

## Decision 1: FastMCP Version & Dual-Transport Architecture

**Decision**: Use `fastmcp>=3.0` — FastMCP 3.0.1 is the current stable release on PyPI (verified 2026-02-22).

**Rationale**: FastMCP v3.0.1 is available. The constitution's "FastMCP v3.0" reference is accurate. The v3.x series provides all required capabilities: MCP tools (`@mcp.tool()`), MCP resources (`@mcp.resource()`), stdio transport for Claude Desktop, and Streamable HTTP transport for HTTP clients. The service-first architecture (shared `TimerService`, thin MCP and REST adapters) is version-agnostic across 2.x and 3.x.

**Dual-transport pattern**: FastMCP does NOT auto-generate REST endpoints from tool definitions. The REST layer is a separate FastAPI application. Both call the same `TimerService` from the shared service layer. The two can be:
- Run as **separate processes** (stdio MCP + `uvicorn` REST) — simplest for local deployment.
- Mounted in a **single ASGI process** (`rest_app.mount("/mcp", mcp.http_app())`) — one `uvicorn` invocation serves both.

**Chosen approach**: Single ASGI process for the HTTP-served mode (reduces deployment complexity); stdio remains the primary Claude Desktop transport and runs as its own process entry point.

**Alternatives considered**:
- SSE transport — being deprecated in the MCP spec; not used.
- GraphQL instead of REST — YAGNI; REST is the simplest documented contract surface.

---

## Decision 2: Timer State Persistence Format

**Decision**: Single JSON file, fully rewritten on every lifecycle event using an atomic write-to-temp-then-`os.replace()` pattern.

**Rationale**:
- Timer state is a single small record (~300 bytes). There is no benefit to SQLite's query capability for a one-row table.
- Atomic rename (`os.replace()`) guarantees readers see either the previous complete state or the new complete state — never a partial write.
- JSON is human-readable and stdlib-only (no added dependencies).
- The file is written exactly once per lifecycle event (not periodically), so write amplification is negligible.

**State schema** (fields stored):

```json
{
  "session_id": "uuid4",
  "session_type": "work | short-break | long-break | null",
  "status": "active | paused | null",
  "configured_duration_s": 1500,
  "started_at_utc": "2026-02-22T10:30:00.000000+00:00",
  "elapsed_at_pause_s": 0.0,
  "paused_at_utc": null,
  "set_position": 2,
  "last_reset_date": "2026-02-22"
}
```

**Remaining time formula**:
- Active: `configured_duration_s - elapsed_at_pause_s - (now_utc - started_at_utc).total_seconds()`
- Paused: `configured_duration_s - elapsed_at_pause_s`

No periodic writes required — wall-clock math reconstructs remaining time from stored timestamps on any read.

**Alternatives considered**:
- SQLite — correct for session history (though JSONL wins there too), unnecessary overhead for single-row hot state.
- JSON Lines for state — appends are not atomic; unsuitable for a record that is always fully replaced.
- Pickle — binary, version-fragile, security risk; rejected.

---

## Decision 3: Session History Storage Format

**Decision**: JSON Lines (`.jsonl`) — one JSON object per line, appended per completed or skipped/interrupted session. Crash-safe via `fsync()` after each append; malformed last line is silently discarded on startup.

**Rationale**:
- Single-user history grows at ~12 records/day maximum. At 200 bytes/record, 10 years of data is ~8.8 MB — an O(n) scan is well under the 1-second query limit (SC-006).
- Appending a single newline-terminated line is crash-safe: a partial write only corrupts the last (incomplete) line, which is detected and discarded on next read.
- JSONL is human-readable, inspectable with `grep` or `jq`, and requires only stdlib.
- Upgrade path to SQLite is straightforward if query volume ever warrants it (YAGNI deferral).

**Alternatives considered**:
- Plain JSON array — entire file must be rewritten on each append; not atomic; rejected.
- SQLite — all strengths (indices, transactions) are irrelevant at this scale; YAGNI.
- CSV — weaker type support; harder to extend schema; rejected.

---

## Decision 4: Data Directory Location

**Decision**: Platform-specific local data directory following XDG Base Directory Specification with macOS fallback.

| Platform | Path |
|---|---|
| macOS | `~/Library/Application Support/pomodoro-mcp/` |
| Linux (XDG set) | `$XDG_DATA_HOME/pomodoro-mcp/` |
| Linux (XDG not set) | `~/.local/share/pomodoro-mcp/` |

Files within that directory:
- `timer_state.json` — live timer state (hot path)
- `session_history.jsonl` — append-only completed session log

**Rationale**: macOS `~/Library/Application Support` is the OS-conventional location for durable app data (backed up by Time Machine, excluded from iCloud sync by default). Linux XDG compliance is the standard expectation for CLI/service tools.

---

## Decision 5: Local Timezone Detection

**Decision**: `tzlocal>=4.0` + `tzdata>=2024.1`. Cache the result at process startup.

**Rationale**:
- `datetime.datetime.now().astimezone().tzinfo` returns a fixed-offset object without DST transition knowledge — insufficient for reliable midnight boundary detection across the year.
- `zoneinfo` stdlib (Python 3.9+) requires knowing the IANA zone name; it does not auto-detect it.
- `tzlocal` reads `/etc/localtime` (macOS/Linux) to derive the IANA zone name and returns a `zoneinfo.ZoneInfo` object — full DST awareness, no network calls, no paid services (MIT licensed, ~10 KB).
- `tzdata` ships the IANA database as a pure-Python package for Linux systems that lack `/usr/share/zoneinfo`.

**Midnight reset logic**: Store `last_reset_date` (ISO date string) in the persisted state. On every lifecycle event, compare `today_local_date > last_reset_date`; if true, reset the set counter and update `last_reset_date`.

**Midnight query boundary**: Convert local midnight to UTC (`midnight_local.astimezone(utc)`) and use as the lower bound for filtering JSONL records by `started_at_utc`. Never filter on UTC date strings directly.

**Alternatives considered**:
- Raw `datetime.astimezone()` — fixed-offset only; DST-unsafe for general use.
- `pytz` — superseded by `zoneinfo`; `tzlocal` 4.x returns `zoneinfo` objects natively.
- User-configured timezone string — unnecessary complexity per YAGNI; system timezone is the correct default.

---

## Decision 6: Python Version

**Decision**: Python >= 3.11.

**Rationale**: FastMCP 2.x requires >= 3.10. Python 3.11 adds `datetime.UTC` constant, improved asyncio, and better error messages. `zoneinfo` is available from 3.9+. Targeting 3.11 gives the best performance and stdlib completeness with no downside for a new project.

---

## Decision 7: Testing Stack

**Decision**: `pytest` + `pytest-asyncio` (for any async FastMCP/FastAPI paths) + `httpx` (for REST endpoint integration tests against a live FastAPI test client).

**Test categories**:
- **Unit**: `TimerService`, `StateStore`, `HistoryStore`, `timezone_utils` — pure logic, no I/O.
- **Integration**: MCP tool calls via FastMCP's built-in test client; REST endpoint calls via `httpx.AsyncClient`.
- **Contract**: Verify MCP tool input/output schemas match REST endpoint request/response shapes for the same operations.

All tests runnable with a single command (`pytest`) from the repository root, satisfying the constitution's project default.
