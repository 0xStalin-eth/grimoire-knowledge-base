---
name: accrued duration instant state trigger
description: Flags permissionless irreversible triggers whose duration test uses stored past time while the current state test can be met by a zero-length same-transaction dip.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [grace-period, liquidation, termination, flash-loan, timing]
related-checks: [spot-price-in-irreversible-permissionless-action]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (H-05)
---

Find clocks that add up time spent in a bad state and let anyone act once the total reaches a grace window.
Grep for: `grace`, `breach`, `belowFor`, `since`, `breachTime`, `due(`, `isDue`, `liquidatable`,
`enforce`, `terminate`, `block.timestamp -`.

For each trigger, find the two predicates it checks:
- the state predicate (for example `below == true` at the moment of the call), and
- the duration predicate (stored total + current stretch >= grace).

Then check:
1. Can the stored total, possibly built from several dips, stay at or above grace after the state has
   recovered? (For example, it only clears after a recovery that lasts a full window.)
2. Can the caller make the state predicate true inside its own transaction (large mint, swap, borrowed
   funds), so the current stretch is 0 seconds, and does the trigger accept that with none of the duration
   coming from the stretch open now?
3. Is the action permanent (termination, liquidation, settlement)? If it also locks a spot price the same
   transaction moved, report that part under the spot-price check.

If 1 and 2 hold, a borrowed transaction can force the action on a healthy position.
Severity: high if the action is permanent or pays the caller. The exposure lasts one grace window after a
recovery from a state that was already due. Medium if the action can be undone.
Benign: the trigger ignores a stretch that opened at the current timestamp after a recovery that held (a
fix that ignores every same-timestamp stretch lets a caller stall a valid close by lifting and re-dipping),
or the total resets on recovery. A breach the caller holds across blocks with its own capital is the
designed cumulative rule, not this bug.
