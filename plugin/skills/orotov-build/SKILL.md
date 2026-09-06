---
name: orotov-build
description: Use when a claimed OROTOV issue has a clear specification and acceptance criteria ready to implement.
metadata:
  version: "0.1.0"
---

# OROTOV: Build & Review

Implement a single issue using TDD. Three isolated subagents handle implementation, spec compliance, and code quality. You are the orchestrator — a thin state machine. You never read, summarize, or pass code. It lives on disk.

## When to Use

When you have a claimed issue with a clear spec and acceptance criteria:

```
/orotov-build 4
```

## What You Do (Orchestrator)

1. Fetch issue context from the tracker
2. Spawn implementer subagent — it writes tests and code to disk
3. Spawn fresh spec reviewer — it reads code from disk, returns a manifest
4. Spawn fresh quality reviewer — it reads code from disk, returns a manifest
5. On rejection: send concise brief to implementer, loop back
6. On both approved: instruct implementer to commit, mark issue done

**You hold only:** issue spec + file paths + manifest JSON. Nothing else ever enters your context.

## Step 1: Fetch Issue Context

Call:
```
mcp__orotov__get_issue_context(issue_id)
```

Extract and hold:
- `spec` — what needs to be built
- `acceptance_criteria` — numbered list of requirements
- `test_requirements` — what tests must exist
- `story_context` — why this feature matters
- `output_file_paths` — where implementation files should be written

Then move the issue into progress and log the session start:
```
mcp__orotov__update_status(issue_id=<issue_id>, status="In Progress")
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="session_started",
  summary="Build session started for issue #<issue_id>"
)
```

> Status values are capitalized: `New`, `In Progress`, `In Review`, `Done`, `Blocked`.

## Step 2: Spawn Implementer Subagent

Spawn a subagent (model: standard) with this prompt — fill in the `<placeholders>`:

```
You are implementing issue #<issue_id>.

Read `implementer-prompt.md` (in this skill's directory).

Issue Spec:
<spec>

Acceptance Criteria:
<acceptance_criteria numbered list>

Test Requirements:
<test_requirements>

Story Context:
<story_context>

Output file paths (write your implementation here):
<output_file_paths>

Follow your instructions exactly. End with an IMPLEMENTER_MANIFEST block.
```

**Wait for** an `IMPLEMENTER_MANIFEST` block in the response.

Save the agent ID — you will resume this agent on rejection.

Log implementer started:
```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="work_note",
  summary="Implementer subagent started"
)
```

## Step 3: Spec Review Loop

**Spawn a fresh subagent** (model: standard) with this prompt:

```
You are the spec reviewer for issue #<issue_id>.

Read `spec-reviewer-prompt.md` (in this skill's directory).

Issue Spec:
<spec>

Acceptance Criteria:
<acceptance_criteria numbered list>

Files to review — read each one directly from disk:
<manifest.files_written as a list>

Review for spec compliance. End with a SPEC_REVIEW_MANIFEST block.
```

**Wait for** a `SPEC_REVIEW_MANIFEST` block.

Parse `verdict`:

**If `verdict = "REJECTED"`:**

Extract `issues[]` from the manifest. Log the rejection:
```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="work_note",
  summary="Spec review round <N>: REJECTED — <len(issues)> issues",
  detail={"round": <N>, "issue_count": <len(issues)>}
)
```

Resume the implementer subagent (SendMessage) with:

```
REJECTION from SPEC_REVIEW — fix these issues then resubmit:

<for each issue in issues[]:>
<issue number>. [<type>] criterion_id=<criterion_id>: <description>
   Fix: <fix>

Return a new IMPLEMENTER_MANIFEST when ready.
```

Wait for a new `IMPLEMENTER_MANIFEST`. Return to the start of Step 3.

**If `verdict = "APPROVED"`:**

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="work_note",
  summary="Spec review: APPROVED"
)
```

Proceed to Step 4.

## Step 4: Quality Review Loop

**Spawn a fresh subagent** (model: most capable) with this prompt:

```
You are the code quality reviewer for issue #<issue_id>.

Read `code-quality-reviewer-prompt.md` (in this skill's directory).

Issue Spec (context only):
<spec>

Files to review — read each one directly from disk:
<manifest.files_written as a list>

Review for production quality. End with a QUALITY_REVIEW_MANIFEST block.
```

**Wait for** a `QUALITY_REVIEW_MANIFEST` block.

Parse `verdict`:

**If `verdict = "REJECTED"`:**

Extract `issues[]`. Log the rejection:
```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="work_note",
  summary="Quality review round <N>: REJECTED — <len(issues)> issues",
  detail={"round": <N>, "issue_count": <len(issues)>}
)
```

Resume the implementer subagent (SendMessage) with:

```
REJECTION from QUALITY_REVIEW — fix these issues then resubmit:

<for each issue in issues[]:>
<issue number>. [<category>] <location>: <description>
   Fix: <fix>

Return a new IMPLEMENTER_MANIFEST when ready.
```

Wait for a new `IMPLEMENTER_MANIFEST`. **Return to Step 3** — spec review reruns because code changes may introduce new spec gaps.

**If `verdict = "APPROVED"`:**

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="work_note",
  summary="Quality review: APPROVED"
)
```

Proceed to Step 5.

## Step 5: Git Commit

Resume the implementer subagent (SendMessage) with:

```
Both reviews passed. Commit your work now.

Run:
git add <manifest.files_written — each path separated by a space>
git commit -m "feat: <issue title> (#<issue_id>)

Criteria met: <manifest.criteria count>/<manifest.criteria count>
Tests passing: <manifest.tests_passing>
Spec review: APPROVED
Quality review: APPROVED"

Reply with "Committed: <commit hash>" when done.
```

Wait for the commit confirmation, then register the commit and the passing tests
with the tracker — the lane policy requires both before an issue may move to Done:

```
mcp__orotov__link_commit(
  issue_id=<issue_id>,
  commit_hash="<commit hash from the implementer>",
  commit_message="feat: <issue title> (#<issue_id>)"
)
```

For each test the implementer reported passing (`manifest.tests_passing`), record it
as a passing test case (use `status="Pass"` so it satisfies the lane policy):
```
mcp__orotov__add_test_case(
  issue_id=<issue_id>,
  name="<test name from manifest>",
  code="<test identifier or path>",
  status="Pass"
)
```

## Step 6: Mark Done

First confirm the issue satisfies the lane policy, then transition it:
```
mcp__orotov__check_lane_requirements(issue_id=<issue_id>, target_status="Done")
```
If `allowed` is false, resolve each `missing_artifacts[]` entry (most often a missing
linked commit or passing test from Step 5) before continuing.

```
mcp__orotov__update_status(issue_id=<issue_id>, status="Done")
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="session_ended",
  summary="Build complete. Spec rounds: <N>. Quality rounds: <M>.",
  detail={"spec_review_rounds": <N>, "quality_review_rounds": <M>}
)
```

## Model Selection

| Role | Model | Reason |
|---|---|---|
| Implementer | Standard | TDD is mechanistic, spec is explicit |
| Spec Reviewer | Standard | Compliance checking is deterministic |
| Quality Reviewer | Most capable | Requires judgment and pattern recognition |

## Context Rules

- **Orchestrator holds:** issue spec + manifests only (~5–15K, flat)
- **Never** pass code text to the orchestrator or between agents — code lives on disk
- **Never** summarize or paraphrase reviewer output — parse the manifest JSON only
- **Implementer context** grows with each rejection round — isolated and acceptable
- **Each reviewer** starts fresh — unbiased, no accumulated review history

## Success Criteria

Issue is complete when all of these are true:
- `IMPLEMENTER_MANIFEST`: all `criteria[].met = true`
- `SPEC_REVIEW_MANIFEST`: `verdict = "APPROVED"`
- `QUALITY_REVIEW_MANIFEST`: `verdict = "APPROVED"`
- Git commit made with both approvals in message
- Commit linked (`link_commit`) and a passing test recorded (`add_test_case status="Pass"`)
- Issue marked `Done` in tracker (lane requirements satisfied)
