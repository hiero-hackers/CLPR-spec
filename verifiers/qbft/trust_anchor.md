# QBFT Trust Anchor

## Overview

The QBFT trust anchor encodes the network's current validator set authority together with the `ClprService` contract identity. It enables the on-chain verifier to authenticate committed seals without reading any chain state — all required information is carried in the trust anchor itself. Rotation is driven by QBFT epoch boundaries: the validator set encoded in each epoch block header's `extraData` becomes the authority for the next epoch.

---

## Wire format

The trust anchor is ABI-encoded as a fixed 128-byte value:

```
abi.encode(
  bytes32 validatorSetHash,  // keccak-256 of the current validator address set
  bytes32 codeHash,          // keccak-256 of the ClprService contract bytecode
  uint64  epochLength,       // number of blocks per epoch (network parameter)
  uint64  epochNumber        // epoch index of the current validator set
)
```

The secondary `trustAnchorId` field stores the epoch number as a compact big-endian byte string (variable length, no leading zeros). The relay's `QbftBundleConstructor` reads `trustAnchorId` to determine whether the remote peer is behind the current epoch.

---

## Initial anchor derivation

At channel setup, `QBFTVerifier.verifyConfig` derives the initial trust anchor:

1. The relay's `EvmQbftLedgerConfigurationProvider` fetches the latest epoch boundary block header from the QBFT network.
2. The validator set is decoded from the epoch header's QBFT `extraData` (ECDSA addresses, RLP-encoded).
3. `verifyConfig` computes `validatorSetHash = keccak256(abi.encodePacked(sortedValidatorAddresses))`.
4. `codeHash` is read from the `ClprService` account state (keccak-256 of deployed contract bytecode).
5. `epochLength` is read from the `ClprLedgerConfiguration` stored in the contract.
6. `epochNumber` is derived from the epoch header's block number: `blockNumber / epochLength`.
7. The trust anchor `abi.encode(validatorSetHash, codeHash, epochLength, epochNumber)` is stored in `channel.trustAnchor`.
8. `trustAnchorId` is set to the big-endian encoding of `epochNumber`.

---

## Rotation lifecycle

### Detection

Rotation is epoch-driven. The relay detects that rotation is needed by comparing the remote peer's epoch (read from `channel.trustAnchorId` on the local chain) with the current epoch on the QBFT network:

```java
// QbftBundleConstructor
long remoteTrustAnchorEpoch = decodeRemoteTrustAnchorEpoch(remoteTrustAnchorId);
long currentEpoch = currentBlockNumber / epochLength;
boolean rotationNeeded = remoteTrustAnchorEpoch < currentEpoch;
```

If `remoteTrustAnchorId` is empty or unparseable (e.g. after a cold start), the relay defaults to the current epoch — no rotation headers are included.

### Bundle inclusion

When `rotationNeeded` is true, the relay fetches epoch boundary block headers from `remoteTrustAnchorEpoch + 1` to `currentEpoch` (inclusive), up to `maxEpochBlockHeadersPerBundle` headers:

```
epochBlockHeaders = [
  blockHeader(epochStart(remoteTrustAnchorEpoch + 1)),
  blockHeader(epochStart(remoteTrustAnchorEpoch + 2)),
  ...
  blockHeader(epochStart(currentEpoch))
]
```

These are included as the second element of the RLP 5-tuple in `proofBytes`. Each epoch header contains the QBFT `extraData` with committed seals from the previous validator set and the new validator set for the following epoch.

If more epochs have passed than `maxEpochBlockHeadersPerBundle` allows, the relay sends the oldest N epochs first and catches up over subsequent bundles.

### On-chain update

`QBFTVerifier.verifyBundle` processes epoch headers in sequence:

1. For each epoch header, verify the QBFT committed seals against the current `validatorSetHash`.
2. Decode the new validator set from the header's `extraData`.
3. Compute `newValidatorSetHash = keccak256(abi.encodePacked(sortedNewValidatorAddresses))`.
4. Advance `epochNumber += 1`.
5. After all epoch headers are processed, return:
   - `newTrustAnchor = abi.encode(newValidatorSetHash, codeHash, epochLength, newEpochNumber)`
   - `newTrustAnchorId = bigEndianBytes(newEpochNumber)`

`BundleProcessor` writes `newTrustAnchor` and `newTrustAnchorId` to `channel.trustAnchor` and `channel.trustAnchorId`. The relay will read the updated `trustAnchorId` on its next cycle.

If `epochBlockHeaders` is empty (no rotation), `verifyBundle` returns empty bytes for both fields and the stored trust anchor is unchanged.

---

## Security properties

- **Validator set binding**: the trust anchor commits to the keccak-256 hash of the validator set; an attacker cannot produce valid committed seals without controlling a threshold of the real validators.
- **Contract identity**: `codeHash` binds the trust anchor to the specific `ClprService` bytecode deployed at channel setup. A contract upgrade changes `codeHash` and would require establishing a new channel.
- **Epoch continuity**: each rotation step requires the previous validator set to sign the epoch boundary header. A single malicious epoch header cannot skip validator set history — the chain of committed seals from the initial epoch to the current one must be unbroken.
- **Rotation lag**: if the relay is offline across many epochs, it may need multiple bundles to catch up. The `maxEpochBlockHeadersPerBundle` parameter bounds the catch-up rate per bundle.
