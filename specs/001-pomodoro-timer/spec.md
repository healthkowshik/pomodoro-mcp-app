# Feature Specification: Pomodoro Timer with Work/Break Cycles and Session Tracking

**Feature Branch**: `001-pomodoro-timer`
**Created**: 2026-02-22
**Status**: Draft
**Input**: User description: "Add a Pomodoro timer with work/break cycles and session tracking"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Start and Complete a Work Interval (Priority: P1)

A user wants to begin a focused work session. They start a Pomodoro and receive confirmation that a 25-minute timer is running. When the interval elapses, they are notified the work is complete and a break is due.

**Why this priority**: This is the atomic unit of the Pomodoro Technique. Without the ability to start and complete a single work interval, no other functionality has value. Every other story builds on this one.

**Independent Test**: Start a timer, simulate time elapsed to completion, and verify the completion notification and state transition to "break due" — delivers complete standalone MVP value.

**Acceptance Scenarios**:

1. **Given** no session is active, **When** a user starts a work session, **Then** a 25-minute countdown begins and the current state reports session type "work" with time remaining.
2. **Given** a work session is running, **When** the 25-minute interval elapses, **Then** the session is marked complete, the completed-work count within the current set increments by 1, and the system signals that a break is due.
3. **Given** a work session is running, **When** the user cancels it, **Then** the session is terminated, state is cleared, and the interval is not counted as completed.

---

### User Story 2 - Take a Break After Work Intervals (Priority: P2)

A user has completed a work interval and needs to take a break. The system determines whether to offer a short break (5 minutes) or a long break (15 minutes, offered after every 4 completed work intervals in a set).

**Why this priority**: The alternating work/break rhythm is the defining structure of the Pomodoro Technique. Without correct break cycles the feature is incomplete and cannot be used as a real productivity tool.

**Independent Test**: Complete work intervals (or simulate completions) and verify the correct break type is offered — short after the 1st, 2nd, and 3rd intervals in a set; long after the 4th.

**Acceptance Scenarios**:

1. **Given** the user has completed 1–3 work intervals in the current set, **When** a work interval ends, **Then** the system signals a 5-minute short break is due.
2. **Given** the user has completed 4 work intervals in the current set, **When** the 4th work interval ends, **Then** the system signals a 15-minute long break is due and the interval counter for the current set resets.
3. **Given** a break timer is running, **When** the break elapses, **Then** the user is notified the break is over and the system is ready to start the next work interval.
4. **Given** a break timer is running, **When** the user skips the break, **Then** the break is cancelled and the user can immediately start the next work interval.

---

### User Story 3 - Pause and Resume a Running Session (Priority: P3)

A user is interrupted mid-session and needs to pause the timer. They can resume later and pick up exactly where they left off, with no time lost.

**Why this priority**: Real-world interruptions are unavoidable. Pause/resume prevents users from losing progress and reduces the friction of adopting the technique.

**Independent Test**: Start a session, pause at a known elapsed time, wait an arbitrary period, resume, and verify the remaining time matches what it was at the moment of pause.

**Acceptance Scenarios**:

1. **Given** a session is active, **When** the user pauses it, **Then** the timer stops and the elapsed time at the moment of pause is preserved.
2. **Given** a session is paused, **When** the user resumes it, **Then** the countdown continues from the preserved remaining time.
3. **Given** a session is paused, **When** the user cancels it, **Then** the session is terminated and state is cleared without counting the interval as completed.

---

### User Story 4 - View Current Status and Session History (Priority: P4)

A user wants to know the current timer state at a glance and to review their completed sessions to assess their productivity.

**Why this priority**: Session tracking closes the feedback loop — users can see their progress and make decisions about their work patterns. Without it, the tool provides no reflection capability.

**Independent Test**: Complete one or more sessions and verify that querying status returns accurate current state, and querying history returns the completed sessions with correct timestamps and types.

**Acceptance Scenarios**:

1. **Given** a session is active, **When** the user queries current status, **Then** the response includes session type, time remaining, and position within the current set (e.g., "2 of 4 work intervals completed").
2. **Given** no session is active, **When** the user queries current status, **Then** the response clearly indicates idle state and shows how many intervals have been completed in the current set.
3. **Given** one or more work intervals have been completed, **When** the user queries session history, **Then** a list of completed sessions is returned with UTC timestamps, session types, and actual durations.
4. **Given** no sessions have been completed, **When** the user queries session history, **Then** an empty history is returned without error.

---

### Edge Cases

- What happens when the server is restarted mid-session and the session has not yet expired? Timer state must be restored from persisted storage so the remaining time is accurate to ±1 second.
- What happens when the server restarts and the persisted session has already passed its expiry (more wall-clock time elapsed than the session's remaining duration)? The session MUST be marked as cancelled/interrupted, not counted as completed, and the system MUST restore to idle state. A structured log entry MUST be emitted for this cancellation with event type "interrupted".
- What happens when the user tries to start a new session while one is already active? The system must reject the request and return a clear message describing the current session state.
- What happens when the user tries to resume when no session is paused? The system must return a clear error indicating there is nothing to resume.
- What happens when the user tries to skip a break when no break is running? The system must return an error indicating the operation is not applicable.
- What happens after a long break completes and the user queries set progress? The counter must reflect the start of a new set (0 intervals completed in the current set).
- What happens at midnight local time? The set counter MUST reset to 0. If a session is active at midnight, it continues uninterrupted; only the completed-interval count resets at the day boundary.
- What happens if two lifecycle events occur in rapid succession (e.g., pause then immediate resume)? Each event must be logged independently and state must remain consistent.
- What happens if the persisted state file is unreadable or corrupted on startup? The system MUST emit a structured warning log entry, discard the corrupted state, and start in idle state. The server MUST NOT refuse to start due to a corrupt state file.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Users MUST be able to start a work session with a default duration of 25 minutes.
- **FR-002**: Users MUST be able to pause a running session (work or break).
- **FR-003**: Users MUST be able to resume a paused session; elapsed time MUST be preserved exactly across the pause.
- **FR-004**: Users MUST be able to cancel an active or paused session at any time; cancelled sessions MUST NOT be counted as completed.
- **FR-005**: When a work interval completes, the system MUST automatically determine the appropriate next break: short break (5 minutes) after the 1st, 2nd, and 3rd intervals in a set; long break (15 minutes) after the 4th.
- **FR-006**: Users MUST be able to start a break session (short or long) after a work interval completes.
- **FR-007**: Users MUST be able to skip a pending or running break and proceed directly to the next work interval. Starting a work session while a break is pending MUST implicitly skip the break — no separate skip command is required. In both the explicit and implicit skip cases, the system MUST record the skipped break in session history with status "skipped" and an actual duration of 0.
- **FR-008**: Users MUST be able to query current session state at any time and receive: session type (work / short-break / long-break / idle), time remaining, and number of completed work intervals in the current set.
- **FR-009**: The system MUST persist timer state (session type, elapsed time, set position) to local storage after every lifecycle event so the state survives a server restart.
- **FR-010**: On startup, if persisted state exists, the system MUST check whether the elapsed wall-clock time since shutdown exceeds the session's remaining duration. If it does not, the session MUST be restored with remaining time accurate to ±1 second. If it does (session has expired while the server was down), the session MUST be discarded as cancelled/interrupted, a structured log entry MUST be emitted with event type "interrupted", and the system MUST start in idle state. If the persisted state file is unreadable or corrupted, the system MUST emit a structured warning log entry, discard the file, and start in idle state — the server MUST NOT refuse to start.
- **FR-011**: The system MUST record each completed session (work intervals and breaks) with a unique session ID, UTC timestamp, session type, configured duration, and actual duration.
- **FR-012**: Users MUST be able to query session history. By default the query returns all sessions from 00:00 of the current day in the user's local timezone. Users MUST also be able to request history for a wider period by specifying a date range. All session records MUST be retained indefinitely on the local filesystem to support arbitrary lookups.
- **FR-013**: The system MUST emit a structured log entry for every timer lifecycle event (start, pause, resume, complete, cancel). Each entry MUST include: UTC timestamp, session ID, event type, and remaining/elapsed duration at the time of the event.
- **FR-014**: The system MUST reject attempts to start a new session when one is already active, returning a message that includes the current session type and time remaining.

### Key Entities

- **Session**: A single timed interval — work or break. Carries: unique session ID, type (work / short-break / long-break), configured duration, elapsed time, status (active / paused / completed / cancelled / skipped / interrupted), and UTC timestamps for start and end events. Skipped breaks have an actual duration of 0; interrupted sessions (expired while server was offline) are not counted as completed.
- **SessionSet**: A group of up to 4 work intervals with interleaved short breaks, capped by a long break. Tracks the count of completed work intervals within the current set. The set counter resets to 0 at midnight in the user's local timezone each calendar day, regardless of whether a long break was taken.
- **SessionHistory**: The append-only, ordered record of all sessions. Supports querying by date range and session type.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Users can start a Pomodoro session within 2 seconds of issuing the start command.
- **SC-002**: Timer duration accuracy is within ±1 second of the configured interval for all session types (work, short break, long break).
- **SC-003**: After a server restart, timer state is restored within 3 seconds, with remaining time accurate to ±1 second of what it was at the moment of shutdown.
- **SC-004**: Users can retrieve current session status at any time and receive a response within 1 second.
- **SC-005**: 100% of timer lifecycle events (start, pause, resume, complete, cancel) produce a retrievable structured log entry with all required fields.
- **SC-006**: Session history is queryable immediately after a session completes — no delay or manual refresh required.
- **SC-007**: Users can complete a full Pomodoro set (4 work intervals + 3 short breaks + 1 long break) using only the available controls, without any workaround or manual state correction.

## Assumptions

- The default session cycle follows the classic Pomodoro Technique: 25-minute work intervals, 5-minute short breaks, and 15-minute long breaks after every 4 completed work intervals. Custom durations are out of scope for this feature.
- The app serves a single user on a local machine; multi-user and concurrent-session scenarios are out of scope.
- Both MCP tools and a REST API are first-class delivery targets as required by the project constitution; this spec makes no assumption about which transport any given user will use.
- Break transitions are user-initiated: the system signals when a work interval is complete but does not auto-start the break timer without user action.
- All timestamps are stored and displayed in UTC.
- All session data is stored exclusively on the local filesystem; no data is transmitted to any remote service.
- A "completed" work interval means the full configured duration elapsed; cancelled or paused-then-cancelled sessions do not count.
- A session that expires while the server is offline is treated as cancelled/interrupted, not completed.

## Clarifications

### Session 2026-02-22

- Q: If the server was down and the persisted session's remaining duration was exceeded by elapsed wall-clock time, what should the system do on restart? → A: Cancel/discard it — mark as interrupted, do not count it, restore to idle state.
- Q: When does the SessionSet counter (completed work intervals toward the next long break) reset? → A: At midnight in the user's local timezone each calendar day.
- Q: If the user issues a "start work" command while a break is pending (break due state), should it be accepted or rejected? → A: Accepted — the pending break is implicitly skipped and the skipped break MUST be recorded in session history with status "skipped" and actual duration of 0.
- Q: If the persisted state file is unreadable or corrupted at startup, should the server refuse to start or recover silently? → A: Emit a structured warning log, discard the corrupted state, and start in idle state — do not block startup.
