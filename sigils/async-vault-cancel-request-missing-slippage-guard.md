---
name: async vault cancel request missing slippage guard
description: Detects async vault cancel/decrease-request functions that compute payout as min(requestedAssets, previewRedeem(...)) but accept no minAmountOut parameter, leaving callers unprotected when silo exchange rates deteriorate between preview and execution.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [erc4626, erc7540, async-vault, slippage, input-validation, cancel-request]
attribution-name: ERC-5143 / LoopFi Code4rena 2024
attribution-url: https://eips.ethereum.org/EIPS/eip-5143
---

Grep for cancel/decrease-request function declarations:

```
function (cancel|decrease).*(Request|Deposit|Redeem|Withdraw)\b
function withdraw(
function redeem(
```

For each candidate function found:

1. **Confirm the payout formula.** Read the function body (and any preview helper it calls). Verify that `actualPayout = min(userRequestedAmount, previewRedeem(shares))` or equivalent. If the silo exchange rate can fall below the user-requested amount, a shortfall is reachable.

2. **Check the function signature for a minimum-output guard.** Read the parameter list. If no parameter named `minAmountOut`, `minAssets`, `minOut`, `minReceived`, or similar exists, proceed.

3. **Check for an inline require guard.** Read the function body after the payout computation. If there is no `require(actualPayout >= callerMinParam)` or equivalent revert on shortfall, the slippage gap is confirmed.

4. **Assess exploitability.** Determine whether the underlying silo/strategy can lose value (lending bad debt, yield vault slashing, AMM impermanent loss). If yes and the async queue window is hours or days, severity is real.

**Severity adjustment:**
- Upgrade to **High** if the underlying is high-risk (leveraged lending, volatile AMM LP) and the exchange rate can crash suddenly.
- Keep **Medium** for cancel/decrease-request flows with realistic but gradual exchange-rate deterioration and multi-epoch queue windows.
- Downgrade to **Low** if the underlying is low-risk and the rate deterioration window is short.
- Downgrade to **Informational** if the protocol documents no slippage protection as intentional.

**Benign cases (dismiss):**
- Function signature includes `minAmountOut`, `minAssets`, `minOut`, or equivalent with an enforcing `require`.
- Payout is always exactly `requestedAssets` — the silo guarantees 1:1 redemption by design (plain ERC-20 escrow, no exchange rate).
- Function is admin-only or otherwise not accessible to end users.
