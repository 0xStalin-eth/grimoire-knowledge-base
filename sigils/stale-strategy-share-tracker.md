---
name: stale strategy share tracker
description: Detects ERC-4626 vaults that track strategy shares in storage but sweep only the excess direction, leaving the deficit unhandled after external slashing or burning.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, share-accounting, dos, tvl-inflation, strategy]
attribution-name: Karak H-03 / ReaperVaultV2 / DittoETH yDUSD
attribution-url: https://github.com/code-423n4/2024-07-karak-findings/issues/98
---

## Patterns

1. Grep for `saturatingSub` in files that also assign a storage share-tracking variable (e.g., `_strategyShareBalance`, `strategyShares`, `stratShares`). A one-directional sweep computes `excess = real.saturatingSub(tracked)` and acts only on positive excess.

2. Grep for `redeem(` near reads of a storage share-tracking variable. Check whether a `balanceOf(` call on the strategy precedes the redeem to cap the amount at the actual on-chain balance.

3. Grep for `convertToAssets(` inside `totalAssets`-style functions that pass a storage variable instead of a live `strategy.balanceOf(vault)` result.

## Assessment

For each match:

1. **Confirm the sweep is one-directional.** Read the sweep/reconcile function. If it handles `real > tracked` (sweeping excess) but has no branch for `real < tracked` (updating tracked downward), the check is triggered.

2. **Confirm the redeem path uses the stale tracked count.** Read the migration or strategy-swap function. If it passes the storage variable directly to `strategy.redeem(trackedCount, ...)` without first capping at `strategy.balanceOf(vault)`, this is a finding.

3. **Confirm the TVL path.** If `totalAssets()` uses `strategy.convertToAssets(trackedStorageVar)` instead of `strategy.convertToAssets(strategy.balanceOf(vault))`, report as a secondary impact (TVL inflation misprices deposits).

**Severity adjustment:**
- Upgrade to **critical** if the DoS is permanent and no admin escape exists (no function that resets the tracker to 0 or to `balanceOf()`).
- Keep at **high** if an admin can manually reset the tracked count or if migration is non-critical.
- Downgrade to **medium** if slashing is explicitly out of scope for the strategy type.

**Benign cases:**
- Both directions handled: `if real > tracked { sweep excess } else { set tracked = real }` or equivalent.
- Redeem path uses `min(tracked, strategy.balanceOf(vault))` as the redemption amount.
- Strategy wraps risk-free assets with no slashing mechanism in its interface.
