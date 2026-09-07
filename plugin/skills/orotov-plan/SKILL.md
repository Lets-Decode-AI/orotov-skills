---
name: orotov-plan
description: Use when clarified requirements need a structured execution plan with testable tasks.
metadata:
  version: "0.2.0"
---

# OROTOV: Execution Planning

Convert clarified requirements from orotov-discover into a detailed, actionable execution plan with clear phases, tasks, and test-driven specifications.

## When to Use

Dispatched by `orotov-scope` after `orotov-discover` completes. Receives clarified requirements and produces a structured plan ready for issue extraction.

## Output Format

The plan follows this structure to ensure clarity, testability, and actionability:

```markdown
# [Feature Name] Execution Plan

## Goal & Success Criteria

[What done looks like — statement of what success means]

Acceptance criteria (user-facing outcomes):
- [Criterion 1]
- [Criterion 2]
- [Criterion 3]

## Architecture

[Design decisions, patterns, technology choices, data models, interface contracts]

## Phases

[High-level breakdown of work into sequential and/or parallel phases]

### Phase 1: [Story Name]

[Overview of what this phase accomplishes and why it comes first]

#### Tasks (Each becomes an Issue)

- **Task 1: [Issue Title]**
  - Description: [What to build, what user sees/does]
  - Acceptance Criteria: [How to know it's done, edge cases covered]
  - Test Spec: [TDD requirements — tests first, exact assertions]
  - Dependencies: [What must be done first]

- **Task 2: [Issue Title]**
  - Description: [What to build]
  - Acceptance Criteria: [How to know it's done]
  - Test Spec: [TDD requirements]
  - Dependencies: [Task 1]

### Phase 2: [Story Name]

[Overview of what this phase accomplishes]

#### Tasks

- **Task 3: [Issue Title]**
  - Description: [What to build]
  - Acceptance Criteria: [How to know it's done]
  - Test Spec: [TDD requirements]
  - Dependencies: [Task 1, Task 2]

...

## Critical Path

[Which tasks must be sequential, which can run in parallel]
```

## Example Output

Here is a complete, realistic example plan for a Real-Time Notifications feature:

```markdown
# Real-Time Notifications Execution Plan

## Goal & Success Criteria

Build a WebSocket-based real-time notification system that allows agents to receive server-sent updates without polling. Agents connect once, authenticate with JWT, and receive events in real-time with automatic reconnection on network failure.

Acceptance criteria:
- Agents can establish WebSocket connection and receive "connected" event
- Agents authenticate with JWT token; invalid tokens are rejected with 401
- System broadcasts events to connected agents in real-time (< 100ms latency)
- Agents automatically reconnect on network failure with exponential backoff
- Notifications include timestamp, type, payload; frontend displays them in notification center
- System handles 1000+ concurrent connections without dropped events
- Connection state is stored in database for agent replay on reconnect

## Architecture

- **Backend:** WebSocket server (Python websockets library, async)
- **Data Model:** 
  - `WebSocketConnection` (agent_id, session_id, connected_at, last_heartbeat)
  - `Notification` (id, type, agent_id, payload, created_at, delivered_at)
- **Frontend:** JavaScript WebSocket client with reconnection logic
- **Authentication:** JWT token in connection query param, verified on handshake
- **Events:** Event types = "task_assigned", "task_completed", "agent_updated", "error"
- **Storage:** Connection state in Redis for fast lookup; notification history in Postgres

## Phases

### Phase 1: WebSocket Server Setup

Build the core WebSocket server infrastructure with authentication and basic message routing.

#### Tasks

- **Task 1: Create WebSocket server endpoint**
  - Description: Implement WebSocket endpoint at `/ws` that accepts connections. Server should verify JWT token in query param and reject with 401 if invalid. Accept valid connections and send back a "connected" event with session ID.
  - Acceptance Criteria:
    - WebSocket endpoint `/ws` listens on port 8000 (or configured port)
    - Client connects with query param `?token=<jwt>`
    - Valid JWT → connection accepted, client receives `{"type": "connected", "session_id": "<uuid>"}`
    - Invalid JWT → connection rejected with 401 Unauthorized
    - Expired JWT → connection rejected with 401
    - Missing token → connection rejected with 401
    - Server logs connection/disconnection events with agent_id and timestamp
  - Test Spec:
    - Test: Connect with valid JWT, assert "connected" event received and session_id is UUID
    - Test: Connect with invalid JWT, assert 401 error
    - Test: Connect with expired JWT, assert 401 error
    - Test: Connect without token, assert 401 error
    - Test: Send invalid JSON to WebSocket, assert server doesn't crash and sends error event
    - Test: Disconnect client, assert server cleans up connection
  - Dependencies: None

- **Task 2: Implement connection state tracking in Redis**
  - Description: When a client connects, store the connection metadata (agent_id, session_id, connected_at, last_heartbeat). When client disconnects, remove it from Redis. Implement a heartbeat mechanism where server sends ping every 30 seconds and expects pong response within 10 seconds, or marks connection as stale.
  - Acceptance Criteria:
    - Connection stored in Redis with key `connection:{session_id}`
    - Redis key contains: agent_id, session_id, connected_at, last_heartbeat (timestamp)
    - Server sends ping event every 30 seconds
    - Client expected to respond with pong event within 10 seconds
    - If no pong received, connection marked as stale and closed after 5 seconds
    - On client disconnect, Redis key deleted
    - Stale connections cleaned up automatically
  - Test Spec:
    - Test: Connect, verify Redis entry created with correct data
    - Test: Receive ping, send pong, verify last_heartbeat updated
    - Test: Receive ping, don't send pong, verify connection closed after 15 seconds
    - Test: Disconnect, verify Redis key deleted
    - Test: Connect 100 clients simultaneously, verify all stored in Redis
  - Dependencies: Task 1

- **Task 3: Add persistent connection state to Postgres**
  - Description: Create a `WebSocketConnection` table to persist connection history and support replay on reconnect. On each connection, insert a record. On disconnect, update disconnected_at. Implement a query to fetch all active connections for a given agent.
  - Acceptance Criteria:
    - Table: `connections` (id, agent_id, session_id, connected_at, disconnected_at, last_event_id)
    - On connect: insert row with connected_at = now()
    - On disconnect: update disconnected_at = now()
    - Query: Get all active connections for agent_id (disconnected_at IS NULL)
    - Query: Get last_event_id for agent to support replay
    - Connections expire after 7 days (cleanup job)
  - Test Spec:
    - Test: Connect, verify Postgres record created with connected_at
    - Test: Disconnect, verify disconnected_at set
    - Test: Query active connections for agent, verify correct count
    - Test: Query last_event_id, verify correct ID returned
    - Test: Create 500 connections, query active count, assert correct number
  - Dependencies: Task 1

### Phase 2: Event Broadcasting & Notification System

Implement event publishing, broadcasting to connected agents, and notification persistence.

#### Tasks

- **Task 4: Create notification event system with typing**
  - Description: Define notification event types (task_assigned, task_completed, agent_updated, error) with TypeScript interfaces and validation. Implement event creation and storage in a Notification table. Events should include timestamp, type, payload, and target agent(s).
  - Acceptance Criteria:
    - Notification types defined: task_assigned, task_completed, agent_updated, error, system_alert
    - Each type has required payload schema (e.g., task_assigned requires {task_id, agent_id, priority})
    - Notification stored in Postgres with: id, type, payload (JSON), target_agent_id, created_at, status
    - Status enum: pending, delivered, failed, expired
    - Payload validated against schema on creation; invalid payload rejected
    - Query: Get all notifications for agent created in last 24 hours
  - Test Spec:
    - Test: Create task_assigned notification, assert payload has {task_id, agent_id, priority}
    - Test: Create notification with missing required field, assert validation error
    - Test: Create notification with invalid type, assert rejected
    - Test: Query notifications for agent, assert correct count and ordering (newest first)
    - Test: Notification expires after 7 days (cleanup)
  - Dependencies: Task 2, Task 3

- **Task 5: Implement broadcast to connected agents**
  - Description: When a notification is created, broadcast it to all connected WebSocket clients for the target agent. If agent not connected, mark as pending in database for replay on reconnect. Use Redis pub/sub to distribute events across multiple server instances (if scaled).
  - Acceptance Criteria:
    - Broadcast event sent to all active WebSocket connections for target_agent_id
    - Event format: `{"type": "<notification_type>", "id": "<uuid>", "payload": {...}, "timestamp": "<iso8601>"}`
    - If agent offline, notification marked as pending in database
    - Pending notifications delivered when agent reconnects
    - Redis pub/sub used for cross-instance broadcasting (optional in Phase 2)
    - Delivery confirmed: status updated to "delivered" after sent
  - Test Spec:
    - Test: Create notification, broadcast to connected agent, assert event received
    - Test: Create notification for offline agent, assert marked as pending
    - Test: Reconnect offline agent, assert pending notifications delivered immediately
    - Test: Broadcast to 10 concurrent connections, assert all receive event
    - Test: Broadcast includes correct timestamp and id
  - Dependencies: Task 4

- **Task 6: Add notification delivery confirmation and logging**
  - Description: Implement delivery confirmation. After WebSocket client receives a notification, it sends back a confirmation with notification_id. Server updates notification.status to "delivered" and logs delivery time. Implement query to get delivery metrics (% delivered, avg delivery time).
  - Acceptance Criteria:
    - Client sends confirm event: `{"type": "confirm", "notification_id": "<id>"}`
    - Server updates notification.delivered_at and status to "delivered"
    - Notifications not confirmed after 5 minutes marked as "failed"
    - Query: Get delivery metrics per agent (total sent, delivered %, avg delivery time)
    - Logs include: notification_id, agent_id, delivery_time, status
  - Test Spec:
    - Test: Send notification, receive confirm, assert status = "delivered"
    - Test: Send notification, no confirm for 5 minutes, assert status = "failed"
    - Test: Query delivery metrics for agent, assert percentages correct
    - Test: Send 100 notifications, query delivery report
  - Dependencies: Task 5

### Phase 3: Client-Side Integration & Frontend Display

Implement the JavaScript WebSocket client and frontend notification UI.

#### Tasks

- **Task 7: Build JavaScript WebSocket client**
  - Description: Create a WebSocket client library (`ws-client.js`) that handles connection, authentication, reconnection with exponential backoff, and event subscription. Expose methods: `connect()`, `disconnect()`, `subscribe(event_type, callback)`, `confirm(notification_id)`. Handle connection failures gracefully.
  - Acceptance Criteria:
    - Client connects to `wss://<host>/ws?token=<jwt>`
    - Emits custom events for connection, disconnection, notification received, error
    - Reconnection with exponential backoff: 1s, 2s, 4s, 8s, max 30s
    - Max 5 reconnection attempts before fail
    - `subscribe()` method allows filtering by event type
    - `confirm()` sends confirmation back to server
    - Client stores last received notification_id for replay on reconnect
    - All errors logged to console (configurable)
  - Test Spec:
    - Test: Connect with token, assert "connected" event fired
    - Test: Receive notification, assert subscribed callback invoked
    - Test: Connection drops, assert reconnect attempts with backoff
    - Test: Max reconnect attempts exceeded, assert "failed" event
    - Test: Confirm notification, assert sent to server
    - Test: Receive message while offline, assert queued and sent on reconnect
  - Dependencies: Task 1

- **Task 8: Create notification UI component (React/Vue)**
  - Description: Build a notification center UI component that displays incoming notifications. Show notification type (icon + color), message, timestamp, and dismiss button. Implement bell icon with unread count. Add toast notifications for real-time display and persistent list for history.
  - Acceptance Criteria:
    - Bell icon in header shows unread count (updates in real-time)
    - Notification center modal shows list of recent notifications (last 24 hours)
    - Each notification displays: type (icon), message, timestamp (relative time, e.g., "2 mins ago")
    - Click notification → dismiss and mark as read
    - Toast notification appears top-right for new incoming events (2 sec auto-dismiss)
    - Notification colors: task_assigned=blue, completed=green, error=red, system_alert=yellow
    - Click "Clear all" → dismiss all notifications
    - Responsive design (mobile-friendly)
  - Test Spec:
    - Test: Receive notification, assert displayed in notification center
    - Test: Bell icon shows unread count, updates on receive
    - Test: Dismiss notification, assert removed from list and count decreases
    - Test: Toast shown for 2 seconds then auto-dismiss
    - Test: Open modal with 10 notifications, assert all visible and scrollable
    - Test: On mobile, notification center responsive
  - Dependencies: Task 7

- **Task 9: Implement notification preferences and filtering**
  - Description: Allow users to configure which notification types they want to receive and how (in-app only, email, SMS). Store preferences in user profile. Client subscribes only to enabled types. Backend respects preferences when broadcasting.
  - Acceptance Criteria:
    - Settings page: toggle for each notification type (task_assigned, task_completed, agent_updated, error, system_alert)
    - Settings stored in user profile (Postgres)
    - Client queries preferences on connect, subscribes only to enabled types
    - Backend queries user preferences before broadcasting
    - Email/SMS channel support (infrastructure setup)
    - Default: all types enabled for new users
  - Test Spec:
    - Test: User disables task_assigned, send notification, assert not received
    - Test: User enables task_completed, send notification, assert received
    - Test: Update preferences, assert client re-subscribes to correct types
    - Test: Query 1000 users' preferences, performance < 500ms
  - Dependencies: Task 8

## Critical Path

**Sequential (must be in order):**
1. Task 1: WebSocket server endpoint (foundation for all others)
2. Task 2: Redis connection tracking (required before Task 3 and Task 4)
3. Task 4: Notification event types (required before Task 5)
4. Task 5: Broadcast system (required before Task 7)
5. Task 7: JavaScript client (required before Task 8)
6. Task 8: UI component (required before Task 9)

**Parallel (can run together):**
- Task 2 and Task 3 can run in parallel (both build on Task 1)
- Task 6 can start after Task 5 (delivery confirmation is optional enhancement)
- Task 9 can start after Task 8 (preferences are optional Phase 3 enhancement)

**Recommended Implementation Order:**
1. Task 1 → Task 2 → Task 4 → Task 5 → Task 7 → Task 8
2. (Parallel) Task 3, Task 6, Task 9 can follow or be deferred to Phase 3.5
```

## Process

The planning agent receives clarified requirements from orotov-discover and follows this process:

1. **Determine Structure**
   - If single story: write one phase with 2-5 tasks
   - If multi-phase: identify natural break points (e.g., infrastructure, feature core, UI, polish)
   - Consider dependencies: what must be sequential vs. parallel?

2. **Write Plan**
   - Use the structure shown in "Output Format" above
   - Write detailed task descriptions (what user sees/does, not how to implement)
   - Define acceptance criteria that cover happy path, edge cases, and failure modes
   - Write TDD specs (test names and assertions) — these will become actual tests

3. **Identify Dependencies**
   - Mark task dependencies clearly (Task 1 → Task 2)
   - Highlight critical path (sequential tasks that block others)
   - Note parallel opportunities (tasks that can run together)

4. **Persist the Plan to the Board**
   - Call the `mcp__orotov__create_planning` MCP tool to save the plan so it appears in the OROTOV UI's Planning view (the constitution requires planning documents to be persisted to the database):
     - `title`: the feature name (e.g. "Real-Time Notifications")
     - `plan_document`: the full plan markdown you wrote in step 2
     - `description` (optional): the one-line goal
     - `interview_transcript` (optional): the clarified requirements/Q&A from orotov-discover, if available
   - Capture the returned `id` (the planning document id) to include in your return.
   - If the `mcp__orotov__create_planning` tool is unavailable (no MCP connection), note that in your return and continue — do not fail the plan.

5. **Return Full Plan**
   - Format as markdown
   - Include all required sections (Goal, Architecture, Phases, Tasks, Critical Path)
   - Ensure tasks are 4-8 hour efforts (not 1-2 week epics, not 30-min tasks)
   - Ensure no ambiguity in acceptance criteria or test specs

## Return

Return the plan as follows:

```
## Execution Plan Created

[Full plan markdown here]

---

Planning document saved to the board (Planning #<id>).

Ready for issue extraction. This plan will be parsed into stories and issues by orotov-scope.
```

If `mcp__orotov__create_planning` was unavailable, replace the "saved to the board" line with a short note that the plan could not be persisted (MCP not connected) so the caller can save it manually.

## Key Principles

### Testability
Every task must have clear, testable acceptance criteria and a complete TDD spec. Someone reading the test spec should be able to implement the task by making the tests pass.

### Clarity
Someone reading the plan (without meeting the user) should understand exactly what to build, why, and how to know it's done. No ambiguity about scope or success.

### Granularity
Each task should be 4-8 hours of work for an experienced engineer. Not 1-2 week epics, not 30-minute tasks. If a task seems larger, break it into multiple tasks.

### Dependencies
Make explicit which tasks depend on which. Highlight what must be sequential vs. parallel to optimize timeline.

### No Ambiguity
Specify error cases, edge cases, and how to handle them. Don't say "handle edge cases" — list the edge cases.
