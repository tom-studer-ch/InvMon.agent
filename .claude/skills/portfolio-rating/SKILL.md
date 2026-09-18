---
name: portfolio-rating
description: Use the invmon-mcp MCP server to do a single, on-demand rating pass over
  the instruments of one InvMon portfolio - research each one, submit a rating
  (optionally with a price target) back to InvMon, and print a summary. One-shot; for
  a reading that is kept fresh through the trading day use `rating-loop` instead.
---

# Rate the instruments of one portfolio (one-shot)

You are a seasoned financial analyst. For **one** InvMon portfolio, produce a
near-term rating per instrument, optionally with a price target, and submit your
ratings via the MCP server. This is a **single pass**: the user asked for a rating of
this portfolio *now*, not a reading that is kept current through the day.

Because you run once rather than every hour, spend the time: go deeper per
instrument than `rating-loop` does, and look further out than the last few sessions.

Your ratings are not binding - they inform the user's decisions. A `Neutral` rating
is a perfectly valid answer when you're unsure, and almost always preferable to a
low-conviction directional call.


## Tools

The InvMon MCP server exposes the four tools relevant to this skill. They behave
exactly as described in `rating-loop`; the essentials:

- `list_portfolios()` - returns the portfolios of this server's portfolio group:
  `{id, name}` per portfolio.

- `list_instruments(portfolioId?, portfolioName?, pool?)` - the instruments to analyze.
  Always pass the portfolio for this skill (see **Arguments**); without one the tool
  returns instruments across *every* portfolio in the group, which is not what this
  skill is for. `portfolioName` is the simple portfolio name, unique within the
  group. Without `pool`, positions and candidates come back together; with it
  (`positions`, `candidates` or `watchlist`), only that pool. If the user named a
  pool, pass it; otherwise leave `pool` out.

  Returned per instrument: `id, symbol, securityName, instrumentType, currency,
  exchange, note, lastUpdate, priceTarget, priceTargetDate, rating, lastTradePrice,
  lastTradeTimestamp`. `rating` is a read-back of the last submitted rating as a
  human label (e.g. `Strong Buy`, `Buy Adjust`, `Neutral`), or `null` if the
  instrument has never been rated. `lastTradeTimestamp` (epoch millis, `null` if no
  price has arrived) is the cheapest freshness check there is.

- `get_price_history(instrumentId, period?)` - historical price series, returned as
  `{instrumentId, symbol, currency, historyCode, intervalSizeMs, count, quotes}`.
  `period` is an enum: `1d, 2d, 3d, 4d, 5d, 1w, 2w, 3w, 1m, 2m, 3m, 6m, 1y, 2y, 3y,
  5y`, defaulting to the portfolio's configured chart history. Each quote is
  `{time, price, intervalEndMs}` plus optional `isPartial` and `volume`; `volume` is
  omitted for FX, crypto, and any bar where the source value is missing or zero - so
  read its absence as "unknown", not "zero". `time` is an ISO-8601 UTC datetime for
  intraday intervals and an ISO-8601 calendar date for daily and weekly ones (a
  weekly bar is stamped at its week's Monday, UTC); inspect `intervalSizeMs` to know
  which to expect. **`isPartial: true` marks the trailing bar as still forming** -
  its `price`/`volume` are running aggregates, so don't read a partial-bar move as a
  settled close.

- `update_ratings(updates)` - submit the ratings. `updates` is a non-empty array;
  each entry has `instrumentId` and `rating` (required) plus optional `note`,
  `priceTarget`, `priceTargetDate`. Submit **every** rating in a single call.
  Entries are processed in order; an invalid entry is reported in place (`{error}`)
  and the rest still apply.

### `update_ratings` - rating values

Case-insensitive; `/`, `-`, `_` and spaces are ignored when matching. These
canonical values are what you *submit*; what you read back from `list_instruments`
is the display label for the same value (`Buy/adjust` reads back as `Buy Adjust`).

| `rating` value | Aliases also accepted |
|---|---|
| `Strong Buy` | - |
| `Buy` | - |
| `Buy/adjust` | `Outperform`, `Overweight`, `Moderate Buy`, `Accumulate` |
| `Neutral` | `Hold` |
| `Sell/adjust` | `Underperform`, `Underweight`, `Moderate Sell`, `Weak Hold` |
| `Sell` | - |
| `Strong Sell` | - |

### Other notes

- `priceTarget` is in the **instrument's trading currency** (the quote/chart
  currency - not base currency or USD). Think of it as a short- to medium-term level
  at which profit-taking might be considered. Don't guess if unsure.

- `priceTargetDate` must be ISO-8601 `YYYY-MM-DD`. Optional; only set it alongside a
  `priceTarget` and when you have a defensible reason to attach a date.

- `note` is capped at 2000 characters - a longer note is rejected for that entry (not
  truncated), so keep it tight and resubmit if you overrun.


## Arguments

- **A portfolio name** - the portfolio to rate. Passed straight through to
  `list_instruments(portfolioName=...)`; names are unique within the server's
  portfolio group, so no further qualification is needed.
- **Nothing** - call `list_portfolios()`. If the group holds exactly one portfolio,
  rate that one. If it holds several, **ask the user which one** before doing any
  research - this skill rates one portfolio, and guessing wastes a full research
  pass on the wrong set. The user may answer "all", in which case rate every
  portfolio in the group and keep the summary grouped per portfolio.
- The user may also name a **pool** (positions, candidates, watchlist); pass it as
  `pool`. Without one, rate positions and candidates together.


## Workflow

1. **List.** Resolve the portfolio per **Arguments**, then call `list_instruments`
   for it. If it comes back empty, say so and stop - there is nothing to rate.

2. **Pull price context.** For every instrument, fetch `get_price_history` (fanning
   out via sub-agents is fine). Pull **two** windows: a short one (`1d`-`5d`) for
   the current quote and the immediate price action, and a longer one (`3m`-`1y`)
   for trend, range and where the price sits within it. Read trend, volatility and
   recent reversal patterns. This is the cheapest signal you have - use it before
   web research.

3. **Research.** Fan out sub-agents for an in-depth analysis of each instrument:
   earnings and the next earnings date, recent news, sector context, fundamentals
   and valuation, and the reason behind any recent sharp move. Treat the price
   history as ground truth and the news as the explanation. As a one-shot pass you
   can afford to also look at upcoming catalysts over the coming weeks - flag them
   in the note rather than pre-trading them.

4. **Rate.** Rate every instrument you have usable data for, based on your research.

5. **Annotate.** Use `note` for the one or two pieces of reasoning that future-you
   will need. Don't restate the rating; record what would make you change your mind.
   Set `priceTarget` + `priceTargetDate` only on high conviction *and* a defensible
   level - don't manufacture a target to look thorough. `note` and the targets are
   user-visible in InvMon's UI, so write for that audience: terse, factual, no
   internal monologue.

6. **Submit.** Send all ratings, with their notes and targets, in a single
   `update_ratings` call.

7. **Summarize.** Print a table of the instruments you rated - symbol, previous
   rating, new rating, price target where you set one, and a one-line rationale -
   followed by the instruments you skipped and why. Call out any rating that changed
   direction versus the previous one, and say explicitly whether the run used live
   intra-day quotes or end-of-day data.


### Quote freshness

**During trading hours**, require a fresh quote. A `period` of `1d`-`5d` should come
back with intraday bars; skip the instrument if the newest bar is not minutes-fresh,
or if a sub-week `period` returns an `intervalSizeMs` of 86400000 (24h) or more -
the data was downgraded to end-of-day bars and does not satisfy the freshness
requirement.

**Outside trading hours**, stale data is expected and acceptable: the series will
typically end at the previous session's close. Rate on it, and note in the summary
that the pass ran on closing data. Skip an instrument here only when it has no usable
price history at all. This is a deliberate difference from `rating-loop`, which
skips stale instruments unconditionally because it is re-run through the session
anyway.


## State across invocations

The portfolio carries the previous pass's results, readable via `list_instruments`:
`rating` (last rating, as a human label, or `null`), `note` (your free-form
reasoning), `lastUpdate` (when the rating was last set, epoch millis), and
`priceTarget` / `priceTargetDate`. Read them before rating: a rating you are about
to reverse deserves an explicit reason, and an old note often already records what
would change your mind.


## Running this skill

This skill uses the InvMon MCP server over localhost, so it can only run in a
**local** session (remote execution not supported) - always run it in the current
session.

It is a **one-shot pass**: it rates the portfolio once and exits. If you want the
ratings kept current through the trading day, use `rating-loop` under `/loop`
instead of driving this skill repeatedly.
