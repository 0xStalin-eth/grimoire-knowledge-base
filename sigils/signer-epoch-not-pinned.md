---
name: signer epoch not pinned
description: Flags signed-claim consumers that let the claim pick its signer set (epoch, key-set id) without requiring the current one, so retired or leaked keys stay valid.
languages: solidity
severity-default: medium
confidence: high
tools: [Grep, Read]
tags: [oracle, reclaim, zk-proof, key-rotation, signature, attestation]
related-checks: [push-oracle-observation-ordering]
attribution-name: Cyfrin FX-stablecoin AMM audit 2026 (M-08 epoch binding)
attribution-url: https://docs.reclaimprotocol.org/
---

Grep for consumers of signed attestations or proofs: `verifyProof`, `signedClaim`, `claim.epoch`, `epoch`,
`currentEpoch`, `fetchEpoch`, `witnesses`, `keySetId`, `signerSetId`, `guardianSetIndex`.

For each consumer:
1. Does the signed payload carry the id of the signer set (epoch, guardian set index, key version)?
2. Does the verifier choose its witnesses or keys from the set the payload names, including old sets?
   (Reclaim `verifyProof` / `fetchEpoch` does; Wormhole is similar.) Check the verifier actually
   deployed (for example on a fork), not only a vendored copy.
3. Does the consumer require that id to equal the verifier's current set (`currentEpoch()`), or does it
   at least check that an old set's expiry has not passed?
4. Is the claim's timestamp checked against the window of the named set?

If neither the verifier nor the consumer pins the set, rotating a leaked key does not help: the key
holder signs fresh claims that name the old epoch. With one witness and a minimum of one witness per
claim, a single leaked key is enough.
Severity: medium if forged claims stay inside downstream guards (band, budget). The exposure is that
rotating the key out no longer ends it. High if the claim directly authorizes transfers or prices with
no bound.
Benign: the consumer checks `claim.epoch == verifier.currentEpoch()`, or the verifier rejects old sets.
