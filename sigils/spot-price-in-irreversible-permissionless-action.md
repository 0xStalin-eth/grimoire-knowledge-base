---
name: spot price in irreversible permissionless action
description: Flags permissionless one-shot actions (epoch strike, termination, rate lock, loss realization) that read an AMM spot price a same-transaction swap can move.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [spot-price, slot0, nav, manipulation, flash-loan, uniswap-v4]
related-checks: [entry-exit-price-source-asymmetry, accrued-duration-instant-state-trigger, live-supply-sized-permissionless-transfer]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-01, M-07, H-05)
---

Grep for spot reads: `slot0`, `getSlot0`, `sqrtPriceX96`, `getSqrtPriceAtTick`, `StateLibrary`,
`getReserves`. Trace them up to valuation helpers: `nav`, `liveNav`, `capacity`, `valuation`,
`totalAssets`, `liquidCapacity`.

Then find the callers that write a value once and never revisit it:
- epoch, batch or redemption strikes (`fulfillEpoch`, `settle`, `strike`, `pricePerShare` snapshots)
- termination or shutdown that locks a rate (`lockedRate`, `lockedD`, `_runTermination`)
- loss realization or forced unwind that decides whether to terminate (`_raise`, `realize`, `forceTermination`)
- fee accrual that raises a high-water mark that never goes down

For each, check:
1. Can anyone call it (keepers, `markYield`, `fulfillEpoch` with no role check)?
2. Does it open its own lock or context, so a swap in an earlier lock of the same transaction is not blocked?
3. Can a user-routed swap, mint or contraction move the spot price before the call and move it back after?
4. Who loses: queued redeemers, holders paid at a locked rate, other pools in a pro-rata split? Another
   trade path that only makes an already-reported spot lock larger is that finding's impact, not a new one.

A spot-moved decision (whether to terminate) and a spot-locked rate are separate findings; neither fix
closes the other.
Severity: medium by default. Raise it to high only if the loss is not capped by the spot-marked share of
the valuation (check whether larger trades add less). Low if the action has a role check or reads a TWAP
or an exit-value figure.
Benign: the valuation reads a figure a reversing trade restores; the function is gated; the only actors who
can bracket the call pay an entry or exit fee sized to the move. A NatSpec warning that the price is spot
and the caller picks the moment does not make it benign.
