# Rating Agent — Set-Up Tutorial

The `rating-agent` skill lets Claude research the instruments of an InvMon portfolio group and submit a rating (optionally with a price target) for each one via InvMon's built-in MCP server. This tutorial walks through the InvMon configuration used to run it: a simulated long/short portfolio group whose candidates are supplied by IB market scanners.

## Prerequisites

- InvMon running with a paid plan (the MCP server and intra-day chart data require it; Arm Ratings and the autonomous trading switches require an AT subscription).
- Interactive Brokers TWS running and connected to your IB account.
- Claude Code with this repository (`InvMon.agent`) as the working directory. The bundled `.mcp.json` expects the MCP server on port `55206` — keep the port below in sync, or adjust `.mcp.json`.

## 1. Create the portfolio group

Create a new portfolio group for the agent to work in.

![Add a Portfolio Group](assets/010-add-portfolio-group.png)

Options worth noting:

- **Simulation** — all trades in this group are simulated. Recommended for a first run.
- **Enable MCP Server** on port `55206` — this is the server the agent connects to (must match `.mcp.json`).
- **Instruments List Mode: Combined** — the agent's `list_instruments` call returns positions and candidates together.
- **Arm Ratings** — allows the agent's submitted ratings to actually drive target weights and close policies.
- **Auto Delta Order Submission** is on, but **Auto IB Order Transfer** is *off* — orders are created and sent to the TWS, but not transferred to IB for execution. Combined with the Simulation flag, no real-money trades can result.

## 2. Add the portfolios

Add two portfolios to the group: a long side and a short side.

![Add Portfolio: Long](assets/020-add-portfolio-1.png)

![Add Portfolio: Short](assets/030-add-portfolio-2.png)

The two are identical (Target Weight *Medium*, 5 target positions, sentiment *Neutral*) except for **Target Hedge %**: 20 for `Long`, 80 for `Short` — so one trades predominantly long, the other predominantly short.

## 3. Portfolio group settings

Open the portfolio group's *Settings…* dialog (context menu on the group in the Portfolios panel), enable editing, and override the account settings as follows.

### Instrument chart range

![Instrument Chart Range](assets/040-edit-portfolio-group-settings.png)

**5 days (15 min resolution)** gives the agent intra-day price bars: `get_price_history` defaults to this range, and short periods return quotes that are only minutes old — the freshness the skill requires before rating an instrument. Intra-day resolutions need a paid InvMon plan or a suitable IB market data subscription.

### Rebalancing settings

![Rebalancing Settings](assets/050-rebalancing-settings.png)

**Enable Short Orders** is required for the `Short` portfolio to work. **Move to Watchlist** keeps closed instruments visible instead of dropping them.

<!-- Editorial note: screenshot numbering jumps from 050 to 070 — is a screenshot missing here (060), or was it removed intentionally? Also: "Enable Rebalancing Orders" is off in this setup while "Auto Delta Order Submission" is on in the group targets; a sentence explaining why threshold-based rebalancing orders stay disabled for this agent setup would help readers. -->

### Order type, size & limit calculation

![Order Type, Size & Limit Calculation](assets/070-order-type.png)

Defaults are fine here: adaptive orders, a 0.5% slippage cap, and trading restricted to regular trading hours.

### Close policies

![Close Policies](assets/080-close-policies.png)

These policies close positions automatically and require an active TWS connection. The two **(MCP)** policies are what make the agent's ratings actionable: **Close on Neutral** and **Close on Counter-Rating** close a position when the agent downgrades it to Neutral or rates it against the position's direction. **EOD Auto-Close** flattens remaining positions 20 minutes before the session ends — this setup is intra-day.

### Autonomous trading gates

![Autonomous Trading Gates](assets/090-autonomous-trading-gates.png)

Sanity limits every automatically created order must pass (fresh quote, tight spread, limited price deviation and exposure overshoot). **Hard Gates** rejects failing orders outright instead of holding them. The autonomous-trading on/off switches themselves live in the group's *Targets & Rebalancing* dialog (step 1).

### MCP server options

![MCP Server Options](assets/100-mcp-options.png)

- **List Limit 10** — caps how many instruments `list_instruments` returns per call, keeping the agent's research fan-out bounded.
- **Find Instruments Using Scanner** — a `list_instruments` call triggers the portfolios' saved market scanners first, so the agent always rates a fresh candidate set (see next step).
- **Random List Order** — shuffles the returned list.

## 4. Configure the market scanners

Each portfolio gets an IB market scanner that supplies its candidates: *Add from Market Scanner…* on the portfolio.

![Long scanner: top gainers](assets/110-long-scanner.png)

![Short scanner: top losers](assets/120-short-scanner.png)

Both scan US *Listed/NASDAQ* corporate stocks by **Top %**, filtered to liquid names (average volume ≥ 2,000,000) that are shortable (shortable shares ≥ 2000), capped at 10 results with **Replace Existing** so the candidate pool rotates instead of growing. The only difference is the sort direction (the arrow next to *Variable*): descending for `Long` (top gainers), ascending for `Short` (top losers).

**Don't run scanner – only save changes** stores the configuration without running it — the scans are executed later by the MCP server (via *Find Instruments Using Scanner*) whenever the agent lists instruments.

## 5. Run the agent

With TWS connected and InvMon running:

```
cd InvMon.agent
claude
```

Then run the skill once:

```
/rating-agent
```

or in a loop during market hours:

```
/loop 1h /rating-agent
```

The agent lists the instruments, pulls price history, researches each name, submits ratings via `update_ratings`, and prints a summary. Ratings, notes, and price targets are visible in InvMon and — with the close policies above — can trigger position closes.

<!-- Editorial note: possible extensions — a screenshot of the InvMon UI after a run (ratings/notes visible on the instruments), a sample agent console summary, and a short troubleshooting section (MCP port mismatch, missing IB market data subscription). -->
