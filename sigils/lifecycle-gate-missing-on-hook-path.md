---
name: lifecycle gate missing on hook path
description: Flags hook, callback or side-pool paths that reach the same assets as gated core entrypoints but skip the terminated/paused gates those entrypoints carry.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [lifecycle, termination, pause, uniswap-v4, hooks, access-control]
related-checks: [entry-exit-price-source-asymmetry]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-05)
---

1. List the lifecycle modifiers on the core entrypoints: grep for `notTerminated`, `whenNotPaused`,
   `whenActive`, `whenTerminated`, `onlyLive`, `require(!terminated`, `require(!paused`.
2. List the other paths that move the same assets or prices: Uniswap v4 hook callbacks
   (`beforeSwap`, `afterSwap`, `_getUnspecifiedAmount`, `_beforeAddLiquidity`), custom-curve hooks,
   flash or lock callbacks, and core functions that hooks call (`sync*`, `settle*`, `onSwap`).
3. For each such path, check whether it reads the same flags. A core callback that returns early on a
   terminated state, instead of reverting, does not block the swap that called it.
4. After termination or pause, the protocol may lock a rate (claims at a fixed price). Can the ungated
   path still trade the asset at a different price (near par on a stable pool, for example)? Who pays
   the difference: LPs, guarantors, or holders? If the same trade gives the same output before and after
   the lock, the lock did not cause the gap. That is a question about the AMM's design, not this check.
5. If the spec declares the gate on the swap and the code does not, flag the gap.

Severity: medium if the asset the ungated path pays out is worth more than the locked claim at the
moment of lock. Low if the gap only comes from price drift after the lock, before counterparties withdraw.
Do not claim the loss grows with price impact: it is capped by the premium.
Benign: the hook reverts on the same flag; the pool's liquidity is pulled when the state changes; a pause
gate that is selective by documented design (wind-down and LP self-exit paths stay open), where a
curve-priced swap or an LP taking its own pro-rata share during a pause is not a loss.
