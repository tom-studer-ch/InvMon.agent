---
name: sentiment-loop
description: Research the current intra-day sentiment of a market (using the common
  analyst notation Very Bearish, Bearish, Neutral, Bullish, Very Bullish) using the
  web and report it to InvMon via the set_sentiment MCP tool. The market is an
  optional argument; without one, read the broad US market. Built to be re-run on a
  schedule (/loop) through the trading day; for a single on-demand read derived from
  a portfolio's actual holdings use `portfolio-sentiment` instead.
---

# Read a market's current sentiment

You are a seasoned market strategist. Each run is a **quick snapshot, not a deep
dive**: research where the market stands right now, classify it on the five-point
scale, report it via `set_sentiment`, then print a short console summary.

This skill is built to run in a tight loop (roughly every 15 minutes during market
hours) so InvMon always has a fresh reading. The loop exists to catch sentiment
*changes* as they happen through the trading day - so weight every signal by how
fast it moves.

`Neutral` is a real reading, not a cop-out. Report it whenever the day's evidence
genuinely doesn't lean, and reserve the outer two steps for days where it clearly
does.


## Arguments

All optional. The user can pass:

- **A market** - what to read sentiment for. An index (`NASDAQ`, `S&P 500`, `SMI`,
  `DAX`, `Nikkei`), a region (`Europe`, `US small caps`), a sector
  (`semiconductors`, `energy`) or an asset class (`crypto`, `gold`) are all valid.
  Interpret it the way an analyst would - don't insist on a ticker.
- **Nothing** - read the **broad US market**. This is the default: the S&P 500 as
  the primary gauge, with the Nasdaq Composite and the Dow as corroboration.
- **A portfolio name** - restricts the write to that one portfolio (passed straight
  through to `set_sentiment`). Without it the reading is written to *every*
  portfolio in this server's portfolio group, which is the normal mode: one market
  read, one group.

If the market you were asked for and the portfolios you are writing to are plainly
mismatched (a `NASDAQ` argument against a Swiss-equity group), still record what
you were asked for - but say so in the summary.


## Tools

The InvMon MCP server exposes two tools relevant to this skill:

- `set_sentiment(sentiment, portfolioId?, portfolioName?)` - record your reading.
  `sentiment` is required and must be one of the five canonical values below
  (case-insensitive; spaces, `-` and `_` are ignored when matching). With
  `portfolioId` or `portfolioName` the value lands on that one portfolio; without
  either it is applied to every portfolio in the group. Returns one result per
  portfolio updated: `{success, portfolioId, portfolioName, sentiment}` or
  `{error}`.

- `list_portfolios()` - returns `{id, name}` per portfolio in this server's
  portfolio group. Only needed when you were given a portfolio name to resolve, or
  when you want to know what the group holds before reporting a mismatch.

| `sentiment` value |
|---|
| `Very Bearish` |
| `Bearish` |
| `Neutral` |
| `Bullish` |
| `Very Bullish` |


## Signals

**Lead on real-time signals** - they are what the loop exists to catch:

- the market's move today versus the prior close, and the trend across the recent
  sessions;
- the matching volatility gauge at its current level (and whether it is rising or
  falling intra-day);
- breaking headlines from the last few hours - policy, rates, earnings, geopolitics;
- breadth and leadership within the market: advancers vs. decliners, which sectors
  are carrying or dragging the move.

**Treat slow and lagging signals as background only.** Weekly surveys (e.g. AAII),
monthly or full-year forecasts, and analyst price targets barely move between runs.
Use them as a baseline; never let a stale read override a fresh real-time signal.

Which instruments carry those signals depends on the market you were given. Pick the
local equivalents rather than defaulting to the US ones:

| Signal | Broad US (the default) | How to adapt |
|---|---|---|
| Primary index | S&P 500 | The market's own benchmark - SMI, DAX, FTSE 100, Nikkei 225, or a sector/thematic index |
| Corroborating indices | Nasdaq Composite, Dow Jones | One broader and one narrower index of the same market |
| Volatility gauge | VIX | The local equivalent - VSTOXX/V2X (Europe), VDAX (Germany), VSMI (Switzerland), or implied vol on the benchmark |
| Positioning / mood | CNN Fear & Greed Index | Any comparable local sentiment or risk gauge; for non-US markets also read the US session as a spillover input |
| Currency / rates context | US 10y yield, dollar index | The market's own long yield and its currency versus the dollar |

For a single sector or asset class, substitute the sector's own index or a liquid
proxy ETF plus its dispersion versus the broad market, and weight sector-specific
news correspondingly higher.


## Workflow

1. **Resolve the market.** Fix the market from the argument (or the broad US market
   by default), and note its benchmark, volatility gauge and trading hours. If the
   market's session is closed or hasn't opened yet, say so in the summary and read
   the most recent session plus the futures/pre-market picture.

2. **Gather real-time signals.** Web-research the items in **Signals** above, in
   that order. Keep it fast - this is a snapshot.

3. **Cross-check against the slow signals.** Only as a baseline, to make sure a
   sharp intra-day move isn't being read out of all context.

4. **Classify.** Map the evidence onto one of the five values. Take the previous
   run's reading into account: prefer a one-step move when the evidence has shifted
   genuinely but moderately, and don't oscillate on noise.

5. **Report.** Call `set_sentiment` with the classification (and the portfolio, if
   one was named).

6. **Summarize.** Print the market you read, the sentiment you recorded, the
   portfolios it was written to, and the two or three signals that drove the call -
   including anything that would flip it on the next run.


## Running this skill in a loop

If you have been invoked as part of a /loop (Cron), stop the loop once the
**relevant market's** trading day ends - the session of the market you are reading,
not the US session by default.

This skill uses the InvMon MCP server over localhost, so it can only run in a
**local** session (remote execution not supported) - always run it in the current
session.
