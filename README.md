# InvMon Agent Skills

> **Disclaimer:** The skills and tutorials in this repository are provided only as an example. They must not be used for real investment decisions — the ratings, price targets and sentiment readings the skills produce are not investment advice.

## Skills

This repository contains a number of Claude Code skills. See folder .claude/skills. 

Start a Claude Code session in this directory and Claude will automatically enumerate/load the skills. 

If you're using a different LLM/Agent system, just ask them to re-organize the file structure according to their requirements.

### Naming convention

Skills whose name ends in `-loop` are built to be **re-run on a schedule** through the trading day (see [Running](#running) below) — each run is a quick refresh that keeps InvMon's view current. Every other skill is **one-shot**: you start it, it does its pass, it prints a summary and exits.

| Skill | Kind | What it does |
|---|---|---|
| `rating-loop` | loop | Researches the instruments of a portfolio group and submits a rating (optionally with a price target) for each, keeping the ratings current through the session. **For simulated portfolio groups only.** |
| `sentiment-loop` | loop | Reads the current sentiment of a market and reports it to InvMon. Takes an optional market argument; without one it reads the broad US market. |
| `portfolio-rating` | one-shot | A single, deeper rating pass over the instruments of one portfolio. |
| `portfolio-sentiment` | one-shot | Derives the market a portfolio actually trades against from its holdings, reads that market's sentiment, and records it on the portfolio. |
| `market-scanner` | one-shot | Sweeps a whole market in price-banded batches through InvMon's IB market scanner, rating each batch, until it has collected N names matching the rating you asked for. |


## Requirements

Every skill in this repository is available to anyone with a **paid InvMon plan**. The skills talk to InvMon through its built-in MCP server, which a paid plan enables. Beyond that:

| Skill | Also needs |
|---|---|
| all | The MCP server enabled on the portfolio group the skill works on: *Enable MCP Server* in the group's *MCP Server Options*, on the port in this repository's `.mcp.json` (`55206`). |
| `rating-loop` | A portfolio group in **simulation mode**. The skill and its tutorial are meant to run on simulated portfolios only; the tutorial walks through the set-up. |
| `market-scanner` | An Interactive Brokers account, with TWS running and connected. Market scanning is IB-only. |
| `rating-loop`, `portfolio-rating`, `market-scanner` | Fresh intra-day quotes when run during trading hours. With an IB connection they follow your IB market data subscriptions; with InvMon's built-in data provider, intra-day chart resolutions need an InvMon subscription. |


## Documentation 

There's a documentation folder under `doc` for some of the skills here (typically a tutorial with screenshots).


## Running

* Copy the InvMon.agent repository to your computer (via zip file download or a git clone).
* Make sure TWS is running, connected to your IB account (see [Requirements](#requirements) for which skills need it).
* Make sure InvMon is running and the MCP server is enabled on your portfolio group (see [Requirements](#requirements)).
* Start claude code in your InvMon.agent directory. 
* then follow the instructions found in the doc directory for the particular skill you want to run.

To run a skill once, use its name as a Claude Code command (example)

```
/portfolio-rating
```

To run a skill in a loop, use the Claude Code command (example)

```
/loop 1h /rating-loop
```
