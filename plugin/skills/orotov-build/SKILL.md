---
name: orotov-build
description: Use when a claimed OROTOV issue has a clear specification and acceptance criteria ready to implement.
metadata:
  version: "0.2.0"
---

# OROTOV: Build & Review

Implement a single issue using TDD. Three isolated subagents handle implementation, spec compliance, and code quality. You are the orchestrator — a thin state machine. You never read, summarize, or pass code. It lives on disk.

## When to Use

When you have a claimed issue with a clear spec and acceptance criteria:

```
/orotov-build 4
```

## What You Do (Orchestrator)

1. Claim the issue, load its context **and its completion policy**, resume any earlier run
2. Spawn implementer subagent — it writes tests and code to disk
3. Spawn fresh spec reviewer — it reads code from disk, returns a manifest
4. Spawn fresh quality reviewer — it reads code from disk, returns a manifest
5. On rejection: send concise brief to implementer, loop back
6. On both approved: commit, then follow the policy — push, PR, CI, human gate
7. Request Done only when `evaluate_transition` says the issue is eligible

**You hold only:** issue spec + file paths + manifest JSON + gate state. Nothing else ever enters your context.

**You never fabricate external state.** A PR, CI result, review or merge exists
only when OROTOV has observed it. If you cannot observe it, say so and stop at
that phase — do not mark anything Done to "finish" the run.

## Step 1: Claim, Load Context, Read the Policy, Resume

```
mcp__orotov__claim_issue(issue_id)
```

- `error_code="blocked"` → stop. Report `constraint.blockers` (issue id + status). Do not work around it.
- `error_code="already_claimed"` → stop and report `holder_ref`.
- Keep `session_id`. Claims renew automatically while you work; they expire after 30 min of no activity.

```
mcp__orotov__get_issue_context(issue_id)
mcp__orotov__get_completion_policy(issue_id=<issue_id>)
mcp__orotov__resume_session(issue_id=<issue_id>)
```

From context, hold: `spec`, `acceptance_criteria`, `test_requirements`, `story_context`, `output_file_paths`.

From the policy, hold `mode` and `requirements`:
- `mode="legacy"` — claims satisfy Done (linked commit + passing test). Say so in your final report.
- `mode="enforced"` — note which of `verified_commit`, `observed_tests`, `pull_request`
  (`target_branch`), `required_checks`, `merge`, `human_approvals` apply. They decide where this run ends.

If `resume_session` returns `progress_snapshot.build`, continue **after**
`last_progress_phase` using `build.artifacts` (commit SHA, branch, PR number) — do not redo finished phases.

Otherwise start the run:
```
mcp__orotov__update_status(issue_id=<issue_id>, status="In Progress")
mcp__orotov__record_build_phase(session_id=<session_id>, phase="implementing")
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="session_started",
  summary="Build session started for issue #<issue_id>"
)
```

> Status values are capitalized: `New`, `In Progress`, `In Review`, `Done`, `Blocked`.

> `log_work` records testimony only: `event_type` is one of `work_note`, `debug_note`,
> `handoff_note`, `session_started`, `session_ended`, and `summary` is at most 500
> characters (put detail in `detail`). State changes are recorded by the tool that
> makes them (`update_status`, `link_commit`, ...), never logged by hand.

**If a subagent dies** (context limit, usage limit, crash, timeout): start a fresh
one from the last recorded phase. Everything it needs is on disk and in
`resume_session`.

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

## Step 5: Commit

After local tests pass: `record_build_phase(session_id, phase="verified_locally", artifacts={"test_command": "<command>"})`.
After both reviews approve: `record_build_phase(session_id, phase="quality_reviewed")`.

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

Reply with "Committed: <full commit hash>" when done.
```

Link the commit and record the local test run:

```
mcp__orotov__link_commit(issue_id=<issue_id>, commit_hash="<hash>", commit_message="feat: <issue title> (#<issue_id>)")
mcp__orotov__add_test_case(issue_id=<issue_id>, name="<test name>", code="<test path>", status="Pass")
mcp__orotov__record_build_phase(session_id=<session_id>, phase="committed", artifacts={"commit_sha": "<hash>"})
```

`link_commit` returns `verification.verification_state`. `claimed` means no
repository could confirm it yet; `not_found` means the SHA does not exist in the
connected repository — stop and fix it; `connector_unavailable` means retry later
with `verify_commit`. Your `add_test_case(status="Pass")` is a **claim**: it
never satisfies an `observed_tests` requirement.

## Step 6: Follow the Completion Policy

Ask the gate what is missing — never guess:

```
mcp__orotov__evaluate_transition(issue_id=<issue_id>, target_status="Done")
```

Each `requirements[]` entry has `state` (`satisfied`, `unsatisfied`, `pending`,
`not_observed`, `not_configured`, `overridden`), `reason`, and `evidence[]` with
`trust` (`claimed`, `observed`, `approved`).

**Legacy policy** (`policy.mode="legacy"`): if `eligible`, go to Step 7.

**Enforced policy:**

1. **Push** (when `pull_request` is required, or your workflow pushes):
   `git push -u origin <branch>` — name the branch `issue-<issue_id>-<slug>` so OROTOV correlates it.
   `record_build_phase(phase="pushed", artifacts={"branch": "<branch>"})`.
2. **Pull request**: open or update a PR into `requirements.pull_request.target_branch`
   with `#<issue_id>` in the title, using the tools available to you (`gh pr create`).
   `record_build_phase(phase="pr_open", artifacts={"pr_number": <n>})`.
3. **Checks**: `record_build_phase(phase="awaiting_checks")`, then re-run
   `evaluate_transition`. `required_checks`:
   - `pending` → CI is running; check again later, do not claim it passed.
   - `unsatisfied` → a required check failed; fix, commit, push, loop.
   - `not_configured` / `not_observed` → OROTOV cannot see CI for this repository.
     **Stop:** `record_build_phase(phase="stopped", note="required checks not observable: <reason>")`
     and report it. Do not substitute your own test run.
4. **Human gate**: if `merge` or any `human_approvals` requirement is not satisfied
   and the policy makes it human-owned, **do not merge**. Record
   `record_build_phase(phase="awaiting_human", artifacts={"pr_number": <n>})`, move the issue to
   `In Review`, release your claim with `release_issue(session_id, outcome="paused")`, and end with:

```
AWAITING_HUMAN_MERGE
issue: #<issue_id>
pr: <url or number>
ci: <required_checks state + reason>
evidence: <verified_commit / observed_tests states>
unresolved: <each blocking requirement key + reason>
```

5. **After the human merges** (a later run, from `resume_session`):
   `record_build_phase(phase="reconciling")` and re-run `evaluate_transition`. The
   connector observes the merge and CI on the merge commit; continue only when
   `eligible` is true. If it is not, report the blocking requirements and stop.

## Step 7: Request Done

```
mcp__orotov__record_build_phase(session_id=<session_id>, phase="completed")
mcp__orotov__update_status(issue_id=<issue_id>, status="Done")
```

The gate re-evaluates independently. If it refuses (`error_code="gate_blocked"`),
report `gate.blocking` and stop — never retry with different claims. A recorded
`completed` phase is the builder's note; it is not evidence and does not make
the issue Done.

On success the claim completes automatically:

```
mcp__orotov__log_work(
  entity_type="issue", entity_id=<issue_id>,
  event_type="session_ended",
  summary="Build complete. Spec rounds: <N>. Quality rounds: <M>.",
  detail={"spec_review_rounds": <N>, "quality_review_rounds": <M>, "policy_mode": "<mode>"}
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

The run ends in exactly one of these, and says which:

- **Done** — `update_status` succeeded because `evaluate_transition` was eligible under the policy in force (name the mode).
- **AWAITING_HUMAN_MERGE** — PR open, required human step outstanding, claim paused, report emitted.
- **Stopped** — a requirement cannot be observed or satisfied; `record_build_phase(phase="stopped")` holds the reason.

Never:
- mark Done because tests passed locally;
- merge a PR the policy reserves for a human;
- describe CI, a PR, a review or a merge you did not observe through OROTOV.
