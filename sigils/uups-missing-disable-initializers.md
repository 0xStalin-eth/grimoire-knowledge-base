---
name: uups missing disable initializers
description: Detects UUPS upgradeable contracts with constructors that omit _disableInitializers(), leaving the implementation directly initializable.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [initialization, uups, upgrade, best-practice]
related-checks: [uups-authorize-upgrade-mutable-delegate]
---

Look for these two signals — both must be present for a candidate:

**Signal 1 — UUPS inheritance:**
- Grep `.sol` files for `UUPSUpgradeable`

**Signal 2 — constructor defined in the same file:**
- For each match from Signal 1, grep the same file for `constructor`

For each file with both signals, read the constructor body and check for `_disableInitializers()`.

## Assessment

Confirm the constructor belongs to this contract (not only to an imported parent). Read the constructor body: if `_disableInitializers()` is present, dismiss.

Check whether a public `initialize()` function guarded by the `initializer` modifier exists. If it does, the raw implementation address is directly initializable — report as a finding.

Severity adjustments:
- Keep at **low** if the codebase uses OZ v5 or later (the `onlyProxy` immutable guard blocks upgrade abuse via the implementation address)
- Upgrade to **medium** if the codebase uses OZ v4 or earlier (no `onlyProxy` immutable, so the UUPS upgrade path on the implementation may remain open)
- Upgrade to **medium** if `initialize()` writes state that the proxy reads through a shared storage layout (initializing the implementation could corrupt proxy storage)

Benign cases (dismiss):
- Constructor already calls `_disableInitializers()`
- No constructor defined in this contract; an inherited parent constructor already calls `_disableInitializers()`
- Contract is not deployed behind a proxy (no proxy deployment in test setup or scripts)
- Abstract contract with no `initialize()` of its own (concrete subclass handles initialization)

Note: grep-based detection cannot inspect constructor bodies automatically. After locating candidates, manually verify that the constructor shown is the one belonging to the UUPS-inheriting contract, not an unrelated contract in the same file. A Semgrep AST rule matching `constructor() { ... }` without a `_disableInitializers()` call is the preferred automated path.
