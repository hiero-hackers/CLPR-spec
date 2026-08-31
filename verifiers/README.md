# CLPR Verifier Specifications

Each subdirectory here documents the verifier specification for one chain. A verifier is the on-chain component that authenticates state proofs submitted by a remote relay and advances the trust anchor for a Channel.

## Interface

All verifiers implement `IClprVerifier`:

```solidity
interface IClprVerifier {
    function verifyConfig(bytes calldata configProofBytes, bytes32 channelId)
        external view returns (
            bytes memory channelContext,
            string memory chainId,
            bytes memory serviceAddress,
            uint96 peerConfigNanos,
            Throttles memory throttles,
            bytes memory initialTrustAnchor,
            bytes memory initialTrustAnchorId,
            Endpoint[] memory seedEndpoints
        );

    function verifyBundle(
        bytes calldata proofBytes,
        bytes calldata trustAnchor,
        bytes calldata channelContext
    ) external view returns (
        QueueMetadata memory metadata,
        bytes[] memory messagePayloads,
        bytes memory newTrustAnchor,
        bytes memory newTrustAnchorId
    );
}
```

`verifyConfig` is called once at channel setup to authenticate the remote chain's configuration and derive the initial trust anchor. `verifyBundle` is called on every bundle submission to authenticate state and advance the trust anchor when the remote chain's signing authority rotates.

Both functions are `view` — all verification is stateless. State transitions are applied by `ClprService` using the return values.

## Directory layout

Each chain directory contains three files:

| File | Contents |
|------|----------|
| `state_proof.md` | Wire format of the proof bytes passed to `verifyBundle` (and `verifyConfig`). Covers cryptographic primitives, proof structure, construction on the relay side, and verification steps on-chain. |
| `verifier_contract.md` | The `IClprVerifier` implementation: what `verifyConfig` and `verifyBundle` validate, inputs and outputs, and the implementation reference. |
| `trust_anchor.md` | Binary layout of the trust anchor, how it is seeded at channel setup, and the full rotation lifecycle — detection, bundle inclusion, and on-chain update. |

## Chains

| Chain | Directory | Trust anchor type | Status |
|-------|-----------|-------------------|--------|
| Hiero | `hiero/` | `ledger_id` bytes (TSS/hinTS) | Production |
| QBFT (Besu) | `qbft/` | ABI-encoded validator + epoch | Production |

## Standard file outlines

### `state_proof.md`
1. **Overview** — what this chain's state proof scheme proves and its role in bundle verification
2. **Cryptographic primitives** — hash functions, signature schemes, curve parameters
3. **Wire format** — exact binary/protobuf/RLP structure of the `proofBytes` argument passed to `verifyBundle`
4. **Storage layout** — which slots or state leaves are proven and how their addresses are derived
5. **Proof construction** — relay-side steps: what RPC calls are made, how the proof is assembled
6. **Proof verification** — on-chain verifier steps: what the contract checks and in what order

### `verifier_contract.md`
1. **Overview** — what chain authority this verifier authenticates
2. **verifyConfig** — inputs, validation steps, outputs (including `initialTrustAnchor`)
3. **verifyBundle** — inputs, validation steps, outputs (including `newTrustAnchor`)
4. **Implementation reference** — Solidity source path and, where applicable, native precompile address

### `trust_anchor.md`
1. **Overview** — what signing authority the trust anchor represents for this chain
2. **Wire format** — byte-level layout with field names, types, and sizes
3. **Initial anchor derivation** — how `verifyConfig` seeds the trust anchor at channel setup from the config proof
4. **Rotation lifecycle**
   - *Detection* — what on-chain event signals rotation; how the relay observes it
   - *Bundle inclusion* — what rotation data is carried in the bundle payload
   - *On-chain update* — how `verifyBundle` validates and returns `newTrustAnchor`
5. **Security properties** — what guarantees the trust anchor provides and its failure modes

## Adding a new chain

1. Create a subdirectory named after the chain (lowercase, no spaces).
2. Add `state_proof.md`, `verifier_contract.md`, and `trust_anchor.md` using the structure described above.
3. Implement `IClprVerifier` in `clpr-smart-contracts/src/verifiers/<chain>/`.
4. Register the verifier address in the `ClprService` deployment for the target network.
5. Add an inbound `BundlePayloadCodec` in `clpr-relay` if the chain can act as a remote peer.
6. Add the chain to the table above with its trust anchor type and status.

Key questions to answer in each file:
- **state_proof.md**: What cryptographic commitment scheme does the chain use? What storage slots or state leaves are proven? How does the relay construct the proof, and how does the verifier check it?
- **verifier_contract.md**: What does `verifyConfig` authenticate? What does `verifyBundle` authenticate? What invariants must hold for a proof to be accepted?
- **trust_anchor.md**: What authority does the trust anchor represent? How is it seeded? What event on the remote chain triggers rotation, and how does the relay detect and carry the rotation through?
