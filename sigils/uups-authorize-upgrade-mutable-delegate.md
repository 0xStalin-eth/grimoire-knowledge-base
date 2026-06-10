---
name: uups authorize-upgrade mutable delegate
description: Detects _authorizeUpgrade implementations that delegate the role check to a mutable external address instead of the contract's own role table.
languages: solidity
severity-default: high
confidence: medium
tools: [Grep, Read]
tags: [access-control, uups, upgrade, privilege-escalation]
related-checks: [uups-missing-disable-initializers]
---

Look for these two patterns — both must be present for a finding:

**Pattern A — external role delegation inside authorizeUpgrade:**
- Grep: `_authorizeUpgrade`
- In each match, read the function body and look for `someStateVariable.hasRole(` or `SomeInterface(someVariable).hasRole(` where the receiver is a state variable (not `address(this)`, not a local constant, not a constructor-immutable)

**Pattern B — setter for the delegated address:**
- Take the state variable name from Pattern A and grep for its assignment (`=`) in the same contract
- Confirm a setter function exists that updates it post-deployment

## Assessment

Confirm both patterns coexist in the same contract. If no setter exists or the address is `immutable`, dismiss.

Confirm the privilege gap: read the setter's access control modifier or inline role check. If the setter requires a different role than the upgrade authority (e.g., `WRAPPER_MANAGER_ROLE` vs `UPGRADER_ROLE`), a holder of the setter role can substitute a malicious contract whose `hasRole` always returns `true`, bypassing upgrade authorization entirely.

Severity adjustments:
- Keep at high if the setter role is operationally separate from the upgrade authority
- Downgrade to medium if the setter requires `DEFAULT_ADMIN_ROLE` or the same highest-level role (no privilege gap, but external delegation remains an anti-pattern)

Benign cases (dismiss without a finding):
- `_authorizeUpgrade` uses `onlyRole(UPGRADER_ROLE)` as a modifier with no external call
- Role check is `hasRole(ROLE, msg.sender)` with no receiver (implicit `this` via inherited AccessControl)
- The delegated address is `immutable` or set only in the constructor and has no setter
- The setter for the external address requires the identical role as the upgrade authority

Note: grep-based detection cannot automatically compare role constants across functions. After finding matches, manually verify the setter role vs upgrade role gap. For automated tracking, a Slither detector is the ideal tool.
