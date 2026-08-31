# Hiero Trust Anchor

## Overview

The Hiero trust anchor is the network's `ledger_id`: a variable-length byte string that is the genesis-rooted identity of the Hiero consensus network's threshold signing key. It binds the CLPR channel to a specific Hiero network at the moment of channel setup and authorizes every subsequent bundle from that network. Any state proof submitted over the channel is accepted only if its TSS signature verifies against this `ledger_id`.

The `ledger_id` is stable across normal operations. Rotation (TSS key succession) is not yet implemented; the infrastructure for it (WRAPS recursive SNARK binding hinTS verification key to `ledger_id`) is in place but `verifyBundle` currently returns empty `newTrustAnchor`.

---

## Wire format

The trust anchor is the raw `ledger_id` bytes with no additional framing:

```
ledger_id   variable    Genesis-rooted TSS identity bytes (BLS12-381 hinTS VK root)
```

There is no secondary `trustAnchorId` field — `verifyConfig` returns empty bytes for `initialTrustAnchorId`.

The `ledger_id` is present in every `BlockStateProof` embedded in a Hiero `StateProof`. During initial channel setup it is read from the config proof; thereafter it is read from the stored channel trust anchor.

---

## Initial anchor derivation

At channel setup, `ClprService.completeChannel` calls `verifyConfig(configProofBytes, channelId)`. The native `VerifyConfigCall`:

1. Deserializes the `StateProof` from `configProofBytes`.
2. Reads `ledger_id` from `BlockStateProof.ledger_id`.
3. Verifies the TSS signature against the block root (self-certifying: the first Channel uses the `ledger_id` from the proof itself, which the caller implicitly trusts by choosing to connect to this network).
4. Returns `ledger_id` bytes as `initialTrustAnchor`.

`ClprService` stores `initialTrustAnchor` in `channel.trustAnchor`. All subsequent `verifyBundle` calls use the stored value — the proof's own `ledger_id` field is ignored after setup to prevent impersonation.

---

## Rotation lifecycle

### Detection

TSS succession on Hiero would be signaled by a network upgrade that replaces the hinTS verification key and, correspondingly, changes the `ledger_id`. The relay has no current mechanism to detect this; it is expected to be an out-of-band event coordinated with network participants.

There is no relay-side rotation trigger today. The relay submits proofs with the current TSS signatures; if the network has rotated and the stored `ledger_id` no longer matches, `verifyBundle` will reject the signature.

### Bundle inclusion

TSS succession is not yet carried in bundle payloads. The planned mechanism is:

1. The new hinTS verification key is committed on-chain (Hiero side) via a network upgrade transaction.
2. A WRAPS recursive SNARK proves that the new VK is the legitimate successor of the old one, rooted at the same `ledger_id`.
3. The WRAPS proof is included in the bundle alongside the normal `StateProof`.

Until this is implemented, `proofBytes` contains only the `StateProof` with no succession proof.

### On-chain update

`VerifyBundleCall` currently returns `newTrustAnchor = bytes(0)` (empty). `BundleProcessor` treats an empty return as "no change" and leaves `channel.trustAnchor` unchanged.

When WRAPS succession is implemented, `VerifyBundleCall` will:
1. Decode the succession proof from `proofBytes`.
2. Verify the WRAPS SNARK binding old VK → new VK → same `ledger_id`.
3. Return the new `ledger_id` as `newTrustAnchor`.
4. `BundleProcessor` writes `newTrustAnchor` to `channel.trustAnchor`, completing the rotation.

---

## Security properties

- **Network binding**: the `ledger_id` is the genesis-rooted TSS identity; a CLPR Channel is bound to exactly one Hiero network. A different network with a different genesis cannot forge a matching signature.
- **Threshold security**: the TSS signature requires a threshold of consensus nodes to cooperate. No single node can forge a bundle.
- **Immutability (current)**: because rotation is not implemented, the trust anchor is immutable for the lifetime of the Channel. A compromised or forked Hiero network would require a new Channel to establish a fresh trust anchor.
- **Self-certification risk**: the `ledger_id` in the initial config proof is trusted on first use. Operators must verify the `ledger_id` out-of-band against a known-good source before accepting the Channel.
