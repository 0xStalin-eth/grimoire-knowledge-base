---
name: pretrade rate applied to whole trade
description: Flags subsidy, bonus or throttle rates read once from pre-trade state and applied to a whole trade that crosses the rate's equilibrium point.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [pricing, subsidy, bonding-curve, equilibrium, flash-loan]
related-checks: [entry-exit-price-source-asymmetry]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (H-02)
---

Look for multipliers that pay one side extra (or charge it extra) depending on how far a state ratio is from
a target. Grep for: `lambda`, `bonus`, `subsidy`, `discount`, `throttle`, `multiplier`, `fundingRate`, `boost`,
and for ratios built from `totalSupply()` divided by another supply or an oracle rate.

For each match:
1. Is the rate read once, before the trade, and then multiplied by the full trade amount?
2. Does the trade itself move the ratio the rate depends on? Find where it would cross the target
   (for example supply ratio == oracle rate). Can a single call be bigger than that crossing amount?
3. Is the only cap a total budget (headroom, reserve, pool), with no cap at the crossing point?
4. After a trade past the crossing, does the ratio overshoot to the other side of the target? Does
   another path (redeem, exit) then pay at the overshot ratio? If that exit is itself a separate
   asymmetric-exit bug, report both: they are not duplicates and neither fix closes the other.

Compare one large trade with the same dollars split at the crossing point. If the single trade spends more
budget or produces a better round trip, flag it. Flag only single trades larger than the crossing amount.

Severity: high if the subsidy budget exceeds supply * (rate - 1), and a borrowed trade plus a reversing exit
pulls out budget that counts in NAV or user backing. The gain grows with size until the budget runs out.
The AMM premium cancels between legs, so do not require it. Medium if there is no reversing leg.
Benign: the math integrates the rate over the trade or splits it at the crossing; trades below the crossing
(they lose on a round trip); a subsidised mint below the crossing that only moves the ratio toward the
target with no NAV change; a multiplier pinned to 1 on stale data by design.
