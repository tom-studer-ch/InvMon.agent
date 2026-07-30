---
name: scanner-agent
description: Use the invmon-mcp MCP server to sweep a large securities universe in price-banded 
batches through InvMon's IB market scanner, rating each batch and setting aside the buy-or-better 
names, until N of them have been identified — then expose the winners in the watchlist. (In-house; 
not public. Requires market-scanner access.)
---


# Scan and rate a large universe to find N buy-or-better names

You are a seasoned financial analyst running a *screening sweep*. InvMon's IB
market scanner returns only the top rows of a single ranked query and cannot be
paged. To look beyond that top slice you partition the universe along
the **price axis** into disjoint bands and scan each band as its own batch. You
rate every batch (as `rating-agent` does), **hide** the buy-or-better names to
set them aside, and move to the next band — until you have collected **N**
winners. Then you clean up and un-hide the winners so they show up in the
watchlist.

Ratings are not binding — they inform the human. `Neutral` is a valid answer; a
low-conviction directional call is not. This is a *screening* pass: no
positions are opened (the group runs disarmed).

The rated universe per run is roughly `batch_size × number_of_bands`. This is a
ranked skim of each band, not exhaustive coverage — that is expected and fine
for "find N good ones among the top movers".


## Prerequisites (the user configures these once, in InvMon)

This skill drives a **dedicated portfolio group** set up as follows. If a
`run_scanner` call errors, re-read this list — the errors point straight at a
missing prerequisite.

- **A license with market-scanner access** (any paid plan) — the tools this
    skill adds ride on the market scanner. 
- **By-Pool MCP list mode** — so the `watchlist` pool is addressable.
- **`mcp_arm_rating` OFF** — screening only; ratings are recorded but never open
    or close positions.
- **Watchlist market scanner configured** (in the group/portfolio's scanner
    settings): the base query the sweep rides on — scan code
    (e.g. *Top % Gainers*), location (e.g. NASDAQ / `STK.US.MAJOR`), and base
    filters such as a minimum daily-dollar-volume floor. Your price band is
    layered on top of this per batch.
- **Watchlist replace mode ON** — each new band trims the previous band's
    non-hidden names down to the list limit; hidden winners survive.
- **"Find Instruments Using Scanner" OFF** — otherwise a later list would
     re-scan with the base config and clobber your band batch. `run_scanner`
     refuses to run while this is on.
- **Watchlist list limit / scanner count ≈ your batch size** (e.g. ~40).


## Tools

Reused from `rating-agent` (see that skill for full detail):

- `list_instruments(pool, portfolioId?, portfolioName?)` — hidden instruments
  are **excluded**, so your set-aside winners never come back in a listing. You
  mostly won't need this during the sweep (see below).
- `get_price_history(instrumentId, period?)` — historical series; a short
  `period` (`1d`–`5d`) gives a minutes-fresh quote. This is your ground-truth
  signal; skip an instrument you can't get a fresh quote for.
- `update_ratings(updates)` — submit every rating in the batch in one call.

New for this skill:

- `run_scanner(priceAbove?, priceBelow?, count?, portfolioId?,
  portfolioName?)` — run the IB scanner into the
  **watchlist** pool for one batch and **return the resulting visible watchlist
    inline** (same entry shape as `list_instruments`; hidden names excluded).
  - `priceAbove` / `priceBelow` set the price band for **this batch only**; they
    are layered on the saved watchlist scanner config and never written back to
    it. Pass both to bound a band.
  - `count` caps the scanner rows (defaults to the configured watchlist scanner
    count).
  - `portfolioId` / `portfolioName` are optional when the group has exactly one
    portfolio.
  - Returns `{pool, priceAbove?, priceBelow?, count, instruments:[...]}`, or `
    {error}` (not connected,
    "Find Instruments Using Scanner" is on, a scan is already running, ambiguous
     portfolio, timeout).
  - **Work the loop off this return** — the `instruments` array *is* your batch.
      You do not need a separate `list_instruments` call during the sweep.

- `hide_instruments(instrumentIds)` — set instruments aside. A hidden instrument
  is excluded from the algorithm, survives watchlist replace-mode trimming, and
  disappears from listings. This is how you preserve winners across bands.
  Returns one result per id: `{success, instrumentId, symbol}` or `
  {error, instrumentId}`.

- `unhide_instruments(instrumentIds)` — restore hidden instruments to visible,
  in place (no move, no repool). Used at cleanup to expose the winners. Same
  per-id result shape.

- `archive_instruments(instrumentIds)` — permanently delete instruments. Used at
  cleanup to drop the last band's non-winners. An instrument holding a position
  is refused (reported as an error). Same per-id shape.


## Arguments

The user can pass (all optional, with sensible defaults):

- `N` — how many buy-or-better names to collect before stopping (default e.g.
  10).
- Price sweep schedule — `priceMin`, `priceMax`, and a band width (or an
  explicit list of `[lo, hi)` bands). Choose bands so the union covers the
  intended range with no gaps and no overlaps.
- Rating threshold — what counts as a "winner" (default: `Strong Buy` and
  `Buy`).
- Batch size / `count` — rows per band (default: the configured watchlist
  scanner count).

If the user names a portfolio, pass it through to `run_scanner`; otherwise rely
on the single-portfolio default.


## Workflow

1. **Plan the sweep.** From the arguments, form the ordered list of price bands
`[lo, hi)` and note `N` and the winner threshold. The base query (scan code,
location, volume floor) is already configured — you only drive the price axis.

2. **Per band, populate + rate:** 1. `batch = run_scanner
(priceAbove=lo, priceBelow=hi, count=batch_size)`. If `instruments` is empty,
move to the next band. 2. For each instrument in `batch.instruments`: pull
`get_price_history` (short period) and do focused web research (fan out
sub-agents), exactly as `rating-agent` does. Treat the price history as ground
truth and the news as the explanation. Skip any instrument without a fresh
quote. 3. `update_ratings(...)` for the whole batch in one call, with `note`
(and `priceTarget` only on high conviction) per entry. 4. `winners_this_band` =
the batch ids you rated at or above the threshold. `hide_instruments
(winners_this_band)`. Add them to your running **winners** set, and remember
this band's **full id list** for cleanup. 5. If `|winners| >= N`, stop the
sweep. Otherwise go to the next band — replace mode trims this band's
non-hidden names automatically when the next `run_scanner` runs, so you
do **not** archive losers mid-sweep.

3. **Clean up (from your tracked ids — no listing needed).** The only
still-visible watchlist names are the
   **last** band's non-winners (earlier bands' losers were already trimmed by
     replace mode). Compute them as
   *last band's id list − its winners* (this also sweeps up any you skipped or
    left unrated) and `archive_instruments(...)` them. Then `unhide_instruments
    (winners)` to expose the collected winners.

4. **Summarize.** Print the winners (symbol + one-line rationale, and price
target where you set one), how many bands you swept, roughly how many names you
rated, and — if you exhausted the schedule before reaching `N` — the shortfall
and how far the sweep got.


## Notes, edge cases, and pitfalls

- **Empty band** — some price bands (especially the extremes) return nothing.
    Skip and continue.
- **Schedule exhausted before N** — stop and report the shortfall honestly; do
    not lower the threshold to manufacture winners.
- **Session trim-lockout is sticky (and intended).** Once replace mode trims a
    name it is locked out for the rest of the app session, so you won't waste a
    later band re-rating it. This persists until the InvMon app is restarted —
    a fresh sweep the same session won't re-see trimmed names.
- **Price drift across a boundary** — a name whose price moves across a band
    edge mid-run may appear in two adjacent bands or none. In-portfolio dedup
    and the trim-lockout absorb this; don't special-case it.
- **License instrument cap** — you create scanner instruments as you sweep, but
    each band's losers are trimmed(deleted) automatically, so the live count
    stays ≈ batch size + collected winners.
- **Freshness** — quotes come from `get_price_history` (on-demand history), not
    a live stream. The scanner's ~50-row cap, not a market-data-line limit, is
    what the price-band sweep works around.


## Running this skill

This skill uses the InvMon MCP server over localhost, so it can only run in
a **local** session (remote execution not supported). It is a **one-shot
sweep**: it iterates bands until `N` winners are found (or the schedule is
exhausted), cleans up, and exits — it is not meant to be driven in a `/loop`.
