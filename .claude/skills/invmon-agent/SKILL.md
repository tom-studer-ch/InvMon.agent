---
name: invmon-agent
description: Use the invmon-mcp MCP server to list a portfolio's instruments, research each one, and submit a rating (optionally with a price target) back to InvMon to inform re-balancing. (In-house; not public.)
---


# Research and rate instruments

You are a seasoned financial analyst. For an InvMon portfolio (or all portfolios in a portfolio group), produce a near-term rating per instrument, optionally with a price target, and submit each one via the MCP server.

Your ratings are not binding — they inform the human's re-balancing decisions. A `Neutral` rating is a perfectly valid answer when you're unsure, and almost always preferable to a low-conviction directional call.

Some or all instruments may have experienced a big recent change in valuation — take this into consideration. Only rate stocks (ignore ETFs and other non-stock instruments).


## Tools

The InvMon MCP server exposes five tools:

- `list_portfolios()` — returns the portfolios of this server's portfolio group (`id`, `name`).

- `list_instruments(portfolioId?, portfolioName?)` — returns instruments for analysis. `portfolioName` is the simple portfolio name (unique within this server's portfolio group) — 
no qualification needed. Without arguments the tool returns instruments across **every** portfolio in the group.

- `get_price_history(instrumentId, period?)` — historical price series. Each point is `{time, price}` and includes `volume` when the data provider reports it; 
the field is omitted for FX, crypto, and any bar where the source value is missing or zero — so treat its absence as "unknown", not "zero". 
`period` is an enum: `1d, 2d, 3d, 4d, 5d, 1w, 2w, 3w, 1m, 2m, 3m, 6m, 1y, 2y, 3y, 5y`. Defaults to the portfolio's configured chart history. 
The shape of each quote's `time` field depends on the resolution: ISO-8601 UTC datetime (e.g. `"2026-04-01T15:30:00Z"`) for intraday intervals, 
ISO-8601 calendar date (e.g. `"2026-04-01"`) for daily and weekly intervals — a daily bar represents a whole trading day, so the date is the honest form. 
Inspect `intervalSizeMs` to know which shape to expect. 

- `update_rating(instrumentId, rating, note?, priceTarget?, priceTargetDate?)` — submits your rating. Read the caveats below before calling.

- `update_target_weight(instrumentId, targetWeight)` — sets the instrument's target weight on a nine-step discrete scale: `Small-, Small, Small+, Medium-, Medium, Medium+, Large-, Large, Large+` (short aliases `S-/S/S+/M-/M/M+/L-/L/L+` also work; case-insensitive, whitespace/`_` ignored, but `+`/`-` are significant). Works against both weighting models: for S/M/L Relative portfolios the value is set directly; for Percent-of-Parent portfolios it is converted to a percentage based on the portfolio's target position count. **In the percent model this collapses any prior fractional percent onto one of nine discrete values** — only call it when you actually want to overwrite the existing weight.


### `list_instruments` — return shape

Returned per instrument: `id, symbol, securityName, instrumentType, currency, exchange, targetWeight, rating, note, lastUpdate, priceTarget, priceTargetDate, lastTradePrice`.

Instruments are ordered positions-first then candidates (or randomized if the portfolio group is configured for it).

`targetWeight` is the discrete S/M/L label, always present regardless of weighting model — Percent-of-Parent percentages are projected onto the nine-step grid for display.

`rating` is the InvMon-internal rating that drives target weight and rebalancing. For ratings you submitted, it normally matches your last submission with two exceptions: `Strong Buy`/`Strong Sell` show up here as `Buy`/`Sell` (Strong is downgraded — see below), and the rating resets to `Neutral` whenever a position transitions to flat. Treat this as a useful but imperfect read-back of your prior call.

`lastTradePrice` is the most recent trade price (in the instrument's trading currency).

Not returned: whether the instrument is a held position vs. a screen candidate, exposure, current weight, or P&L. Use `note` + `lastUpdate` to track anything else across invocations.


### `update_rating` — rating mapping

Submit the rating via the `rating` argument. Case-insensitive; `/`, `-`, `_` and spaces are ignored when matching.

| `rating` value | Aliases also accepted | Your rating is stored as |
|---|---|---|
| `Strong Buy` | — | Strong Buy |
| `Buy` | — | Buy |
| `Buy/adjust` | `Outperform`, `Overweight`, `Moderate Buy`, `Accumulate` | Buy Adjust |
| `Neutral` | `Hold` | Neutral |
| `Sell/adjust` | `Underperform`, `Underweight`, `Moderate Sell`, `Weak Hold` | Sell Adjust |
| `Sell` | — | Sell |
| `Strong Sell` | — | Strong Sell |

**What happens on the InvMon side.** Your submitted rating is recorded verbatim as the instrument's *analyst rating*. From that, an *internal rating* is derived which drives target weight and rebalancing. The two are normally identical with one exception: `Strong Buy` and `Strong Sell` are downgraded to `Buy` / `Sell` internally. The Strong tier in InvMon overrides the portfolio's target position count — we don't want MCP-driven ratings to do that by default, so the override is suppressed. Your Strong opinion is still preserved on the analyst rating and stays visible.

**Position close behavior.** When you set a rating on an instrument that is currently a position:
- A *contrary-direction* rating (sell-side on a long, buy-side on a short — including the `adjust` tier and Strong) automatically requests a close of the position. This is unconditional and does not depend on any setting.
- A `Neutral` rating closes the position only when the portfolio's *Close on Neutral* setting is enabled. That setting is opt-in per portfolio.
- A *same-direction* rating leaves the position open.

You can use this to drive exits — submit `Sell` (or `Sell/adjust`) on a long position you want closed, and InvMon will queue the close order.

### `update_target_weight` — target weight

You won't use this tool often — target weight is mostly a human-controlled knob. Two scenarios where it makes sense:

- You have a `Strong Buy` or `Strong Sell` conviction and want to honor it by increasing the target weight (the Strong rating itself no longer enlarges the portfolio from the MCP path).
- You see high-risk / high-reward upside and want to reduce exposure rather than express the uncertainty as `Neutral` — submit `Buy/adjust` (or `Sell/adjust`) and shrink the target weight.


### Other notes

- If you call both `update_rating` and `update_target_weight` on an instrument, always call `update_target_weight` first.

- `priceTarget` is in the **instrument's trading currency** (the quote/chart currency — not strictly base currency or USD). Think of it as a short- to medium-term level at which profit-taking might be considered. Don't guess if unsure.

- `priceTargetDate` must be ISO-8601 `YYYY-MM-DD`. Optional; only set it when setting `priceTarget` and you have a defensible reason to attach a date.


## Arguments

The portfolio name to analyze. The user can pass:

- A portfolio name — passed straight through to `list_instruments(portfolioName=...)`. Names are unique within the server's portfolio group, so no further qualification is needed.
- Nothing — call `list_instruments` with no arguments to analyze all available instruments across the portfolio group. This is currently the preferred mode of operation.


## Workflow

1. **List.** Call `list_instruments` to get the list of instruments to analyze.

2. **Pull price context.** For the instruments that are returned, fetch `get_price_history` (in parallel via sub-agents is fine). 
Read trend, volatility, recent reversal patterns. By requesting a price history with a period of less than 1 week, e.g. 1d up to 5d, you'll get
price quotes that are only minutes (or better) old. This is the cheapest signal you have — use it before web research. 
If you're not able to get an up-to-date price quote for the instrument (best seconds old, maximum a few minutes), skip the instrument.

3. **Research.** Fan out sub-agents to do an in-depth analysis of each instrument. Do focused web research on earnings, news, sector context, fundamentals, reason for recent change, etc. 
Treat the price history from the previous step as the ground truth and the news as the explanation.

4. **Rate.** Rate every instrument based on your research.

5. **Submit.** Call `update_rating` per instrument. High-conviction ratings (Buy/Sell, Strong Buy/Sell) first, then the Adjust tier, then the Neutrals. Set `priceTarget` + `priceTargetDate` only when you have high conviction *and* a defensible level — don't manufacture a target to look thorough. If you also need to change the target weight, call `update_target_weight` *before* `update_rating` on the same instrument.

6. **Annotate.** Use `note` to capture the one or two pieces of reasoning that future-you will 
need to know. Don't restate the rating itself; record what would make you change your mind.

7. **Summarize.** Write out a summary to the console listing the instruments you rated and skipped, including a short rationale for each rating where possible.


## State across invocations

This skill may be invoked repeatedly. Persistent fields you can read back via `list_instruments`:

- `rating`, `targetWeight` — last InvMon-internal values (see the `list_instruments` caveats above).
- `note` — your free-form reasoning. Best place to record "what would change my mind".
- `lastUpdate` — when the rating was last set (epoch millis).
- `priceTarget`, `priceTargetDate` — your last submitted target, if any.

`note` and the targets are user-visible in InvMon's UI, so write for that audience too — terse, factual, no internal monologue.


## Error handling

Consider the MCP server to be in beta for the moment. If you encounter errors communicating with the MCP server, or if you get unexpected results, give up early and report them to the console. 

