---
name: async oracle NAV fee claim split inflation
description: Detects async vault fee-claim functions that read an external oracle for total assets without a staleness guard while burning shares after the payout sub-function returns, enabling split-claim inflation of curator payments.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [erc4626, async-vault, oracle, fee-accounting, inflation, share-accounting]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

1. **Oracle NAV read inside fee/claim functions.** Grep for these patterns inside function bodies whose names match `claimFees`, `executeFeesTransfer`, `_claimFees`, `collectFees`, `withdrawFees`, `transferFees`, `_feePayout`, `claimManagementFee`, `claimPerformanceFee`:

```
oracle\.totalAssets(
\.nav(
\.totalValue(
```

2. **Burn-after-call ordering.** Grep for `_burn(` in the same file. Check whether the outer fee-claim function calls a payout sub-function first and then burns — `_burn` in the caller after the sub-function call, not inside the sub-function itself.

3. **Missing staleness guard.** Within 10 lines of the oracle read, grep for `lastUpdate`, `lastOracleTimestamp`, `oracleTimestamp`, `isOracleFresh`, `staleAfter`. Absence of these patterns is the signal.

## Assessment

1. **Locate the oracle read.** Confirm `oracle.totalAssets()` (or equivalent NAV call) is the sole source of `totalAssets` in the payout formula `claimedAssets = totalAssets × shares / totalSupply`.

2. **Check for staleness guard.** Read the 10 lines before the oracle call. If no `require(block.timestamp - oracle.lastUpdated() <= maxStaleness)` or equivalent reverts on stale data, proceed.

3. **Check burn ordering.** Read both the outer fee-claim function and the inner payout sub-function. If `_burn` is in the outer caller and executes after the sub-function returns, the `totalSupply` denominator seen by the oracle read on the next call is smaller — flag this.

4. **Check call multiplicity.** Verify the fee-claim function accepts partial `sharesFeeToClaim` amounts and has no per-settlement lock, nonce, or minimum-consumed guard. If splits are possible, flag split-claim inflation.

5. **Severity adjustment:**
   - Upgrade to **medium** if settlement is infrequent (oracle can be stale days/weeks), or the fee-claim role is held by a non-curator party, or inflation compounds significantly over realistic split counts.
   - Keep **low** if the claim role requires admin/curator trust and settlement is frequent.
   - Downgrade to **informational** if a staleness check exists but is soft (event-only, no revert).

6. **Benign cases (dismiss):**
   - Sync-mode path (`isVaultOpen == true`) that re-reads live silo balances via `silo.previewRedeem()` before each call.
   - Oracle read guarded by a hard staleness check that reverts on expiry.
   - Fee payout computed from a fixed snapshot stored at vault-close time, not a live oracle call.
   - `_burn` is inside the payout sub-function before it returns (denominator shrinks before payout).
   - Protocol enforces full consumption of `unclaimedSharesFee` in a single call (no partial splits).
