---
name: pausable accrual no sync on toggle
description: Detects pausable contracts with a time-based accrual mechanism where the pause-toggle function does not call the accrual sync before changing pause state.
languages: solidity
severity-default: low
confidence: medium
tools: [Grep, Read]
tags: [pause, accrual, state-sync, erc4626, defi, time-based]
related-checks: [erc4626-live-balance-accrual-base, stale-timestamp-on-mode-transition, pausable-unpause-accrues-paused-interval, oz-pausable-paused-override-expanded-semantics]
---

Look for contracts that combine pause control with time-based accrual:

- Pause toggle functions: `togglePause`, `pause()`, `_pause()`, or `setPaused`
- Time-based accrual indicators: `_accrue`, `accrueInterest`, `lastUpdate`, or `block.timestamp - lastUpdate`
- The `whenNotPaused` modifier applied to state-changing functions that invoke the accrual function

For each match, read the pause-toggle function body and assess:

1. Does the contract have both a pause mechanism and a time-based accrual?
2. Are state-changing functions gated by `whenNotPaused` (meaning accrual is blocked during pause)?
3. Does the pause-toggle function call `_accrue()` (or equivalent) before mutating the pause flag?

Decision:

- (1) yes AND (2) yes AND (3) no → Low — the stale accrual gap accumulates during the full pause duration; see related check for live-balance amplification
- (1) yes AND (2) yes AND (3) yes → Dismiss — accrual is synced before the state change; mitigated
- (1) yes AND (2) no → Informational — accrual callers are not blocked during pause; gap is bounded by normal usage

Note: `stale-timestamp-on-mode-transition` covers a related but distinct pattern — that check targets transitions that omit resetting a time variable, front-loading fees onto post-transition holders. This check targets the missing pre-toggle accrual call that creates a donation-amplifiable stale gap during pause.
