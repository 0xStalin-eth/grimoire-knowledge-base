---
name: lifecycle-gated refund with no terminal-state sweep
description: Detects user backing committed to a resting commitment that becomes unrecoverable once a lifecycle gate flips, because the terminal-state function does not sweep open commitments.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [funds-loss, lifecycle, escrow, finalization, order-book, auction]
---

Look for these patterns:

- A modifier that closes a window: `isNotFinalized`, `notFinalized`, `whenOpen`,
  `onlyActive`, `notExpired`, `beforeDeadline`, `notSettled`, `notClosed`, `isOpen`,
  `inAuction`. Find via `grep -rnE "modifier (isNot|when|only|before|not)[A-Z][A-Za-z]*"`.
- The **same** modifier applied to *both* an entry function (`create*`, `place*`,
  `deposit*`, `bid*`, `subscribe*`, `open*`) **and** an exit/cancel function (`cancel*`,
  `withdraw*`, `refund*`, `revoke*`, `exit*`). Grep callers of each modifier.
- Inside the entry function, *commitment-at-placement* writes — backing moves irreversibly
  into a resting struct without minting a position: `userShares[...] -=`, `balanceOf[...] -=`,
  `deposit(`, `safeTransferFrom(msg.sender, address(this),`, fields like `order.sharesUsed =`
  or `bid.escrow =`.
- The terminal-state function (flips `isFinalized` / `closed` / `settled`) and the payout
  function. Confirm neither iterates the open commitments — no loop over order book / bid
  list and no call back into the cancel/refund helper.

Assessment:

1. **Confirmed trap.** Shared modifier on entry + cancel, entry transfers backing at
   placement, terminal-state function does not iterate the commitment list, no alternative
   post-terminal recovery path → **high**; **critical** if it destroys already-settled value
   (e.g., a closing/exit commitment on a winning side).
2. **Partial trap.** Entry escrows fungible collateral but position is minted continuously,
   so payout against the user's *balance* recovers part of the value → medium.
3. **No trap.** Any of: cancel lacks the lifecycle modifier; terminal function sweeps open
   commitments; a separate `reclaim*` / `sweepOrders*` / rescue function exists → dismiss.

Keep severity high when the gate is flipped permissionlessly, off a schedule the user does
not control, or synchronously with the payout call (e.g., `claim*` itself calls `finalize*`).
