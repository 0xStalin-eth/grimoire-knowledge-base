---
name: accrual timestamp not set in initializer
description: Detects yield-bearing contracts where the timestamp anchor used in elapsed-time accrual is not initialized to block.timestamp, leaving it at zero (Unix epoch) until first accrue.
languages: solidity
severity-default: informational
confidence: medium
tools: [Grep, Read]
tags: [initialization, timestamp, accrual, erc4626, yield-vault]
---

Grep for `block.timestamp - last` to find contracts with elapsed-time accrual expressions.
For each match, grep the same file for `function initialize` or the `initializer` modifier.

If both are present, identify the timestamp state variable used in `block.timestamp - <var>`.

**Assessment:**

1. Read `initialize()` and every function it calls. If the timestamp variable is assigned
   (e.g. `lastUpdate = uint40(block.timestamp)`) anywhere in that call tree → DISMISS.

2. If the timestamp variable is never assigned during initialization:
   - Read the accrual/compute function. Does it short-circuit when a rate or multiplier is
     zero (e.g. `if (ratePerSecond == 0) return currentBalance`)? If yes → flag as
     **informational**: the contract is safe-by-accident; the protection relies on a
     convention (rate is written only after accrual), not an enforced invariant. Any upgrade
     or governance helper that sets the rate without calling accrue first will compute
     `block.timestamp - 0` (≈1.7×10⁹ seconds) of yield.
   - If no short-circuit exists → escalate to **low**: any non-zero rate at first accrue
     will compute epoch-wide yield and pass it to the mint function.

3. Note as a secondary observation: off-chain consumers reading the timestamp variable will
   see `0` until the first accrual interaction, misrepresenting vault age.
