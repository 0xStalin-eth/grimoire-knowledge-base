---
name: performance fee hwm no admin reset
description: Detects monotonically-ratcheting high-water mark variables that gate performance fee collection without any admin reset mechanism, allowing an extreme initial value to permanently suppress fee revenue.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [erc4626, fees, performance-fee, high-water-mark, defi]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

Grep for HWM variable names: `HighWaterMark`, `highWaterMark`, `ppsHighWater`, `hwm`, `HWM`, `peakPps`, `highPps`.

For each match, grep for the ratchet-up assignment: a conditional block comparing a new value against the HWM variable then assigning it, e.g., `if (updatedPps > ppsHighWaterMark)` or `if (newPps > hwm)`.

Grep for the HWM variable appearing in a boolean condition that controls fee minting, `isProfit`, or share dilution logic.

## Assessment

**Step 1 — Enumerate all write sites.** Read every location that assigns to the HWM variable. If any write site can decrease the value, dismiss — a downward write path means the protocol can recover.

**Step 2 — Check for admin reset function.** Grep for any `public` or `external` function that writes to the HWM variable. If one accepts an arbitrary input or resets to current PPS on demand, dismiss.

**Step 3 — Confirm the gate is financially material.** Verify the HWM variable is read in the performance fee condition. If the gated function controls fee collection, the suppression impacts protocol revenue.

**Step 4 — Check initialization timing.** Identify where the HWM is first set. If it can be set before any user has deposited — e.g., via an oracle or pre-seeding mechanism reporting non-zero `totalAssets` while `totalSupply` is near-zero — flag as the higher-risk sub-scenario.

**Severity adjustment:**
- Keep **low** when no recovery path exists and the gate is purely over fee revenue.
- Upgrade to **medium** if the HWM also gates user-facing claim eligibility or deposit/redeem access.

**Benign cases:** admin reset function exists and is callable by a governance role; HWM is set once at deployment to a safe constant not derived from live supply or oracle; performance fee rate is zero.
