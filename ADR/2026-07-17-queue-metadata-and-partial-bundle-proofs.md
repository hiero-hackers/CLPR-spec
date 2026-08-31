# Restructuring and Renaming ClprQueueMetadata to a State Proven ChannelSyncData

This ADR renames `ClprQueueMetadata` to `ChannelSyncData` and reduces its content to the minimal necessary state 
proven fields, decoupled from the full `Channel` record. This ADR further ensures that bundles may contain a proper 
subsequence of the available messages to send.  Some implementations are coded to send every pending message, 
which will freeze the connection if the size of the pending messages is larger than bundle throttles allow. 

---

## 1. Problem

### 1.1 A bundle cannot express a subsequence of a large backlog

Two constraints currently force a bundle to represent "everything queued," never a bounded slice of it:

- `clpr-service-spec.md` §4.2 Step 3 requires "the last message's ID MUST equal `ClprQueueMetadata.next_message_id - 1`" 
  — the metadata is *defined in terms of* whatever the bundle happens to contain, which reads as general but
  hard-pins the two together.
- On EVM chains (the shared verifier base behind QBFT, Sei, and EthMainnet), the anchor message's running hash is
  proven at a storage slot computed from the Channel's own `nextMessageId` — the channel's true latest
  message, not an arbitrary bundle-scoped one. A bundle can only ever be "everything up to my true latest," never a
  bounded prefix of it. A channel whose backlog exceeds `max_messages_per_bundle` or `max_sync_bytes` cannot be
  synced at all on these chains today — the anchor-proof slot and the true backlog boundary are the same slot by
  construction.

Hiero doesn't have this specific failure mode — each message is its own Merkle leaf, so any prefix is already
selectable — but it has the inverse problem, covered next.

### 1.2 ChannelSyncData should be a first class data structure. 

**Hiero** proves the entire `ClprChannel` leaf to obtain the data that was in `ClprQueueMetadata`, sending 
unnecessary fields that consume call data gas on EVM based chains.  The renamed `ChannelSyncData` should be a
separate state proven data structure in the source chain so it can be sent separately.

---

## 2. Options Considered

### 2.1 Continue proving every message individually — rejected

The running hash chain (§4.1) already gives every message a verifiable link to the one before it:
`running_hash = SHA-256(previous_running_hash || SHA-256(serialized_payload))`. Once the chain's starting hash is
trustworthy and its ending hash is state-proven, recomputing it over the intervening messages — supplied as
ordinary bytes, not individually proven — either reproduces the proven ending hash (contents are exactly as
claimed) or it doesn't (bundle rejected, per existing §4.2 Step 4). Individually proving every intermediate message
leaf, as Hiero does today, adds proof cost without adding assurance the chain doesn't already provide.

**Fix:** only the Anchor Message — the last message in a bundle's sequence — is individually state-proven.
The running hash computation extends the anchor message's trust to the preceding messages in the bundle.

### 2.2 Keep `ChannelSyncData` folded into `Channel` — rejected

Keeping `ChannelSyncData`'s fields as part of the same leaf/slots as the rest of `Channel` makes it impossible to 
send just the needed data.  The whole `Channel` is sent in the state proof, which increases the call data gas on EVM 
chains. 

**Fix:** give `ChannelSyncData` its own hashed, independently provable representation on every chain, 
sized to exactly what a peer's CLPR Service needs.

---

## 3. Proposed Solution

### 3.1 new `ChannelSyncData` definition

Each side of a Channel maintains exactly one `ChannelSyncData` instance, describing its own queue state, and it is
proven and exchanged on every bundle. Its fields are limited to the minimum necessary of what the *receiving* CLPR 
Service needs to see as state proven values for the channel.

| Field | Meaning                                                           |
|---|-------------------------------------------------------------------|
| `received_message_id` | Highest ID of the *peer's* messages received and processed        |
| `status` | this ledger's own current channel status                          |
| `next_message_id` | The ID to be assigned to the next outgoing message on this ledger |

The receiving CLPR Service acts on `received_message_id`, advancing its own `acked_message_id`, and `status`, which may
cause its own channel state transitions. The state-proven `next_message_id` lets the peer determine whether the
outbound queue has any messages to deliver, enabling an empty-bundle short-circuit without ambiguity.

The other fields for tracking message queue progress are only needed for the local channel's bookkeeping, held on
`Channel` and never state-proven.

| Field               | Meaning                                                                            |
|---------------------|------------------------------------------------------------------------------------|
| `received_running_hash` | Indicates where to start the running hash computation for the next incoming message |
| `sent_running_hash` | Indicates the resultant running hash after the last outgoing message in the queue  |
| `acked_message_id`  | Highest ID of this ledger's own outgoing messages confirmed received by the peer   |

### 3.2  Bundle Format 

The following data has their hashes state proven in the sync bundle: 
1. The ChannelSyncData
2. Any Trust Anchor Updates
3. Any Endpoint Manifest Updates
4. The Anchor Message, including its message id and resulting running hash.  

Messages included in the bundle prior to the last message are secured through the state proven running hash 
on the last message and SHOULD NOT be state proven in the bundle to save on verification computation costs.  

When constructing a bundle in response to a bundle request, the message sequence
- **MUST** contain no messages when `BundleRequest.current_received_message_id + 1 ≥` this ledger's
  `next_message_id`. 
- **SHOULD** start at `(peer's BundleRequest.current_received_message_id) + 1` — the freshest information
  available, avoiding replay of messages the peer has already told this endpoint it holds.
- **MAY** start at `ClprChannel.acked_message_id + 1` — this ledger's own last confirmed ack point, safe by
  construction but potentially stale, since a chain's own committed `acked_message_id` only advances when *it* processes
  a proven bundle carrying the peer's `received_message_id`.
- **MUST NOT** start at or before `ClprChannel.acked_message_id` — resending anything already reflected in this
  ledger's own confirmed ack point is pure waste.

#### 3.2.1 Bundle Progress and Handling

One or both of the following two progress criteria must be met for accepting the `ChannelSyncData` in a bundle:
1. **Acking Messages**: The bundle's `ChannelSyncData.received_message_id` MUST be `>` than the local channel's
   stored `ClprChannel.acked_message_id`.
2. **Channel Status Update**: The bundle's `ChannelSyncData.status` MUST cause a state change to the local
   channel's `ChannelSyncData.status`.

If acking messages made progress:
- The local channel's `ClprChannel.acked_message_id :=` the bundle's `ChannelSyncData.received_message_id`.

If a channel status update results, the existing handling procedures are followed.

### 3.3 Managing this on Hiero
The `ChannelSyncData` is a distinct data structure stored as a separate leaf in the state from the `ClprChannel` in 
order to receive its own state proof.  The data should not be replicated in the ClprChannel to ensure it is
represented and updated in a single location.  

### 3.4 Managing this on EVM chains

On EVM chains, the ChannelSyncData can be packed into storage slots in any way desired.  The contents are hashed, 
and the resulting hash is stored in its own slot.  State proofs are to the stored hash value and verified by hashing 
the data on the wire in the same way to confirm the content matches the hash.  

### 3.5 Verifier Interface Update

The `verifyBundle` method gains two return values, `anchor_message_id` and `anchor_running_hash`.

If a bundle does not contain any messages, the return value for `anchor_message_id` and `anchor_running_hash` are the 
smart contract's standard values for representing absence or null.  On Hiero it is Empty Bytes in protobuf.  On EVM 
chains it may be sentinel values.  

---

## 4. Code Artifacts

### 4.1 `ChannelSyncData`

```protobuf
// State-proven sync data for one side of a Channel, exchanged on every bundle.
message ChannelSyncData {
  // Highest ID of the peer's messages the local end of the channel
  // has received and processed.
  uint64 received_message_id = 1;

  // The status of the local end of the channel.
  ClprChannelStatus status = 2;

  // The next message ID to be assigned on this ledger's outbound queue.
  uint64 next_message_id = 3;
}
```

### 4.2 `Channel` — updated (`clpr-service-spec.md` §2.1)

```
Channel {
  // --- Identity ---
  channel_id              : bytes(32)   // primary key; arbitrary 32-byte value chosen by registrant
  chain_id                : string      // CAIP-2 identifier of the peer chain

  // --- Peer Configuration ---
  peer_config_timestamp   : Timestamp   // timestamp of the last known peer configuration

  // --- Verifier ---
  verifier_contract       : bytes       // address of the locally deployed verifier (immutable after registration)
  verifier_fingerprint    : bytes       // code hash of verifier_contract at registration time (informational)
  trust_anchor            : bytes       // opaque signing-authority material for this Channel; initialized from
                                        // verifyConfig's initial_trust_anchor return; updated in-band when
                                        // verifyBundle returns a non-empty new_trust_anchor. For Hiero TSS this
                                        // is the peer ledger_id, allowing a single generic verifier contract to
                                        // serve all Hiero networks. Only verifiers with no rotating-authority
                                        // concept store empty bytes. Only verifyBundle may
                                        // cause an update; no admin or governance path modifies this field.
  trust_anchor_id         : bytes       // opaque identifier for trust_anchor; initialized from verifyConfig's
                                        // initial_trust_anchor_id return; updated in-band alongside trust_anchor.
  channel_context      : ChannelContext // constant data returned by verifyConfig and passed to every verifyBundle
                                        // call: channel_id (echoed) and service_address (peer CLPR Service).
                                        // Immutable after completeChannel. See §3.1.

  // --- Peer Endpoint Manifest ---
  endpoint_manifest         : ClprEndpointManifest  // Cached peer endpoint manifest.

  // --- Config Propagation ---
  last_config_timestamp  : timestamp   // consensus_timestamp of the last config propagated on this Channel
  // When last_config_timestamp < current_config.consensus_timestamp, a
  // ConfigUpdate is lazily enqueued on the next interaction (see §1.3).

  // --- Outbound Queue Data ---
  acked_message_id       : uint64      // highest ID confirmed received by peer
  sent_running_hash      : bytes(32)   // cumulative SHA-256 of all enqueued outgoing messages

  // --- Inbound Queue Data ---
  received_running_hash  : bytes(32)   // cumulative SHA-256 of all received messages
}
```

### 4.3 `verifyBundle` — updated (`clpr-service-spec.md` §3.1)

```
  // Verify a bundle payload against the Channel's current trust anchor.
  // Returns verified sync data, an ordered array of message payloads,
  // a state-proven identity for the last message in that array, a
  // successor trust anchor, and an updated endpoint manifest (empty if none).
  //
  // The verifier is responsible for validating that the bundle data belongs to the
  // (trust_anchor, ctx.service_address, ctx.channel_id) triple. Other data-integrity
  // checking (running-hash chain, message ordering, replay defense) is performed by the
  // CLPR Service after this call returns and is out of scope for the verifier.
  //
  //   bundle_payload   : bytes             — state proof and message data (format is verifier-specific).
  //   trust_anchor     : bytes             — the Channel's current signing-authority material,
  //                                         as stored by the CLPR Service. Mutable: updated in-band
  //                                         whenever a prior verifyBundle call returned a non-empty
  //                                         new_trust_anchor. Empty for verifiers with no
  //                                         rotating-authority concept.
  //   ctx              : ChannelContext — constant channel-scoped data (channel_id,
  //                                         service_address). Passed to all verifiers uniformly;
  //                                         each verifier consumes only the fields its proof
  //                                         system requires. EVM/QBFT verifiers use both fields
  //                                         to derive storage slot keys and account proof paths.
  //                                         Hiero verifiers authenticate via state proof: the
  //                                         proof directly authenticates the ClprChannel
  //                                         leaf's service_address field rather than deriving
  //                                         it from ctx. Never mutated; the same value is
  //                                         passed on every call for a given Channel.
  //
  //   ChannelSyncData     — verified queue state: the sender's received_message_id, status,
  //                         and next_message_id.
  //   ClprMessagePayload[]  — ordered message payloads proven by the bundle.
  //   anchor_message_id     : uint64 — ID of the last message in the sender's outbound
  //                          queue included in this bundle. Chain-specific absent/sentinel
  //                          value when messages is empty.
  //   anchor_running_hash   : bytes — the sender's running_hash_after_processing at
  //                          anchor_message_id. Chain-specific absent/sentinel value
  //                          when messages is empty.
  //   new_trust_anchor      : bytes — successor signing authority if the bundle contains
  //                          state-proven rotation evidence; empty bytes if none. MUST be
  //                          state-proven against trust_anchor or MUST revert (§3.2, §4.2).
  //   new_trust_anchor_id   : bytes — opaque identifier for new_trust_anchor. MUST be
  //                          non-empty iff new_trust_anchor is non-empty.
  //   new_endpoint_manifest : ClprEndpointManifest — verified manifest if the bundle
  //                          contains a manifest proof; empty (field not set) if none.
  //
  // Used during:
  //   - submitBundle (on-chain bundle processing)
  //
  // MUST revert if verification fails.
  // SHOULD fail fast on obviously malformed inputs before expensive cryptographic operations.
  function verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)
    returns (ChannelSyncData, ClprMessagePayload[],
             uint64 anchor_message_id, bytes anchor_running_hash,
             bytes new_trust_anchor, bytes new_trust_anchor_id,
             ClprEndpointManifest new_endpoint_manifest)
```
