---
name: unchecked-claim-accumulator-subtraction
description: Detects async vault claim paths that subtract a cross-epoch ratio-derived settled value from a global share accumulator inside an unchecked block, where the ratio can exceed 1 if the global was independently reset after the request was stored.
languages: solidity
severity-default: high
confidence: low
tools: [Grep, Read]
tags: [erc4626, async-vault, unchecked, share-accounting, underflow, migration]
attribution-name: OWASP SC-02 / Perennial V2 Sherlock 2024 #26 / T3tris Vault Cyfrin Audit 2026
attribution-url: https://github.com/sherlock-audit/2024-02-perennial-v2-3-judging/issues/26
related-checks: [async-vault-epoch-reset-stale-claim, async-vault-donation-bridges-inflated-silo-claim]
---

## Patterns

1. Grep for `unchecked` blocks inside functions named `claimRedeem`, `claimWithdraw`, `_claimRedeem`, `_settleRedeem`, or `processRedeem`. Within those blocks, look for subtraction patterns: `total.*TrueShares\s*-\s*settled`, `total.*Shares\s*-=`, or the storage-write form `s\.total.*Shares\s*=\s*total.*Shares\s*-`.

2. Inside the same function scope, grep for `fullMulDiv(` or `mulDiv(` where the three arguments reference (a) a per-request stored field (`req\..*Shares`, `req\..*Virtual`), (b) a global accumulator (`total.*TrueShares`, `total.*VirtualShares`), and (c) another global accumulator. This is the settled-value computation that can exceed the global if accumulators were reset independently.

## Assessment

For each match:

1. **Confirm the settled value is a cross-global ratio.** Read the settled-value expression. It should be of the form `fullMulDiv(req.storedField, globalA, globalB)`. Verify that `globalA` and `globalB` are stored separately and can reach the state `globalA / globalB > 1` — most commonly when `globalB` (the virtual denominator) is reset to 0 and then seeded with a fresh small value while `globalA` (the true numerator) retains a large pre-reset value, or vice versa.

2. **Confirm the subtraction is unchecked.** If the `settled` value can exceed `globalA`, the `unchecked` subtraction wraps to a near-`MAX_UINT` value, corrupting the global accumulator for every subsequent claimer.

3. **Confirm no pre-subtraction cap.** Look for `min(settled, globalA)` or a `require(settled <= globalA)` guard applied before the unchecked block. If absent, the check fires.

**Severity adjustments:**
- Keep at **high** if the corrupted global persists across all future claims, enabling the global to be drained to zero or wrapped.
- Downgrade to **medium** if the protocol applies `Math.min(settled, totalShares)` before entering the unchecked block, bounding settled to at most the current global.

**Benign cases:**
- The subtraction is inside a checked block (no `unchecked` wrapper), so overflow reverts cleanly.
- A `require(settled <= totalShares)` or equivalent guard precedes the subtraction in the same call frame.
- The two global accumulators are always updated atomically (same transaction, same storage slot) so their ratio is bounded to [0, 1].
