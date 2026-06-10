---
name: reward index carry-less accumulator
description: Detects per-share reward index accumulations that floor-divide by a supply denominator with no carry-forward, permanently stranding yield tokens already transferred into the contract.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [precision-loss, yield, reward-index, defi]
related-checks: [share-pool-fixed-1to1-deposit-rate]
---

Grep for reward index accumulation lines:

- `rewardIndex +=`
- `rewardPerToken +=`
- `accRewardPerShare +=`
- `rewardPerShare +=`
- `rewardPerTokenStored +=`
- `rewardIndex=` (no-space variant)

For each match, read the enclosing function body (~20 lines) and assess:

1. **Confirm floor-division shape** — RHS must be `(amount * PRECISION) / supply` where `supply` tracks depositor shares or total staked value.
2. **Check for carry-forward** — if any variable accumulates `% supply` remainder across calls (e.g., `pendingYield`, `remainder`, `dustAccum`), mark benign.
3. **Confirm token pre-transfer** — look for a yield claim, `transferFrom`, or `safeTransferFrom` before the index update in the same function. If tokens enter the contract before the division and no carry exists, this is a true positive; truncated dust is a real token loss.
4. **Assess PRECISION magnitude** — if PRECISION >= 1e27 and totalSupply is ~1e18 order, per-call dust approaches sub-wei; adjust severity to Low. For PRECISION = 1e18 with large totalSupply, strand can exceed 1 token per call; keep Medium.

Benign cases: (a) carry variable confirmed present, (b) PRECISION/totalSupply ratio makes dust sub-wei, (c) no token transfer precedes the index update and no tokens are held by the contract at that point.
