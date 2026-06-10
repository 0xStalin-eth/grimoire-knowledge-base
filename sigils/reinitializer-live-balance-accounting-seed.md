---
name: reinitializer live balance accounting seed
description: Detects reinitializer functions that seed an accounting baseline state variable from the contract's live ERC20 balance, enabling a pre-upgrade donation to permanently inflate it.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [upgradeable, initializer, accounting, erc4626, donation-attack, upgrade-safety]
related-checks: [reinitializer-missing-access-control, erc4626-live-balance-accrual-base]
---

Grep for `reinitializer(` in Solidity files to locate candidate functions.

For each match, read the full function body and search for any assignment whose right-hand side contains `balanceOf(address(this))` (or `balanceOf(` with any argument resolving to `address(this)`).

**Assess each match against these criteria:**

1. Does the reinitializer body contain an assignment from an ERC20 `balanceOf(address(this))` call to a named state variable?
2. Before that assignment, does the function call any internal settlement or accrual function (e.g., `_accrue()`, `_settle()`, `_update()`, or any prefixed `_` function that precedes the assignment)?
3. Does the function accept an expected-balance parameter and compare it against the live balance before assigning?

- (1) yes AND (2) no AND (3) no → Low — raw live balance is written as accounting baseline; pre-upgrade donation permanently inflates it
- (1) yes AND (2) yes AND (3) no → Informational — yield staleness is addressed but the donation front-run window remains
- (1) yes AND (3) yes → Dismiss — caller-supplied guard closes the attack
- (1) no (balance used only in a `require`/`assert`, not assigned to state) → Dismiss

**Severity adjustment:** Upgrade to Medium if the written state variable is used as the denominator in share-price or withdrawal calculations AND the contract has mint rights on the underlying token (donation-amplified minting is possible).

**Benign cases:**
- `balanceOf` read only inside a sanity `require` check, with no assignment to a state variable
- The written variable is a fee accumulator that is only ever incremented, never used as a withdrawal cap or share-price denominator
- The upgrade is performed via a non-proxy constructor where mempool front-running is impossible
