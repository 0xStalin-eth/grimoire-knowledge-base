---
name: external vault value not bounded
description: Flags deposits and exact withdrawals into external ERC-4626 venues where value received or consumed is measured but never bounded against the amount sent.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, inflation, donation, adapter, yield, friction]
related-checks: [accounting-mark-reset-on-withdrawal]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (H-04, M-09)
---

Find adapters or strategies that move protocol funds into an external vault.
Grep for: `.deposit(` and `.mint(` with an unused return value, `.withdraw(` with an exact asset amount,
`convertToAssets(balanceOf(`, `beforeValue`, `afterValue`, `friction`, `consumed`.

For each deposit:
1. Are the shares returned ignored? Is there a check for zero shares or a minimum number of shares?
2. Is the value change (after - before) compared with the amount deposited? Or is any shortfall booked
   as friction or loss with no limit?
3. Can the venue be listed while it has low supply and no virtual shares or decimals offset? Can anyone
   donate to it, or supply on its behalf, before the protocol's first deposit? A public listing before
   the first deposit needs no mempool access.

For each exact-asset withdrawal (`withdraw(assets, ...)`, which rounds the shares burned up):
4. Is the value consumed (before - after) compared with the amount paid out, with a small tolerance? Can a
   permissionless payout (claims, redemptions, rent) send repeated small withdrawals here? On an inflated
   venue each one burns a whole share unit.

Also check that a `principal` or cost basis credited for the full amount can still be redeemed when 0
shares were minted. If it cannot, delisting or emptying the venue is blocked.

Severity: high if a donor can take a deployment. Medium for a drain through repeated payouts that reach the
venue only when idle funds and earlier venues cannot pay, bounded by the exposure cap. Low only when one
share unit is worth less than one raw asset unit (for example a 1e12 virtual-share offset). Virtual shares
with a zero decimals offset still lose a few percent, and an empty one can mint 0 shares.
Benign: a minimum-shares or maximum-friction require; a tolerance of a few raw units sized to the adapter's
own valuation rounding; a fixed raw-unit friction per deposit that the design accepts; a venue inflated only
after the protocol already holds a position (the donor ends at a loss).
