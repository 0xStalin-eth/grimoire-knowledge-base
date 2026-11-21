---
name: entry exit price source asymmetry
description: Flags issue and redeem routes (mint/burn, OTC fills, alt venues) priced from different rate sources, so a round trip returns more than it paid.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [pricing, oracle, arbitrage, round-trip, flash-loan, otc]
related-checks: [pretrade-rate-applied-to-whole-trade, spot-price-in-irreversible-permissionless-action]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (H-01, H-03, M-03, M-04)
---

List every route that turns the base asset into the protocol token, and every route back: primary mint/redeem,
expansion/contraction, OTC or desk fills, auto-routed side venues, hook-attached pools.
Grep for: `mint(`, `redeem(`, `expand`, `contract`, `fill`, `forward`, `reverse`, `desk`, `otc`, `premium`.

For each route, write down which price it uses: oracle (`rate(`, `latestRoundData`, `getPrice`),
internal supply ratio (`totalSupply` / `totalSupply`), AMM spot (`slot0`, `getSlot0`, `sqrtPriceX96`), or a sum of these.

For every (entry route, exit route) pair, check:
1. Can both legs run in the same transaction with no role? Can borrowed funds pay for them?
2. Can an accepted oracle print, a spot move or the caller's own leg make tokens per dollar on entry higher than
   tokens per dollar burned on exit? Typical case: mint at the oracle, redeem at the supply ratio.
3. Side venue (OTC/desk): the canonical marks include the AMM premium (entry ~ oracle + subsidy * premium,
   exit ~ ratio + premium). Does the venue's trigger or price compare with the bare ratio instead? Check
   each bound at fill time and report each one that is missing as its own finding: (a) a buyer routed to the
   venue gets fewer tokens per dollar than the entry mark it displaces (user loss), and (b) the venue sells
   above, or buys below, the exit mark (round-trip gain). The premium moves with every trade, including the caller's own.
4. Does a side venue read the oracle while the canonical exit does not? Then any accepted print, right or
   wrong, reaches the protocol's book through that venue unless it is capped at the canonical exit.
5. Do the fees on both legs exceed the largest possible gap, at the fee values in the shipped configs?

Write out the gain per round trip as a formula in the rates and confirm it is positive in a reachable state.

Severity: high if the gap opens with no role and borrowed funds. An admin-set premium does not lower it when
ordinary trades move the AMM premium into the losing band. Medium if the loss falls on ordinary users with
no attacker gain and is bounded by inventory, or if each exit narrows the gap so the window closes. Low if
fees always cover the gap.
Benign: both legs read the same rate; the exit charges at least max(entry rates); a rounding leftover of at
most one raw unit per trip that goes to the protocol. Refilling a mispriced venue's bucket is that venue's
finding, not a new one.
