---
name: sync-vault loss-state accumulator drift
description: Detects ERC-4626 sync-mode vaults where deposit adds a nominal amount to a recovery-ceiling accumulator but the loss-branch withdraw subtracts a proportionally-scaled amount that exceeds the nominal add, silently shrinking the accumulator on each round-trip until the recovery ceiling is pinned at the loss-state value forever.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, vault, accounting, sync-flow, loss-state, recovery-ceiling, griefing]
attribution-name: T3tris Vault Cyfrin Audit 2026
attribution-url:
---

## Patterns

1. Grep for `fullMulDiv` or `fullMulDivUp` in functions named `*withdraw*`, `*redeem*`, `*syncOutflow*`, or `*release*`, where one operand is a state variable matching `*initialAssets*`, `*initialBalance*`, `*seedAssets*`, or `*recoveryAssets*` and the result is subtracted back into that same variable (look for `rawSub`, `-=`, or `s.<var> = <var> - ...`).

2. Grep for `rawAdd` or `+=` on the same variable identified in pattern 1 inside functions named `*deposit*`, `*mint*`, `*syncInflow*`, or `*feature*`. Confirm the addition is the raw incoming `assets` or `amount` value without any scaling.

## Assessment

For each pair of matches (deposit path + withdraw loss branch operating on the same accumulator variable):

1. **Confirm the asymmetric arithmetic.** Read both the deposit and withdraw functions. In the deposit path, verify the accumulator is incremented by `assets` directly (no scaling). In the withdraw loss branch, verify the accumulator is decremented by `fullMulDiv(accumulator, assets, currentAssets)` (or equivalent), where `currentAssets` is the live silo or strategy balance.

2. **Confirm the loss-state condition.** Check whether `currentAssets < accumulator` is possible at the time of the withdraw. If the vault tracks a drawdown (strategy value fell below the recorded high-water mark), the ratio `accumulator / currentAssets > 1` and the scaled subtraction exceeds the nominal deposit, causing net downward drift per round-trip.

3. **Confirm the accumulator controls a downstream clamp.** Check whether `totalAssets` or an equivalent PPS numerator is clamped to `min(accumulator, currentAssets)`. If so, driving the accumulator down to `currentAssets` pins PPS at the loss-state ratio permanently, even after full strategy recovery.

4. **Assess profit-routing impact.** Check whether the profit branch of the withdraw skims `currentAssets - accumulator` to a treasury or fee recipient. If yes, once the accumulator is artificially depressed, all subsequent strategy recovery is routed to the treasury and existing LPs cannot reach their pre-drawdown PPS regardless of strategy performance.

**Severity adjustments:**
- Keep at **high** if the vault is open to permissionless depositors and the round-trip costs only gas (attacker nets zero position, bears no economic risk).
- Downgrade to **medium** if a deposit whitelist restricts who can open a position (attack requires a whitelisted actor with sufficient capital, limiting realistic exploitability).
- Downgrade to **medium** if a separate guard prohibits deposits when `currentAssets < accumulator` (loss-state deposit block removes the precondition).

**Benign cases:**
- The deposit path also scales the accumulator addition by `accumulator / currentAssets` when in loss state, making deposit and withdraw symmetric.
- The accumulator is recomputed from scratch each time (not persistent state) so round-trip drift cannot accumulate across transactions.
- The vault enforces `require(currentAssets >= accumulator)` before allowing deposits, preventing round-trips in loss state.
- Deposit and withdraw paths are proven to be inverses via a formal invariant test that checks `accumulator` is unchanged after a deposit-then-full-withdraw cycle.
