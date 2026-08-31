# QBFT State Proof

## Overview

QBFT state proofs authenticate the storage state of the `ClprService` smart contract on a Besu-based QBFT network. Each bundle proof combines an EIP-1186 Merkle Patricia Trie (MPT) account and storage proof with the canonical RLP-encoded block header (and, during trust anchor rotation, one or more epoch boundary headers). The block header's `extraData` field carries the QBFT committed seals that bind the header to the validator set.

---

## Cryptographic primitives

| Primitive | Details |
|-----------|---------|
| State trie | Merkle Patricia Trie (MPT), keccak-256 node hashes |
| Block hash | keccak-256 over RLP-encoded block header |
| Storage slot | keccak-256 mapping layout (EIP-1967 / Solidity standard) |
| QBFT consensus | ECDSA SECP256K1 committed seals in block `extraData` |
| EIP-1186 | `eth_getProof` RPC returns account proof + per-slot storage proofs |

---

## Wire format

`proofBytes` is an RLP-encoded list of five top-level items:

```
RLP [
  currentBlockHeader,       // RLP-encoded EVM block header (canonical field order)
  epochBlockHeaders,        // RLP list of epoch-boundary block headers (may be empty)
  clprServiceAccountProof,  // RLP list of MPT account proof nodes (from eth_getProof)
  clprServiceStorageProofs, // RLP list of [key, [proofNodes...]] entries
  serializedBundleContent   // RLP byte string wrapping protobuf(ClprBundleContent)
]
```

`currentBlockHeader` follows Besu's canonical RLP field order including optional fork fields (EIP-1559, EIP-4895, EIP-4844, EIP-4788, EIP-7685). Any field present in a deployed Besu build must appear in the correct position; omitting or reordering a field changes the keccak and breaks QBFT seal recovery.

`ClprBundleContent` is a protobuf containing `ClprQueueMetadata` (nextMessageId, sentRunningHash, receivedMessageId, receivedRunningHash, status, trustAnchorId) and the ordered list of `ClprMessagePayload` entries.

---

## Storage layout

Five storage slots in `ClprService` are proven per bundle. Slot addresses follow Solidity mapping layout (`keccak256(abi.encode(key, baseSlot))`):

| Slot expression | Content | Base slot |
|----------------|---------|-----------|
| `keccak256(chanId, 17) + 1` | `status` (packed) + `nextMessageId` | CHANNELS_BASE_SLOT = 17 |
| `keccak256(chanId, 17) + 2` | `ackedMessageId` + `receivedMessageId` (packed) | 17 |
| `keccak256(chanId, 17) + 4` | `sentRunningHash` | 17 |
| `keccak256(chanId, 17) + 5` | `receivedRunningHash` | 17 |
| `keccak256(keccak256(chanId, 1), messageId) + 1` | last message `runningHashAfterProcessing` | MESSAGE_QUEUES_BASE_SLOT = 1 |

For `verifyConfig`, the slot proven is:

| Slot expression | Content | Base slot |
|----------------|---------|-----------|
| `keccak256(0, 23)` | `_config.serviceAddress` | CONFIG_SERVICE_ADDRESS_SLOT = 23 |

---

## Proof construction

The relay's `QbftBundleConstructor` assembles the proof when triggered by new block events:

1. **Determine epoch headers**: read `remoteTrustAnchorId` from the on-chain `Channel` and interpret it as a big-endian epoch number. If `remoteTrustAnchorEpoch < currentEpoch`, fetch up to `maxEpochBlockHeadersPerBundle` epoch-boundary headers via `eth_getBlockByNumber`.

2. **Compute storage slots**: call `ClprServiceStorageLayout.calculateChannelFieldStorageSlot(channelId, fieldOffset)` for the five channel slots and `calculateMsgRunningHashStorageSlot(channelId, messageId)` for the last message's running hash.

3. **Fetch EIP-1186 proofs**: call `eth_getProof(clprServiceAddress, slotsToProve, blockTag)` via `EvmJsonRpcClient`. The response includes the MPT account proof and one storage proof per slot.

4. **Sanity check**: verify `storageProof.size() == slotsToProve.size()` and that the last message's proven running hash matches the in-memory `ClprMessageValue.runningHashAfterProcessing`.

5. **Assemble RLP**: encode the five-element RLP tuple using `AbiCodec.rlpList`. The `serializedBundleContent` item wraps `ClprBundleContent.PROTOBUF.toBytes(content)` as an RLP byte string.

6. **Pre-verify**: `EvmBundlePreVerifier` simulates `ClprService.submitBundle(channelId, proofBytes)` via `eth_call` before the paid transaction. If the simulation reverts, the submission is skipped.

---

## Proof verification

`QBFTVerifier.sol` verifies the proof in the following order:

1. **Decode RLP**: split the five-element outer list; decode `currentBlockHeader` fields including `extraData`.
2. **Verify QBFT committed seals**: recover signer addresses from the QBFT `extraData` committed seals using ECDSA. Confirm that a threshold of recovered addresses matches the current validator set in the trust anchor.
3. **Verify block hash**: keccak-256 the RLP-encoded header and confirm the MPT account proof is rooted at `currentBlockHeader.stateRoot`.
4. **Verify account proof**: walk the MPT account proof to confirm `ClprService` account inclusion at `stateRoot`.
5. **Verify storage proofs**: for each slot, walk the storage MPT proof rooted at the account's `storageRoot` to confirm each slot value.
6. **Process epoch headers** (if present): verify committed seals on each epoch header against the validator set encoded in its `extraData`; advance the validator set to produce `newTrustAnchor`.
7. **Decode bundle content**: deserialize `ClprBundleContent` from the fifth RLP item; return `QueueMetadata` and message payloads.
