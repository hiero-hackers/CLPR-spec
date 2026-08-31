# Hiero Verifier Contract

## Overview

The Hiero verifier authenticates state proofs from a Hiero consensus network. It relies on the Hiero node's native TSS/hinTS threshold signature scheme, exposing verification through a precompile rather than pure Solidity cryptography. The EVM-facing `HieroVerifier.sol` delegates all work to the precompile at address `0x16e`, which is implemented natively in Java (`VerifyConfigCall`, `VerifyBundleCall`).

Implementation: `clpr-smart-contracts/src/verifiers/hiero/HieroVerifier.sol`
Native implementation: `clpr-hiero/hedera-node/hedera-clpr-service-impl/.../VerifyConfigCall.java`, `VerifyBundleCall.java`

---

## verifyConfig

### Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `configProofBytes` | `bytes` | Serialized `StateProof` protobuf proving a `ClprLedgerConfiguration` leaf |
| `channelId` | `bytes32` | The channel being established (used to scope the config proof) |

### Validation steps

1. Deserialize `StateProof` from `configProofBytes`.
2. Extract the `ClprLedgerConfiguration` leaf via `ClprProofExtraction`, reading leaf tag for configuration entities.
3. Recompute the SHA-384 Merkle path from the config leaf to the block root.
4. Verify the TSS signature in `BlockStateProof` against the computed block root using the `ledger_id` embedded in the proof.
5. Validate that `ClprLedgerConfiguration.chainId` is non-empty and `serviceAddress` is well-formed.

### Outputs

| Return value | Source |
|--------------|--------|
| `channelContext` | Empty bytes (Hiero uses no per-channel context) |
| `chainId` | `ClprLedgerConfiguration.chainId` |
| `serviceAddress` | `ClprLedgerConfiguration.serviceAddress` |
| `peerConfigNanos` | Timestamp from `BlockStateProof` block root |
| `throttles` | `ClprLedgerConfiguration.throttles` |
| `initialTrustAnchor` | `ledger_id` bytes from `BlockStateProof` |
| `initialTrustAnchorId` | Empty bytes (Hiero trust anchor has no secondary id) |
| `seedEndpoints` | `ClprLedgerConfiguration.seedEndpoints` |

---

## verifyBundle

### Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `proofBytes` | `bytes` | Serialized `StateProof` protobuf proving Channel and ClprMessageValue leaves |
| `trustAnchor` | `bytes` | Current `ledger_id` bytes stored in the Channel |
| `channelContext` | `bytes` | Empty for Hiero |

### Validation steps

1. Deserialize `StateProof` from `proofBytes`.
2. Verify the TSS signature using `ledger_id` from `trustAnchor` (not from the proof itself — the stored anchor is the authority).
3. Extract the `Channel` leaf (tag 482): read `nextMessageId`, `sentRunningHash`, `receivedMessageId`, `receivedRunningHash`, `status`.
4. Extract each `ClprMessageValue` leaf (tag 498): read payload bytes and `runningHashAfterProcessing`.
5. Validate running-hash chain: each message's `runningHashAfterProcessing` must equal `SHA-384(prevRunningHash || messagePayload)`.
6. Validate that the proven `Channel.nextMessageId` and running hashes match the `QueueMetadata` returned.

### Outputs

| Return value | Source |
|--------------|--------|
| `metadata` | Decoded from `Channel` leaf |
| `messagePayloads` | Decoded from `ClprMessageValue` leaves, in order |
| `newTrustAnchor` | Empty bytes — TSS succession not yet implemented |
| `newTrustAnchorId` | Empty bytes |

When `newTrustAnchor` is empty, `BundleProcessor` leaves the stored trust anchor unchanged.

---

## Implementation reference

| Component | Location |
|-----------|----------|
| Solidity entry point | `clpr-smart-contracts/src/verifiers/hiero/HieroVerifier.sol` |
| Precompile dispatch | `clpr-hiero/.../service/clpr/impl/evm/EvmClprVerifier.java` |
| verifyConfig native | `clpr-hiero/.../service/clpr/impl/evm/VerifyConfigCall.java` |
| verifyBundle native | `clpr-hiero/.../service/clpr/impl/evm/VerifyBundleCall.java` |
| Proof extraction | `clpr-hiero/.../service/clpr/impl/evm/ClprProofExtraction.java` |
| Merkle verification | `clpr-hiero/.../service/clpr/impl/evm/StateProofVerifier.java` |
| TSS verification | `clpr-hiero/.../service/clpr/impl/evm/NativeTssVerifier.java` |
| Proof construction | `clpr-hiero/.../service/clpr/impl/evm/ClprStateProofManager.java` |
