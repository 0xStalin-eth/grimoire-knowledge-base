---
name: liquidity waterfall no retry
description: Flags multi-venue funding loops that ask a venue for the whole remaining amount when its liquidity report is zero or unknown, skip it on revert, and never retry.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [liquidity, dos, erc4626, maxWithdraw, try-catch, morpho]
related-checks: [external-vault-value-not-bounded]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-14)
---

Find loops that pay out by pulling from several venues or strategies. Grep for `for (` together with
`try` ... `withdraw` / `redeem` and `catch` ... `continue`, plus `maxWithdraw`, `availableLiquidity`, and a
comment that treats a reported zero as "unknown, not illiquid" next to `if (withdrawable != 0)`.

For each loop:
1. How is the pull sized? Is it capped by the venue's liquidity report only when that report is non-zero?
2. Which venues report 0, or a value too high, while a withdrawal would still succeed?
   (Morpho Vaults V2 `maxWithdraw` returns 0. Other async vaults overreport.)
3. For such a venue, is it asked for the whole remaining shortfall? If it can pay only part, does the
   call revert, so the venue gives nothing?
4. Does the loop make a single pass with no retry at a smaller amount after later venues have paid?
5. Which payers cannot split their exit: rent or claim functions that take no amount, fixed-amount
   integrators, `maxRedeem` / `maxWithdraw` values that ignore venue liquidity?

Severity: medium if fixed-size payouts revert while the venues together hold enough. Preconditions: a
second venue behind the default one, the default venue short of liquidity, and an exit larger than the
later venues can pay. Low if every caller can split the exit.
Benign: a retry at smaller sizes, or liquidity reports that are exact or an upper bound for every listed
venue. A second pass alone is not enough when a single venue is partly liquid. A documented caveat naming
the zero report and the revert is not benign if it leaves out that the venues together could pay.
