# QBFT Verifier Contract

## Overview

The QBFT verifier authenticates state proofs from a Besu-based QBFT network. Trust is anchored in the QBFT validator set, which is encoded in block `extraData` (committed seals). The verifier reads the genesis block `extraData` to seed the initial validator set and advances it through epoch boundary headers carried in bundle proofs.

Implementation: `clpr-smart-contracts/src/verifiers/qbft/QBFTVerifier.sol`

---

## verifyConfig

### Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `configProofBytes` | `bytes` | Proof bytes from `EvmQbftLedgerConfigurationProvider`; contains epoch block header, `ClprLedgerConfiguration` storage proof, current block header, and account proof |
| `channelId` | `bytes32` | The channel being established |

### Validation steps

1. Decode the config proof structure (epoch block header, current block header, account proof, storage proof for `_config.serviceAddress` at slot 23).
2. Verify QBFT committed seals on the epoch block header using the initial validator set decoded from the genesis block's `extraData`.
3. Verify the MPT account and storage proofs rooted at `currentBlockHeader.stateRoot` to confirm the `ClprService` address and `serviceAddress` slot value.
4. Compute the initial trust anchor from the epoch validator set, `ClprService` code hash, `epochLength`, and `epochNumber`.

### Outputs

| Return value | Source |
|--------------|--------|
| `channelContext` | ABI-encoded `epochLength` and initial epoch number (used by `verifyBundle`) |
| `chainId` | From proven `ClprLedgerConfiguration` |
| `serviceAddress` | From proven storage slot 23 |
| `peerConfigNanos` | Block timestamp from epoch header |
| `throttles` | From `ClprLedgerConfiguration` |
| `initialTrustAnchor` | `abi.encode(validatorSetHash, codeHash, epochLength, epochNumber)` — 128 bytes |
| `initialTrustAnchorId` | Big-endian epoch number bytes |
| `seedEndpoints` | From `ClprLedgerConfiguration` |

---

## verifyBundle

### Inputs

| Parameter | Type | Description |
|-----------|------|-------------|
| `proofBytes` | `bytes` | RLP 5-tuple (see `state_proof.md`) |
| `trustAnchor` | `bytes` | Current ABI-encoded `(validatorSetHash, codeHash, epochLength, epochNumber)` |
| `channelContext` | `bytes` | ABI-encoded epoch metadata from `verifyConfig` |

### Validation steps

1. RLP-decode the five-element proof structure.
2. Decode the trust anchor to obtain current `validatorSetHash`, `codeHash`, `epochLength`, and `epochNumber`.
3. Verify QBFT committed seals on `currentBlockHeader.extraData` using the validator set identified by `validatorSetHash`.
4. Verify the MPT account proof rooted at `currentBlockHeader.stateRoot`.
5. Verify the five storage proofs against the account's `storageRoot`.
6. **Epoch rotation** (if `epochBlockHeaders` is non-empty): for each epoch header in sequence, verify its QBFT seals against the current validator set; decode the new validator set from that header's `extraData`; advance `validatorSetHash` and `epochNumber`.
7. Decode `ClprBundleContent` from the fifth RLP item; validate `runningHashAfterProcessing` chains.

### Outputs

| Return value | Source |
|--------------|--------|
| `metadata` | From proven Channel storage slots |
| `messagePayloads` | From `ClprBundleContent.messages` |
| `newTrustAnchor` | Updated `abi.encode(newValidatorSetHash, codeHash, epochLength, newEpochNumber)` if rotation occurred; empty bytes otherwise |
| `newTrustAnchorId` | Big-endian new epoch number bytes if rotation occurred; empty bytes otherwise |

---

## Implementation reference

| Component | Location |
|-----------|----------|
| Solidity verifier | `clpr-smart-contracts/src/verifiers/qbft/QBFTVerifier.sol` |
| Relay bundle constructor | `clpr-relay/clpr-relay-evm/src/main/java/org/hiero/clpr/relay/evm/QbftBundleConstructor.java` |
| Relay proof codec | `clpr-relay/clpr-relay-evm/src/main/java/org/hiero/clpr/relay/evm/QbftProofCodec.java` |
| Config provider | `clpr-relay/clpr-relay-evm/src/main/java/org/hiero/clpr/relay/evm/EvmQbftLedgerConfigurationProvider.java` |
| Storage layout | `clpr-relay/clpr-relay-evm/src/main/java/org/hiero/clpr/relay/evm/storage/ClprServiceStorageLayout.java` |
| ABI / RLP codec | `clpr-relay/clpr-relay-evm/src/main/java/org/hiero/clpr/relay/evm/AbiCodec.java` |
