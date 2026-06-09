---
name: async-vault-epoch-reset-stale-claim
description: Detects async vaults that zero a global virtual-share accumulator on migration/insolvency and bump an epoch counter, but whose claim path divides req.storedVirtual by the (now-tiny) post-reset global without checking the request epoch against the migration watermark.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, async-vault, epoch, share-accounting, stale-state, migration]
attribution-name: Perennial BalancedVault Sherlock 2023 #45 + T3tris Vault Cyfrin Audit 2026
attribution-url: https://github.com/sherlock-audit/2023-05-perennial-judging/issues/45
related-checks: unchecked-claim-accumulator-subtraction
---

## Patterns

1. Grep for assignments of the form `total.*Virtual.*Shares\s*=\s*0` or `total.*True.*Shares\s*=\s*0` in the same function that contains `++` on a variable named `currentRedeemRequestId`, `currentEpoch`, `epochId`, or `requestId`. These are migration or insolvency settlement functions that reset global accumulators and advance the epoch.

2. Grep for `req\.totalVirtualSiloShares` or `req\.virtualShares` inside functions named `claimRedeem`, `claimWithdraw`, `_claimRedeem`, `_settleRedeem`, or `processRedeem`. Read the surrounding math: look for a division or `fullMulDiv` against the current global `totalVirtualShares`. Verify whether a guard like `require(req.epochId >= lastMigrationEpoch)` or `require(req.epochId == currentEpoch - 1)` precedes the calculation.

## Assessment

For each match:

1. **Confirm the epoch-reset pair.** Read the migration/insolvency function. Verify `totalVirtualShares = 0` (or equivalent) and `++epochId` (or equivalent) both execute in the same transaction or atomically linked path.

2. **Confirm the missing epoch guard in the claim path.** Read the claim function. If the ratio `req.storedVirtual / globalTotalVirtual` is computed without first asserting that `req.epochId` falls within the post-migration epoch range, the check fires: a pre-migration request carries an unbounded stale numerator against a near-zero post-migration denominator.

3. **Assess inflated settlement.** Determine whether the inflated ratio can exceed 1 (settled shares > global shares). If so, the claimant receives more assets than the global pool holds, draining other users or reverting in the subtraction (see related check).

**Severity adjustments:**
- Keep at **high** if migration is admin-triggered but the insolvency settlement path is permissionlessly reachable (any user can call settle after the insolvency threshold is crossed).
- Downgrade to **medium** if migration requires admin action AND a separate guarded insolvency settlement function resolves all pre-migration claims atomically before resetting the accumulator.

**Benign cases:**
- Claim path includes `require(req.epochId == currentEpoch)` or an equivalent epoch-range guard that rejects cross-epoch claims.
- Global accumulator is never set to 0; it is only incremented or replaced with a fresh non-zero seed during migration.
- Pre-migration requests are fully settled (accumulator drained to 0 via iteration) before the epoch reset executes.
