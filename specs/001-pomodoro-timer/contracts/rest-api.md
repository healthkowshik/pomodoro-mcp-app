# REST API Contract: Pomodoro Timer

**Feature**: `001-pomodoro-timer`
**Base URL**: `http://localhost:8000` (local deployment default)
**Date**: 2026-02-22

All endpoints delegate to `TimerService`. No business logic lives in route handlers. Response shapes are identical to MCP tool outputs for the same operation.

---

## Session Actions

### `POST /sessions/work`

Start a 25-minute Pomodoro work session.

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "active",
  "configured_duration_s": 1500,
  "remaining_s": 1500.0,
  "set_position": 1,
  "message": "Work session started. 1 of 4 in current set."
}
```

**Response `409 Conflict`** (session already active):
```json
{
  "error": true,
  "code": "SESSION_ALREADY_ACTIVE",
  "message": "A session is already running.",
  "current_status": { "...": "same shape as GET /sessions/current" }
}
```

---

### `POST /sessions/pause`

Pause the currently active session.

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "paused",
  "remaining_s": 1234.5,
  "message": "Session paused. 20m 34s remaining."
}
```

**Response `409 Conflict`**: `NO_ACTIVE_SESSION` or `SESSION_ALREADY_PAUSED`

---

### `POST /sessions/resume`

Resume a paused session.

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "active",
  "remaining_s": 1234.5,
  "message": "Session resumed. 20m 34s remaining."
}
```

**Response `409 Conflict`**: `NO_PAUSED_SESSION`

---

### `POST /sessions/cancel`

Cancel the current session (active or paused).

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "cancelled",
  "actual_duration_s": 265.3,
  "message": "Session cancelled after 4m 25s."
}
```

**Response `409 Conflict`**: `NO_SESSION`

---

### `POST /sessions/break`

Start the pending break.

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "short-break",
  "status": "active",
  "configured_duration_s": 300,
  "remaining_s": 300.0,
  "message": "Short break started. Take 5 minutes."
}
```

**Response `409 Conflict`**: `NO_PENDING_BREAK`

---

### `POST /sessions/break/skip`

Skip the pending or running break.

**Request body**: None

**Response `200 OK`**:
```json
{
  "session_id": "uuid4",
  "session_type": "short-break",
  "status": "skipped",
  "actual_duration_s": 0.0,
  "message": "Break skipped. Ready to start next work session."
}
```

**Response `409 Conflict`**: `NO_BREAK_TO_SKIP`

---

## Session Queries

### `GET /sessions/current`

Retrieve the current timer state.

**Response `200 OK`** (same shape as `pomodoro://status` MCP resource):
```json
{
  "status": "active",
  "session_type": "work",
  "session_id": "uuid4",
  "remaining_s": 1234.5,
  "elapsed_s": 265.5,
  "configured_duration_s": 1500,
  "set_position": 2,
  "max_set_position": 4,
  "pending_break_type": null,
  "message": "Work session 2/4 in progress. 20m 34s remaining."
}
```

---

### `GET /sessions/history`

Retrieve session history.

**Query parameters**:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `from_date` | `YYYY-MM-DD` | today | Start of date range (inclusive, local timezone) |
| `to_date` | `YYYY-MM-DD` | today | End of date range (inclusive, local timezone) |
| `session_type` | `work \| short-break \| long-break` | all | Filter by session type |
| `status` | `completed \| cancelled \| skipped \| interrupted` | all | Filter by terminal status |

**Response `200 OK`** (same shape as `pomodoro://history/{date}` MCP resource):
```json
{
  "query_from": "2026-02-22",
  "query_to": "2026-02-22",
  "local_timezone": "America/New_York",
  "total_sessions": 5,
  "completed_work_intervals": 3,
  "sessions": [
    {
      "session_id": "uuid4",
      "session_type": "work",
      "status": "completed",
      "configured_duration_s": 1500,
      "actual_duration_s": 1500.0,
      "started_at_utc": "2026-02-22T15:00:00+00:00",
      "ended_at_utc": "2026-02-22T15:25:00+00:00"
    }
  ]
}
```

---

## Shared Error Response Shape

```json
{
  "error": true,
  "code": "NO_ACTIVE_SESSION",
  "message": "No session is currently active."
}
```

HTTP status codes:
- `200 OK` — operation succeeded
- `409 Conflict` — operation not valid given current state (not a client input error)
- `500 Internal Server Error` — unexpected server fault (logged; client receives generic message)
