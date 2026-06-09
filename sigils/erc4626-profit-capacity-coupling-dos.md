---
name: erc4626 profit-capacity coupling dos
description: Detects ERC-4626 capacity formulas that subtract accumulated profit from strategy liquidity without guarding for pnl > liquidity, blocking principal-only withdrawals when strategy is under-liquid.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [erc4626, dos, profit-extraction, withdrawal-capacity, yield-vault]
related-checks: [erc4626-fullmuldiv-maxuint-ratio-overflow]
---

## Patterns

Grep for `zeroFloorSub` in Solidity files. Read each caller and confirm the two operands are (a) a `maxWithdraw()` result and (b) a profit or pnl variable computed from `previewRedeem`, `totalRedeemable`, or similar unconstrained source.

Grep for `capacity.*pnl`, `pnl.*capacity`, `maxWithdraw.*pnl`, or `pnl.*maxWithdraw` in Solidity files. Read the surrounding function to identify the minuend and subtrahend semantics.

Grep for `isProfit` or `isPnlPositive` in Solidity files. Read the profit branch to check whether `pnl` is deducted from a `maxWithdraw` result without an upper-bound guard.

## Assessment

Read the capacity function. Identify operand A (liquidity-constrained, from `maxWithdraw()`) and operand B (profit, from `previewRedeem()` or `totalRedeemable()`). Verify that A and B can diverge — A is bounded by withdrawable liquidity while B is not.

Check for a guard before the subtraction: `if (pnl >= maxW) return maxW` or equivalent that caps B at A. If the guard exists, the match is benign.

If no guard: the formula returns 0 whenever `pnl > maxW`, blocking all withdrawals regardless of available principal liquidity. Confirm the path can be reached by a normal utilization spike in the underlying strategy.

Raise to **Medium** if the underlying strategy is a lending protocol (Aave, Compound, Morpho) that routinely reaches high utilization, making `maxWithdraw < accumulatedProfit` a realistic steady-state condition.

**Benign cases:** (a) Both operands share the same liquidity measurement (e.g., both use `maxWithdraw`) so they cannot diverge. (b) The vault design explicitly documents that withdrawals are blocked when strategy liquidity falls below accumulated profit — report as design risk only.
