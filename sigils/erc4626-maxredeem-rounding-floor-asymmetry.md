---
name: erc4626 maxRedeem floor-floor rounding asymmetry
description: Detects ERC-4626 maxRedeem/maxWithdraw that compute share caps via floor (assets-to-shares) paired with redeem/withdraw that enforce caps via floor (shares-to-assets), creating an off-by-one gap where redeem(maxRedeem()+1) can succeed.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [erc4626, rounding, spec-compliance]
related-checks: [erc4626-bootstrap-accounting-dos]
---

## Patterns

Grep for `maxRedeem` or `maxWithdraw` function definitions in Solidity files.

Grep for `convertToShares` or `mulDiv` or `fullMulDiv` inside `maxRedeem`/`maxWithdraw` bodies (assets-to-shares conversion, typically `assets * totalSupply / totalAssets`).

Grep for `convertToAssets` or `mulDiv` or `fullMulDiv` inside `redeem`/`withdraw` bodies paired with a `require`, `revert`, or custom error enforcing an asset-denominated upper bound (e.g., `Silo__RedeemMoreThanMaxAssets`, `<= maxAssets`, `> availableAssets`).

## Assessment

Read both `maxRedeem` (or `maxWithdraw`) and `redeem` (or `withdraw`) in the same contract.

Confirm `maxRedeem` converts available-assets to shares using **floor** (round-down). If it calls `fullMulDivUp`, `mulDivUp`, or any ceiling variant, dismiss — the share cap is already conservative and no gap exists.

Confirm `redeem` converts input-shares to assets using **floor** (round-down) and then compares that asset value against an asset-denominated cap. If the conversion uses ceiling, or the cap is share-denominated, dismiss — the asymmetry does not arise.

If both use floor in their respective directions, flag: the gap exists when `(availableAssets × totalShares) mod totalAssets >= totalAssets − totalShares + 1`, allowing `redeem(maxRedeem()+1)` to succeed without revert, violating ERC-4626 §4.8.

Adjust severity to **informational** if the vault has no external ERC-4626 composability surface (fully internal, no integrators).

Adjust severity to **medium** if a smart-contract caller in the same repo explicitly uses `maxRedeem`/`maxWithdraw` as an input bound passed to `redeem`/`withdraw`.
