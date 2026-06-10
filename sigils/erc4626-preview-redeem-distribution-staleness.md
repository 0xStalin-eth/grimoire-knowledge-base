---
name: erc4626 preview redeem distribution staleness
description: Detects ERC-4626 settlement functions that compute asset distributions from previewRedeem before the actual redeem call, then discard or ignore the actual return, causing surplus assets to accrue to privileged recipients instead of following the intended distribution.
languages: solidity
severity-default: low
confidence: low
tools: [Grep, Read]
tags: [erc4626, accounting, settlement, async-vault, previewRedeem, surplus-misallocation]
related-checks: [async-vault-cancel-request-missing-slippage-guard]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

Grep 1 (primary): search for `previewRedeem(` in Solidity source files (`.sol`).

Grep 2 (confirmation): in each matching file, look for a line containing `.redeem(` or `_safeRedeem(` that is a bare statement — i.e., the line does NOT begin with an assignment token (`=`, `uint`, `address`, or a variable name followed by `=`).

## Assessment

1. **Scope the function.** Read the full body of the function containing the `previewRedeem` call. Confirm it is state-changing (not `view`/`pure`) and performs transfers.

2. **Trace the preview value.** Verify the `previewRedeem` return is stored and used as input to at least one downstream computation that determines transfer amounts (fees, payouts, `safeTransfer` arguments, or distribution structs).

3. **Locate the paired redeem call.** Find the `redeem()` or `_safeRedeem()` call in the same function. Check whether its return value is captured AND used to recompute the same distribution amounts from step 2. If the return is discarded or downstream amounts are never recalculated from it, this is a **true positive**.

4. **Identify the surplus recipient.** Determine where the `actual − preview` surplus ends up (stranded on contract, routed to curator/treasury/owner, etc.).

**Severity adjustment:**
- Upgrade to **Medium** if the surplus flows to a privileged actor (curator, owner, admin, treasury) who also controls silo or strategy selection — this creates a self-rewarding misreporting path.
- Keep **Low** for compliant ERC-4626 silos where the delta is rounding-level dust (≤1 wei per call) and no privileged routing occurs.
- Downgrade to **Informational** if the protocol explicitly documents that `previewRedeem` is used as-is and surplus handling is acknowledged separately.

**Benign cases (dismiss):**
- `previewRedeem` appears only in `view`/`pure` functions or preview-only helpers with no paired `redeem()` in the same execution path.
- The `redeem()` return IS captured and used to recompute all downstream distribution amounts before any `safeTransfer` executes — this is the correct pattern.
- `previewRedeem` is used only for capacity/guard checks (slippage floor, maxDeposit gate) and not as the direct basis for transfer amounts.

**Note on ideal tool:** This pattern requires taint tracking (preview value → distribution struct → transfer; redeem return → not used). A CodeQL inter-procedural taint query detects this with high precision. Agentic grep + read is the fallback.
