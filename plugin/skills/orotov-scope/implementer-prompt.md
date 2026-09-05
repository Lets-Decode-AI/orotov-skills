# orotov-scope Implementation Context

You are implementing the entry point skill for the OROTOV planning system.

## Your Job

Given a user's work description, orchestrate the planning workflow:

1. **Extract Intent** — Parse what the user wants to build (feature, bug, design, etc.)
2. **Dispatch orotov-discover** — Interview the user relentlessly about requirements
3. **Dispatch orotov-plan** — Create a structured execution plan
4. **Call extract-issues-from-plan** — Convert plan to stories/issues via MCP
5. **Show Preview** — Display generated stories/issues for user approval
6. **Commit on Approval** — Create stories/issues in OROTOV via API

## Key Principles

- **Be transparent** — User sees the interview and plan as they happen
- **Iterate if needed** — If user doesn't like the result, they can revise the description and rerun
- **Store everything** — Planning doc goes to git, key decisions go to database
- **No external context** — All information needed to implement is stored in database (StoryContext, IssueSpecification)

## Constraints

- Do NOT assume what the user wants — ask clarifying questions via orotov-discover
- Do NOT create stories/issues without user approval preview
- Do NOT skip the planning phase — every work item should have a plan behind it

## Success Criteria

- [ ] User describes work
- [ ] System interviews user (orotov-discover)
- [ ] System creates plan (orotov-plan)
- [ ] System extracts stories/issues
- [ ] User approves preview
- [ ] Stories/issues created in database
- [ ] Planning doc saved to git
- [ ] User can see all context in OROTOV
