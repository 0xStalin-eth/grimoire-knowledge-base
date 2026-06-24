---
name: Spark-PSM3 basin liquidity sourcing
description: Why naive `IERC20(collateralToken).balanceOf(basin)` undercounts fillable swap output on a Spark-PSM3-derived basin whose swap-token pocket mints collateral on demand via an external venue (e.g. MakerDAO PSM)
vulnerability-class: integration-misestimation
languages: [solidity]
tags: [spark-psm3, grove-basin, integration, liquidity-estimation, pocket, psm]
---

# Spark-PSM3 Basin Liquidity Sourcing

## What this captures

In any Spark-PSM3-derived basin (Spark PSM3, Grove Basin, future forks) the output side
of a `swapExactIn`/`swapExactOut` for the **collateral asset** is funded by a two-tier
source model:

```
fillable collateralOut ≈ collateral.balanceOf(basin)
                       + collateral.balanceOf(pocket)
                       + (swapToken.balanceOf(pocket) convertible 1:1 via external venue, if open)
```

If the swap-token pocket can mint collateral on demand by exchanging the pocket's idle
swap-token balance through an external venue (e.g. MakerDAO PSM `buyGem` with `tout == 0`),
then **the dominant source of fillable collateral is the pocket's swap-token balance, not
the basin's own collateral balance**.

Integrators that estimate available liquidity using `IERC20(collateralToken).balanceOf(basin)`
will under-estimate by orders of magnitude in steady state. The basin's own collateral
balance is usually a small residue — by design, idle inventory rests in the pocket as the
swap token to capture external yield.

## Why it matters

- Integrators routing swaps will reject or shrink trades the basin could trivially fill,
  pushing flow to inferior venues.
- Reviewers auditing integrator code may not flag the wrong estimator because the cap is
  a *safe lower bound* — it never over-promises — only an under-bound.
- The naive estimator becomes correct only when the external mint-on-demand path is closed
  (e.g. `PSM.tout != 0`, drained PSM reserves). A robust estimator must gate accordingly.

## Code shape (Grove Basin example)

`_withdrawLiquidityInPocket` (`GroveBasin.sol:886-901`) for `assetOut == collateralToken`:

```solidity
} else if (asset == collateralToken) {
    uint256 basinBalance = IERC20(asset).balanceOf(address(this));
    if (basinBalance < amount) {
        uint256 deficit = amount - basinBalance;
        uint256 withdrawn = IGroveBasinPocket(pocket).withdrawLiquidity(deficit, asset);
        if (withdrawn < deficit) revert InsufficientFunds();
    }
}
```

The pocket's `withdrawLiquidity` (e.g. `UsdsUsdcPocket.withdrawLiquidity(USDC)` at
`L91-115`) closes the deficit by:
1. Optional gating check on the external venue (e.g. `PSM.tout() == 0`).
2. Approving swap-token to venue, calling the venue's mint/buy entry point (e.g. `PSM.buyGem(self, remainder)`).
3. Transferring the full `amount` of collateral to Basin.

Note `pocket.availableBalance(collateralToken)` returns only the pocket's **resting**
collateral balance (`UsdsUsdcPocket.sol:132-137`) — it does **not** include the
venue-convertible swap-token reserve. So even querying the pocket directly understates
capacity for forms with on-demand minting.

## What to look for

In integrator / router / aggregator code that swaps against a basin (or other
PSM3-derived liquidity venue with pocket-mediated mint-on-demand):

1. Liquidity caps written as `IERC20(<collateralToken>).balanceOf(<basin>)` — likely too low.
2. `pocket.availableBalance(<collateralToken>)` used as the upper bound — also too low when
   the pocket converts swap-token via an external venue.
3. Conservative estimators with no gating on the external venue's state (`tout`, reserve
   depletion, paused withdrawal). Estimator is conditionally correct, but unconditionally
   used.
4. Reverse direction (swap-out swap-token) is symmetric only when the swap token is also
   pocket-deployed (Morpho/Aave pockets); the deficit branch for swap token is different
   (see Grove `_withdrawLiquidityInPocket` swap-token branch L889-890 — pulls from pocket
   unconditionally via `withdrawLiquidity`, not via a deficit calc).

## Recommended estimator

In order of reliability:

1. **`eth_call` / staticcall the actual `swapExactIn(...)`** — single deterministic answer
   that captures basin balance, pocket balance, external venue conversion, swap-size cap,
   pause flags, and rate staleness. Requires simulating from an address with input-asset
   approval, or using state overrides.
2. **Source-model estimate**, gated on external venue state:
   ```
   cap = collateral.balanceOf(basin) + collateral.balanceOf(pocket)
   if external venue is open (e.g. PSM.tout == 0):
       cap += min(swapToken.balanceOf(pocket), venueReserveOfCollateral)
   cap = min(cap, maxSwapSize converted to collateral terms)
   ```
3. **`balanceOf(basin)` alone** — defensible only as a deliberately ultra-conservative floor.

## Variant scope

Within Grove Basin specifically: only one `collateralToken` deficit branch exists, so the
on-codebase variant analysis is closed. However, **any new pocket implementation that wraps
an external mint-on-demand venue will exhibit the same `availableBalance` undercount**.
When auditing or reviewing a new pocket:

- Confirm `availableBalance(collateralToken)` accounts for venue-convertible swap-token
  reserves, OR
- Confirm callers do not use `availableBalance` as an upper bound on collateral capacity.

Across the Spark-PSM3 family this applies to:
- Original Spark PSM3 with USDS↔USDC PSM.
- Any future fork that introduces a yield-bearing pocket with an external 1:1 conversion
  path for the collateral asset (e.g. Curve stable pools, Aave reserves with collateral
  swap, Morpho with collateral underlying).

## Assessment guidance for new findings citing this pattern

A finding is **valid** when:
- Integrator code uses `balanceOf(basin)` or `pocket.availableBalance(collateral)` as a
  decisive upper bound on `swapExactIn`/`swapExactOut` output (not as a floor), AND
- The pocket's `withdrawLiquidity(collateral)` implements an external mint-on-demand path
  whose throughput meaningfully exceeds the basin's resting collateral balance, AND
- The external venue is operational under default conditions (`tout == 0`, reserves
  available).

A finding should be **downgraded** when:
- The cap is a deliberate conservative floor with documented rationale.
- The external venue is closed / discontinued (the naive cap becomes correct in this
  degenerate state).
- The integration is intra-protocol with stronger atomicity guarantees (e.g. same
  multicall as basin liquidity provisioning).

## References

- Project tome: `grimoire/tomes/swapexactin-buidl-usdc-liquidity.md` (Grove Basin-specific walkthrough with worked example).
- Grove Basin core flow: `grimoire/tomes/grovebasin-core.md` (sections on swap and pocket integration).
- Grove Basin pocket details: `grimoire/tomes/pockets.md`.
