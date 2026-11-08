# InvMon Agent Skills

This repository holds skills for driving **InvMon**, a desktop portfolio manager, through its built-in MCP server. You (the agent) research
instruments and markets on the web and report ratings, price targets and sentiment back to InvMon.

## Skills

The skills follow the open Agent Skills format: one folder per skill under `.agents/skills/`, each with a `SKILL.md`.

| Skill | Kind | What it does |
|---|---|---|
| `rating-loop` | re-run on a schedule | Rates the instruments of a portfolio group. **Simulated portfolio groups only.** |
| `portfolio-rating` | one-shot | A single, deeper rating pass over one portfolio. |
| `portfolio-sentiment` | one-shot | Derives a portfolio's market from its holdings and records that market's sentiment. |
| `market-scanner` | one-shot | Sweeps a market through InvMon's IB market scanner until it has found N names with the requested rating. |

When the user asks you to run one of these skills and your agent does not load skills on its own, read
`.agents/skills/<name>/SKILL.md` and follow it. Treat whatever the user typed after the skill name as the skill's arguments.

## Arguments from InvMon

InvMon can start you with a skill and its arguments already filled in. The arguments then begin with the context the user launched from,
followed by the user's own instructions after a semicolon:

    /portfolio-rating portfolio "Tech Leaders" (id 42); pool candidates only

- `portfolio "<name>" (id <id>)` names one portfolio of this server's portfolio group. Pass the id as `portfolioId` and leave
  `portfolioName` out; the name is there for you and the user to read. If the tools report no portfolio with that id, stop and say so -
  don't fall back to the name.
- No portfolio part means the user launched from the portfolio group: apply the skill's default for "no portfolio given".
- Everything after the semicolon is the user's own instructions, in plain language, as if they had typed them.

## The InvMon MCP server

- Server name `invmon-mcp`, Streamable HTTP on `localhost`. Its URL is in this directory's agent config (port 55206 unless the
  environment variable `INVMON_MCP_URL` says otherwise). InvMon serves it on the user's own computer, so you must run on that computer too.
- The server is up only while InvMon is running and *Enable MCP Server* is on for the portfolio group. If its tools are missing, tell the
  user to check exactly that; don't work around it.
- Use only the MCP tools a skill names, and only as the skill describes. Never change InvMon data by any other route.
- A tool call answered with "This agent run was stopped in InvMon." means the user stopped you in InvMon. Stop at once: don't retry,
  don't call other tools, and if you are repeating a run, stop repeating it. Tell the user that InvMon stopped the run.

## Ground rules

- The skills are examples. Their ratings, price targets and sentiment readings are not investment advice; say so if the user treats them
  as such.
- Run `rating-loop` against a portfolio group in simulation mode only. If you learn that the group is not a simulation, stop without
  submitting ratings.
- Notes and price targets you submit are shown in InvMon's UI: keep them terse and factual.
