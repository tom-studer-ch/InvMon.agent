# Screening Agent — Set-Up Tutorial

This tutorial explains a possible scenario for using the scanner-agent skill.

> The InvMon configuration explained here requires **market-scanner access** (any paid plan).

The `scanner-agent` skill lets Claude screen a large securities universe and hand you a shortlist. A single IB market scan returns only the top rows of one ranked query and cannot be paged, so the skill partitions the universe along the **price axis** into disjoint bands and scans each band as its own batch. It rates every batch, **hides** the winners to set them aside, and moves to the next band — until **N** winners have been collected. Then it deletes the leftovers and un-hides the winners, leaving your watchlist holding exactly the shortlist.

The setup below configures a dedicated, **disarmed** portfolio group for that sweep: ratings are recorded, nothing is armed, and no orders are ever created.

> This is a *screening* pass, not a trading scenario. Where the rating-agent setup wires ratings through to target weights, close policies and delta orders, this one deliberately cuts that wire — **Arm Ratings** stays off. The output is a rated watchlist for you to review.


## Prerequisites

- InvMon running on a plan that includes the market scanner (any paid plan — the MCP server only exposes the scanner tools when the license allows it).
- Interactive Brokers TWS running and connected to your IB account. Market scanning is IB-only; other providers are refused.
- Claude Code with this repository (`InvMon.agent`) as the working directory. The bundled `.mcp.json` expects the MCP server on port `55206` — keep the port below in sync, or adjust `.mcp.json`.


## 1. Create the portfolio group

Create a new portfolio group dedicated to screening. Don't reuse a trading group: the sweep churns the watchlist hard and deletes what it doesn't keep.

![Add a Portfolio Group](assets/010-add-portfolio-group.png)

<!-- SCREENSHOT NEEDED — assets/010-add-portfolio-group.png
     The *Add Portfolio Group…* dialog, with: Simulation on, Enable MCP Server on port 55206,
     Instruments List Mode set to "By Pool", and — if the option is present — Arm Ratings visibly OFF. -->

Options worth noting:

- **Simulation** — nothing in this scenario trades, but a simulation group keeps the setup harmless by construction.
- **Enable MCP Server** on port `55206` — this is the server the agent connects to (must match `.mcp.json`).
- **Instruments List Mode: By Pool** — this is the important difference from the rating-agent setup. The skill needs the `watchlist` pool to be addressable by name, which only *By Pool* mode offers. In *Combined* mode there is no watchlist pool argument and the skill cannot run.
- **Arm Ratings** — if this option is present in your dialog, leave it **off**. This is a screening pass: ratings should be recorded, never armed for trading. If it isn't present, ratings are recorded-only anyway and there is nothing to do.
- **Auto Delta Order Submission** — off. There is nothing to submit.


## 2. Add the scanner portfolio

Add a single portfolio to the group. The skill scans into one portfolio; when the group holds exactly one, the agent doesn't even need to name it.

![Add Portfolio](assets/020-add-portfolio.png)

<!-- SCREENSHOT NEEDED — assets/020-add-portfolio.png
     The *Add Portfolio…* dialog for the single screening portfolio. -->

The weighting and target-position settings don't matter here — with the group disarmed, nothing reads them. What matters is that this portfolio owns the watchlist scanner configured in step 4.


## 3. Portfolio group settings

Open the portfolio group's *Settings…* dialog (context menu on the group in the Portfolios panel), enable editing, and override the account settings as follows.

### Instrument chart range

![Instrument Chart Range](assets/030-instrument-chart-range.png)

<!-- SCREENSHOT NEEDED — assets/030-instrument-chart-range.png
     The Instrument Chart Range setting in the group settings dialog, set to 5 days / 15 min. -->

**5 days (15 min resolution)** gives the agent intra-day price bars: `get_price_history` defaults to this range, and short periods return quotes that are only minutes old.

Unlike the rating-agent, this skill tolerates stale data **outside** trading hours — a sweep run in the evening rates on the previous close, by design. During trading hours it still requires a fresh quote and skips instruments that only return end-of-day bars.

### MCP server options

![MCP Server Options](assets/040-mcp-options.png)

<!-- SCREENSHOT NEEDED — assets/040-mcp-options.png
     The MCP Server Options section of the group settings dialog, showing List Limit 40,
     Find Instruments Using Scanner OFF, Random List Order. -->

- **List Limit 40** — this is the number that makes the sweep work. It caps the watchlist at 40 entries *and* is the level the scanner's **Replace Existing** option trims back to. Set it to your intended batch size and keep the scanner's *Max Results* (step 4) in step with it. Going much higher buys you nothing: IB caps a single scan at roughly 50 rows, which is precisely why the sweep partitions by price instead of just asking for more. Going lower is reasonable if TWS market-data lines are tight — every visible watchlist name consumes one.
- **Find Instruments Using Scanner** — must be **off**. With it on, a `list_instruments` call would re-run the scanner with the *base* configuration (no price band) and clobber the batch the agent just built. `run_scanner` detects this and refuses to run rather than corrupt the sweep silently.
- **Random List Order** — optional; shuffles the returned batch, potentially lowering rating bias.


## 4. Configure the watchlist market scanner

The sweep rides on a scanner saved against the **watchlist** pool. Select the portfolio's *Watchlist* panel and choose *Add from Market Scanner…* — invoked from the Watchlist panel, the results land in the watchlist pool rather than in candidates.

![Watchlist scanner: scan criteria](assets/050-watchlist-scanner-criteria.png)

<!-- SCREENSHOT NEEDED — assets/050-watchlist-scanner-criteria.png
     The Market Scanner dialog, scan-criteria section: Region "Stocks", Exchange "US Major",
     Variable "Top % Gainers" sorted descending, and an average-volume filter. -->

Scan US *Stocks* on **US Major** (`STK.US.MAJOR`) by **Top % Gainers**, descending, filtered to liquid names (e.g. average volume ≥ 2,000,000).

> **Do not add a price filter here.** The price band is the one axis the agent drives: it supplies `priceAbove` / `priceBelow` per batch, layered on top of this saved configuration and never written back to it. A price filter you set here becomes a base constraint that silently narrows every band.

![Watchlist scanner: result handling](assets/060-watchlist-scanner-results.png)

<!-- SCREENSHOT NEEDED — assets/060-watchlist-scanner-results.png
     The Market Scanner dialog, result-handling section: Max Results 40, Replace Existing checked,
     "Don't run scanner - only save changes" checked. (May be the lower half of the same dialog
     as 050 — feel free to make this one screenshot instead of two.) -->

- **Max Results 40** — keep in step with the *List Limit* from step 3.
- **Replace Existing** — must be **on**. This is the engine of the sweep: each new band's scan trims the previous band's non-hidden names back down to the list limit, so losers clear themselves out and the agent never has to delete anything mid-run. Hidden names — the winners the agent has already set aside — survive the trim and are not returned again.
- **Don't run scanner – only save changes** — check this. You are storing a base configuration for the MCP server to use later, not populating the watchlist now.


## 5. Run the agent

With TWS connected and InvMon running:

```
cd InvMon.agent
claude
```

Then run the skill:

```
/scanner-agent
```

The defaults sweep **$5 to $100 in $5-wide bands** looking for **10** names rated `Strong Buy` or `Buy`. All of it is overridable in plain language:

```
/scanner-agent find 15 strong-buy names between $20 and $200, bands of $10
```

This is a **one-shot sweep** — it iterates bands until `N` winners are found (or the schedule is exhausted), cleans up, and exits. Don't drive it with `/loop`: the session trim-lockout (see below) means a second sweep in the same session sees a shrinking universe.


## 6. What you see while it runs

During the sweep the watchlist visibly churns: each band replaces the previous band's leftovers, so at any moment it holds roughly one batch. The winners collected so far are *hidden* and therefore invisible — a watchlist that looks small mid-run is the expected picture, not a sign the sweep is losing its work.

![Watchlist during the sweep](assets/070-watchlist-mid-sweep.png)

<!-- SCREENSHOT NEEDED — assets/070-watchlist-mid-sweep.png
     The Watchlist panel mid-sweep: one band's worth of instruments, some already rated. -->

When the sweep finishes it archives the last band's leftovers and un-hides everything it kept, so the watchlist ends up holding exactly the shortlist — each name carrying the agent's rating, note, and price target where it set one.

![Watchlist after the run](assets/080-watchlist-after-run.png)

<!-- SCREENSHOT NEEDED — assets/080-watchlist-after-run.png
     The Watchlist panel after a completed sweep: the N winners with their Rating / Note /
     Price Target columns populated. -->

The agent also prints a summary: the winners with a one-line rationale each, how many bands it swept, roughly how many names it rated, and — if it ran out of schedule before reaching `N` — the shortfall and how far it got.

![Agent summary](assets/090-agent-summary.png)

<!-- SCREENSHOT NEEDED — assets/090-agent-summary.png
     The Claude Code console at the end of a run, showing the winners table / summary. -->


## 7. Troubleshooting

Most failures are a missing prerequisite, and `run_scanner` names the one it hit.

| What you see | Cause | Fix |
|---|---|---|
| `run_scanner is not available.` | The license has no market-scanner access. | Requires a paid plan. |
| `run_scanner requires 'Find Instruments Using Scanner' to be OFF…` | Auto-scan-on-list is on and would clobber the band batch. | Turn it off in *MCP Server Options* (step 3). |
| `Account is not connected; cannot run the scanner.` | TWS not running or not connected. | Start TWS and confirm the account connects. |
| `The account's provider does not support market scanning.` | The portfolio's account isn't an IB account. | Point the group at an IB account. |
| `A scanner run is already in progress for this portfolio.` | A scan (manual or from a previous call) is still running. | Wait for it to finish and re-run. |
| `Portfolio not found; specify portfolioId or portfolioName…` | The group holds more than one portfolio. | Name the portfolio when starting the skill, or keep the group to one. |
| `Scanner timed out after 60 seconds.` | IB did not answer the scan in time. | Usually transient; check the TWS connection and re-run. |
| `This tool call is currently limited to the watchlist pool.` | On this license the hide / un-hide / archive tools accept watchlist instruments only. | Expected — the sweep only touches the watchlist. If you see it, something pointed the tools elsewhere. |
| The agent reports a missing `watchlist` pool, or no pool argument | The group is in *Combined* list mode. | Switch **Instruments List Mode** to *By Pool* (step 1). |
| Bands come back empty that shouldn't be | Either a price filter is baked into the saved scanner config, or the names were trimmed earlier in this session. | Remove the base price filter (step 4). For the lockout, see below. |

**Session trim-lockout.** Once replace mode trims a name, it stays locked out for the rest of the application session and later scans won't re-add it. Within a single sweep this is exactly what you want — no band re-rates what an earlier band already discarded. Across sweeps it means a second run in the same session works from a shrunk universe. **Restart InvMon before starting a fresh sweep**, or restore individual names via *Restore Hidden Instrument…* (from the Delete menu).

<!-- Editorial note: possible extensions — a worked example run (bands swept vs. names rated vs. winners
     found, with timings), and guidance on choosing a band schedule for non-US or non-equity universes. -->
