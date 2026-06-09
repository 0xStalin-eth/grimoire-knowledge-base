---
name: erc4626 dilutive fee mint net vs gross supply mismatch
description: Detects ERC-4626 fee-as-dilutive-mint implementations that use netSupply instead of grossSupply as the multiplier, or totalAssets instead of netAssets as the denominator, causing systematic fee underpayment when unclaimed fee shares are present.
languages: solidity
severity-default: medium
confidence: low
tools: [Grep, Read]
tags: [erc4626, fees, dilutive-mint, rounding]
related-checks: [erc4626-maxredeem-rounding-floor-asymmetry]
attribution-name: MetaMorpho (Morpho Labs)
attribution-url: https://github.com/morpho-org/metamorpho
---

## Patterns

1. **Fee share mint via fullMulDiv:** Grep for `fullMulDiv` or `fullMulDivUp` in functions whose name contains `fee` or in blocks with nearby `// fee` comments. These are candidate dilutive mint computations.

2. **Net supply variable:** Grep for `totalSupply() -`, `totalNetSupply`, `netSupply`, `grossSupply`, or `unclaimedFeeShares` — any expression that splits gross from net supply. Confirms the contract tracks unclaimed fee shares separately.

3. **Net supply as numerator:** Grep for `fullMulDiv` calls where the second argument (numerator) is `netSupply`, `totalNetSupply`, or a variable assigned `totalSupply() - X` where X relates to fees. This is the management fee bug pattern.

## Assessment

**Step 1 — Confirm gross/net split exists.** Read the contract's state variables and supply-accounting logic. If the contract never separates gross from net supply (no `unclaimedFeeShares` or equivalent storage), dismiss — the bug only manifests when unclaimed fee shares can accumulate.

**Step 2 — Identify the correct formula.** The correct dilutive mint for a fee of `feeAssets` is:
`feeShares = feeAssets × grossSupply / (netAssets - feeAssets)`
where `netAssets = totalAssets × netSupply / grossSupply`.

**Step 3 — Check the numerator (management fee bug).** For each fee mint formula, identify the supply multiplier:
- Numerator = `grossSupply` or `totalSupply()` — correct, dismiss.
- Numerator = `netSupply` / `totalNetSupply` — flag: under-mints because dilution spreads over `grossSupply` but compensation is sized against the smaller `netSupply`.

**Step 4 — Check the denominator (performance fee bug).** Identify the denominator:
- Denominator = `netAssets - feeAssets` — correct, dismiss.
- Denominator = `totalAssets - feeAssets` with unclaimed fee shares present — flag: denominator is inflated (includes assets attributed to unclaimed fee shares), shrinking the mint below what compensates the fee recipient.

**Severity adjustment:**
- Keep **medium** when unclaimed fee shares can accumulate across settlement periods (normal operation).
- Downgrade to **low** if the design forces the fee recipient to claim before every settlement (forced-claim), preventing accumulation.

**Benign cases:**
- (a) Single supply variable — no gross/net split and `unclaimedFeeShares` is always zero between settlements. Verify there is no storage variable tracking pending or unclaimed fee shares.
- (b) Fee is a direct asset transfer to the recipient, not a share mint — formula does not apply.
