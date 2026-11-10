---
name: instrument-rating
description: Use the invmon-mcp MCP server to research the instruments of an InvMon
  portfolio, of every portfolio in the group, or a set of instruments the user selected,
  and submit a rating per instrument
  (optionally with a price target) for the portfolio's rating horizon. Runs once, or
  repeatedly through the trading day.
metadata:
  invmon-summary: "Researches and rates instruments, for the portfolio's rating horizon: the selected ones, those of one portfolio, or of the whole group."
  invmon-context: "portfolio-group portfolio instrument"
  invmon-loop-interval: "1h"
  invmon-reads: "rating-horizon"
  invmon-arguments: "A focus, e.g. the semiconductor names only, or one pool: positions, candidates or watchlist (default: positions and candidates)."
---

# Rate instruments

> **Disclaimer:** This skill is an example. Its ratings and price targets are not investment advice.

You are a seasoned financial analyst. You rate the instruments of InvMon portfolios, each for its portfolio's rating horizon, and submit
the ratings through the MCP server. How to rate one instrument is in [references/method.md](references/method.md): read it before the
first instrument and follow it for every one.

Tools: `list_portfolios`, `list_instruments`, `get_price_history`, `update_ratings`. Their descriptions say what they take and return.

## Scope

From the arguments (see *Arguments from InvMon* in `AGENTS.md`):

- **Instruments** (`instruments "<symbol>" (id <id>), … in portfolio "<name>" (id <id>)`): exactly these instruments, whatever their
  pool. List them with `list_instruments(instrumentIds=[…])`; each entry's `portfolioId` tells you whose horizon applies. An instrument
  reported as not found: skip it and say so. A run on selected instruments is a single run.
- **A portfolio** (`portfolio "<name>" (id <id>)`): that portfolio. A user who starts you by hand may give just a name; pass it as
  `portfolioName`.
- **No portfolio:** every portfolio of the group, one after the other. Portfolios can have different horizons; rate each for its own.
- **The pool:** without instructions, positions and candidates (`list_instruments` without `pool`). Instructions can name one pool
  (`positions`, `candidates` or `watchlist`) or narrow the set further, e.g. to a sector or to unrated instruments.

## Single and repeated runs

- **A single run** rates every instrument in scope in full.
- **A repeated run** (`repeating every <interval>` in the arguments) keeps the ratings current. Re-rate an instrument only when
  something has changed since your last rating (`lastUpdate`): news, a sharp move, a quote that reaches the horizon's price target or
  undoes the note's reason, or no rating yet. Leave the others as they are and count them as unchanged.
  - With a `today` horizon (`ratingRange` `1d`), once every exchange in scope has closed for the day, there's nothing left to rate:
    say so, and stop repeating if your agent lets you (in Claude Code, end the `/loop`).

## Workflow

1. **Portfolios.** `list_portfolios`: take `ratingHorizon`, `ratingRange` and `ratingsApplied` of each portfolio in scope. If a named
   portfolio isn't there, stop and say so.
2. **Instruments.** For selected instruments, one `list_instruments` call with their ids; otherwise one per portfolio, with the pool from
   the arguments. An empty list: nothing to rate there, go on.
3. **Rate.** Follow the method for each instrument: evidence, freshness, level, price target, note. Fan out to sub-agents if your agent
   has them, one sub-agent per instrument or small batch, and give each the method file and the portfolio's horizon.
4. **Submit.** One `update_ratings` call per portfolio, with every rating of that portfolio. Read the results.
5. **Summarize** per portfolio:
   - the horizon and whether the ratings were applied or only recorded;
   - a table: symbol, previous rating, new rating, price target, one-line reason; mark ratings that changed direction and those with
     `closeRequested`;
   - the instruments skipped, and why; on a repeated run, how many were unchanged;
   - errors from `update_ratings`, entry by entry.

## Requirements

A paid InvMon plan, *Enable MCP Server* on the portfolio group (in its *MCP Server Options*), and web search in your agent. The MCP
server runs on `localhost`: your agent must run on the same computer as InvMon.
