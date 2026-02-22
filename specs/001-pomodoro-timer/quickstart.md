# Quickstart: Pomodoro Timer

**Feature**: `001-pomodoro-timer`
**Date**: 2026-02-22

---

## Prerequisites

- Python >= 3.11
- [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`

---

## Install & Run

```bash
# Clone / enter the repo
cd pomodoro-mcp-app

# Install dependencies
uv sync

# Run all tests (must pass before any other step)
pytest

# --- Option A: MCP server (stdio, for Claude Desktop) ---
uv run pomodoro-mcp

# --- Option B: REST server (HTTP, for Postman / web clients) ---
uv run pomodoro-rest

# --- Option C: Both in a single ASGI process ---
uv run uvicorn pomodoro.asgi:app --host 127.0.0.1 --port 8000
# MCP Streamable HTTP served at http://localhost:8000/mcp
# REST API served at http://localhost:8000/sessions/...
```

---

## Claude Desktop Integration

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "pomodoro": {
      "command": "uv",
      "args": ["--directory", "/path/to/pomodoro-mcp-app", "run", "pomodoro-mcp"]
    }
  }
}
```

Restart Claude Desktop. The following tools become available:

| Tool | What it does |
|---|---|
| `start_work` | Start a 25-minute Pomodoro |
| `pause_session` | Pause the running timer |
| `resume_session` | Resume where you left off |
| `cancel_session` | Abandon the current session |
| `start_break` | Begin the pending break |
| `skip_break` | Skip the break and stay ready |

Resources available for reading:

| Resource URI | What it returns |
|---|---|
| `pomodoro://status` | Current timer state |
| `pomodoro://history/today` | Today's completed sessions |
| `pomodoro://history/YYYY-MM-DD` | Sessions for a specific date |

---

## REST API Quick Reference

```bash
# Start a work session
curl -X POST http://localhost:8000/sessions/work

# Check current status
curl http://localhost:8000/sessions/current

# Pause / resume / cancel
curl -X POST http://localhost:8000/sessions/pause
curl -X POST http://localhost:8000/sessions/resume
curl -X POST http://localhost:8000/sessions/cancel

# Start or skip the pending break
curl -X POST http://localhost:8000/sessions/break
curl -X POST http://localhost:8000/sessions/break/skip

# Today's history
curl "http://localhost:8000/sessions/history"

# History for a date range
curl "http://localhost:8000/sessions/history?from_date=2026-02-01&to_date=2026-02-22"

# Only completed work intervals
curl "http://localhost:8000/sessions/history?session_type=work&status=completed"
```

---

## Where Data Is Stored

| Platform | Path |
|---|---|
| macOS | `~/Library/Application Support/pomodoro-mcp/` |
| Linux | `$XDG_DATA_HOME/pomodoro-mcp/` or `~/.local/share/pomodoro-mcp/` |

Files:
- `timer_state.json` — live timer state (safe to inspect; do not edit while server is running)
- `session_history.jsonl` — append-only session log (one JSON object per line)

---

## Running Tests

```bash
# All tests
pytest

# Unit tests only (fast, no I/O)
pytest tests/unit/

# Integration tests (starts in-process test server)
pytest tests/integration/

# Contract tests (MCP ↔ REST schema parity)
pytest tests/contract/
```
