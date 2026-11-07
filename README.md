# InvMon Agent Skills

> **Disclaimer:** The skills and tutorials in this repository are provided only as an example. They must not be used for real investment decisions — the ratings, price targets and sentiment readings the skills produce are not investment advice.

Skills that let an AI agent research instruments and markets on the web and report ratings, price targets and sentiment back to InvMon through its built-in MCP server.

The skills use the open [Agent Skills](https://agentskills.io) format (a folder per skill with a `SKILL.md`, under `.agents/skills/`). They work with any agent that can connect to an MCP server over HTTP and search the web. This repository comes ready to use with **Claude Code**, **Codex CLI** and **Gemini CLI**; other agents need a few minutes of set-up (see [Other agents](#other-agents)).


## Skills

Skills whose name ends in `-loop` are built to be **re-run on a schedule** through the trading day — each run is a quick refresh that keeps InvMon's view current. Every other skill is **one-shot**: you start it, it does its pass, it prints a summary and exits.

| Skill | Kind | What it does |
|---|---|---|
| `rating-loop` | loop | Researches the instruments of a portfolio group and submits a rating (optionally with a price target) for each, keeping the ratings current through the session. **For simulated portfolio groups only.** |
| `portfolio-rating` | one-shot | A single, deeper rating pass over the instruments of one portfolio. |
| `portfolio-sentiment` | one-shot | Derives the market a portfolio actually trades against from its holdings, reads that market's sentiment, and records it on the portfolio. |
| `market-scanner` | one-shot | Sweeps a whole market in price-banded batches through InvMon's IB market scanner, rating each batch, until it has collected N names matching the rating you asked for. |


## Requirements

Every skill in this repository is available to anyone with a **paid InvMon plan**. The skills talk to InvMon through its built-in MCP server, which a paid plan enables. Beyond that:

| Skill | Also needs |
|---|---|
| all | The MCP server enabled on the portfolio group the skill works on: *Enable MCP Server* in the group's *MCP Server Options*, on port `55206` by default (for another port, see *Changing the port* below). |
| all | An agent with web search, running on the same computer as InvMon. The MCP server only listens on `localhost`, so cloud or remote agent sessions can't reach it. |
| `rating-loop` | A portfolio group in **simulation mode**. The skill and its tutorial are meant to run on simulated portfolios only; the tutorial walks through the set-up. |
| `market-scanner` | An Interactive Brokers account, with TWS running and connected. Market scanning is IB-only. |
| `rating-loop`, `portfolio-rating`, `market-scanner` | Fresh intra-day quotes when run during trading hours. With an IB connection they follow your IB market data subscriptions; with InvMon's built-in data provider, intra-day chart resolutions need an InvMon subscription. |


## Agent set-up

| | Claude Code | Codex CLI | Gemini CLI |
|---|---|---|---|
| Skills | `.claude/skills/` (links to `.agents/skills/`) | `.agents/skills/` | `.agents/skills/` |
| Instructions | `CLAUDE.md`, which imports `AGENTS.md` | `AGENTS.md` | `AGENTS.md`, via `.gemini/settings.json` |
| MCP server config | `.mcp.json` | `.codex/config.toml` | `.gemini/settings.json` |
| On first start | Approve the `invmon-mcp` server | Trust the project (Codex ignores `.codex/config.toml` otherwise) | Trust the folder (Gemini disables the MCP server and skills otherwise) |
| Run a skill once | `/portfolio-rating` | `$portfolio-rating` | `Run the portfolio-rating skill` |
| Re-run on a schedule | `/loop 1h /rating-loop` | From a scheduler, see below | From a scheduler, see below |

Arguments go after the skill name, in plain language: `/market-scanner find 5 names rated Sell or Strong Sell`.

**Re-running without a built-in loop.** Codex CLI and Gemini CLI have no interval command. Start them non-interactively from a scheduler (cron, launchd, Windows Task Scheduler), e.g. `codex exec "Run the rating-loop skill"` or `gemini -p "Run the rating-loop skill"`. An unattended run can't answer approval prompts, so allow the `invmon-mcp` tools first; how that works depends on your agent, see its documentation on MCP tool approvals.

**Changing the port.** If your portfolio group's MCP server uses a port other than `55206`, set the environment variable `INVMON_MCP_URL` before you start Claude Code or Gemini CLI, e.g. `export INVMON_MCP_URL=http://localhost:55207/mcp` (PowerShell: `$env:INVMON_MCP_URL = "http://localhost:55207/mcp"`). Codex CLI doesn't read environment variables in its config: change the URL in `.codex/config.toml`, or pass it on start with `codex -c 'mcp_servers.invmon-mcp.url="http://localhost:55207/mcp"'`.

**Windows and Claude Code.** The entries in `.claude/skills/` are symbolic links to `.agents/skills/`. A zip download, or a git clone without symlink support, turns them into small text files and Claude Code finds no skills. Replace them with copies, from the repository folder in PowerShell:

```
Remove-Item .claude\skills -Recurse -Force; Copy-Item .agents\skills .claude\skills -Recurse
```

### Other agents

Most other coding agents (Cursor, GitHub Copilot, …) read `AGENTS.md`, which tells them where the skills are and how to run one. Add the InvMon MCP server to the agent's MCP configuration — server name `invmon-mcp`, Streamable HTTP, URL `http://localhost:55206/mcp` — and ask the agent to run a skill by name.


## Running

* Copy the InvMon.agent repository to your computer (via zip file download or a git clone).
* Make sure TWS is running, connected to your IB account (see [Requirements](#requirements) for which skills need it).
* Make sure InvMon is running and the MCP server is enabled on your portfolio group (see [Requirements](#requirements)).
* Start your agent in the InvMon.agent directory (`claude`, `codex` or `gemini`).
* Then follow the tutorial in the `doc` directory for the skill you want to run, if there is one.


## Documentation

There's a documentation folder under `doc` for some of the skills here (typically a tutorial with screenshots). The tutorials show Claude Code; with another agent, start it and run the skill as shown in [Agent set-up](#agent-set-up).


## Adding a skill

Create `.agents/skills/<name>/SKILL.md` (the folder name must match the skill's `name`), link it for Claude Code with `ln -s ../../.agents/skills/<name> .claude/skills/<name>`, and add it to the skill tables in this README and in `AGENTS.md`.
