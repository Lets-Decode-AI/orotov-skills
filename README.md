# OROTOV skills

The board workflow skills for [OROTOV](https://orotov.com) — scope, discover,
plan, build and debug, driven against an OROTOV MCP board.

```
/plugin marketplace add Lets-Decode-AI/orotov-skills
/plugin install orotov
```

The skills call the OROTOV MCP tools, so they need a workspace API key. Get one
at [orotov.com](https://orotov.com), then point your tool at the board:

```
npx @orotov/connect <API_KEY>
```

## This repository is a mirror

The skills are developed in the OROTOV product repository and published here
automatically. Do not send pull requests against this repo — they will be
overwritten by the next sync. File issues at
<https://github.com/Lets-Decode-AI/orotov-skills/issues> and they will be triaged upstream.
