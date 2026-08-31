# Hiero State Proof

## Overview

Hiero state proofs authenticate a snapshot of the consensus node's virtual Merkle tree at a specific block boundary. For CLPR, two leaf types are proven per bundle: the `Channel` record (leaf tag 482) confirming the Channel's queue metadata, and one `ClprMessageValue` record (leaf tag 498) per message confirming its payload and running hash. For `verifyConfig`, a `ClprLedgerConfiguration` leaf is proven instead. In both cases the proof is bound to a block root that is in turn signed by the network's threshold signature scheme (TSS).

---

## Cryptographic primitives

| Primitive | Details |
|-----------|---------|
| Merkle hash | SHA-384 with domain separation bytes |
| Block root signature | BLS12-381 aggregate via hinTS threshold signature scheme |
| hinTS | Blind-rotation-based threshold signature; verification key anchored to `ledger_id` via WRAPS recursive SNARK |
| Curve | BLS12-381 (G1 for public keys, G2 for signatures) |

### SHA-384 Merkle domain prefixes

| Prefix byte | Node type |
|-------------|-----------|
| `0x00` | Leaf node |
| `0x01` | Unary internal node (single child) |
| `0x02` | Binary internal node (two children) |

Each node hash is `SHA-384(prefix_byte || left || right)` for binary nodes, `SHA-384(prefix_byte || child)` for unary nodes, and `SHA-384(0x00 || leaf_serialization)` for leaves.

---

## Wire format

The `proofBytes` argument (passed as `configProofBytes` in `verifyConfig` or `proofBytes` in `verifyBundle`) is a serialized `StateProof` protobuf:

```protobuf
message StateProof {
  BlockStateProof block_state_proof = 1;  // TSS signature over the block root
  repeated MerkleLeafProof leaf_proofs = 2;  // one entry per proven leaf
}

message BlockStateProof {
  bytes ledger_id         = 1;  // genesis-rooted TSS identity bytes
  bytes tss_signature     = 2;  // BLS12-381 aggregate signature over block root
  bytes block_root        = 3;  // 48-byte SHA-384 Merkle root of the full state tree
  MerklePath block_path   = 4;  // optional path extending state root to block root
}

message MerkleLeafProof {
  uint32 leaf_tag         = 1;  // entity type tag (482 = Channel, 498 = ClprMessageValue, etc.)
  bytes  leaf_value       = 2;  // serialized protobuf of the proven entity
  MerklePath path         = 3;  // sibling hashes from leaf to state root
}

message MerklePath {
  repeated bytes siblings = 1;  // 48-byte SHA-384 hashes, ordered leaf-to-root
}
```

The `StateProof` is the single opaque `bytes` field the relay submits and the verifier decodes.

---

## Storage layout

Hiero state is a virtual Merkle tree over typed entity maps. Each entity type has a fixed leaf tag. CLPR reads the following leaves:

| Leaf tag | Entity type | Content proven |
|----------|-------------|----------------|
| 482 | `Channel` | `nextMessageId`, `sentRunningHash`, `receivedMessageId`, `receivedRunningHash`, channel `status` |
| 498 | `ClprMessageValue` | Message payload bytes and `runningHashAfterProcessing` |
| (config) | `ClprLedgerConfiguration` | `protocolVersion`, `chainId`, `serviceAddress`, `throttles`, endpoint roster |

Leaf addresses within the tree are deterministic: `(leaf_tag, entity_id)` tuples map to fixed positions in the entity map for that leaf type. The path from any leaf to the state root is given by the siblings in `MerkleLeafProof.path`.

The block root extends the state root by including a timestamp node: the path is `state_root_siblings || empty_node || SHA-384(timestamp_bytes)`.

---

## Proof construction

The relay does not construct Hiero proofs — the Hiero consensus node builds and signs them natively. The relay retrieves the completed `StateProof` via the Hiero mirror node or state proof API for the target block. The relay's `HieroProofCodec` parses and validates the structure before forwarding but does not reassemble it.

On the Hiero node side, `ClprStateProofManager` drives proof construction:

1. Calls `MerklePathBuilder.buildPath(snapshot, leafTag, entityId)` for each required leaf, obtaining sibling hashes from the live Merkle state.
2. Extends the state root path to the block root by appending the empty-node sibling and the hashed timestamp from `snapshot.path().siblings()`.
3. Packages the leaf proofs and `BlockStateProof` (with TSS signature over block root) into a `StateProof` protobuf.
4. Serializes to bytes for inclusion in the `ClprBundleContent`.

---

## Proof verification

`VerifyBundleCall` (precompile at `0x16e`) and `VerifyConfigCall` perform verification in the following order:

1. **Deserialize** `StateProof` from `proofBytes`.
2. **Recompute the Merkle root** for each `MerkleLeafProof`: walk siblings bottom-up, applying `SHA-384(domain_prefix || left || right)` at each level, using the `leaf_tag` as the leaf domain separator.
3. **Extend to block root**: apply the block path siblings from `BlockStateProof.block_path` to reach the block root.
4. **Verify TSS signature**: call `NativeTssVerifier.verifyTSS(blockRoot, tssSignature, ledgerIdFromTrustAnchor)` — this checks the BLS12-381 hinTS aggregate signature, binding the signature to the `ledger_id` via the WRAPS-anchored verification key.
5. **Extract leaf values**: for each proven leaf tag, decode the `leaf_value` bytes as the expected protobuf type.
6. **Validate semantics**: check that `Channel.nextMessageId` and running hashes match the values in `QueueMetadata`; check each `ClprMessageValue.runningHashAfterProcessing` chains correctly.

Verification is a `view` function — no state is written. All return values are derived solely from the verified proof contents.
