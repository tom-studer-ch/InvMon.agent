---
name: portfolio-sentiment
description: Work out which market an InvMon portfolio actually trades against, research
  that market's current sentiment (Very Bearish, Bearish, Neutral, Bullish, Very
  Bullish) on the web, and record it on the portfolio via the set_sentiment MCP tool.
  One-shot; for a reading kept fresh through the trading day use `sentiment-loop`
  instead.
---

# Read the sentiment of the market a portfolio trades against (one-shot)

You are a seasoned market strategist. For **one** InvMon portfolio, determine the
market it is actually exposed to, research that market's current sentiment, classify
it on the five-point analyst scale, and record it on that portfolio via
`set_sentiment`.

The difference from `sentiment-loop` is where the market comes from: there it is an
argument (defaulting to the broad US market); here you **derive it from what the
portfolio holds**. Everything downstream - the signals, the scale, the write - is the
same.

This is a single pass, so it can be a proper read rather than a snapshot: look at the
portfolio's actual exposure, not just the headline index.

`Neutral` is a real reading, not a cop-out. Report it whenever the evidence genuinely
doesn't lean.


## Tools

- `list_portfolios()` - returns `{id, name}` per portfolio in this server's portfolio
  group. Use it to resolve the portfolio when none was named.

- `list_instruments(portfolioId?, portfolioName?, pool?)` - the portfolio's instruments;
  this is what you derive the market from. Each entry carries `symbol`,
  `securityName`, `instrumentType`, `currency`, `exchange`, `lastTradePrice` and
  `lastTradeTimestamp`, which between them tell you the venue, currency and asset-class
  mix. Pass `pool: "positions"` to read the actual exposure, and fall back to
  `pool: "candidates"` when the portfolio holds no positions yet.

- `get_price_history(instrumentId, period?)` - optional here. A short `period`
  (`1d`-`5d`) on two or three of the portfolio's largest names is a quick sanity
  check that what you read in the market is what the portfolio is actually feeling.

- `set_sentiment(sentiment, portfolioId?, portfolioName?)` - record the reading.
  `sentiment` is required and must be one of the five canonical values below
  (case-insensitive; spaces, `-` and `_` are ignored). **Always pass the portfolio**
  in this skill - without `portfolioId`/`portfolioName` the value is applied to
  *every* portfolio in the group, which is not what a per-portfolio read should do.
  Returns one result per portfolio updated: `{success, portfolioId, portfolioName,
  sentiment}` or `{error}`.

| `sentiment` value |
|---|
| `Very Bearish` |
| `Bearish` |
| `Neutral` |
| `Bullish` |
| `Very Bullish` |


## Arguments

All optional. The user can pass:

- **A portfolio name** - the portfolio to read and write. Names are unique within the
  server's portfolio group.
- **Nothing** - call `list_portfolios()`. One portfolio in the group: use it. Several:
  read and record each one in turn (they may well sit in different markets), and
  report them separately. If the user prefers a single portfolio, they will say so.
- **A market override** - e.g. `/portfolio-sentiment Long NASDAQ`. Then skip the
  derivation and read that market, but still write only to the named portfolio. Say in
  the summary that the market was given rather than derived, and flag it if it
  contradicts what the portfolio holds.


## Deriving the market

From the instrument list, work out what the portfolio is actually exposed to:

- **Venue and currency mix** - `exchange` and `currency` across the holdings. A
  portfolio that is overwhelmingly NASDAQ/USD reads against the Nasdaq Composite; one
  on SIX/CHF reads against the SMI; a spread across venues reads against the region,
  or against the broad US market when US names dominate.
- **Asset class** - `instrumentType`. Equities, FX, crypto and commodities have
  different sentiment gauges; a portfolio that is mostly one of them should be read on
  that asset class's own gauge, not on an equity index.
- **Sector concentration** - `securityName` plus what you know of the symbols. If the
  holdings cluster in one sector (semis, energy, banks), read the sector alongside the
  broad market and say which one drove the call.
- **Direction** - a portfolio configured predominantly short still gets an honest read
  of the market. Do **not** invert the sentiment to match the portfolio's direction;
  InvMon interprets the sentiment relative to the portfolio's own configuration.

Where the portfolio is genuinely mixed and no single market dominates, read the
broadest market that covers most of the exposure (typically the broad US market) and
say so.


## Signals

**Lead on real-time signals:** the market's move today versus the prior close and the
recent session trend; the matching volatility gauge and its direction; breaking
headlines from the last few hours; breadth and sector leadership.

**Treat slow and lagging signals as background only:** weekly surveys (e.g. AAII),
monthly or full-year forecasts, and analyst price targets. Use them as a baseline;
never let a stale read override a fresh real-time signal.

Pick the local instruments for the market you derived rather than the US defaults:

| Signal | Broad US | How to adapt |
|---|---|---|
| Primary index | S&P 500 | The market's own benchmark - Nasdaq Composite, SMI, DAX, FTSE 100, Nikkei 225, or a sector index |
| Corroborating indices | Nasdaq Composite, Dow Jones | One broader and one narrower index of the same market |
| Volatility gauge | VIX | VSTOXX/V2X (Europe), VDAX (Germany), VSMI (Switzerland), or implied vol on the benchmark |
| Positioning / mood | CNN Fear & Greed Index | A comparable local gauge; for non-US markets also read the US session as a spillover input |
| Currency / rates context | US 10y yield, dollar index | The market's own long yield and its currency versus the dollar |

Because this is a one-shot read, also weigh the portfolio's own exposure: if its
largest holdings are diverging from the benchmark, that divergence belongs in the
call and in the summary.


## Workflow

1. **Resolve the portfolio** per **Arguments**, then `list_instruments` for it.

2. **Derive the market** from the holdings as described above. If the portfolio is
   empty, fall back to the broad US market and say so.

3. **Research.** Gather the real-time signals for that market, then cross-check
   against the slower ones. Optionally pull `get_price_history` on the largest
   holdings as a reality check. If the market's session is closed, read the most
   recent session plus the futures/pre-market picture and say so.

4. **Classify.** Map the evidence onto one of the five values.

5. **Record.** Call `set_sentiment` with the classification **and** the portfolio.

6. **Summarize.** Print, per portfolio: the market you derived and what in the
   holdings pointed there, the sentiment recorded, the two or three signals that drove
   it, and what would flip it. State whether the read used live intra-day data or the
   last close.


## Running this skill

This skill uses the InvMon MCP server over localhost, so it can only run in a
**local** session (remote execution not supported) - always run it in the current
session.

It is a **one-shot read**: it records the sentiment once and exits. To keep a
portfolio's sentiment current through the trading day, use `sentiment-loop` under
`/loop` instead of driving this skill repeatedly.
