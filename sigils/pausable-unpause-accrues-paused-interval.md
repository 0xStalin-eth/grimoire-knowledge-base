---
name: pausable unpause accrues paused interval
description: Detects unpause() functions that call a yield/reward accrual function over the full elapsed time since last checkpoint, silently minting the paused-window yield instead of suppressing it.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [pausable, yield-accrual, erc4626, emissions, logic-error]
related-checks: [pausable-accrual-no-sync-on-toggle, oz-pausable-paused-override-expanded-semantics]
---

## Patterns

Grep for `function unpause` in Solidity files.

For each match, read the function body and look for calls to accrual-like functions:
`_accrue`, `accrue(`, `_settle`, `settle(`, `_checkpoint`, `checkpoint(`, `_syncRewards`,
`_updateRewards`, `_distributeRewards`.

For each accrual call found, read the called function and check whether it:
- Uses `block.timestamp - lastUpdate` (or equivalent timestamp delta) as input
- Performs a state-mutating operation: minting tokens, updating a balance, writing a storage variable

## Assessment

A match is a finding if all three conditions hold:
1. `unpause()` calls an accrual function
2. The accrual function computes elapsed time that spans the paused duration (no guard skips it)
3. The accrual function mutates state (mint, balance update, storage write)

**Severity adjustments:**
- Downgrade to informational if the contract or function has inline documentation explicitly stating that deferral (not suppression) is the intended behavior.

**Dismiss if any of the following:**
- The accrual function has a pause guard that returns early: `if (paused) return;` or equivalent
- `unpause()` resets `lastUpdate` to `block.timestamp` BEFORE the accrual call, making elapsed time zero
- `pause()` zeros out the rate or emission variable so accrued amount is zero regardless
- The function named `unpause` is a modifier or emits an event only, with no accrual call in its body
