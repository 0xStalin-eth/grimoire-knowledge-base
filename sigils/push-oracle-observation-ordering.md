---
name: push oracle observation ordering
description: Flags permissionless push oracles that accept any signed observation newer than the stored one, or make it the ordering floor before the guard judges it, so a caller picks or freezes the rate.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [oracle, push-oracle, zk-proof, signed-price, replay, ordering]
related-checks: [oracle-deviation-anchor-lockout, signer-epoch-not-pinned]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-10, M-11)
---

Find functions where anyone can submit a signed or proven price. Grep for: `submitPrice`, `updatePrice`,
`pushPrice`, `verifyProof`, `publishedAt`.

For each submit path, check:
1. Ordering rule: is the observation only required to be newer than the STORED one? Then a holder of two
   genuine readings, at most one band apart and the newer one not yet on chain, can publish the older one,
   trade at it, and publish the newer one in the same transaction. Can the price be read in the same
   transaction it is published? A keeper that publishes every reading promptly closes this window.
2. Floor poisoning: does the observation become the ordering floor before a downstream guard (band,
   budget) accepts or rejects it, and does a rejection not revert? Then publishing a newer out-of-band
   reading first blocks the in-band intermediate reading. A band rejection, unlike a budget rejection,
   does not clear when the window rolls. (A failed signature check reverts, so storing before
   `verify` is not this issue.)
3. Is there an activation delay or a pending slot, so a print cannot be used in the transaction that
   publishes it?
4. What is the ordering floor: the last stored print or the last accepted print?

Severity: medium if the selected or frozen rate prices mints, redemptions or swaps that move value,
even when it needs a lagging keeper or two quick source moves. Low only if every value path halts on a
stale or frozen rate.
Benign: ordering against the last accepted publication; a delay before activation; a gated (keeper-only)
publisher; a rejected print adopted later when anyone re-judges it after the window rolls.
