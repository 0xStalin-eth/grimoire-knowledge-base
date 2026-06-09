---
name: eip712 version hash constant in dual inheritance
description: Detects contracts inheriting both ERC20PermitUpgradeable and a custom EIP-3009 implementation where a hardcoded VERSION_HASH constant can diverge from EIP712Upgradeable storage on a version-bumping upgrade.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [eip712, eip3009, eip2612, permit, domain-separator, upgrade-safety]
related-checks: []
---

Look for contracts combining two independent EIP-712 domain separator sources:

- A `private constant` named `VERSION_HASH` (or similar) holding a hardcoded `keccak256` of a version string
- The same file or a direct parent also inheriting `ERC20PermitUpgradeable` or `EIP712Upgradeable`
- A `DOMAIN_SEPARATOR()` override that calls the custom `_domainSeparator()` rather than OZ's `_domainSeparatorV4()`

For each match, read the contract and assess:

1. Does it inherit both `ERC20PermitUpgradeable` (or `EIP712Upgradeable`) AND a custom EIP-3009 abstract with its own `_domainSeparator()`?
2. Does the custom `_domainSeparator()` use the `VERSION_HASH` constant instead of calling `_EIP712Version()`?
3. Does the public `DOMAIN_SEPARATOR()` route to the constant-based path?

Decision:

- (1) yes AND (2) yes AND (3) yes → Low — `permit()` uses OZ storage; EIP-3009 ops and `DOMAIN_SEPARATOR()` use the constant; a version-bumping upgrade silently breaks `permit()` for all wallets using the getter
- (1) yes AND (2) yes AND (3) no → Informational — `DOMAIN_SEPARATOR()` returns OZ's path; `permit()` and EIP-3009 ops disagree internally but the public getter is correct
- (2) no — `_domainSeparator()` reads version from storage dynamically → Dismiss
