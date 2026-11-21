---
name: residual loss lands on single exit
description: Flags a loss above the first-loss capital that only one payout path holds back, so that claimant class takes the whole residual while the others are paid in full.
languages: solidity
severity-default: medium
confidence: low
tools: [Grep, Read]
tags: [loss-allocation, solvency, waterfall, redemption, impairment]
related-checks: [accounting-mark-reset-on-withdrawal]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-15)
---

Find where the protocol computes a loss it has not booked (booked claims above assets held). Grep for:
`residual`, `impairment`, `shortfall`, `unbooked`, `deficit`, `badDebt`, `lossMarked`.

1. List every payout path (claims, rent, fees, live exits). For each one, does it check the residual or
   global solvency, or only its own bucket (`amount > bucket`)?
2. If only ONE path subtracts the residual, that claimant class takes all of it by default. Is that class
   meant to be protected (a reservation for already-priced redeemers marked "price never changes")? Does
   the capped bucket hold a protected reservation that the loss stages skip? Once the free part is spent,
   the capped bucket is all reservation.
3. Does the residual go down as the other classes are paid? If each payout lowers claims and assets by
   the same amount, it does not, and the loser is fixed whatever order the claims come in.
4. Can a residual above the loss-absorbing layers (float, buffers, unwind, termination) ever be booked?
   If not, a "temporary delay" is a permanent freeze.
5. Compare with the documented loss order (for example vault capital, then guarantors, then holders).

Severity: medium for a loss-allocation inversion caused by an outside event. High if an attacker can
cause the venue loss.
Benign: an explicit write-down across classes in the documented order; every payout reads the residual; a
shortfall that only delays a senior nominal claim while no other class is paid ahead of it; claimants not
yet priced who carry the loss through the share price.
Out of scope: a residual larger than the capped bucket, which other claimants race for. That comes from
missing write-down stages, not from this pattern.
