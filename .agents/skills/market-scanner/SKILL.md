---
name: market-scanner
description: Use the invmon-mcp MCP server to sweep a large securities universe in
  price-banded batches through InvMon's IB market scanner interface, rating each batch
  and setting aside the instruments that match a specific rating, until N of them have
  been identified - then expose the winners in the watchlist. The rating (or ratings)
  that make an instrument a winner can be passed as an argument; by default a winner is
  rated `Strong Buy` or `Buy`. One-shot; not meant to be driven in a loop.
  Requires a paid InvMon plan and a connected Interactive Brokers account.
metadata:
  invmon-summary: "Scans a wide market in price bands for instruments that earn a given rating and puts them on the watchlist."
  invmon-context: "portfolio"
  invmon-requires: "ib-connection"
  invmon-reads: "rating-horizon"
  invmon-arguments: "How many names to find (default 10), which ratings count (default Strong Buy and Buy), price range and band width (default $5 to $100 in $5 bands)."
  invmon-tutorial: "doc/market-scanner/market-scanner.md"
---

# Scan a market for N names with a given rating

> **Disclaimer:** This skill is an example. Its ratings and price targets are not investment advice.

You are a seasoned financial analyst running a screening sweep. InvMon's IB market scanner returns only the top rows of one ranked
query and can't be paged. To look beyond that slice, you split the universe along the price axis into disjoint bands `[lo, hi)` and
scan each band as one batch. You rate every batch, hide the winners to set them aside, and go on to the next band until you have **N**
winners. Then you clean up and un-hide the winners, so that they show in the watchlist.

Rate every instrument by the method in `../instrument-rating/references/method.md`: read it before the first batch, and rate for the
portfolio's horizon (`ratingHorizon`, `ratingRange` from `list_portfolios`). This is a screening pass: no positions are opened.

Tools: `list_portfolios`, `run_scanner`, `get_price_history`, `update_ratings`, `hide_instruments`, `unhide_instruments`,
`archive_instruments`. Their descriptions say what they take and return. `run_scanner` returns the batch inline: you don't need
`list_instruments` during the sweep.

## Prerequisites

The user sets these up once, in InvMon, on a dedicated portfolio. When `run_scanner` returns an error, it points at one of them.

- **A paid InvMon plan** that includes market-scanner access; without it, the scanner tools are missing.
- **The MCP server enabled** on the portfolio group (*Enable MCP Server* in the group's *MCP Server Options*).
- **Web search** enabled in your agent.
- **An IB account, connected.** Other providers don't scan.
- **The watchlist market scanner saved** on the portfolio (*Add from Market Scanner…* in its Watchlist panel): scan code (e.g.
  *Top % Gainers*), location (e.g. `STK.US.MAJOR`) and base filters such as a minimum daily dollar volume. `run_scanner` refuses to
  run without one. The sweep adds its price band per batch.
- **Watchlist replace mode on:** each new band trims the previous band's visible names down to the pool's size; hidden winners stay.
- **"Trigger Scanner" off.** `run_scanner` refuses to run otherwise.
- **The watchlist scanner count about the batch size** (e.g. 40, less when TWS market-data lines are tight: each visible watchlist
  name takes one). A *List Limit* in the portfolio's MCP options overrides it; leave it empty unless the two should differ.

## Arguments

All optional, in the instructions (see *Arguments from InvMon* in `AGENTS.md`); the portfolio comes as the context:

- **N**, how many winners to collect. Default 10.
- **The bands:** a price range and a band width, or a list of `[lo, hi)` bands without gaps or overlaps. Default $5 to $100 in $5
  bands (19 bands). The floor keeps penny stocks out. The sweep stops at N winners, so a generous schedule costs nothing.
- **The winning ratings**, a set, not a threshold. Default `Strong Buy` and `Buy`; `Buy/adjust` doesn't count unless named.
- **The batch size**, passed as `count`. Default: the watchlist scanner count.

Pass the portfolio to `run_scanner` (`portfolioId`). Without one, `run_scanner` works only in a group with one portfolio.

## Workflow

1. **Plan.** `list_portfolios` for the horizon. Form the ordered list of bands, N and the winning set.
2. **Per band:**
   1. `run_scanner(priceAbove=lo, priceBelow=hi, count=…)`. No instruments: next band.
   2. Rate each instrument of the batch by the method (sub-agents if your agent has them). Skip those the method says to skip.
   3. One `update_ratings` call for the batch.
   4. Hide the batch's winners (`hide_instruments`), add them to your winners, and keep the band's full id list.
   5. N winners reached: stop. Otherwise the next band. Don't archive losers in between: replace mode trims them at the next scan.
3. **Clean up** from the ids you kept, no listing needed: archive the last band's ids minus its winners (the only visible names left,
   including any you skipped), then un-hide all winners.
4. **Summarize:** the winners (symbol, rating, price target, one-line reason), the horizon, whether the ratings were applied or only
   recorded, the bands swept and about how many names rated, the skipped names and why, and the shortfall when the schedule ran out
   before N.

## Edge cases

- **Empty bands**, mostly at the extremes: skip them.
- **Schedule exhausted before N:** report the shortfall. Never widen the winning set to make up the number.
- **Why bands, not a bigger `count`:** the IB scanner returns about 50 rows per query at most; a higher `count` doesn't get more.
- **Trimmed names stay out** for the rest of the app session, so no later band re-rates them, also in a second sweep before InvMon
  restarts.
- **A price crossing a band edge** mid-run can put a name into two bands or none. Deduplication and the trim lock absorb it.
- **Instrument count:** each band's losers are trimmed, so the portfolio holds about one batch plus the winners.

## Running this skill

A one-shot sweep: it runs bands until it has N winners or the schedule ends, cleans up and exits. Don't repeat it. The MCP server runs
on `localhost`: your agent must run on the same computer as InvMon.
