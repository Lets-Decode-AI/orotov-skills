---
name: orotov-debug
description: Use when investigating a bug, unexpected behavior, or performance regression with OROTOV board tracking.
metadata:
  version: "0.1.0"
---

# OROTOV: Debug

Investigate a bug end-to-end. Track every finding on the Board. Produce linked fix issues.

## When to Use

```
/orotov-debug [symptom description]
```

Use for any bug, unexpected behavior, or performance regression. This skill:
1. Creates a debug story on the Board (permanent investigation record)
2. Runs `superpowers:systematic-debugging` (root cause first — no guessing)
3. Logs every phase finding to the Board as an activity event
4. Creates fix issues linked back to the debug story
5. Closes the debug story when fixes are complete

## Step 1: Create Debug Story and Root Issue

```
mcp__orotov__create_story(
  title="Debug: <symptom — one line>",
  labels="debug"
)
```

Save `debug_story_id`.

Always ask the user:
> "Is this bug related to a known feature story? If so, provide its story ID (or skip)."

Create the root investigation issue:
```
mcp__orotov__create_issue(
  story_id=debug_story_id,
  title="Investigate: <symptom>",
  labels="debug,investigation"
)
```

Save `root_issue_id`.

If the user provided a related story ID:
```
mcp__orotov__add_issue_link(
  source_issue_id=root_issue_id,
  target_issue_id=<feature_issue_id>,
  link_type="relates-to",
  note="Debug investigation related to this feature"
)
```

Then:
```
mcp__orotov__update_status(issue_id=root_issue_id, status="In Progress")
mcp__orotov__log_work(
  entity_type="story", entity_id=debug_story_id,
  event_type="session_started",
  summary="Debug session opened: <symptom>"
)
```

## Step 2: Run Systematic Debugging

Invoke: `superpowers:systematic-debugging`

This skill runs four phases (Root Cause → Pattern → Hypothesis → Implementation).
At the boundary of each phase, log findings to the Board before continuing:

### After Phase 1 (Root Cause Identified)

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=root_issue_id,
  event_type="debug_note",
  summary="Root cause identified: <one-liner>",
  detail={"phase": 1, "finding": "<full finding text>", "evidence": "<what was observed>"}
)
mcp__orotov__log_decision(
  issue_id=root_issue_id,
  decision_text="Root cause: <finding>. Evidence: <what proved this>. Ruled out: <alternatives considered>",
  category="Architecture"
)
```

### After Phase 2 (Pattern Found)

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=root_issue_id,
  event_type="debug_note",
  summary="Pattern: <diff between working and broken>",
  detail={"phase": 2, "pattern": "<description>", "working_example": "<ref>", "broken_example": "<ref>"}
)
```

### After Phase 3 (Hypothesis Confirmed)

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=root_issue_id,
  event_type="debug_note",
  summary="Hypothesis confirmed: <hypothesis>",
  detail={"phase": 3, "hypothesis": "<text>", "test_performed": "<what was changed to verify>"}
)
mcp__orotov__log_decision(
  issue_id=root_issue_id,
  decision_text="Confirmed: <hypothesis>. Test: <minimal change that proved it>. Rejected alternatives: <list>",
  category="TradeOff"
)
```

### After Phase 4 (Fix Verified)

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=root_issue_id,
  event_type="debug_note",
  summary="Fix verified: <what was changed>",
  detail={"phase": 4, "fix": "<description>", "files_touched": ["<path1>", "<path2>"]}
)
```

## Step 3: Record Findings on Story Context

```
mcp__orotov__set_story_context(
  story_id=debug_story_id,
  overview="Bug: <symptom>. Root cause: <cause in one sentence>",
  architecture_notes="Propagation path: <how the bug flows through the system>. Trigger: <exact condition>",
  constraints="Must not regress: <what to protect — e.g., existing behaviour X, test Y>"
)
```

## Step 4: Create Fix Issues

**Determine fix scope:**
- **Single fix** — root cause touches ≤ 2 files AND the fix is fully described by the Phase 3 hypothesis: create one issue, run `/orotov-build`
- **Multi-issue fix** — touches > 2 files OR requires design decisions: run `/orotov-scope` with the debug story as context

### Single Fix Path

```
mcp__orotov__create_issue(
  story_id=debug_story_id,
  title="Fix: <root cause short title>",
  labels="debug,fix",
  priority="High"
)
```

Save `fix_issue_id`.

```
mcp__orotov__set_issue_spec(
  issue_id=fix_issue_id,
  acceptance_criteria="<from Phase 3 hypothesis — what done looks like>",
  implementation_notes="<from Phase 4 verified fix — what to change and where>",
  test_requirements="Regression test: reproduce the original symptom (test must FAIL before fix, PASS after)",
  references="Debug story #<debug_story_id>, root issue #<root_issue_id>"
)
mcp__orotov__add_issue_link(
  source_issue_id=fix_issue_id,
  target_issue_id=root_issue_id,
  link_type="caused-by",
  note="Fix for root cause found in debug investigation"
)
```

Run: `/orotov-build <fix_issue_id>`

### Multi-Issue Fix Path

Run: `/orotov-scope` with the debug story context pre-filled as requirements.

For each fix issue created by orotov-scope:
```
mcp__orotov__add_issue_link(
  source_issue_id=<fix_issue_id>,
  target_issue_id=root_issue_id,
  link_type="caused-by",
  note="Fix issue linked to debug root cause"
)
```

## Step 5: Close Debug Story

When all fix issues reach Done status:

```
mcp__orotov__update_status(story_id=debug_story_id, status="Approved")
mcp__orotov__log_work(
  entity_type="story", entity_id=debug_story_id,
  event_type="session_ended",
  summary="Debug complete. Root cause: <one-liner>. Fix: Issue #<N>",
  detail={"fix_issue_ids": [<id1>, <id2>], "phases_completed": 4}
)
```

## Querying Past Debug Investigations

To search past debug stories on the Board, filter by `label:debug`. Each debug story contains:
- Full activity timeline (phase findings, decisions, session events)
- Root cause in story context overview
- Fix issues linked via `caused-by` links

## Key Principles

- **Root cause first** — superpowers:systematic-debugging enforces this. Never skip Phase 1.
- **Log before moving** — always call `log_work` at the phase boundary BEFORE advancing to the next phase
- **One fix issue per root cause** — if multiple things are broken, each gets its own debug story
- **Regression test is mandatory** — every fix issue spec must include a test that reproduces the symptom
