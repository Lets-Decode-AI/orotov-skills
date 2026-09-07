---
name: orotov-discover
description: Use when requirements need a deep-dive interview before OROTOV planning.
metadata:
  version: "0.2.0"
---

# OROTOV: Requirement Discovery

Interview user relentlessly to understand the problem, requirements, constraints, and scope.

## When to Use

Spawned by `orotov-scope`. Not called directly.

## Questions Asked

Always ask in this order, one at a time:

1. **Scope Question** — "Is this a single story (one issue/task) or a feature with multiple stories?"
2. **Problem Statement** — "What problem does this solve? Why does it matter?"
3. **Success Criteria** — "How will you know it's done? What does success look like?"
4. **Constraints** — "Are there any hard limits? (e.g., no external dependencies, performance requirements, backwards compatibility)"
5. **Architecture Hints** — "Any architecture or design preferences? (e.g., WebSocket vs polling, monolith vs microservice)"
6. **Timeline** — "When is this needed? Any deadline?"
7. **Dependencies** — "Does this depend on other work? What's the critical path?"
8. **Other Considerations** — "Anything else I should know?"

## Output

Return to orotov-scope:
- Clarified requirements
- Scope determination (single vs. multi-story)
- Key constraints and assumptions
- Interview transcript (verbatim)

## Example

```
User: Build a real-time notification system

Discover Agent:
Q1: "Is this a single story or feature with multiple stories?"
User: Feature with multiple phases

Q2: "What problem does this solve?"
User: Agents need a way to receive real-time updates from the system without polling

Q3: "How will you know it's done?"
User: Agents can connect, receive events, reconnect on disconnect, authenticate with JWT

...continues...
```
