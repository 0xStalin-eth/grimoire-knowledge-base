---
name: oracle deviation anchor lockout
description: Flags oracle deviation guards anchored to the last accepted value with a hard band cap, so a real move past the cap is never accepted and the feed goes stale.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [oracle, circuit-breaker, deviation, liveness, stale]
related-checks: [push-oracle-observation-ordering]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-02)
---

Grep for deviation guards: `deviation`, `band`, `bandBps`, `maxDeviation`, `moveBudget`, `acceptedRate`,
`lastPrice`, `lastGoodPrice`.

For each guard:
1. What is the new print compared with? Is it the last ACCEPTED value, which changes only when a print passes?
2. Is the band's maximum a hard-coded constant? If the source moves further than that maximum
   (devaluations, depegs), can any governance retune accept it?
3. Is there an on-chain path that walks the anchor toward the source in steps, using values the source publishes?
4. Can an attacker push the anchor down within budget with poisoned prints, so the honest value is now
   past the cap?
5. Can a budget retune leave the window budget below what the window has already spent? Then even an
   unchanged print is refused. It goes stale only if the window plus the maximum observation age can
   exceed the staleness limit, and the guard validator does not stop that.
6. Can the budget be set below the band? Then a real move between the two is refused in every window.
7. When the feed goes stale, which paths keep running on the frozen value (mints, redemptions)?

Severity: medium if the feed can get stuck for good while a value-moving path (for example a mint)
reads the rate with no staleness check. Low if every path halts when the feed is stale, or if the stuck
state needs a governance retune inside the compiled ranges, another retune undoes it, and the shipped
config does not use those values.
Benign: the anchor moves toward the source by at most the band each window; governance can set the
anchor under a timelock; fixed-window budget accounting that allows up to 2x the budget across a window
boundary while each print stays in band; a trusted retune of staleness parameters that revives a proven print.
