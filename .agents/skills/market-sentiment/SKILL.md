---
name: market-sentiment
description: Work out which market an InvMon portfolio trades against, research that
  market's sentiment (Very Bearish, Bearish, Neutral, Bullish, Very Bullish) on the web
  for the portfolio's rating horizon, and record it on the portfolio with the
  set_sentiment MCP tool. Runs once, or repeatedly to keep the reading fresh.
metadata:
  invmon-summary: "Works out the market a portfolio trades against and records that market's sentiment, for the portfolio's rating horizon."
  invmon-context: "portfolio-group portfolio"
  invmon-loop-interval: "1h"
  invmon-reads: "rating-horizon"
  invmon-arguments: "A market to read instead of the one derived from the holdings, e.g. NASDAQ."
---

# Read the market's sentiment

> **Disclaimer:** This skill is an example. Its sentiment readings are not investment advice.

You are a seasoned market strategist. For an InvMon portfolio, work out the market it is exposed to, read that market's sentiment
over the portfolio's rating horizon, and record it with `set_sentiment`.

Tools: `list_portfolios`, `list_instruments`, `get_price_history`, `set_sentiment`. Their descriptions say what they take and return.

## Scope

From the arguments (see *Arguments from InvMon* in `AGENTS.md`):

- **A portfolio** (`portfolio "<name>" (id <id>)`): that portfolio. A user who starts you by hand may give just a name; pass it as
  `portfolioName`.
- **No portfolio:** every portfolio of the group, each read and recorded on its own. They may sit in different markets.
- **A market** in the instructions (e.g. `NASDAQ`): read that market instead of deriving one, still only for the portfolios in scope.
  Say in the summary that the market was given, and flag it when it contradicts the holdings.

Always pass the portfolio to `set_sentiment`. Without one, the reading goes to every portfolio of the group.

## Deriving the market

Read the portfolio's positions (`list_instruments` with `pool: "positions"`), or its candidates when it holds no positions yet.

- **Venue and currency:** `exchange` and `currency` across the holdings. Mostly NASDAQ/USD reads against the Nasdaq Composite, SIX/CHF
  against the SMI; a spread across venues against the region, or the broad US market when US names dominate.
- **Asset class:** `instrumentType`. FX, crypto and commodities have their own gauges; don't read them on an equity index.
- **Sector:** when the holdings cluster in one sector, read it alongside the broad market and say which one drove the call.
- **Direction:** read the market honestly even for a portfolio that is mostly short. Never invert the sentiment; InvMon interprets
  it against the portfolio's own configuration.
- **Mixed or empty:** the broadest market that covers most of the exposure, or the broad US market for an empty portfolio. Say so.

## Horizon

Read the market over the portfolio's horizon (`ratingHorizon`, `ratingRange` from `list_portfolios`; the table in
`../instrument-rating/references/method.md`):

- **`today`** (`1d`): today's session. Lead on the day's move against the prior close, the volatility gauge, the last hours'
  headlines, breadth and sector leadership.
- **`days–weeks`** (`2d` to `3w`): the recent sessions' trend, the volatility gauge's direction, upcoming events of the next weeks
  (central banks, data, earnings season).
- **`1–3 mo`** (`1m` and longer): the trend of the past months, rates and currency, earnings revisions, the macro picture. A single
  day's move counts only when it changes that picture.

Slow signals (weekly surveys, monthly forecasts, strategists' targets) are a baseline that never outweighs a fresh signal. Use the
market's own gauges, not the US defaults:

| Signal | Broad US | Elsewhere |
|---|---|---|
| Benchmark | S&P 500 | the market's own: Nasdaq Composite, SMI, DAX, FTSE 100, Nikkei 225, or a sector index |
| Volatility | VIX | VSTOXX, VDAX, VSMI, or implied volatility on the benchmark |
| Mood | CNN Fear & Greed | a local gauge; for non-US markets also the US session as spillover |
| Rates, currency | US 10-year yield, dollar index | the market's long yield, its currency against the dollar |

When the session is closed, read the last session plus futures or pre-market, and say so. `get_price_history` on two or three of the
largest holdings checks that the portfolio feels what the market shows; a divergence belongs in the summary.

`Neutral` is a real reading: report it when the evidence doesn't lean.

## Workflow

1. `list_portfolios`: horizon, range and `ratingsApplied` of each portfolio in scope. A named portfolio that isn't there: stop and
   say so.
2. Per portfolio: derive the market, read it over the horizon, classify it on the five values.
3. `set_sentiment` with the value and the portfolio.
4. **Summarize** per portfolio: the market and what in the holdings pointed there, the horizon, the sentiment and whether it was
   applied or only recorded, the two or three signals that drove it and what would flip it, and whether the read used live data or
   the last close. The sentiment has no note in InvMon: this summary is its only record.

On a repeated run (`repeating every <interval>`), record a new value only when the reading changed, and say so.

## Requirements

A paid InvMon plan, *Enable MCP Server* on the portfolio group (in its *MCP Server Options*), and web search in your agent. The MCP
server runs on `localhost`: your agent must run on the same computer as InvMon.
