---
name: bridge evm address encoding without chain guard
description: Flags cross-chain bridge functions that EVM-pad an address to bytes32 without validating the target chain is EVM-compatible.
languages: solidity
severity-default: medium
confidence: medium
tools: [Grep, Read]
tags: [bridge, cross-chain, wormhole, address-encoding, input-validation]
---

## Patterns

Primary — grep for the EVM address-to-bytes32 encoding pattern:

`bytes32(uint256(uint160(`

Secondary — narrow matches to functions that also accept a chain ID parameter. Look for co-occurrence of the above pattern with parameter names such as `uint16 _targetChain`, `uint16 targetChain`, or `uint16 chainId` within the same function body.

## Assessment

For each match, read the enclosing function and assess:

1. Check whether the function accepts a chain ID or target chain parameter (`uint16` or similar). If not, deprioritize — the encoding may be intra-contract only.
2. Check whether the encoded value is forwarded as a destination address in a cross-chain call (e.g., passed to a bridge send, transferTokens, or publishMessage call). If it stays on-chain, dismiss.
3. Check whether a guard appears BEFORE the encoding that validates the target chain is EVM-compatible — a `require`/`revert` against an allowlist, a chain-type mapping lookup, or an enum check. If a guard exists, dismiss.
4. If no such guard exists and the result is sent cross-chain, flag as a finding.

**Severity adjustment:**
- Upgrade to High if the bridge flow is burn-and-mint or lock-and-mint (permanent fund loss on mismatch).
- Keep Medium if the flow supports retry or refund.

**Benign cases to dismiss:**
- Encoding used purely for local storage or intra-contract hashing, not forwarded cross-chain.
- A chain-type guard or EVM-chain allowlist is enforced before or at the encoding site.
- The function is demonstrably restricted to EVM-only callers by the broader protocol (e.g., no non-EVM chain IDs are registered in the bridge configuration).
