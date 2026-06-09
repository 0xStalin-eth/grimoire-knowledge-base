---
name: async vault donation bridges inflated silo claim
description: Detects async ERC-4626 vault claim paths where a ratio-derived withdrawal amount is passed directly to a silo withdrawal call without being capped to the vault's logical silo entitlement, allowing a permissionless ERC-20 donation to the vault address to make an inflated withdrawal succeed and trigger an unchecked accumulator underflow.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, async-vault, silo, donation, share-accounting, underflow, migration]
related-checks: [async-vault-epoch-reset-stale-claim, unchecked-claim-accumulator-subtraction]
attribution-name: T3tris Vault Cyfrin Audit 2026 / Silo Finance Code4rena Mar 2025
attribution-url: https://code4rena.com/reports/2025-03-silo-finance
---

## Patterns

Grep for ratio-derived settlement in claim/settle functions: `fullMulDiv(` or `mulDiv(` where operands include a per-request stored field alongside a global accumulator variable.

In the same function scope, grep for a silo withdrawal call — `silo.withdraw(`, `silo.redeem(`, `_safeWithdraw(`, `_safeRedeem(` — where the amount argument traces directly to the ratio result.

Check whether a `min(settledShares,` or `require(settledShares <=` guard appears between the ratio computation and the silo call. Absence of such a cap is the core signal.

## Assessment

1. Confirm the withdrawal amount is ratio-derived. Read the claim function; identify `settledShares = fullMulDiv(req.stored, globalA, globalB)` and verify that `globalB` (the virtual denominator) can be smaller than `req.stored` after a migration or insolvency reset (see `async-vault-epoch-reset-stale-claim`).

2. Confirm no entitlement cap before the silo call. Verify there is no `min(settledShares, totalTrueShares)` or equivalent applied to the withdrawal argument. If a cap exists, the donation path cannot cause the underflow — dismiss.

3. Confirm silo shares are a standard ERC-20. If the silo share token is non-transferable or access-controlled, permissionless donation is impossible — dismiss or downgrade to low.

4. Assess donateability. Check the silo ERC-20 `transfer` for transfer restrictions. Standard ERC-20 and Aave StataToken allow this; downgrade to medium if a restriction is present.

Severity adjustments:
- Keep high when the silo ERC-20 is freely transferable, no entitlement cap exists, and the underflow subtraction is in an `unchecked` block (see `unchecked-claim-accumulator-subtraction`).
- Downgrade to medium if the underflow subtraction is in a checked block (over-settlement reverts cleanly instead of wrapping).
- Downgrade to low if triggering the path requires the attacker to donate substantial silo share value.

Benign: entitlement cap applied before silo call; silo shares non-transferable or whitelisted; virtual accumulator always populated after last migration so `req.stored <= globalVirtual` by construction.
