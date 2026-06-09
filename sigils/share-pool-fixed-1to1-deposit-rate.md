---
name: share pool fixed 1:1 deposit rate
description: Detects deposit functions that credit shares at a raw 1:1 rate without exchange-rate scaling when pool backing can be reduced independently of totalSupply.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [accounting, shares, defi, dilution]
---

Look for these two patterns — both must be present for a finding:

**Pattern A — flat deposit credit:**
- `totalSupply +=` or `shares +=` set to the raw deposited amount with no `* totalSupply / poolValue` (or equivalent `mulDiv`) scaling
- Deposit or add-liquidity function where `userShares = depositedAmount` rather than `userShares = depositedAmount * totalSupply / poolBacking`

**Pattern B — backing reduction without supply adjustment:**
- A separate function in the same contract that decreases the pool's backing value (split, burn, fee extraction, slashing, NFT devaluation) without a matching `totalSupply -=`
- Grep for: `totalSupply` in the file, then check whether every code path that decreases backing also decreases `totalSupply`

## Assessment

Confirm both patterns coexist: trace Pattern A's deposit function and Pattern B's backing-reduction function in the same contract. If only one is present, dismiss.

Confirm dilution is real: after Pattern B executes, verify that `sharePrice = poolBacking / totalSupply` falls below 1. A new depositor via Pattern A then receives `totalSupply += depositedAmount` while their redeemable value is `depositedAmount * poolBacking / totalSupply < depositedAmount`.

Severity adjustments:
- Raise to critical if backing-reduction is permissionless or callable by any user
- Lower to medium if Pattern B is owner-only, infrequent, and governance-gated with timelock

Benign cases (dismiss without a finding):
- ERC-4626 vaults using `convertToShares` that correctly scales by `totalAssets()`
- Yield-only pools where backing only increases; Pattern B never fires in practice
- First-deposit bootstrap that uses 1:1 only when `totalSupply == 0`; check that subsequent deposits switch to the scaled formula
