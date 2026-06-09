---
name: erc4626 deposit profit extraction capacity mismatch
description: Detects ERC-4626 vaults where maxDeposit() reports only deposit-side strategy capacity while deposit() silently extracts accumulated profit via a strategy withdrawal, reverting for any deposit below the outstanding profit threshold when withdrawal liquidity is insufficient.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [erc4626, dos, deposit, profit-extraction, maxdeposit, yield-vault]
related-checks: [erc4626-profit-capacity-coupling-dos]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

Grep for `maxDeposit` in Solidity files. Read each implementation body and flag functions that call `silo.maxDeposit(` or `strategy.maxDeposit(` but do not call `silo.maxWithdraw(` or `strategy.maxWithdraw(`.

Grep for `pnl > assets`, `profit > assets`, `excessProfit`, or `_safeWithdraw(` in Solidity files. Read the enclosing function; confirm it is inside a `deposit` or `_deposit` body (not a `withdraw` or `redeem` body). Flag the pattern when the withdrawal-style call fires on the condition that accumulated profit exceeds the deposited amount.

Grep for `syncSilo.withdraw(`, `strategy.withdraw(`, or `silo.redeem(` inside any function whose name contains `deposit`. Read the branch and confirm it is triggered by a profit-vs-deposit comparison rather than by user intent.

## Assessment

Locate the `maxDeposit` implementation. Confirm it consults only the deposit capacity of the underlying strategy or silo. If it also queries `maxWithdraw()` and uses that to reduce the reported cap, dismiss — the mismatch is already handled.

Locate the `deposit` / `_deposit` function. Confirm a branch calls `_safeWithdraw(excessProfit)` or equivalent when `accumulatedPnl > depositedAssets`. Check whether that branch first guards with `strategy.maxWithdraw(vault)` and caps or skips the extraction. If the guard exists, dismiss.

Assess exploitability: determine whether the strategy can reach a state where `accumulatedPnl > strategy.maxWithdraw(vault)`. Lending protocols (Aave, Compound, Morpho) routinely do under high utilization — confirm the gap grows automatically as yield accrues.

Raise to **High** if the DoS is permanent without admin intervention and the strategy is a core lending market where high utilization is an expected steady-state.

Downgrade to **Low** if the profit-extraction branch requires admin action to trigger, or if the strategy has guaranteed 1:1 liquidity (plain ERC-20 escrow, liquid staking with instant withdrawal).

**Benign cases:** (a) `maxDeposit()` reduces reported capacity by the pending profit to account for the required withdrawal. (b) The profit-extraction branch uses `min(excessProfit, strategy.maxWithdraw(vault))` to cap or defer the excess rather than reverting. (c) The vault documents that deposits below the outstanding profit threshold are intentionally rejected. (d) The underlying strategy is a plain ERC-20 escrow with no utilization risk.
