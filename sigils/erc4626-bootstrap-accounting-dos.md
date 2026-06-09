---
name: erc4626-bootstrap-accounting-dos
description: Detects ERC-4626 aggregator vaults where a dust deposit on the bootstrap path triggers the silo recovery path, leaving totalSupply > 0 with totalAssets = 0 and permanently bricking all future deposits.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, dos, bootstrap, recovery-path, accounting-before-interaction]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

1. **Bootstrap 1:1 guard near previewDeposit:** Grep for `totalSupply == 0` (or `totalSupply() == 0`) in Solidity files that also contain `previewDeposit`. Candidate sites where a 1:1 bootstrap branch coexists with an inner silo deposit call.

2. **Recovery path on zero expectedShares:** Grep for `expectedShares == 0` (or `p.expectedShares == 0`) near a `safeTransfer` or `transfer` call. Confirms assets are silently forwarded to a recovery receiver instead of reverting when the silo returns 0 shares.

3. **Accounting increment before silo interaction:** Grep for `initialAssets +=` (or `totalAssets +=`) in deposit-related functions. Read the surrounding context: if the increment precedes a call to `previewDeposit` or `deposit` on a sub-vault/silo, the accounting is updated before the silo interaction is validated.

4. **previewDeposit inside deposit helper:** Grep for `previewDeposit(` inside functions named `_deposit`, `deposit`, `_safeDeposit`, or similar. The return value (`expectedShares`) should be checked for zero with a revert — if the only zero-shares handler is a recovery transfer, the bootstrap path can reach it without reverting.

## Assessment

**Confirming exploitability:** Trace the deposit call and verify all three conditions hold simultaneously:
- (a) `initialAssets` (or equivalent accumulator) is incremented **before** the silo's `previewDeposit` result is checked for zero.
- (b) A recovery path transfers assets to a receiver when `previewDeposit` returns 0 instead of reverting, making the 0-share deposit invisible to the vault.
- (c) A bootstrap guard returns `assets` as vault shares when `totalSupply == 0`, bypassing any downstream zero-shares revert.

When all three hold, depositing `< silo_PPS` wei passes the vault guard (bootstrap mints `assets` shares), hits the recovery path (assets leave the vault), and leaves the vault with `totalSupply > 0` and `totalAssets = 0`. All future `deposit()` and `mint()` calls are blocked by the insolvent guard.

**Severity adjustments:**
- **High:** All three conditions present, silo PPS can grow above 1 through normal yield accrual, and the vault is permissionlessly accessible (no depositor whitelist).
- **Medium:** Attack requires a curator/admin to first seed the silo to PPS > 1, or a depositor whitelist restricts who can make the initial deposit.

**Benign cases:**
- (a) A `require(expectedShares > 0)` guard (or equivalent) fires before any accounting increment on the bootstrap path — no recovery path exists for the zero-shares case.
- (b) The silo's PPS is permanently pegged to 1 (e.g., wrapped stablecoin with no yield), so `previewDeposit(1)` always returns >= 1 share.
- (c) The vault enforces a minimum first deposit (`assets >= minFirstDeposit`) exceeding the silo's maximum share price, guaranteeing `previewDeposit(assets) >= 1` on the bootstrap call.
