---
name: erc4626 curator recovery-receiver zero-cost mint
description: Detects ERC-4626 aggregator vaults where a privileged actor (curator, admin, or owner) is set as the recovery receiver for failed sub-vault deposits, enabling that actor to acquire vault shares at zero net token cost when a deposit triggers the zero-shares recovery path.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [erc4626, vault, dos, bootstrap, curator, privileged-actor, recovery-path, zero-cost-mint, accounting-before-interaction]
related-checks: [erc4626-bootstrap-accounting-dos, erc4626-split-deposit-accounting-inflation]
attribution-name: T3tris Vault Cyfrin Audit 2026
---

## Patterns

1. **Privileged actor as `recoveryReceiver` in a deposit context.** Grep for `recoveryReceiver.*curator`, `recoveryReceiver: s\.curator`, `recoveryReceiver.*admin`, `recoveryReceiver.*owner` in Solidity files. Read the enclosing function: flag only when it appears inside a deposit, mint, or syncInflow path — not in withdraw, redeem, or migration paths.

2. **Share issuance before sub-vault validation.** In the same deposit function (or its caller), look for `_mint(`, share assignments (`shares =`, `sharesToMint =`), or ERC-4626 `deposit`/`mint` return values committed before the `_safeDeposit` call or before the `expectedShares` result is checked. Confirms the privileged actor receives shares regardless of sub-vault outcome.

3. **Caller eligibility on deposit entry point.** Check whether the contract allows the privileged actor named in `recoveryReceiver` to call `deposit()` or `mint()` directly. Look for access-control modifiers or whitelist checks on those entry points. If the curator role is explicitly excluded from depositing, the zero-cost path is blocked.

## Assessment

Confirm all three conditions hold simultaneously:
1. `recoveryReceiver` resolves to a privileged insider (curator, admin, owner) — not `msg.sender`, not a burn address, not an immutable fee treasury.
2. Vault shares are minted or committed to the caller BEFORE the sub-vault result is validated (accounting-before-interaction ordering).
3. The privileged actor named as `recoveryReceiver` is also permitted to invoke `deposit()` or `mint()`.

**When all three hold:**
- **Bootstrap path** (`totalSupply == 0`): one call permanently bricks the vault at zero cost — vault enters `totalSupply > 0, totalAssets = 0`, blocking all future deposits.
- **Loss-state / profit-split path**: loopable in a single transaction to continuously dilute shareholders.

**Severity adjustments:**
- Keep **High** if the bootstrap path is reachable: silo PPS can grow above 1 via normal yield AND no minimum deposit floor prevents `assets < silo_PPS` deposits.
- Keep **High** if the loss-state griefing loop is reachable: vault can be in loss state AND curator can loop in one transaction.
- Downgrade to **Medium** if curator is excluded from depositing by enforced access control.
- Downgrade to **Medium** if silo PPS is permanently pegged at 1:1 (yield accrual impossible), so `previewDeposit(N) == 0` can never occur for any `N >= 1`.

**Benign cases:**
- `recoveryReceiver` is `msg.sender` — any depositor has equal access; no insider advantage.
- `recoveryReceiver` is a burn address or immutable fee sink that cannot act as a depositor.
- A `require(expectedShares > 0)` guard (or equivalent) reverts the full deposit before any share mint or accounting update when the sub-vault returns 0 shares.
- The curator role is permanently and provably excluded from calling `deposit()` or `mint()` by access control.
