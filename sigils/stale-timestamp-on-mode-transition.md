---
name: stale timestamp on mode transition
description: Detects state transition functions that reset domain variables but omit a time-tracking variable used by the next mode for fee/reward accrual, front-loading accumulated fees onto post-transition holders.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [fee-accounting, timestamps, state-machine, mode-transition]
---

Grep for mode-transition function bodies using these patterns:

```
isVaultOpen\s*=\s*false
isClosed\s*=\s*true
isPaused\s*=\s*true
isOpen\s*=\s*false
function (close|_close|shutdown|pause|deactivate|endPeriod|finalize|disableEndOfFund|disableVault)\(
```

For each candidate transition function found:

1. **List every storage variable written** — identify balance resets (`= 0`), share resets, flag flips.
2. **Search the contract for a time-tracking variable** — look for `lastSettlementTimestamp`, `lastAccruedAt`, `lastAccrualTimestamp`, `lastUpdateTimestamp`, `lastHarvestTimestamp`, `lastHarvestTime`, `lastFeeTimestamp`, `lastClaimTime`, `lastRewardTimestamp`. If one exists and is NOT written inside the transition function, flag it as a candidate.
3. **Confirm the prior mode omits updates** — read all user-facing functions active before the transition (deposits, withdrawals, transfers). Verify none call a settle/accrue/update function that writes the time-tracking variable. If the prior mode updates it regularly, dismiss.
4. **Confirm the next mode reads it for accrual** — locate fee/reward/interest functions that run after the transition. Check whether they compute `timeElapsed = block.timestamp - <time-tracking variable>`. If no such computation exists post-transition, dismiss.

**Severity adjustment:**
- Upgrade to **High** if the prior mode can run for months without touching the timestamp.
- Keep **Medium** for typical windows of days or weeks.
- Downgrade to **Low** if a maximum open period is enforced on-chain.

**Benign cases (dismiss):**
- The transition function calls settle/accrue before resetting state (timestamp is current at transition).
- The next mode computes fees from a separately tracked variable that IS reset.
- The prior mode has per-transaction accrual, keeping the timestamp always fresh.
- A guaranteed initialization function immediately after the transition resets the variable.
