---
name: orotov-scope
description: Use when a feature, bug, or design needs clarification and decomposition into OROTOV board work.
---

# OROTOV: Scope & Plan

Turn a natural-language work description into well-scoped stories and issues in the
OROTOV database, with full context (architecture, constraints, TDD specs) captured
for the agents who will implement them.

## When to Use

Use `/orotov-scope [description]` when you have a feature, bug, or design to break
into stories and issues for agents to claim and build.

## Process

```
Description
  → SPAWN orotov-discover   (interview: scope, problem, success, constraints, deps)
  → SPAWN orotov-plan       (writes plan doc: phases → tasks, acceptance + TDD specs)
  → YOU parse the plan          (phases → stories, tasks → issues, derive dependencies)
  → SHOW preview, get approval  (approve / iterate / start over)
  → SAVE plan to git            (.claude/plans/<date-name>.md)
  → CREATE in OROTOV            (MCP tools below)
  → DONE
```

There is **no "extract issues" tool** — you read the plan markdown and structure the
stories/issues yourself, then create them with the MCP tools.

To break the plan into independently-grabbable vertical slices, use the **`to-issues`**
skill's tracer-bullet approach for the parsing step.

## Creating stories and issues

Use the OROTOV MCP tools (`mcp__orotov__*`). The stack is **SQLite** behind a
FastAPI app — no external services to configure.

| Step | Tool | Key args |
|------|------|----------|
| One story per phase | `create_story` | `title`, `description`, `labels` → returns `id` |
| Story-level context | `set_story_context` | `story_id`, `overview`, `architecture_notes`, `constraints`, `tech_stack`, `examples` |
| One issue per task | `create_issue` | `story_id`, `title`, `description`, `priority`, `labels` → returns `id` |
| TDD spec per issue | `set_issue_spec` | `issue_id`, `acceptance_criteria`, `requirements`, `test_requirements`, `definition_of_done` |
| Dependencies | see below | link issues that must run in order |

**Dependencies** — two equivalent options:
- REST: `POST /api/issues/{issue_id}/blockers` with body `{"depends_on_id": <other_issue_id>}`
  (issue is blocked by the other). *Not* `/dependencies`.
- MCP: `add_issue_link(source_issue_id, target_issue_id, link_type="blocks", note=...)`.
  Valid `link_type`s: `blocks`, `relates-to`, `caused-by`, `duplicates`, `is-fixed-by`, `clones`.

Create issues first, then add blockers/links once both IDs exist.

## Preview format

Show the structure before creating anything:

```
I'd create:

Story 1: <Phase name>
  Issue 1: <Task title>
    Tests: <test names from the plan's TDD spec>
  Issue 2: <Task title>   (depends on Issue 1)
    Tests: ...
Story 2: <Phase name>
  ...

Approve? (yes / iterate / start over)
```

- **Approve** → save plan, create entities.
- **Iterate** → ask what to change, re-run discover/plan as needed.
- **Start over** → reset to the description.

## Example

```
/orotov-scope Fix: agents get silently disconnected when their JWT expires
mid-session — auto-refresh or show a clear error.

→ orotov-discover: single story; must not break live sessions; refresh endpoint
  already exists but isn't wired up; cover expiry mid-request and the double-refresh race.
→ orotov-plan: one phase, 3 tasks with TDD specs.
→ You parse → preview:

  Story 1: Fix JWT token refresh
    Issue 1: Auto-refresh expired token in API middleware
      Tests: refresh_on_expired_token, refresh_failure_fallback, concurrent_refresh_race
    Issue 2: Client retries original request after 401  (depends on Issue 1)
      Tests: 401_triggers_refresh, successful_retry, permanent_401, network_error
    Issue 3: Error messaging + logging  (depends on Issue 2)
      Tests: error_message_display, log_entry_created

  Approve? → yes

✓ Saved .claude/plans/2026-06-14-fix-jwt-refresh.md
✓ create_story → Story 1; set_story_context
✓ create_issue ×3; set_issue_spec ×3
✓ POST /api/issues/2/blockers {depends_on_id: 1}; POST /api/issues/3/blockers {depends_on_id: 2}

Done! 1 story, 3 issues. Agents can claim Issue 1 and run /orotov-build <id>.
```

## Key Principles

- **Transparent** — user approves the full structure before anything is created.
- **Self-contained issues** — each issue carries its description, acceptance criteria, and
  TDD spec so an agent can implement it without the planning context.
- **Dependency-explicit** — blockers/links capture sequencing so parallel work stays safe.
- **Persistent** — the plan lives in git and the work in the database, not in the session.

## Related Skills

- **`orotov-discover`** — interview sub-skill (spawned here).
- **`orotov-plan`** — writes the execution plan (spawned here).
- **`to-issues`** — tracer-bullet slicing used to parse the plan into issues.
- **`orotov-build`** — agents run this to implement a claimed issue (TDD + spec/quality review built in).
