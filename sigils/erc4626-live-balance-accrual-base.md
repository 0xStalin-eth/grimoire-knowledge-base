---
name: erc4626 live balance accrual base
description: Detects interest-accruing ERC-4626 vaults that use the raw token balance as the compounding base, enabling flash loan donations to inflate minted interest.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [erc4626, vault, defi, interest-accrual, donation-attack]
related-checks: [pausable-accrual-no-sync-on-toggle, rounding-inflation-attack]
---

Look for accrual functions:

- Functions named `_accrue`, `accrue`, or `accrueInterest`
- Inside those functions: `super.totalAssets()`, `asset().balanceOf(address(this))`, or `IERC20(asset()).balanceOf(address(this))` used as the compounding base
- A `rate` or `ratePerSecond` multiplied against that live balance to compute earned interest
- `_mint(address(this), earned)` or `mint(address(this), earned)` called with the computed delta

For each match, assess:

1. Does the function use a live ERC-20 balance (not a stored snapshot such as `lastTotalAssets`) as the compounding base?
2. Is the vault pausable? Look for `whenNotPaused`, `paused`, or `togglePause` on state-changing functions.
3. Are there share-supply protections? Look for dead shares, minimum deposit guards, or virtual offsets.

Decision:

- (1) live balance AND (2) pausable → Low — a donation during the stale accrual gap created by a pause can amplify minted interest; see related check for the pause-gap amplification path
- (1) live balance AND (2) not pausable → Informational — normal user interactions keep accrual current; donation window is negligible
- Stored snapshot used instead of live balance → Dismiss
