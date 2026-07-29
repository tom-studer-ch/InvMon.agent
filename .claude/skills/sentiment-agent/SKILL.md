---
name: sentiment-agent description: Research the current intra-day sentiment of
the NASDAQ (using the common analyst notation Very Bearish, Bearish, Neutral,
Bullish, Very Bullish) using the web and report it to InvMon via the
set_sentiment tool.
---

This skill runs in a tight loop (roughly every 15 minutes during market hours)
so InvMon always has a fresh NASDAQ sentiment reading. Each run is a quick
snapshot, not a deep dive — the goal is to catch sentiment *changes* as they
happen through the trading day. Research the current sentiment, classify it on
the 5-point scale, report it via `set_sentiment`, then print a quick console
summary.

Currently hard-coded for the NASDAQ (needs to be adjusted for other
portfolios).

Because the loop exists to catch intra-day shifts, weight signals by how fast
they move:

- **Lead on real-time signals:** today's index move vs prior close and the
    recent session trend, the current VIX level, the CNN Fear & Greed Index,
    and breaking headlines from the last few hours.
- **Treat slow/lagging signals as background only:** weekly surveys (e.g. AAII),
    monthly/full-year forecasts and price targets barely move between runs —
    use them as a baseline, but never let a stale read override fresh real-time
    signals.


**Running this skill in a loop**

This skill uses the InvMon MCP server, accessed via localhost. This skill can
only run in a local session, so always run this skill in the current session
only (remote execution not supported).
