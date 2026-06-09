---
name: erc4626 split-deposit accounting inflation
description: Detects ERC-4626 aggregator vaults where an internal accounting accumulator is incremented by the full deposit amount before a silo deposit helper silently recovers zero-share amounts to a non-vault receiver, leaving the accumulator inflated and PPS permanently depressed.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, vault, accounting, pps-dilution, split-deposit, recovery-path, griefing, loss-state]
related-checks: [erc4626-bootstrap-accounting-dos, sync-vault-loss-state-accumulator-drift]
attribution-name: T3tris Vault Cyfrin Audit 2026 / Silo Finance Code4rena M-06 Mar 2025
attribution-url: https://code4rena.com/reports/2025-03-silo-finance
---

## Patterns

1. **Accounting-before-interaction:** Grep for `rawAdd` or `+=` on variables matching `*initialAssets*`, `*initialBalance*`, `*seedAssets*`, or `*depositedAssets*` inside functions containing `deposit` or `_deposit`. Read context: flag when the increment precedes a call to `_safeDeposit`, `previewDeposit`, or an inner silo's `deposit`.

2. **Silent recovery on zero expected shares:** Grep for `expectedShares == 0` (or `p.expectedShares == 0`) combined with a `safeTransfer` or `transfer` call to a `recoveryReceiver`, `curator`, or `treasury` variable. Flag when this is inside a helper called from a deposit path (not a withdraw path).

3. **Profit-split deposit split:** Grep for `assetsToDeposit` assignments of the form `assets - pnl`, `assets - profit`, or `amount - excessProfit` inside functions named `*deposit*` or `*_deposit*`. Check whether the accumulator increment uses `assets` (pre-split full amount) rather than `assetsToDeposit` (post-split reduced amount).

## Assessment

Confirm all three conditions hold simultaneously:
1. The accumulator is incremented by the **full pre-split** `assets`, not by `assetsToDeposit`.
2. The deposit helper has a recovery path that sends assets out of the vault when `previewDeposit` returns 0, without reverting the entire transaction.
3. There is no rollback of the accumulator when recovery fires (no `try/catch` subtraction, no revert on zero shares).

**Severity adjustments:**
- Keep at **High** if the vault is permissionlessly accessible AND the loss-state curator loop is present (depositor pays N wei, recovery returns N wei to depositor, depositor nets free vault shares — loopable in a single transaction).
- Downgrade to **Medium** if only the profit-split path is affected (requires silo PPS > 1 AND deposit within a narrow range just above accumulated profit).
- Downgrade to **Medium** if a deposit whitelist restricts who can trigger the recovery path.

**Benign cases:**
- A `require(siloShares > 0)` guard (or equivalent) reverts the entire deposit when the silo returns 0 shares — no silent recovery path exists.
- The accumulator increment is rolled back when `_safeDeposit` returns 0 shares (whole deposit function reverts).
- The silo PPS is permanently pegged to 1:1, making `previewDeposit(x) >= x` always true.
- The accumulator is incremented by `assetsToDeposit` (the post-split amount) rather than `assets`.
