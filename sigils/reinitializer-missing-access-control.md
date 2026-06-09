---
name: reinitializer missing access control
description: Detects reinitializer functions that write critical state without an access control modifier, while sibling governance functions in the same contract carry one.
languages: solidity
severity-default: informational
confidence: medium
tools: [Grep, Read]
tags: [access-control, upgradeable, initializer, upgrade-safety]
---

Grep for `reinitializer(` in Solidity files to locate candidate functions.

For each match, read the full function signature and check for co-present access control modifiers: `onlyOwner`, `restricted`, `onlyRole`, `onlyGovernor`, `onlyAdmin`, or any project-specific role modifier.

Read the same contract file and identify all other external/public state-writing functions. Check whether those sibling functions carry an access control modifier.

**Assess each match against these criteria:**

1. Does the reinitializer write to one or more state variables?
2. Does the function signature include an access control modifier alongside `reinitializer`?
3. Do sibling external/public state-writing functions carry an access control modifier?

- (1) yes AND (2) no AND (3) yes → Informational — reinitializer relies solely on the one-shot guard; a non-atomic upgrade execution opens a caller-race window between the implementation upgrade and the initialization call
- (1) yes AND (2) yes → Dismiss — access control is present
- (1) no → Dismiss — no state written; one-shot guard alone is sufficient
- (3) no → Dismiss — no established access control pattern in the contract to compare against

**Severity adjustment:** Upgrade to Low if the state written affects critical accounting variables (balances, prices, accumulated assets) AND the upgrade procedure is not documented to be atomic (e.g., via `upgradeToAndCall`).

**Benign cases:**
- All state-writing functions in the contract omit access control — consistent pattern, not a deviation
- The reinitializer takes no parameters and writes only a fully deterministic value with no user-supplied input
