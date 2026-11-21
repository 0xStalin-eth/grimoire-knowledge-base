---
name: accounting mark reset on withdrawal
description: Flags watermark or mark bookkeeping that is zeroed on a withdrawal or payout path, dropping an unbooked loss or gain the next mark can no longer see.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [accounting, watermark, yield, loss, mark-to-market, adapter]
related-checks: [external-vault-value-not-bounded]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-13)
---

Find per-venue or per-strategy marks that the next yield or loss mark compares with live value. Grep for:
`marked[`, `watermark`, `lastValue`, `highWater`, `bookedValue`, `costBasis`, `principal`, set to `0` or
reduced inside withdrawal, payout or funding functions (`fund`, `withdraw`, `pull`, `deallocate`).

For each write on a payout path:
1. Is the mark set to 0 (or reduced by more than the value consumed) on conditions such as
   `principal() == 0`, `consumed >= marked`, or "pull exceeds mark"? Principal can reach 0 two ways: a pull
   that reaches the admitted value, or a proportional basis cut that rounds a dust principal down to 0.
2. Before the reset, was there a difference between mark and live value that no mark had booked yet
   (an unbooked loss or gain)?
3. After the reset, does the mark function see live == mark == 0, so it never books that difference?
4. Can a permissionless payout (claims, redemptions) reach this path between the value change and the
   next permissionless mark? A partly liquid earlier venue can route the payout here with no forced revert.
5. What happens downstream: does an unbooked residual block deposits or epoch pricing, or does it
   misprice shares?

Severity: medium if an erased loss blocks deposits or pricing. A residual of 1 raw unit is enough to do
that, so dust does not lower it. Low if the cleared mark only lets an unbooked gain arrive as a plain asset.
Benign: the reset first books mark minus consumed through the normal loss or gain path; the payout marks
the venue first; other unrecognized backing (for example assets sent straight to the ledger) still covers
mark minus consumed.
