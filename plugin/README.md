# OROTOV skills plugin

Installs the five OROTOV board-workflow skills into Claude Code (and
plugin-capable tools):

- `orotov-scope` — interview + plan + generate stories/issues
- `orotov-discover` — deep-dive requirement interview
- `orotov-plan` — structured execution plan with TDD specs
- `orotov-build` — implement one issue with TDD + review subagents
- `orotov-debug` — systematic debugging with Board tracking

## Install

```
/plugin marketplace add Lets-Decode-AI/orotov-skills
/plugin install orotov
```

These skills drive work against an OROTOV MCP board. Connect the board
first with `npx @orotov/connect <API_KEY>` (see `packages/connect`).

The installed commands are `/orotov-scope`, `/orotov-discover`, `/orotov-plan`,
`/orotov-build`, and `/orotov-debug`.
