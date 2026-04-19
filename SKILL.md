---
name: invmon-agent
description: Use the invmon-mcp MCP server to list stocks to analyze, analyze them, determine a rating and set that rating on the stock (in-house; not public)
---

# Research and rate instruments

List the instruments of an InvMon portfolio and research them with the goal to set a likely near term price direction (up, down or neutral). 

## Arguments

The name of the portfolio whose instruments are to be analyzed. The user can either provide a fully qualified name here (Account, Group, Portfolio) or 
simply pass the name of the portfolio, in which case you can use the list_portfolios tool to resolve the portfolio's Id (assuming the name is unique). If list_portfolios
only lists one portfolio, then this argument can be omitted.

## Task

Get the list of instruments from the MCP server and research each instrument individually (perhaps in parallel using sub-agents). 
Consider yourself a seasoned financial analyst when doing so. Try to come up with an estimated near term price direction for each instrument (up, down or neutral).
If possible, qualify each estimate with a level of confidence (low or high). 

Your guess is not binding. You cannot go wrong. Your input will help to determine
a portfolio's near term risk profile and help with re-balancing decisions. Feel free to set a direction of neutral if you're unsure (much better than taking a blind guess). 
Instead of rating each instrument in isolation, feel free to do an internal pre-rating round and then compare your pre-rating results among all of the instruments of the portfolio. This type
of relative comparison might help you to set the final priceDirection and directionConfidence parameters for your update_rating tool call. 
You can optionally set a priceTarget and priceTargetDate on an instrument. 

## Preserving state across invocations

This skill will be invoked repeatedly. When calling the update_rating tool, you can set a note on the instrument. This note will be later available when
you call list_instruments and may help you with your research across multiple invocations.

