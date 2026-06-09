---
name: erc4626-fullmuldiv-maxuint-ratio-overflow
description: Detects ERC-4626 mint/maxMint paths that scale type(uint256).max via fullMulDiv without guarding for numerator > denominator, causing permanent DoS when any divergence (fee minting, rounding recovery, PnL extraction, or first-deposit PPS mismatch) inflates the numerator above the denominator.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, dos, fullmuldiv, arithmetic-overflow, maxmint, fee-dilution]
attribution-name: Revert Lend Code4rena 2024 M-14 / Sherlock new-scope 2024 #431 / T3tris Vault Cyfrin Audit 2026
attribution-url: https://github.com/code-423n4/2024-03-revert-lend-findings/issues/249
---

## Patterns

1. **fullMulDiv in maxMint/maxDeposit context:** Grep for `fullMulDiv` in Solidity files, then confirm the same file or contract also contains `maxMint` or `maxDeposit`. These are candidate sites where an unbounded capacity value may be scaled by a ratio.

2. **type(uint256).max near fullMulDiv:** Grep for `type(uint256).max` within 20 lines of `fullMulDiv`. Direct evidence that the sentinel unlimited-capacity value is passed as the first argument to the multiply-divide.

3. **Subordinate maxMint/maxDeposit return flowing into fullMulDiv:** Grep for call sites of `maxMint(` or `maxDeposit(` whose return value is stored in a local variable that subsequently appears as the first argument of a `fullMulDiv` call. Confirms the delegation chain from a subordinate silo's capacity to the scaling formula.

4. **totalSupply or share-count numerator in fullMulDiv:** Grep for `totalSupply` (or variables named `*Shares`, `*Supply`) appearing as the second argument to `fullMulDiv`. Confirms the numerator is a share count that can be inflated by fee minting or interest accrual.

## Assessment

**Confirming exploitability:** Read the fee-accrual, settlement, and deposit logic. Verify any path that causes `numerator > denominator`: (a) fee minting inflates `totalSupply` without touching the strategy-share denominator; (b) deposit rounding where the sub-vault mints 0 shares for a non-zero asset amount (recovery path), crediting vault shares with no denominator increase; (c) PnL extraction (`_safeWithdraw` / `claimSilosPnl`) burns strategy shares, shrinking the denominator while `totalSupply` stays fixed; (d) **fresh-vault / external-silo PPS mismatch**: when a new vault's first deposit is issued at 1:1 (standard ERC-4626 zero-supply behavior) but the sub-silo has an established PPS above 1:1 from external deposits, the silo mints fewer shares than the vault issues — creating structural divergence from the very first deposit, with no adversarial actor. When `numerator > denominator`, `fullMulDiv(type(uint256).max, numerator, denominator)` overflows 256 bits and reverts with `FullMulDivFailed` (`0xae47f702`), permanently DoSing `mint()` and `maxMint()` in violation of ERC-4626 §4.9.

**Severity adjustments:**
- **High:** Overflow is reachable through normal protocol operation (e.g., automatic fee accrual on any deposit/withdraw) with no admin action required, and it permanently bricks `mint()` or `maxMint()`.
- **Medium:** The trigger requires a specific admin action (e.g., manually calling a fee-settlement function) or the DoS is reversible by an admin.

**Benign cases:**
- (a) A guard `if (numerator >= denominator) return type(uint256).max;` (or equivalent cap) appears before the `fullMulDiv` call.
- (b) The first argument is bounded well below `type(uint256).max` (e.g., capped at `totalAssets()` or a protocol maximum), making 256-bit overflow impossible in practice.
- (c) No fee-minting, interest-accrual, or other dilutive share-inflation path exists in the contract, so `numerator` can never exceed `denominator`.
