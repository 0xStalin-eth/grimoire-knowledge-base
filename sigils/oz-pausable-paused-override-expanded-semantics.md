---
name: oz pausable paused override expanded semantics
description: Detects contracts that override OZ Pausable paused() to include external state without also overriding the inherited guard mechanics, breaking pause and unpause under the expanded condition.
languages: solidity
severity-default: informational
confidence: medium
tools: [Grep, Read]
tags: [pausable, oz-deviation, standards-violation, access-control]
related-checks: [pausable-accrual-no-sync-on-toggle, pausable-unpause-accrues-paused-interval]
attribution-name: OpenZeppelin PausableUpgradeable
attribution-url: https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/v5.3.0/contracts/utils/PausableUpgradeable.sol
---

## Patterns

Grep for contracts overriding `paused()`:
- `function paused()` + `override`
- `PausableUpgradeable` or `import.*Pausable` in the same file

## Assessment

Read the override body.

1. Does `paused()` return anything beyond the local `_paused` storage slot (e.g., an external call, OR-condition with a second flag, protocol-level check)? If not — dismiss.

2. If yes, check whether `_pause()`, `_unpause()`, `_requireNotPaused()`, and `_requirePaused()` are also overridden to decouple their guards from the expanded result.

3. If any of those four are inherited without override, flag as finding:
   - `_pause()` carries `whenNotPaused` → calls `paused()` (override) → reverts with `EnforcedPause()` if external state is true, blocking local pause activation.
   - `_unpause()` carries `whenPaused` → calls `paused()` (override) → proceeds on external-only pause, writes `_paused = false` (no-op) and emits `Unpaused` spuriously.

4. Severity adjustment:
   - Stay `informational` if the gap is purely observational (spurious event, minor UX friction).
   - Raise to `low` if local pause cannot be set while external pause is active (operational gap for incident response).
   - Raise to `medium` if the spurious `Unpaused` event or broken `_pause()` revert can be triggered by an unprivileged actor or causes downstream access-control bypass.
