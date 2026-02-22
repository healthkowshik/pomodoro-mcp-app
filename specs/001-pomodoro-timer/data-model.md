# Data Model: Pomodoro Timer

**Feature**: `001-pomodoro-timer`
**Date**: 2026-02-22

---

## Entities

### Session

A single timed interval — either a work interval or a break. The atomic unit of the Pomodoro system.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `session_id` | `str` (UUID4) | Required, unique | Immutable identifier assigned at session creation |
| `session_type` | `enum` | Required | One of: `work`, `short-break`, `long-break` |
| `status` | `enum` | Required | One of: `active`, `paused`, `completed`, `cancelled`, `skipped`, `interrupted` |
| `configured_duration_s` | `int` | Required, > 0 | Duration the session was configured to run (seconds). Work = 1500, short-break = 300, long-break = 900 |
| `actual_duration_s` | `float` | Required for terminal statuses | Actual elapsed duration at the time the session entered a terminal status. 0 for `skipped`. |
| `started_at_utc` | `datetime (UTC)` | Required on creation | UTC timestamp when the session was started or when a paused session was resumed |
| `elapsed_at_pause_s` | `float` | Default 0.0 | Cumulative elapsed seconds across all prior active periods (updated on each pause) |
| `paused_at_utc` | `datetime (UTC) \| null` | Null when active | UTC timestamp of the most recent pause event |
| `ended_at_utc` | `datetime (UTC) \| null` | Set on terminal status | UTC timestamp of completion, cancellation, skip, or interruption |

**Status lifecycle**:
```
idle → [start] → active → [complete] → completed (terminal)
                         → [pause]   → paused → [resume] → active
                                               → [cancel] → cancelled (terminal)
                → [cancel] → cancelled (terminal)
                → [interrupt on restart] → interrupted (terminal)
pending-break → [start break] → active (break session)
             → [skip break / start work] → skipped (terminal, actual_duration_s = 0)
```

**Computed field (not persisted, derived on read)**:
- `remaining_s`:
  - If `active`: `configured_duration_s - elapsed_at_pause_s - (now_utc - started_at_utc).total_seconds()`
  - If `paused`: `configured_duration_s - elapsed_at_pause_s`
  - If terminal: `0`

---

### SessionSet

Tracks the position within the current Pomodoro set (1–4 work intervals before a long break). Persisted as part of the timer state file, not as a standalone entity.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `completed_in_set` | `int` | 0–4 | Number of work intervals completed in the current set. Resets to 0 after a long break completes OR at midnight local time. |
| `last_reset_date` | `date (ISO 8601)` | Required | The local calendar date (`YYYY-MM-DD`) on which `completed_in_set` was last reset. Used to detect day rollover at midnight. |

**Break type determination**:
- `completed_in_set` ∈ {1, 2, 3} after a work interval → next break is `short-break` (300 s)
- `completed_in_set` = 4 after a work interval → next break is `long-break` (900 s); reset `completed_in_set` to 0

**Midnight reset**:
- On every lifecycle event, compare `today_local_date` > `last_reset_date`.
- If true: set `completed_in_set = 0`, update `last_reset_date = today_local_date`.
- An active session at midnight continues uninterrupted; only the counter resets.

---

### TimerState (persisted to `timer_state.json`)

The full mutable state of the server, written atomically on every lifecycle event.

| Field | Type | Description |
|---|---|---|
| `session_id` | `str \| null` | ID of the current session (null when idle) |
| `session_type` | `enum \| null` | Type of the current session (null when idle) |
| `status` | `enum \| null` | Status of the current session (null when idle). `"pending-break"` indicates work ended but break not yet started. |
| `configured_duration_s` | `int \| null` | Configured duration of the current session |
| `started_at_utc` | `str \| null` | ISO 8601 UTC string of session start / last resume |
| `elapsed_at_pause_s` | `float` | Cumulative elapsed seconds across all prior active periods |
| `paused_at_utc` | `str \| null` | ISO 8601 UTC string of last pause (null if active or idle) |
| `set_position` | `int` | `completed_in_set` value (0–4) |
| `last_reset_date` | `str` | ISO date string `"YYYY-MM-DD"` of last midnight reset |
| `pending_break_type` | `enum \| null` | `"short-break"` or `"long-break"` when status is `"pending-break"`; null otherwise |

**Null state** (idle, no session):
```json
{
  "session_id": null,
  "session_type": null,
  "status": null,
  "configured_duration_s": null,
  "started_at_utc": null,
  "elapsed_at_pause_s": 0.0,
  "paused_at_utc": null,
  "set_position": 0,
  "last_reset_date": "2026-02-22",
  "pending_break_type": null
}
```

---

### SessionRecord (persisted to `session_history.jsonl`)

An immutable log entry written for every session that reaches a terminal status (completed, cancelled, skipped, interrupted). One JSON object per line.

| Field | Type | Description |
|---|---|---|
| `session_id` | `str` | UUID4, matches the Session entity |
| `session_type` | `enum` | `work`, `short-break`, `long-break` |
| `status` | `enum` | Terminal status: `completed`, `cancelled`, `skipped`, `interrupted` |
| `configured_duration_s` | `int` | Configured duration in seconds |
| `actual_duration_s` | `float` | Elapsed seconds at terminal event. 0 for `skipped`. |
| `started_at_utc` | `str` | ISO 8601 UTC string |
| `ended_at_utc` | `str` | ISO 8601 UTC string |
| `set_position_at_end` | `int` | Value of `completed_in_set` at the time this record was written |

**Example (completed work interval)**:
```json
{"session_id":"a1b2c3d4-...","session_type":"work","status":"completed","configured_duration_s":1500,"actual_duration_s":1500.0,"started_at_utc":"2026-02-22T10:30:00.000000+00:00","ended_at_utc":"2026-02-22T10:55:00.000000+00:00","set_position_at_end":1}
```

**Example (skipped short break)**:
```json
{"session_id":"e5f6g7h8-...","session_type":"short-break","status":"skipped","configured_duration_s":300,"actual_duration_s":0.0,"started_at_utc":"2026-02-22T10:55:00.000000+00:00","ended_at_utc":"2026-02-22T10:55:00.000000+00:00","set_position_at_end":1}
```

---

## State Transitions Summary

```
IDLE
  │  start_work()
  ▼
ACTIVE (work)
  │  elapsed == configured_duration_s
  ▼
PENDING-BREAK (work completed, break not yet started)
  │  start_break()           │  start_work() [implicit skip]
  ▼                          ▼
ACTIVE (break)             SKIP recorded → ACTIVE (work)
  │  elapsed == configured   │  cancel()
  ▼                          ▼
IDLE (ready for next)     CANCELLED → IDLE

ACTIVE (any) → pause() → PAUSED → resume() → ACTIVE
ACTIVE (any) → cancel() → CANCELLED → IDLE
PAUSED (any) → cancel() → CANCELLED → IDLE
ACTIVE (on restart, expired) → INTERRUPTED → IDLE
```

---

## History Query Filters

Supported query dimensions for `SessionRecord` lookups:

| Filter | Type | Behaviour |
|---|---|---|
| Default (no params) | — | All records from 00:00 today in local timezone (UTC boundary computed at query time) |
| `from_date` | ISO date `YYYY-MM-DD` | Records with `started_at_utc >= midnight(from_date, local_tz)` |
| `to_date` | ISO date `YYYY-MM-DD` | Records with `started_at_utc < midnight(to_date + 1 day, local_tz)` |
| `session_type` | enum | Filter by `work`, `short-break`, or `long-break` |
| `status` | enum | Filter by `completed`, `cancelled`, `skipped`, `interrupted` |
