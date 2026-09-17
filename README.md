# InvMon Agent Skills

## Skills

This repository contains a number of Claude Code skills. See folder .claude/skills. 

Start a Claude Code session in this directory and Claude will automatically enumerate/load the skills. 

If you're using a different LLM/Agent system, just ask them to re-organize the file structure according to their requirements.

### Naming convention

Skills whose name ends in `-loop` are built to be **re-run on a schedule** through the trading day (see [Running](#running) below) — each run is a quick refresh that keeps InvMon's view current. Every other skill is **one-shot**: you start it, it does its pass, it prints a summary and exits.

| Skill | Kind | What it does |
|---|---|---|
| `rating-loop` | loop | Researches the instruments of a portfolio group and submits a rating (optionally with a price target) for each, keeping the ratings current through the session. |
| `sentiment-loop` | loop | Reads the current sentiment of a market and reports it to InvMon. Takes an optional market argument; without one it reads the broad US market. |
| `portfolio-rating` | one-shot | A single, deeper rating pass over the instruments of one portfolio. |
| `portfolio-sentiment` | one-shot | Derives the market a portfolio actually trades against from its holdings, reads that market's sentiment, and records it on the portfolio. |
| `market-scanner` | one-shot | Sweeps a whole market in price-banded batches through InvMon's IB market scanner, rating each batch, until it has collected N names matching the rating you asked for. |


## Documentation 

There's a documentation folder under `doc` for some of the skills here (typically a tutorial with screenshots).


## Running

* Copy the InvMon.agent repository to your computer (via zip file download or a git clone).
* Make sure TWS is running, connected to your IB account.
* Make sure InvMon is running with a running MCP server (requires paid InvMon plan, enable the server in your Portfolio Group Target settings).
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
