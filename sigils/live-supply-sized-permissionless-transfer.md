---
name: live supply sized permissionless transfer
description: Flags permissionless one-way transfers between internal buckets sized from a live totalSupply or balance that a reversible round trip can lower first.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [griefing, totalSupply, round-trip, buckets, cap]
related-checks: [spot-price-in-irreversible-permissionless-action]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-12)
---

Find external functions with no role gate that move value from one internal bucket to another (escrow to
hub, reserve to treasury, excess to surplus). Grep for: `convertExcess`, `sweepExcess`, `skim`, `excess`,
`surplus`, `retained`, `cap`, together with `totalSupply()` or `balanceOf(address(this))`.

For each one:
1. Is the amount kept computed as factor * live supply (or balance) read at call time?
2. Can a user lower that supply with a reversible action (redeem, burn, contract, withdraw) and restore it
   right after the call (mint, expand, deposit) for little cost?
3. With factor > 1, lowering the supply by d lowers the kept amount by factor * d but the bucket by only d,
   so (factor - 1) * d becomes "excess". Confirm this on the real formula.
4. Is the transfer one-way for users, so only governance can move the funds back?

Severity: medium if it moves value that later users depend on (subsidy, reserve) and can be repeated at
near-zero cost after every refill, even when the funds stay in the protocol's books and the loss per user
is small. Low if it only changes accounting labels and no one can be paid less.
Benign: the function has a role gate, or it measures against a supply a caller cannot lower (a high-water
mark over a window, or a snapshot from an earlier block); a documented vent that moves only excess that
really exists. If the moved funds then feed a mispriced venue, report it under that venue.
