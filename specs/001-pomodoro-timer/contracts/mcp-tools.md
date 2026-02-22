# MCP Tool Contracts: Pomodoro Timer

**Feature**: `001-pomodoro-timer`
**Transport**: stdio (Claude Desktop), Streamable HTTP
**Date**: 2026-02-22

All tools delegate to `TimerService`. No business logic lives in tool definitions. Inputs are validated by Pydantic before reaching the service layer.

---

## Tools (State-Mutating Actions)

### `start_work`

Start a 25-minute Pomodoro work session.

**Input**: None

**Output**:
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

**Errors**:
- `SESSION_ALREADY_ACTIVE`: A session is currently running. Returns current status.

---

### `pause_session`

Pause the currently active session (work or break).

**Input**: None

**Output**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "paused",
  "remaining_s": 1234.5,
  "message": "Session paused. 20m 34s remaining."
}
```

**Errors**:
- `NO_ACTIVE_SESSION`: No session is running.
- `SESSION_ALREADY_PAUSED`: Session is already paused.

---

### `resume_session`

Resume a paused session from where it was paused.

**Input**: None

**Output**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "active",
  "remaining_s": 1234.5,
  "message": "Session resumed. 20m 34s remaining."
}
```

**Errors**:
- `NO_PAUSED_SESSION`: No session is paused.

---

### `cancel_session`

Cancel the current session (active or paused). Not counted as completed.

**Input**: None

**Output**:
```json
{
  "session_id": "uuid4",
  "session_type": "work",
  "status": "cancelled",
  "actual_duration_s": 265.3,
  "message": "Session cancelled after 4m 25s."
}
```

**Errors**:
- `NO_SESSION`: Nothing to cancel.

---

### `start_break`

Start the pending break (short or long) after a work interval completes.

**Input**: None

**Output**:
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

**Errors**:
- `NO_PENDING_BREAK`: No break is due (work session hasn't completed or break already started).

---

### `skip_break`

Skip the pending or running break and return to idle (ready for next work interval).

**Input**: None

**Output**:
```json
{
  "session_id": "uuid4",
  "session_type": "short-break",
  "status": "skipped",
  "actual_duration_s": 0.0,
  "message": "Break skipped. Ready to start next work session."
}
```

**Errors**:
- `NO_BREAK_TO_SKIP`: No pending or active break exists.

---

## Resources (Read-Only Data)

### `pomodoro://status`

Current timer state.

**URI**: `pomodoro://status`
**MIME type**: `application/json`

**Response (active session)**:
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
  "message": "Work session 2/4 in progress. 20m 34s remaining."
}
```

**Response (pending break)**:
```json
{
  "status": "pending-break",
  "session_type": null,
  "session_id": null,
  "remaining_s": null,
  "set_position": 3,
  "pending_break_type": "short-break",
  "pending_break_duration_s": 300,
  "message": "Work interval complete. Short break (5 min) is due."
}
```

**Response (idle)**:
```json
{
  "status": "idle",
  "session_type": null,
  "session_id": null,
  "remaining_s": null,
  "set_position": 0,
  "pending_break_type": null,
  "message": "No active session. Ready to start."
}
```

---

### `pomodoro://history/{date}`

Session history for a given date or date range.

**URI template**: `pomodoro://history/{date}`

**URI values**:
- `today` → sessions from 00:00 local time today
- `YYYY-MM-DD` → sessions from 00:00 local time on that date
- `YYYY-MM-DD/YYYY-MM-DD` → sessions in date range (inclusive)

**MIME type**: `application/json`

**Response**:
```json
{
  "query_date": "2026-02-22",
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

## Error Response Shape (all tools)

```json
{
  "error": true,
  "code": "NO_ACTIVE_SESSION",
  "message": "No session is currently active."
}
```

All error codes are uppercase snake-case strings. Each tool section above lists its possible error codes.
