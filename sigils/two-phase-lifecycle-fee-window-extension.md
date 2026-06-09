---
name: two-phase lifecycle fee window extension
description: Detects paired enable/disable lifecycle functions where the enable phase keeps the system operating with no fee settlement, silently extending the fee-accumulation window before the disable function transitions the mode without backfilling the timestamp.
languages: solidity
severity-default: medium
confidence: low
tools: [Grep, Read]
tags: [fee-accounting, timestamps, state-machine, lifecycle]
related-checks: [stale-timestamp-on-mode-transition]
---

Grep for phase-1 flag-setter functions using these patterns:

```
function\s+(enable|arm|prepare|schedule|flag|mark)[A-Z]
endOfFund\s*=\s*true
isPendingClose\s*=\s*true
isScheduledForClose\s*=\s*true
isFlaggedForClose\s*=\s*true
```

For each candidate phase-1 function found:

1. **Confirm it is phase-1 of a two-phase lifecycle** — the function sets a boolean flag but does NOT change the primary operating-mode flag (`isVaultOpen`, `isOpen`, `isPaused`, etc.). The system continues operating after this call.

2. **Find the paired phase-2 executor function** — search for a corresponding `disable*`, `execute*`, `trigger*`, `close*`, or `complete*` function that checks or clears the phase-1 flag and performs the actual mode transition.

3. **Check for timestamp backfill in phase-2** — verify whether `lastSettlementTimestamp` (or equivalent: `lastAccruedAt`, `lastFeeTimestamp`, `lastUpdateTimestamp`) is written inside the phase-2 function BEFORE or during the mode transition. If absent, the unbilled window extends back to the last update before phase-1 was called — potentially the full preceding operating period.

4. **Confirm post-transition fee accrual uses the stale variable** — locate fee/reward/interest functions active after the transition and check whether they compute `timeElapsed = block.timestamp - <time-tracking variable>`. If yes and the variable is stale, flag.

**Severity adjustment:**
- Upgrade to **High** if the enable-to-disable window is unbounded or can span months.
- Keep **Medium** for time-bounded windows (days to weeks).
- Downgrade to **Low** if the maximum enable window is enforced on-chain.

**Benign cases (dismiss):**
- Phase-1 function triggers a fee settlement before setting the flag.
- Phase-2 function calls settle/accrue before resetting state.
- The operating mode between phase-1 and phase-2 has per-transaction fee accrual that keeps the timestamp current.
- A guaranteed initialization call immediately after phase-2 resets the timestamp.
