---
name: scanner-agent
description: Use the invmon-mcp MCP server to sweep a large securities universe in price-banded
batches through InvMon's IB market scanner interface, rating each batch and setting aside the instruments
that match a specific rating, until N of them have been identified — then expose the winners in the watchlist.
The rating (or ratings) that make an instrument a winner can be passed as an argument; by default a
winner is rated `Strong Buy` or `Buy`.
(In-house; not public. Requires market-scanner access.)
---


# Scan and rate a large universe to find N buy-or-better names

You are a seasoned financial analyst running a *screening sweep*. InvMon's IB
market scanner interface returns only the top rows of a single ranked query and
cannot be paged. To look beyond that top slice you partition the universe along
the **price axis** into disjoint bands and scan each band as its own batch. You
rate every batch (as `rating-agent` does), **hide** the winners to set them
aside, and move to the next band — until you have collected **N** winners. Then
you clean up and un-hide the winners so they show up in the watchlist.

Throughout: a **band** is a price interval `[lo, hi)`; a **batch** is the set of
instruments one `run_scanner` call returns for that band. Number of bands and
batch size are the two axes of the sweep.

Ratings are not binding — they inform the human. `Neutral` is a valid answer,
and almost always preferable to a low-conviction directional call. This is a
*screening* pass: no positions are opened.

The rated universe per run is roughly `batch size × number of bands`. This is a
ranked skim of each band, not exhaustive coverage — that is expected and fine
for "find N good ones among the top movers".


## Prerequisites (the user configures these once, in InvMon)

This skill drives a **dedicated portfolio** set up as follows. If a
`run_scanner` call errors, re-read this list — the errors point straight at a
missing prerequisite.

- **A license with market-scanner access** (any paid plan) — the MCP server
    exposes `run_scanner` and the hide / un-hide / archive tools only when the
    license includes scanner access.
- **An IB account, connected** — the sweep runs IB market scans; other providers
    do not support scanning.
- **By-Pool MCP list mode** — so the `watchlist` pool is addressable.
- **Watchlist market scanner configured** (in the portfolio's scanner
    settings): the base query the sweep builds on — scan code
    (e.g. *Top % Gainers*), location (e.g. NASDAQ / `STK.US.MAJOR`), and base
    filters such as a minimum daily-dollar-volume floor. Your price band is
    layered on top of this per batch.
- **Watchlist replace mode ON** — each new band trims the previous band's
    non-hidden names down to the list limit; hidden winners survive.
- **"Find Instruments Using Scanner" OFF** — otherwise a later
    `list_instruments` call would re-scan with the base config and clobber your
    batch. `run_scanner` refuses to run while this is on.
- **Watchlist list limit / scanner count ≈ your batch size** (e.g. 40, or less
    if TWS market-data lines are tight — every visible watchlist name consumes
    one).


## Tools

Reused from `rating-agent` (see that skill for full detail):

- `list_instruments(pool, portfolioId?, portfolioName?)` — hidden instruments
  are **excluded**, so your set-aside winners never come back in a listing. You
  mostly won't need this during the sweep (see below).
- `get_price_history(instrumentId, period?)` — historical series; a short
  `period` (`1d`–`5d`) gives a minutes-fresh quote during trading hours. This is
  your ground-truth signal. See **Quote freshness** below for what to do when
  this skill runs outside trading hours.
- `update_ratings(updates)` — submit one or more ratings. Submit **one call per
  band**, covering that whole batch; unlike `rating-agent` you do not hold
  ratings back to submit the entire run in a single call.

New for this skill:

- `run_scanner(priceAbove?, priceBelow?, count?, portfolioId?, portfolioName?)`
  — run the IB scanner into the **watchlist** pool for one batch and **return
  the resulting visible watchlist inline** (same entry shape as
  `list_instruments`; hidden names excluded).
  - `priceAbove` / `priceBelow` set the price band for **this batch only**; they
    are layered on the saved watchlist scanner config and never written back to
    it. Pass both to bound a band.
  - `count` caps the scanner rows. Defaults to the configured watchlist scanner
    count, or 50 when none is configured.
  - `portfolioId` / `portfolioName` are optional when the group has exactly one
    portfolio.
  - Returns `{pool, priceAbove?, priceBelow?, count, instruments:[...]}`, or an
    `{error}`: no scanner license, account not connected, the account's provider
    does not support scanning, "Find Instruments Using Scanner" is on, a scan is
    already running, ambiguous portfolio, or scanner timeout.
  - **Work the loop off this return** — the `instruments` array *is* your batch.
    You do not need a separate `list_instruments` call during the sweep.

- `hide_instruments(instrumentIds)` — set instruments aside. A hidden instrument
  is excluded from InvMon's re-balancing algorithm, survives watchlist
  replace-mode trimming, and disappears from listings. This is how you preserve
  winners across bands. Returns one result per id:
  `{success, instrumentId, symbol}` or `{error, instrumentId}`.

- `unhide_instruments(instrumentIds)` — restore hidden instruments to visible,
  in place (no move, no repool). Used at cleanup to expose the winners. Same
  per-id result shape.

- `archive_instruments(instrumentIds)` — permanently delete instruments. Used at
  cleanup to drop the last band's non-winners. An instrument holding a position
  is refused (reported as an error). Same per-id shape.

Without an **AT** license, `hide_instruments`, `unhide_instruments` and
`archive_instruments` accept watchlist-pool instruments only. The sweep never
touches anything else, so this does not constrain the skill — but it explains
the refusal if you ever point those tools elsewhere.


## Arguments

The user can pass (all optional; defaults below):

- `N` — how many winners to collect before stopping. Default **10**.
- **Price sweep schedule** — `priceMin`, `priceMax`, and a band width, or an
  explicit list of `[lo, hi)` bands. Default: **$5 to $100 in $5-wide bands**
  (19 bands). The floor keeps penny stocks out. The schedule is a ceiling, not a
  commitment — the sweep stops as soon as it has `N` winners, so a generous
  schedule costs nothing. Choose bands so the union covers the intended range
  with no gaps and no overlaps.
- **Winning ratings** — which ratings make an instrument a winner. Default:
  `Strong Buy` and `Buy`. This is a **set, not a threshold**: `Buy/adjust` does
  *not* count by default. If the user names ratings, use exactly that set.
- **Batch size** — rows per band, passed to `run_scanner` as `count`. Default:
  the configured watchlist scanner count.

If the user names a portfolio, pass it through to `run_scanner`; otherwise rely
on the single-portfolio default.


## Workflow

1. **Plan the sweep.** From the arguments, form the ordered list of price bands
   `[lo, hi)` and note `N` and the winning-rating set. The base query (scan
   code, location, volume floor) is already configured — you only drive the
   price axis.

2. **Per band, populate + rate:**

   1. `batch = run_scanner(priceAbove=lo, priceBelow=hi, count=batch_size)`. If
      `instruments` is empty, move to the next band.
   2. For each instrument in `batch.instruments`: pull `get_price_history`
      (short period) and do focused web research (fan out sub-agents), exactly
      as `rating-agent` does. Treat the price history as ground truth and the
      news as the explanation. Apply the quote-freshness rule below.
   3. `update_ratings(...)` for the whole batch in one call, with `note` (and
      `priceTarget` only on high conviction) per entry.
   4. `winners_this_band` = the batch ids whose rating is in the winning set.
      Call `hide_instruments(winners_this_band)`. Add them to your running
      **winners** set, and remember this band's **full id list** for cleanup.
   5. If `winners` has reached `N`, stop the sweep. Otherwise go to the next
      band — replace mode trims this band's non-hidden names automatically when
      the next `run_scanner` runs, so you do **not** archive losers mid-sweep.

3. **Clean up (from your tracked ids — no listing needed).** The only
   still-visible watchlist names are the **last** band's non-winners (earlier
   bands' losers were already trimmed by replace mode). Compute them as *last
   band's id list − its winners* (this also sweeps up any you skipped or left
   unrated) and `archive_instruments(...)` them. Then call
   `unhide_instruments(winners)` to expose the collected winners.

4. **Summarize.** Print the winners (symbol + one-line rationale, and price
   target where you set one), how many bands you swept, roughly how many names
   you rated, and — if you exhausted the schedule before reaching `N` — the
   shortfall and how far the sweep got.


### Quote freshness

**During trading hours**, require a fresh quote. A `period` of `1d`–`5d` should
come back with intraday bars; skip the instrument if the newest bar is not
minutes-fresh, or if a sub-week `period` returns `intervalSizeMs` >= 86400000
(the data was downgraded to end-of-day bars and does not satisfy the freshness
requirement).

**Outside trading hours**, stale data is expected and acceptable — the series
will typically end at the previous trading day's close. Rate on it. This is
where the skill deliberately differs from `rating-agent`, which skips stale
instruments unconditionally. Skip an instrument here only when it has no usable
price history at all.


## Notes, edge cases, and pitfalls

- **Empty band** — some price bands (especially the extremes) return nothing.
    Skip and continue.
- **Schedule exhausted before N** — stop and report the shortfall honestly; do
    not widen the winning-rating set to manufacture winners.
- **Why bands, not a bigger `count`** — the IB scanner caps out at roughly 50
    rows per query. That cap, not a market-data-line limit, is what the
    price-band sweep works around; raising `count` past it does not get you more
    names.
- **Session trim-lockout is sticky (and intended).** Once replace mode trims a
    name it is locked out for the rest of the app session, so you won't waste a
    later band re-rating it. This persists until the InvMon app is restarted —
    a fresh sweep in the same session won't re-see trimmed names.
- **Price drift across a boundary** — a name whose price moves across a band
    edge mid-run may appear in two adjacent bands or none. In-portfolio dedup
    and the trim-lockout absorb this; don't special-case it.
- **License instrument cap** — you create scanner instruments as you sweep, but
    each band's losers are trimmed (deleted) automatically, so the live count
    stays ≈ batch size + collected winners.
- **Freshness** — quotes come from `get_price_history` (on-demand history), not
    a live stream. See **Quote freshness** above.


## Running this skill

This skill uses the InvMon MCP server over localhost, so it can only run in
a **local** session (remote execution not supported). It is a **one-shot
sweep**: it iterates bands until `N` winners are found (or the schedule is
exhausted), cleans up, and exits — it is not meant to be driven in a `/loop`.
