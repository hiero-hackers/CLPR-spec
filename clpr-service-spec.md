# CLPR Protocol Specification

This document is the cross-platform technical specification for the CLPR (Cross Ledger Protocol). It defines the wire
formats, verification interfaces, state models, and algorithms required to implement CLPR on any ledger. Platform-
specific APIs (HAPI transactions, Solidity contract interfaces, Solana program instructions) are out of scope — this
document provides pseudo-API descriptions for those operations, from which platform-specific specifications will be
derived.

For the architectural rationale, design decisions, and conceptual overview, see the companion
[CLPR Design Document](clpr-service.md).

---

## Notation

- **MUST**, **SHOULD**, **MAY** follow [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) semantics.
- Protobuf definitions use `proto3` syntax.
- `bytes` fields are opaque unless otherwise specified. Byte lengths are noted where protocol-relevant.
- Pseudo-API sections use language-neutral function signatures. Platform-specific specs map these to native constructs
  (HAPI transactions, Solidity functions, Solana instructions, etc.).

---

# 1. Protobuf Definitions

All protobuf types in this section define the canonical wire format for cross-platform interoperability. Implementations
MUST serialize these types using standard protobuf encoding. Implementations MUST reject messages containing
unrecognized fields.

## 1.1 Ledger Identity and Configuration

```protobuf
syntax = "proto3";

// The static configuration for a ledger participating in CLPR.
message ClprLedgerConfiguration {
  // Protocol version. Implementations MUST reject configurations with
  // an unrecognized protocol_version.
  uint32 protocol_version = 1;

  // CAIP-2 chain identifier (e.g., "hedera:mainnet", "eip155:1").
  string chain_id = 2;

  // On-ledger address of this ledger's CLPR Service.
  // On EVM chains: the contract address. On Hiero: a well-known constant.
  // Included so that verifyConfig() can provide the service address to
  // the caller during channel registration.
  bytes service_address = 3;

  // Consensus timestamp of the transaction that last modified this configuration.
  // Monotonically increasing. Used to determine configuration freshness.
  Timestamp timestamp = 4;

  // Capacity limits advertised by this ledger.
  ClprThrottles throttles = 5;

  // Initial trust anchor for this ledger. For ledgers where the signing authority
  // is a stable property of the ledger configuration (e.g., Hiero TSS ledger_id),
  // this field MUST be populated by the source ledger. verifyConfig() returns it
  // directly as initial_trust_anchor. For ledgers where the signing authority is
  // state external to the configuration (e.g., Ethereum sync committees), this
  // field is empty and the verifier derives the trust anchor from a separate state
  // proof embedded in the same proof_bytes.
  bytes initial_trust_anchor = 6;

  // Opaque identifier for initial_trust_anchor. MUST be non-empty iff
  // initial_trust_anchor is non-empty. Format is verifier-defined.
  bytes initial_trust_anchor_id = 7;
}

// Consensus timestamp. Defined here rather than importing google.protobuf.Timestamp
// to avoid a dependency in constrained environments (e.g., on-chain verifiers).
// seconds MUST be non-negative. nanos MUST be in range [0, 999_999_999].
message Timestamp {
  int64 seconds = 1;
  int32 nanos = 2;
}

// Platform binding note — EVM timestamp storage:
// The wire format above is unchanged across platforms. The EVM CLPR Service stores
// timestamps internally as a single uint96 value equal to seconds * 1_000_000_000 + nanos
// (nanoseconds since Unix epoch). This fits any plausible consensus timestamp in ~96 bits
// (the year 2554 corresponds to ~5×10^19 ns, well within uint96 range). When the EVM
// service reads or compares timestamps it always works in this nanos-since-epoch form;
// conversion to/from the two-field Timestamp protobuf happens only at the wire boundary.

// Capacity limits published in the ledger's configuration.
// These are ledger-wide values advertised to all peers. Each Channel
// independently enforces them. Sending ledgers MUST respect these limits
// when constructing messages and bundles.
message ClprThrottles {
  // Hard cap: maximum messages in a single bundle.
  uint32 max_messages_per_bundle = 1;

  // Maximum payload size in bytes for a single message.
  // Enforced by both source (at enqueue) and destination (at bundle processing).
  uint32 max_message_payload_bytes = 2;

  // Maximum gas (or ops budget) allocated to processing a single message.
  uint64 max_gas_per_message = 3;

  // Maximum unacknowledged messages in the outbound queue per Channel.
  // When the queue is full, new messages are rejected until the peer catches up.
  uint32 max_queue_depth = 4;

  // Maximum total size of a serialized ClprSyncPayload (the gRPC message).
  // Endpoints MUST reject messages exceeding this limit.
  //
  // Sizing constraint: this value MUST be greater than max_message_payload_bytes plus the
  // protocol overhead required to deliver a single message (proof bytes, sync data, and
  // message framing). Set generously — if the next message to send is larger than the bundle
  // payload can carry, the Channel deadlocks. A safe rule of thumb is
  // max_sync_bytes >= max_message_payload_bytes * max_messages_per_bundle + proof_overhead,
  // with proof_overhead chosen for the worst-case verifier output.
  uint64 max_sync_bytes = 5;

  // Maximum number of endpoints that may appear in the live ClprEndpointManifest
  // for this CLPR Service. Excess registrations are rejected.
  uint32 max_local_endpoints = 6;

  // Maximum number of peer endpoints this ledger will cache per Channel
  // from the remote ClprEndpointManifest. Zero means no limit.
  // The first max_peer_endpoints are cached locally if the remote's 
  // ClprEndpointManifest has more than max_peer_endpoints entries. 
  uint32 max_peer_endpoints = 7;
}
```

## 1.2 Endpoint Identity

```protobuf
// An endpoint participating in CLPR syncs for this CLPR Service.
// Used in ClprEndpointManifest.endpoints — the service-level on-ledger endpoint
// manifest (see §2.4.2, §6.5).
message ClprEndpoint {
  // Network address and port. Optional; omit for private networks that
  // only initiate outbound syncs.
  ServiceEndpoint service_endpoint = 1;

  // DER-encoded self-signed ECDSA P-384 X.509 CA certificate: the root of
  // trust for mTLS between endpoints. Content is minimized — minimal
  // issuer/subject DN (a single RDN such as CN=CLPR; some implementations
  // reject an entirely empty DN), minimal serial, basicConstraints CA:TRUE,
  // keyUsage
  // keyCertSign+cRLSign, no path-building or revocation extensions. MUST NOT
  // exceed 512 bytes; the CLPR Service MUST reject admission of a
  // ClprEndpoint whose tls_certificate exceeds this size. Not presented on
  // the wire: each endpoint holds a leaf certificate signed by this CA's key
  // and presents that leaf at the mTLS handshake.
  bytes tls_certificate = 2;

}

// Network address for an endpoint.
message ServiceEndpoint {
  // IPv4 or IPv6 numeric address (no DNS hostnames).
  string ip_address = 1;
  uint32 port = 2;
}

// The authoritative, on-ledger endpoint set for this CLPR Service.
// Maintained as distinct CLPR Service state, separate from ClprLedgerConfiguration.
// Service-scoped: all admitted endpoints should serve all Channels on this CLPR Service.
message ClprEndpointManifest {
  // Monotonically increasing counter. MUST be strictly positive (>= 1).
  // Starts at 1. Incremented by 1 on each change to the endpoint set.
  // Has no correlation to any other version numbers in the system.
  uint64 version = 1;

  // On-ledger address of the CLPR Service this manifest belongs to.
  // Binds the manifest to a specific CLPR Service instance.
  // A verifier MUST reject a manifest proof whose service_address does not
  // match ctx.service_address (the peer's service address in the Channel's channel_context).
  bytes service_address = 2;

  // Current endpoint set. May be empty: version >= 1 with no endpoints is valid.
  repeated ClprEndpoint endpoints = 3;
}
```

## 1.3 Control Messages

```protobuf
// Control message payload variants. These are protocol-level messages that
// manage Channel state rather than carrying application data.
// Control Messages do not involve Connectors, are not dispatched to
// applications, and do not generate responses.
//
// Forward compatibility: if a receiver encounters a ClprControlMessage with
// no oneof variant set (indicating an unknown control message type from a
// newer protocol version), it MUST reject the entire bundle. Silently
// skipping unknown control messages could cause state divergence.
message ClprControlMessage {
  oneof payload {
    ClprConfigUpdate config_update = 1;
  }
}

// Carries updated configuration parameters.
//
// Config propagation uses lazy enqueue: when the admin updates the local
// configuration, the CLPR Service stores it with its consensus_timestamp
// (O(1) operation). When a Channel next processes a bundle or enqueues a
// message, the service checks whether the Channel's last_config_timestamp
// is behind the current configuration's consensus_timestamp. If so, a
// ConfigUpdate Control Message is enqueued on that Channel at that point.
// This ensures:
//   - Config updates are O(1) for the admin regardless of Channel count.
//   - Dead or bogus Channels (with no traffic) never incur cost.
//   - Total ordering is preserved: the ConfigUpdate appears at a specific,
//     consensus-determined point in the message stream.
//
// The receiving side MUST verify that the enclosed configuration's timestamp
// is strictly greater than the stored peer_config_timestamp. Since control
// messages are ordered in the queue, this check is a consistency safeguard
// rather than a reordering defense.
message ClprConfigUpdate {
  ClprLedgerConfiguration configuration = 1;
}
```

## 1.4 Message Queue

```protobuf
// Composite key for a queued message.
message ClprMessageKey {
  // Channel ID identifying the Channel this message belongs to.
  // MUST be exactly 32 bytes.
  bytes channel_id = 1;

  // Monotonically increasing sequence number within the Channel.
  uint64 message_id = 2;
}

// Stored value for a queued message.
message ClprMessageValue {
  // The message payload (Data, Response, or Control).
  ClprMessagePayload payload = 1;

  // Cumulative SHA-256 hash after processing this message.
  // SHA-256(previous_running_hash || serialized_payload).
  bytes running_hash_after_processing = 2;
}

// All message classes share the same queue, running hash chain,
// and bundle transport. The oneof distinguishes them at processing time.
message ClprMessagePayload {
  oneof payload {
    ClprMessage message = 1;             // Data Message — application content
    ClprMessageReply message_reply = 2;  // Response Message — outcome of a Data Message
    ClprControlMessage control = 3;      // Control Message — protocol management
  }
}

// Data Message: application-level content sent from one ledger to another.
message ClprMessage {
  // 32-byte derived Connector ID identifying which Connector authorized this message.
  // Derived as keccak256(channel_id || public_key || salt) on registration; identical
  // on source and destination ledgers (see §2.2). Resolved on the destination via
  // (channel_id, connector_id) → local Connector.
  bytes connector_id = 1;

  // Address of the destination application to dispatch to.
  bytes target_application = 2;

  // Source-chain address of the caller, stamped by the CLPR Service at enqueue time.
  bytes sender = 3;

  // Opaque application payload.
  bytes message_data = 4;
}

// Structured outcome of processing a Data Message on the destination ledger.
enum ClprMessageReplyStatus {
  // Application processed the message successfully.
  SUCCESS = 0;

  // Application reverted — not the Connector's fault, no slash.
  APPLICATION_ERROR = 1;

  // Connector missing on destination — slash source Connector.
  CONNECTOR_NOT_FOUND = 2;

  // Connector couldn't pay — slash source Connector.
  CONNECTOR_UNDERFUNDED = 3;
}

// Response Message: the outcome of a previously received Data Message.
// Every Data Message produces exactly one Response Message, in order.
message ClprMessageReply {
  // ID of the originating Data Message this responds to, from the
  // source ledger's outbound queue (the queue that sent the Data Message).
  uint64 message_id = 1;

  // Structured outcome — determines slash decision on the source ledger.
  ClprMessageReplyStatus status = 2;

  // Opaque application response (empty on protocol errors).
  bytes message_reply_data = 3;
}
```

## 1.5 Sync Protocol

The sync protocol defines the wire format for endpoint-to-endpoint communication. A sync cycle is a bidirectional
stream of `ClprSyncPayload` messages scoped to a single Channel; each side requests a bundle describing its
current queue state and returns a bundle carrying opaque proof bytes that the receiving ledger's verifier contract
will interpret.

```protobuf
// One message in the bidirectional sync stream, scoped to a single
// Channel (see ClprEndpointService.sync).
message ClprSyncPayload {
  // Channel ID identifying the Channel this message belongs to.
  // MUST be exactly 32 bytes.
  bytes channel_id = 1;

  // A request describing the requestor's own current Channel state, used by
  // the responder to shape a minimal, progress-making bundle. Absent when the
  // requestor has no further request this cycle.
  BundleRequest bundle_request = 2;

  // A response to a previously received BundleRequest. Absent when this
  // message carries no bundle this cycle.
  BundleResponse bundle_response = 3;
}

// Unproven request describing the requestor's own current Channel
// state, used by the responder to build a minimal, progress-making
// bundle. It informs bundle creation only and MUST NOT cause a state
// change: verifyBundle's replay defense, hash-chain check, and
// trust-anchor update (§4.2) remain the sole authorities on state and
// run when the requestor submits the returned bundle. Each field lets
// the responder build against the requestor's current state (§2.1.2).
message BundleRequest {
  // The requestor's claimed current ChannelSyncData.received_message_id. 
  // This value +1 determines where to start the message sequence in the response bundle.
  uint64 current_received_message_id = 1;

  // The requestor's current ChannelSyncData.status.
  ClprChannelStatus current_status = 2;

  // The requestor's current Channel.trust_anchor_id.
  bytes current_trust_anchor_id = 3;

  // The requestor's current Channel.endpoint_manifest.version.
  uint64 current_endpoint_manifest_version = 4;
}

// A bundle for the peer's verifier contract.
message BundleResponse {
  // Opaque payload for the receiving ledger's verifier contract. Contains
  // the state proof, the message data it attests to, and whatever else
  // the verifier needs to extract and verify sync data and messages
  // (Merkle paths, ZK proofs, TSS signatures, BLS aggregate signatures, etc.).
  bytes bundle_payload = 1;
}

// The lifecycle state of a Channel, propagated to the peer during sync.
enum ClprChannelStatus {
  PENDING = 0;
  ACTIVE = 1;
  PAUSED = 2;
  CLOSING = 3;
  DRAINED = 4;
  CLOSED = 5;
}

// Sync data extracted and verified by a verifier contract from bundle_payload.
// The verifier contract returns this as part of verifyBundle().
//
// Each side of a Channel maintains exactly one ChannelSyncData instance,
// describing its own queue state, proven and exchanged on every bundle.
//
// Field usage:
//   - received_message_id:  provided as an ACK indicating which messages have been received
//   - status:               provided to drive Channel status transitions
//   - next_message_id:      provided to indicate the upperbound on ids of sent messages
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

### gRPC Endpoint Service

Every CLPR endpoint exposes this gRPC service. It is the endpoint-to-endpoint protocol — separate from the on-ledger
CLPR Service API. Implementations MUST configure gRPC max message sizes to accommodate `max_sync_bytes`.

```protobuf
service ClprEndpointService {
  // Bidirectional sync over a stream of ClprSyncPayload messages scoped to a
  // single Channel: each side sends a BundleRequest describing its current
  // queue state and returns a BundleResponse shaped by the peer's request. One
  // stream is one sync cycle; a message with both fields absent closes it. The
  // underlying connection and its TLS session MAY persist across cycles.
  rpc sync(stream ClprSyncPayload) returns (stream ClprSyncPayload);
}
```

## 1.6 Misbehavior Detection (Local Only)

Misbehavior detection and enforcement are **strictly local** — each ledger detects and responds to misbehavior
it observes on its own chain. There is no cross-ledger misbehavior reporting protocol.

---

# 2. On-Ledger State Model

This section defines the logical state that every CLPR Service implementation MUST maintain, regardless of platform.
How this state is physically stored (Merkle tree, contract storage, Solana accounts) is platform-specific.

> **Note:** Connector and endpoint bond data never crosses the wire in cross-platform protobuf messages. The state
> model below describes on-ledger storage only. Balance and stake field widths are platform-specific (e.g., `uint64`
> on Hiero, `uint256` on EVM chains).

## 2.1 Channel

Each Channel is keyed by its **Channel ID** — exactly 32 bytes, chosen by the registrant
(e.g., randomly generated). The same Channel ID is used on both ledgers, which is the fundamental
mechanism that ties the two sides of a Channel together. Because the Channel ID is an independent
value — not derived from any key — the registrant can use different keypairs on each ledger. This is
essential because different ledgers may require different key types (e.g., ECDSA secp256k1 on Ethereum,
Ed25519 on other platforms). The protocol does not mandate a particular signature scheme, allowing
platforms to adopt post-quantum signature algorithms as they become available.

Channel registration uses a **commit-reveal** scheme to prevent cross-ledger front-running (channel ID
squatting). In the commit phase, the registrant submits an **ownership commitment** —
`keccak256(channel_id || public_key)` — which hides both the Channel ID and the key. In the reveal
phase, the registrant reveals the Channel ID and public key, and the system verifies the commitment.
See §5.1 for the full registration lifecycle.

For the purposes of this specification, the Channel data is presented as if it is stored all in one 
data structure.  Different chains may choose to break up the storage of the data into pieces to enable 
efficient CLPR state proofs of ChannelSyncData.  

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

`status`, `received_message_id`, and `next_message_id` are not Channel's own fields — each side's current
values for these are held solely in its own `ChannelSyncData` instance, state-proven and exchanged
on every bundle. Wherever this document refers to `Channel.status`, `Channel.received_message_id`, or 
`Channel.next_message_id`, read this side's own `ChannelSyncData` instance instead.

**Initial values.** When a Channel is created, `acked_message_id` = 0, both running hashes are 32 bytes
of zeros, and `endpoint_manifest` is empty. The initial `trust_anchor_id` is also empty or a sentinel
value of all zeros. For this side's own `ChannelSyncData`: `received_message_id` = 0,
`next_message_id` = 1. No outbound Data Messages exist, so no responses are expected. The first Data
Message ever enqueued will have ID 1, and the first Response Message received from the peer MUST
reference that ID.

**Peer manifest** is cached as `Channel.endpoint_manifest` (see §2.4.2). Initialized at `completeChannel`
from the `ClprEndpointManifest` returned by the
`verifyConfig(config_proof_bytes, channel_id, endpoint_manifest_proof_bytes)` call. Refreshed
automatically through manifest-update bundle payloads (§4.2 Step 1b).

**Message queue** entries are stored separately, keyed by `(channel_id, message_id)`.

### 2.1.1 Channel Status Transitions

| From | To | Type | Trigger | Notes |
|---|---|---|---|---|
| (new) | `PENDING` | Admin | `registerChannel` succeeds (commit phase) | Channel ID claimed. No messaging until revealed. |
| `PENDING` | `ACTIVE` | Admin | `completeChannel` succeeds (reveal phase) | Public key revealed and verified. Messaging enabled. |
| `PENDING` | (deleted) | Admin | Admin calls `closeChannel` | Cleanup of abandoned claims. Immediate deletion (no drain needed). |
| `ACTIVE` | `PAUSED` | Bundle | Inbound bundle contains out-of-order responses (§4.5) | Auto-triggered. No new outbound messages. Syncs continue. |
| `ACTIVE` | `CLOSING` | Admin or Bundle | Admin calls `closeChannel`, or inbound bundle has `sync_data.status ∈ {CLOSING, DRAINED, CLOSED}` (§4.2 Step 5a) | Graceful drain. No new messages from applications. All inbound messages still dispatched normally. |
| `PAUSED` | `ACTIVE` | Bundle | Inbound bundle has correctly ordered responses (§4.5) | Auto-resumes if admin hasn't closed. |
| `PAUSED` | `CLOSING` | Admin or Bundle | Admin calls `closeChannel`, or inbound bundle has `sync_data.status ∈ {CLOSING, DRAINED, CLOSED}` (§4.2 Step 5a) | Channel drains once peer fixes ordering. |
| `CLOSING` | `DRAINED` | Bundle | No Data Messages with ID > `acked_message_id` remain in the outbound queue after processing inbound bundle (§4.2 Step 5b) | All Data Messages delivered and acknowledged. Response Messages generated during CLOSING (for the peer's remaining Data Messages) may still be in flight and drain through the DRAINED state. |
| `DRAINED` | `CLOSED` | Bundle | Inbound bundle has `sync_data.status ∈ {DRAINED, CLOSED}` AND `acked_message_id ≥ next_message_id − 1` (§4.2 Step 5b), including close-notification bundles from the remote endpoint (§3.4) | Terminal state. All processing stops. |
| `DRAINED` | `CLOSED` | Admin | Admin calls `closeChannel` (recovery path when close-notification cannot be delivered — see §3.4) | Terminal state. Outbound queue is already fully acknowledged; nothing is abandoned. |

**Status behavior for incoming bundles:**
- **`PENDING`**: All bundle submissions and `sendMessage` calls are rejected. The Channel exists only as a
claimed ID — no messaging, no syncing.
- **`ACTIVE`**: Bundles accepted and processed normally.
- **`PAUSED`**: Inbound bundles containing out-of-order responses are rejected — nothing is dispatched,
no acknowledgements updated, no hash chain advanced. New `sendMessage` calls are rejected. Existing queued
messages remain and continue syncing. Auto-resumes to ACTIVE when a bundle arrives with correctly ordered
responses (if admin hasn't closed). Admin may close a PAUSED Channel; the Channel always transitions to
`CLOSING` (if the outbound queue is already empty, Step 5b will immediately advance it to `DRAINED` in the same
processing step — there is no direct admin PAUSED→DRAINED path).
- **`CLOSING`**: Inbound bundles are still accepted and processed normally — Data Messages are dispatched to the
application and generate Response Messages; Response Messages are delivered to the originating application. No
new Data Messages are accepted from the local application (`sendMessage` is rejected). Acks flow and queues
drain. Transitions to `DRAINED` when no Data Messages with ID > `acked_message_id` remain in the outbound
queue (§4.2 Step 5b) — all Data Messages have been delivered to and acknowledged by the peer. Response
Messages generated during CLOSING (for the peer's remaining Data Messages) may still be unacknowledged at
DRAINED entry; they drain through the DRAINED state.
- **`DRAINED`**: All Data Messages this side has sent have been received and acknowledged by the peer — no
Data Messages with ID > `acked_message_id` remain in the outbound queue. `sendMessage` is blocked; no new
locally-originated Data Messages are possible. Response Messages generated during CLOSING may still be in the
outbound queue and are not required to be acknowledged at DRAINED entry — they drain before the Channel
reaches `CLOSED`. Inbound bundles are still accepted: the peer may still be draining its own queue,
so arriving Data Messages are dispatched normally and generate Response Messages. These Response Messages are
added to the outbound queue and delivered before the Channel reaches `CLOSED`; the outbound queue may be
temporarily non-empty during this final drain. From the application's perspective, DRAINED means all Data
Messages sent by this side have been delivered; the side is now waiting only for responses to those Data
Messages to return. The `ChannelSyncData.status` carries `DRAINED` so the peer observes it during sync.
Transitions to `CLOSED` when an inbound bundle carries `sync_data.status ∈ {DRAINED, CLOSED}` AND
`acked_message_id ≥ next_message_id − 1` (§4.2 Step 5b), ensuring all Response Messages generated for the
peer's final in-flight Data Messages have also been delivered before the Channel closes. If the remote
endpoint cannot deliver its close-notification, the admin MAY call `closeChannel` on the stuck `DRAINED`
Channel to transition it directly to `CLOSED` (see §3.4). See §2.1.2 for bundle progress criteria while
DRAINED.
- **`CLOSED`**: All bundle submissions are rejected. No further processing occurs.

### 2.1.2 Bundle Progress Criteria

A bundle **makes progress** if at least one of the following conditions holds. The CLPR Service rejects
bundles satisfying none of them (§4.2 Step 1a). CLPR Endpoints MUST NOT submit bundles that make no progress.

| # | Condition | Holds when… |
|---|-----------|-------------|
| 1 | **New messages** | The bundle's anchor message ID is greater than this side's own stored `ChannelSyncData.received_message_id` |
| 2 | **Trust anchor advancement** | The verifier returns `new_trust_anchor_id ≠ Channel.trust_anchor_id` |
| 3 | **Endpoint manifest advancement** | `new_endpoint_manifest` is non-empty and `new_endpoint_manifest.version > Channel.endpoint_manifest.version` |
| 4 | **Acknowledgement progress** | `sync_data.received_message_id > Channel.acked_message_id` |
| 5 | **Channel state transition** | At least one of: **(a)** `sync_data.status ∈ {CLOSING, DRAINED, CLOSED}` with this side's own `ChannelSyncData.status ∈ {ACTIVE, PAUSED}` — triggers `CLOSING` (§4.2 Step 5a); **(b)** this side's own `ChannelSyncData.status = CLOSING` and no Data Messages with ID > `acked_message_id` remain in the outbound queue — triggers `DRAINED` (§4.2 Step 5b); **(c)** this side's own `ChannelSyncData.status = DRAINED`, `sync_data.status ∈ {DRAINED, CLOSED}`, and the acknowledgement point covers every response generated for the peer's remaining Data Messages — triggers `CLOSED` (§4.2 Step 5b) |

**Endpoint pre-check.** Before building a bundle for submission, the endpoint evaluates these conditions
against its cached view of the destination Channel. For Condition 1, it uses the destination's
last-known `received_message_id`. For Condition 2, it compares against the destination's last-known
`trust_anchor_id`. For Condition 3, it compares the remote manifest version last reported in
`current_endpoint_manifest_version` against the stored `Channel.endpoint_manifest.version`. If no condition holds, the endpoint suppresses the
bundle until the next sync cycle produces a progress-bearing payload.

**Service post-check.** Step 1a (§4.2) applies the same check using the Channel's actual on-chain
values after the verifier call returns. A bundle rejected at Step 1a costs the submitting account its
transaction fee; the endpoint pre-check is the primary gate.

## 2.2 Connector

Each Connector is registered on a specific Channel and has a counterpart on the peer ledger.

**Connector ID derivation.** Each Connector is identified by a **Connector ID** — exactly 32 bytes —
derived deterministically from the registrant's public key, the Channel ID, and an optional salt:

    connector_id = keccak256(channel_id || public_key || salt)

where `public_key` is the registrant's full public key in platform-specific encoding and `salt` is an
optional 32-byte label (defaults to 32 bytes of zeros) for operators who need multiple Connectors on
the same Channel. Because the formula is deterministic, the Connector ID is identical on every
ledger where the same operator registers this Connector for this Channel. This stable cross-ledger
identity is what allows applications and the cross-chain dispatch path to refer to a Connector by a
single ID regardless of which ledger they are on.

```
Connector {
  connector_id        : bytes(32)   // derived ID; primary key (with channel_id)
  channel_id       : bytes(32)   // Channel this Connector operates on
  connector_contract  : bytes       // address of the Connector's authorization contract on this ledger
  admin               : bytes       // admin authority (can top up, adjust, shut down)
  balance             : uint        // available funds for message execution (native tokens)
  locked_stake        : uint        // stake locked against misbehavior (slashable)
}
```

The CLPR Service stores Connectors keyed by `(channel_id, connector_id)`. The
`connector_contract` field is recorded so that the Service can invoke the Connector's authorization
contract (`IClprConnectorAuth.authorizeMessage`) on send and charge it on receive.

**Cross-chain dispatch.** When a Data Message arrives on the destination ledger, its
`ClprMessage.connector_id` field carries the source-side Connector ID — which, because the derivation
is deterministic, is identical to the destination-side Connector ID for the same operator. The
destination Service looks up the local Connector via `(channel_id, connector_id)` directly, with no
address translation required.

**Bilateral requirement.** For a Connector to function end-to-end, it MUST be registered on **both**
ledgers under the same `channel_id`, with the same `public_key` and `salt` (so the derived
`connector_id` matches). The source-side registration provides the authorization contract; the
destination-side provides the balance and stake for message execution. If a Connector exists only on
the source side, messages will be authorized and enqueued but will fail with `CONNECTOR_NOT_FOUND` on
the destination.

**Many-to-many.** The relationship is many-to-many: multiple Connectors may serve the same
Channel (different operators, or the same operator with different salts), and a single operator
keypair may register Connectors across multiple Channels.

## 2.3 Endpoint Bond

On ledgers where endpoint registration is permissionless (e.g., Ethereum), each endpoint posts a bond. The bond
acts as a quality filter: it makes registering throwaway endpoints economically unattractive. The bond structure
is determined by the CLPR Service, who is the custodian and the sole authority for holding and returning it.
The bond is always returned in full when an endpoint exits, whether by self-removal (`removeEndpoint`) or admin
eviction (`addEndpoint` rejection or `removeEndpoint` by admin). 

## 2.4 Endpoint Manifest

The CLPR Service maintains its own **local** `ClprEndpointManifest` indicating the endpoints able to receive sync bundles 
for the CLPR Service's channels.  Each Channel maintains a cache of the remote **peer** endpoints that the local 
endpoints can send sync bundles to.  

### 2.4.1 Local Endpoint Manifest

The local endpoint manifest tracks endpoints registered on **this** ledger's CLPR Service. It is
on-ledger state indexed by `registrant_account`.

```
EndpointManifestEntry {
  endpoint             : ClprEndpoint               // see §1.2 for fields
  registrant_account   : bytes                      // on-ledger account of the registrant; bond refund
                                                    // recipient; set from caller at registerEndpoint.
                                                    // MUST be unique within the endpoint set.
                                                    // Length is platform-dependent (e.g., 20 bytes for EVM/Hiero).
  status               : enum { pending, live }     // pending: registered, awaiting admin confirmation;
                                                    // live: admitted to ClprEndpointManifest
  bond                 : uint                       // bond posted (where applicable; see §2.3)
  registered_at        : Timestamp                  // consensus timestamp of registration
}
```

The CLPR Service stores manifest entries keyed by `registrant_account`.

> **Note:** `registrant_account` need not be the same account used to submit bundle transactions. Bundle
> submission is permissionless — any account may call `submitBundle` regardless of manifest membership.
> The CLPR Service does not enforce or verify any relationship between `registrant_account` and the
> account that submits bundles on behalf of that endpoint. If a CLPR Service Admin wishes to track which
> registered endpoints are actively submitting, that association must be maintained out of band; it is
> not part of this specification.

**Population.** On platforms where endpoints are derived automatically from the consensus roster
(e.g., Hiero), the CLPR Service maintains the manifest as a mirror of the consensus node set, with
all entries in `live` state. On permissionless platforms, endpoints are admitted through a two-step
process: any caller registers via `registerEndpoint` (entering `pending` state with bond held); the
CLPR Service Admin confirms via `addEndpoint` (moving the endpoint to `live` state and adding it to
the `ClprEndpointManifest`). Only `live` endpoints appear in `ClprEndpointManifest.endpoints`.

### 2.4.2 Peer Manifest (on-ledger, per Channel)

Each Channel stores an on-ledger `ClprEndpointManifest` (see §1.2) as its authoritative directory
of remote endpoints to contact for gRPC sync. This manifest is stored as `Channel.endpoint_manifest`,
with its version tracked internally as `Channel.endpoint_manifest.version` — there is no separate
top-level version field on Channel.

**Population.** At `completeChannel`, the CLPR Service calls the extended
`verifyConfig(config_proof_bytes, channel_id, endpoint_manifest_proof_bytes)` on the verifier contract, which
returns — among other values — a `ClprEndpointManifest` at version ≥ 1 (see §3.1). The Service stores
this as `Channel.endpoint_manifest`, truncated to `ClprThrottles.max_peer_endpoints` if the endpoint list
exceeds that limit.

**Refresh.** When `verifyBundle` returns a `new_endpoint_manifest` with `version > Channel.endpoint_manifest.version`,
the CLPR Service performs a manifest update: atomically replace `Channel.endpoint_manifest` with the new manifest
(truncated to `ClprThrottles.max_peer_endpoints` if the endpoint list exceeds that limit). See §4.2 Step 1b.

**Empty manifest.** A manifest at version ≥ 1 with no endpoints is valid. `completeChannel` MUST
NOT revert solely because the remote manifest's endpoint list is empty. A Channel with an empty peer
manifest depends on the remote CLPR Endpoints to initiate syncs with the local ledger's endpoints. If
the local ledger also has 0 CLPR Endpoints in its manifest, the Channel will sit idle until someone 
proactively submits bundles to either ledger that makes progress on the Channel.  

**Supplemental discovery.** Supplemental off-chain discovery protocols are possible independently of
this specification, since bundle submission is permissionless.

---

# 3. Verification Interfaces

CLPR is proof-system-agnostic. All cryptographic verification is delegated to verifier contracts. The CLPR Service
never interprets proof bytes directly.

## 3.1 Verifier Contract Interface

Every verifier contract deployed for a Channel MUST implement two methods. The method signatures below are
language-neutral; platform-specific specs define the concrete ABI.

The two interface methods MUST be read-only with respect to CLPR Service state. Per-channel mutable trust
material (signing authority, validator sets, sync committees) is passed in and returned via the `trust_anchor` /
`new_trust_anchor` / `new_trust_anchor_id` parameters rather than stored in the verifier. Constant Channel data
(channel_id, service_address) is passed via `ChannelContext`. Verifiers MAY maintain global state that is
not channel-specific (e.g., an admin-updated emergency revocation list).

`ChannelContext` carries the constant, channel-scoped data that identifies which Channel a
bundle belongs to. It is assembled by the CLPR Service from fields fixed at `completeChannel` time
and is passed to every verifier call unchanged for the lifetime of the Channel. The interface is
uniform across all verifier implementations; individual verifiers consume only the fields their proof
system requires. EVM/QBFT verifiers use `channel_id` and `service_address` to derive storage slot
keys and account proof paths when validating Merkle-Patricia-Trie state proofs. Hiero verifiers
authenticate via state proof: the proof directly authenticates the specific `ClprChannel` leaf's
`service_address` field rather than deriving it from `ChannelContext`.

```
struct ChannelContext {
  bytes channel_id    // 32-byte Channel identifier, same on both ledgers
  bytes service_address  // on-ledger address of the peer's CLPR Service
}
```

```
interface IClprVerifier {

  // Verify a configuration proof and endpoint manifest proof. Returns the peer's verified
  // configuration and initial endpoint manifest.
  //
  // Inputs:
  //   config_proof_bytes            : bytes     — proof of the remote ClprLedgerConfiguration.
  //   channel_id                 : bytes(32) — the Channel identifier. Echoed verbatim
  //                                               in the returned ctx.
  //   endpoint_manifest_proof_bytes : bytes     — proof of the remote ClprEndpointManifest.
  //                                               MUST be verifiable against the initial trust
  //                                               anchor established by this call.
  //
  // Returns:
  //   ctx               : ChannelContext — constant channel-scoped data: channel_id
  //                                 (echoed from input) and service_address (peer CLPR Service
  //                                 address). Stored as Channel.channel_context; passed
  //                                 to every verifyBundle call for the Channel's lifetime.
  //   chain_id          : string  — CAIP-2 identifier of the peer ledger.
  //   peer_config_nanos : uint96  — config version timestamp in nanoseconds since
  //                                 Unix epoch (seconds * 1_000_000_000 + nanos).
  //                                 Stored on the Channel as peerConfigTimestamp
  //                                 and used to detect stale CONTROL config updates.
  //   throttles         : ClprThrottles — peer's capacity limits, stored on the
  //                                 Channel as peerThrottles and enforced during
  //                                 §4.3 step-4 payload-size and queue-depth checks.
  //   initial_trust_anchor : bytes — opaque starting trust anchor for the Channel.
  //                                 Extracted from ClprLedgerConfiguration.initial_trust_anchor
  //                                 when the source ledger publishes its signing authority in
  //                                 its configuration (e.g., Hiero TSS ledger_id). For ledgers
  //                                 where signing authority is state external to the
  //                                 configuration (e.g., Ethereum sync committees), the verifier
  //                                 derives this value from a separate state proof within
  //                                 proof_bytes. Empty bytes for verifiers with no rotating-
  //                                 authority concept.
  //   initial_trust_anchor_id : bytes — opaque identifier for initial_trust_anchor.
  //                                 Extracted from ClprLedgerConfiguration.initial_trust_anchor_id
  //                                 when present, or computed by the verifier. Initializes
  //                                 Channel.trust_anchor_id. Empty bytes when
  //                                 initial_trust_anchor is empty.
  //   endpoint_manifest     : ClprEndpointManifest — verified manifest at version >= 1.
  //                                   MUST NOT revert solely because endpoints is empty.
  //
  // MUST revert if:
  //   - any proof verification fails
  //   - manifest.service_address does not match ctx.service_address
  //   - manifest.version is 0
  //
  // Used during:
  //   - completeChannel to verify and extract the peer's configuration
  function verifyConfig(bytes config_proof_bytes, bytes channel_id, bytes endpoint_manifest_proof_bytes)
    returns (ChannelContext ctx, string chain_id, uint96 peer_config_nanos,
             ClprThrottles throttles, bytes initial_trust_anchor,
             bytes initial_trust_anchor_id, ClprEndpointManifest endpoint_manifest)

  // Verify a bundle payload against the Channel's current trust anchor.
  // Returns verified sync data, an ordered array of message payloads,
  // a state-proven identity for the last message in that array, a successor
  // trust anchor, and an updated endpoint manifest (empty if none).
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
  //   ChannelSyncData     — verified queue state: the sender's received_message_id,
  //                          status, and next_message_id.
  //   ClprMessagePayload[]  — ordered message payloads proven by the bundle.
  //   anchor_message_id     : uint64 — ID of the last message in the sender's outbound
  //                          queue included in this bundle. Chain-specific absent/sentinel
  //                          value when messages is empty.
  //   anchor_running_hash   : bytes — the sender's running_hash_after_processing at
  //                          anchor_message_id. Chain-specific absent/sentinel value
  //                          when messages is empty.
  //   new_trust_anchor      : bytes — successor signing authority if the bundle contains
  //                          state-proven rotation evidence; empty bytes if none. MUST be
  //                          state-proven against trust_anchor or MUST revert.
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
}
```

## 3.2 Endpoint Trust Anchor Rotation Obligations

This section specifies endpoint behaviour during trust anchor transitions. It is normative for endpoint implementations;
the on-chain CLPR Service only enforces what the verifier returns.

### Per-channel obligation

For every Channel it maintains, a CLPR endpoint is responsible for ensuring the remote ledger's stored trust anchor
is current enough to verify the state proofs it intends to submit. Every received sync payload carries the peer's
current `trust_anchor_id` — the opaque identifier for the remote ledger's `Channel.trust_anchor`, self-reported and
unproven. On each sync cycle the endpoint performs the following:

1. Read `trust_anchor_id` from the most recently received sync payload for this Channel.
2. Look up the identifier in the endpoint's local trust anchor record to identify which signing authority is installed
   on the remote ledger. If the identifier is unrecognized, fetch the full `Channel.trust_anchor` bytes from the
   remote chain's state once and cache them by identifier.
3. Determine whether the endpoint can produce source chain state proofs verifiable under that signing authority.
4. If yes — submit a normal bundle containing state proofs of pending messages.
5. If no — construct and submit one or more transition bundles, each embedding rotation evidence as a state-proven
   commitment inside `bundle_payload`, until the remote ledger's trust anchor reaches signing authority for which the
   endpoint can produce valid proofs. Then proceed to step 4.

**Endpoint local record.** Each CLPR endpoint MUST maintain, per Channel, a local ordered sequence of
`trust_anchor_id` values it has encountered — which authority succeeded which — derived from its observation of the
source chain. This ordering determines which transition bundles to construct and in what sequence when step 5 applies.
The on-chain CLPR Service stores only the current trust anchor and its identifier; history and ordering are exclusively
the endpoint's responsibility.

An endpoint MAY additionally maintain a local mapping of `trust_anchor_id → trust_anchor_bytes`. Storing the bytes
locally avoids an on-demand remote chain read in step 2 and enables pre-verification of a `bundle_payload` against the
local copy before submission. Endpoints that do not maintain this mapping must fetch `Channel.trust_anchor` from the
remote chain's state when the identifier is unrecognized (step 2). Both approaches are protocol-correct; the mapping is
a local implementation detail with no observable effect on the CLPR Service or the peer ledger.

### Triggering

A transition bundle is required in two situations:

1. **Proactive** — The endpoint detects that the source chain's signing authority has changed (e.g., a new sync
   committee in the Ethereum beacon state, a validator-set-change transaction on an IBFT chain, or a ledger-ID
   succession commitment in the Hiero state tree). The remote anchor is now behind the source chain; the endpoint
   constructs and submits the transition bundle before the next application bundle.

2. **Reactive** — The endpoint reads `trust_anchor_id` from a received sync payload and, after consulting its local
   record, determines it cannot produce state proofs verifiable under the identified authority (e.g., after a restart or
   monitoring gap). It must advance the remote anchor before submitting application bundles.

### Transition bundle

A transition bundle:

1. Has a `bundle_payload` whose state proof is verifiable by an authority encoded in the current
   `trust_anchor`.
2. Embeds a state-proven commitment to a new or additional authority. The verifier validates this commitment
   and returns `new_trust_anchor` and `new_trust_anchor_id`. If no valid commitment is present the verifier MUST revert.
3. MAY carry zero application messages (trust-only bundle — see §2.1.2 Bundle Progress Criteria, Condition 2).
4. After the transition bundle is accepted, `Channel.trust_anchor` holds whatever the verifier returned, and
   `Channel.trust_anchor_id` holds the returned `new_trust_anchor_id`. All subsequent
   bundles are verified against the new value.

### Timing depends on trust anchor encoding

The protocol itself imposes no wall-clock deadline on when the transition bundle must be submitted. The timing
constraints that matter are entirely a product of the trust anchor's format, as determined by the source chain's proof
system:

**Single-authority encoding.** The trust anchor holds exactly one valid signing authority. After the transition
bundle updates it, proofs from the prior authority are immediately rejected. The endpoint must submit the
transition bundle at the exact source chain authority boundary, with no prior-authority proofs in-flight. This
is the most constrained design.

**Window encoding.** The trust anchor encodes a set of currently valid authorities (e.g., `{ current_committee,
next_committee }` for Ethereum). The verifier accepts proofs signed by any authority in the set. A transition
bundle advances the window when a proof includes a state-proven commitment to an additional authority. This
decouples endpoint submission from the source chain boundary — the endpoint can submit the window-advance bundle at
any point while the current window covers all in-flight proofs. Verifier implementers SHOULD use this pattern
for proof systems with frequent predictable rotations.

### Proof system compatibility requirement

For any proof system compatible with CLPR, a proof MUST be constructable that is:
(a) verifiable by an authority currently encoded in `trust_anchor`, and
(b) contains a state-proven commitment to the new authority.

A source ledger whose authority rotation can only be attested by the new authority, with no overlap where the
old authority attests to the new authority's commitment, cannot support the CLPR trust anchor model.

### Catch-up across multiple missed rotations

If the endpoint was offline across N rotations, it submits N consecutive transition bundles before resuming
application bundles. Each is verified against the trust anchor installed by the previous transition. Historical
boundary blocks remain available in the source chain's immutable record. With a window encoding, the endpoint can
additionally batch window-advances into regular application bundles rather than requiring separate
trust-only bundles.

A verifier MAY support batching multiple rotations into a single `bundle_payload`. In this case the verifier
walks the chain of rotation proofs internally — each sub-proof verified against the authority derived by the
previous step — and returns only the final `new_trust_anchor` and `new_trust_anchor_id`. The CLPR Service sees
a single `verifyBundle` call; the intermediate authorities are never stored on-chain. This optimization reduces
the number of on-chain transactions required for catch-up from N+1 to 1. Application messages proven under the
final authority MAY be included in the same bundle.

## 3.3 Connector Authorization Interface

When the CLPR Service processes a message send, it calls the Connector's authorization contract.

```
interface IClprConnectorAuth {

  // Called by the CLPR Service when an application submits a message via
  // this Connector. Returns true if the Connector authorizes the message.
  //
  // The Connector may inspect any field and apply arbitrary authorization
  // logic (allow-lists, rate limits, payment requirements, etc.).
  //
  // By returning true, the Connector is making a commitment: it asserts that
  // its counterpart on the destination ledger has sufficient funds to pay for
  // execution, and that it itself can cover the cost of handling the response.
  // This commitment is the contractual basis for slashing (§4.6).
  //
  // MUST NOT have side effects that modify CLPR Service state.
  //
  // The message_size parameter is provided as a gas optimization — Connectors
  // that only need to check size thresholds can avoid reading the full payload.
  function authorizeMessage(
    bytes sender,              // source-chain address of the caller
    bytes target_application,  // destination app address
    uint64 message_size,       // payload size in bytes (convenience; equals message_data.length)
    bytes message_data         // opaque application payload
  ) returns (bool authorized)
}
```

## 3.4 Endpoint Close-Notification Obligation

When the CLPR Service transitions a local Channel to `CLOSED`, the endpoint MUST submit one final
**close-notification bundle** to the remote CLPR Service. This bundle is a normal bundle — same wire
format as any other — constructed from the local ledger state at or after the `CLOSED` transition block.
Its `ChannelSyncData` carries `status = CLOSED` along with the final `received_message_id`; the bundle's
anchor message carries the final message ID and running hash.

**Purpose.** Once a Channel reaches `CLOSED`, normal sync cycles end. The close-notification is the
endpoint's terminal sync obligation: it delivers the final state proof to the remote and drives both sides
to mutual closure. The endpoint retries until the remote either accepts the bundle (transitioning to
`CLOSED`) or rejects it because the Channel is already `CLOSED`. Only then does the endpoint cease sync
activity for this Channel. If the endpoint is permanently unable to deliver the close-notification,
the remote Channel remains stuck at `DRAINED`; the remote admin MAY call `closeChannel` on the
stuck `DRAINED` Channel to transition it directly to `CLOSED` (§2.1.1, §5.3) — the outbound queue is
fully acknowledged, so nothing is abandoned.

**What the remote CLPR Service does with it.** The remote accepts the close-notification (DRAINED Channels
still accept inbound bundles) and processes it through the standard verification algorithm (§4.2):

- If the remote is `CLOSING` and the final `sync_data.received_message_id` completes its drain:
  Step 5b sub-check 1 fires (`→ DRAINED`), then sub-check 2 fires (`sync_data.status = CLOSED → CLOSED`).
  The remote closes in a single bundle submission.
- If the remote is already `DRAINED`: Step 5b sub-check 2 fires directly (`→ CLOSED`).
- If the remote is `ACTIVE` or `PAUSED` (endpoint was offline while local closed): Step 5a fires
  (`→ CLOSING`), and subsequent processing continues the normal drain-to-close sequence.

**Remote already CLOSED.** If the remote CLPR Service rejects the close-notification because the Channel
is already `CLOSED`, both sides are confirmed closed and the endpoint ceases all sync activity for this
Channel.

**NoProgress exemption.** A close-notification bundle satisfies Bundle Progress Criterion 5 (§2.1.2) in all
cases where it can usefully be processed: `sync_data.status = CLOSED` satisfies sub-condition (a) when the
remote status is `ACTIVE` or `PAUSED`, and sub-condition (c) when the remote status is `DRAINED`. If the
remote is `CLOSING` and has not yet seen the final ack, the bundle may also satisfy Criterion 4 (ack
progress) or sub-condition (b) (if the ack data completes the drain). A close-notification that carries no
new information — because the remote is already `CLOSED` — is rejected because the Channel is CLOSED, not
with `NoProgress`.

---

# 4. State Transition Algorithms

## 4.1 Running Hash Computation

The running hash chain uses **SHA-256**. Each message's running hash is computed as:

```
running_hash = SHA-256(previous_running_hash || serialized_payload)
```

where `||` denotes byte concatenation and `serialized_payload` is the canonical protobuf serialization of the
`ClprMessagePayload`.

**Initial value.** When a Channel is first created, both `sent_running_hash` and `received_running_hash` are
initialized to 32 bytes of zeros (`0x00 * 32`). Both sides of the Channel MUST agree on this initial value.

**Hash algorithm rationale.** SHA-256 is chosen for universal platform availability: EVM `sha256` precompile, Hiero
native, Solana `sol_sha256` syscall. Under Grover's algorithm, SHA-256 retains 128-bit preimage resistance, adequate
for the running hash chain's security requirements. The hash algorithm can be upgraded via a protocol version bump
(see `ClprLedgerConfiguration.protocol_version`) and Channel renegotiation — the `running_hash` fields are opaque
`bytes`, so no wire format change is needed.

## 4.2 Bundle Verification Algorithm

When a bundle is submitted to the CLPR Service (post-consensus):

**Step 1 — Verifier call.** Pass `bundle_payload`, the Channel's current `trust_anchor`, and the Channel's stored `channel_context` to the Channel's verifier contract via `verifyBundle(bundle_payload, trust_anchor, channel_context)`. The verifier returns
`(ChannelSyncData sync_data, ClprMessagePayload[] messages, uint64 anchor_message_id, bytes anchor_running_hash, bytes new_trust_anchor, bytes new_trust_anchor_id, ClprEndpointManifest new_endpoint_manifest)`.
If the verifier reverts or returns an error, reject the entire bundle. The submitting account pays the transaction
cost.

**Step 1a — NoProgress check.** Reject the bundle with `NoProgress` if none of the five Bundle Progress
Criteria (§2.1.2) holds. Concretely, reject if all of the following are true simultaneously:

- `messages.length == 0`, or `anchor_message_id ≤` this side's own stored `ChannelSyncData.received_message_id` — no new messages
- `new_trust_anchor_id == Channel.trust_anchor_id` (or `new_trust_anchor` is empty) — trust anchor not advanced
- `new_endpoint_manifest` is empty bytes, or non-empty but `new_endpoint_manifest.version ≤ Channel.endpoint_manifest.version` — manifest not advanced (Criterion 3)
- `sync_data.received_message_id <= Channel.acked_message_id` — no acknowledgement progress
- None of the channel-state-transition sub-conditions in Criterion 5 (§2.1.2) hold

A bundle satisfying at least one criterion is valid. In particular, a bundle that advances the trust anchor
but carries no application messages is a valid rotation-only proof (Criterion 2). A bundle carrying only a
manifest proof with an advancing version satisfies Criterion 3 and is valid.

**Step 1b — Manifest update.** If `new_endpoint_manifest` is non-empty and `new_endpoint_manifest.version > Channel.endpoint_manifest.version`,
perform a manifest update as defined in §2.4.2.
If `new_endpoint_manifest` is empty bytes, or non-empty but `new_endpoint_manifest.version ≤ Channel.endpoint_manifest.version`, silently skip — do not revert.

**Step 1c — Trust anchor update.** If `new_trust_anchor.length > 0`, atomically set
`Channel.trust_anchor = new_trust_anchor` and `Channel.trust_anchor_id = new_trust_anchor_id`
**before** processing any message in this bundle. There is no window in which the old anchor applies to one
message and the new anchor to another within the same bundle. This guarantees that all messages in the bundle
are attributed to the authority that produced them.

**Step 2 — Bundle size check.** If the number of messages returned by the verifier exceeds `max_messages_per_bundle`,
reject the entire bundle. If any individual message payload exceeds `max_message_payload_bytes`, reject the entire
bundle. (Per the design document §3.2.5, an oversized payload is evidence of a dishonest source — the entire bundle
is tainted.)

**Step 3 — Replay defense.** For every message returned by the verifier:
- The first message's ID MUST equal `received_message_id + 1`.
- Subsequent message IDs MUST be contiguous and ascending (each ID = previous ID + 1).
- If any constraint is violated, reject the entire bundle.

**Step 4 — Running hash verification.** Starting from the Channel's current `received_running_hash`, recompute
the hash chain for each message in the bundle sequentially: apply `SHA-256(prev_hash || serialized_payload)`.
The final computed hash MUST equal the `anchor_running_hash` returned by the verifier.
(`anchor_running_hash` is the cumulative hash through the anchor message — the last message in the bundle —
not necessarily through all messages the sender has ever enqueued.) If they do not match, reject the entire
bundle.

**Step 5 — Acknowledgement update.** Update `acked_message_id` from the verifier-returned
`ChannelSyncData.received_message_id`. Delete acknowledged Response Messages and Control Messages from the outbound
queue (neither generates a further response). Retain acknowledged Data Messages until their corresponding response
arrives (see §4.5).

**Step 5a — Queue status check.** If `ChannelSyncData.status` is `CLOSING`, `DRAINED`, or `CLOSED` and
this side's own `ChannelSyncData.status` is `ACTIVE` or `PAUSED`, transition to `CLOSING`. Set the local `ChannelSyncData.status` to
`CLOSING` so the peer observes it on the next sync. The `CLOSED` case handles close-notification bundles
(§3.4) arriving at a Channel that has not yet begun closing locally.

**Step 5b — Drain check.** Two independent sub-checks, evaluated in order:

1. If this side's own `ChannelSyncData.status` is `CLOSING` and no Data Messages with ID > `acked_message_id` remain in the outbound
   queue (peer has acknowledged all Data Messages, using the `acked_message_id` already updated in Step 5),
   transition the Channel to `DRAINED`. Response Messages generated during CLOSING (for the peer's remaining
   Data Messages) are not required to be acknowledged for this transition — they may still be in flight and
   will drain through the DRAINED state. This condition may fire on the same bundle that triggered the CLOSING
   transition in Step 5a — a Channel with no unacknowledged Data Messages transitions directly through
   CLOSING to DRAINED in a single step.
2. If this side's own `ChannelSyncData.status` is `DRAINED` (whether just transitioned in sub-check 1 or already `DRAINED` prior to
   this bundle), `sync_data.status ∈ {DRAINED, CLOSED}`, AND `acked_message_id >= next_message_id - 1`
   (all Response Messages generated for the peer's final in-flight Data Messages have been acknowledged),
   transition the Channel to `CLOSED`.

**Step 5c — Lazy config propagation.** If the Channel's `last_config_timestamp` is less than the current
configuration's `consensus_timestamp`, enqueue a `ConfigUpdate` Control Message (see §1.3) on this Channel's
outbound queue and update `last_config_timestamp` to match. This ensures the ConfigUpdate appears at a deterministic,
consensus-determined point in the message stream — specifically, after the acknowledgement update and before
any new messages generated by this bundle's dispatch. `ConfigUpdate` carries throttle and protocol version changes
only; endpoint manifest changes are propagated through bundle payloads (§4.2 Step 1b), not through `ConfigUpdate`.

**Step 6 — Message dispatch.** For each message in order, dispatch by type:

|   Message Type   |                                                                                                                                                                                           Processing                                                                                                                                                                                           |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Control Message  | Apply directly (store new config values). Advance `received_message_id` and `received_running_hash`. No response generated.                                                                                                                                                                                                                                     |
| Data Message     | Resolve `connector_id` to local Connector via cross-chain mapping. Charge Connector the full per-message amount: `actual_gas_used * (1 + connector.margin_percent / 100)`. The entire charge is paid to the submitting account; there is no protocol treasury. Dispatch to target application. Generate Response Message. Advance `received_message_id` and `received_running_hash`. |
| Response Message | Deliver to originating application. Verify ordering per §4.5. Charge Connector the full per-message amount: `actual_gas_used * (1 + connector.margin_percent / 100)`. The entire charge is paid to the submitting account. Advance `received_message_id` and `received_running_hash`.                                                                                                                                                                                                                                                                       |

A failure on one message does NOT stop processing of remaining messages in the bundle. Each message is processed
independently — if one Connector is underfunded, a `CONNECTOR_UNDERFUNDED` response is generated for that message
while other messages (including from the same Connector) continue processing normally.

## 4.3 Message Enqueue Algorithm

When a message send is processed:

1. Look up the Channel by `channel_id`. Reject if status is not `ACTIVE`.
   1a. **Lazy config propagation.** If the Channel's `last_config_timestamp` is less than the current
   configuration's `consensus_timestamp`, enqueue a `ConfigUpdate` Control Message (see §1.3) on this
   Channel's outbound queue before the application's message and update `last_config_timestamp` to match.
   This ensures config changes are
   propagated before new Data Messages.
2. Look up the Connector by `(channel_id, connector_id)`. Reject if not found.
3. Call `IClprConnectorAuth.authorizeMessage()` on the Connector's authorization contract. Reject if not authorized.
4. Validate payload size: `len(message_data) <= peer_config.throttles.max_message_payload_bytes`,
   where `peer_config` is the most recent verified peer configuration stored on the Channel.
   Reject if exceeded. The source ledger MAY additionally enforce a stricter local limit, but it
   MUST NOT enqueue any message that exceeds the destination's advertised `max_message_payload_bytes`.
5. Validate queue depth: `next_message_id - acked_message_id < max_queue_depth`. Reject if full.
6. Construct `ClprMessage` with `connector_id`, `target_application`, `sender` (stamped from transaction caller),
   and `message_data`.
7. Compute `running_hash = SHA-256(sent_running_hash || serialized_payload)`.
8. Store the message in the queue keyed by `(channel_id, next_message_id)`.
9. Update Channel: `sent_running_hash = running_hash`. Update ChannelSyncData: `next_message_id += 1`.

## 4.4 Message Lifecycle

**Response Messages** in the outbound queue are deleted when the peer acknowledges them (ack covers their ID).

**Control Messages** in the outbound queue are also deleted on ack — they do not generate responses and require no
further action once acknowledged.

**Data Messages** are retained after acknowledgement because they serve as the ordering reference for response
verification (§4.5). They are deleted only when their corresponding response has been received and matched.

## 4.5 Response Ordering Verification

When a Response Message arrives in a bundle on the source ledger:

1. Walk the outbound queue of retained Data Messages (skipping Response Messages and Control Messages).
2. The incoming response's `message_id` MUST match the oldest unresponded Data Message's ID.
3. **Match:** deliver the response to the originating application, read the originating application address from
   `ClprMessage.sender`, and delete the matched slot.
4. **Mismatch:** the peer has violated the ordering guarantee. Set Channel status to `PAUSED`. Reject new
   `sendMessage` calls on this Channel.

**Initial state.** When a Channel is first created, no outbound Data Messages exist, so no responses
are expected and the walk in step 2 finds nothing. The first Response Message received MUST match the
first Data Message ever sent on this Channel (ID 1, since `next_message_id` is initialized to 1).
Response ordering is tracked implicitly by walking the retained outbound Data Messages — there is no
separate counter.

**PAUSED recovery.** A PAUSED Channel auto-resumes when the next inbound bundle contains correctly ordered
responses (if the admin hasn't closed it). The ordering violation indicates a peer-side bug in response
generation — the peer must fix it (which may require a contract upgrade on platforms like Ethereum). While
PAUSED, existing queued messages remain and continue syncing, but inbound bundles with out-of-order responses
are rejected outright — nothing is dispatched, acknowledged, or advanced. New `sendMessage` calls are also
blocked. The admin may close a PAUSED Channel (`closeChannel` transitions it to CLOSING). While
CLOSING, bundles with out-of-order responses are still rejected; once the peer fixes the ordering, bundles
are dispatched normally and queues drain through DRAINED → CLOSED.

**Distinction from bad inbound bundles.** If a peer sends bundles that fail verification (bad hash chain, replay,
oversized payloads), the CLPR Service simply rejects them — no pause, no state change. The Channel remains
ACTIVE and will accept valid bundles as soon as the peer fixes the issue. PAUSED is reserved exclusively for
response ordering violations, which indicate corruption in the peer's outbound queue state.

## 4.6 Slashing Decision

Slashing is two-sided — both the destination and source ledger enforce penalties independently, so that the endpoint
that did the work on each side is compensated on its own ledger.

When a Data Message is processed on the **destination** ledger:

|         Outcome         |                                         Destination-Side Action                                         |
|-------------------------|---------------------------------------------------------------------------------------------------------|
| Connector found, funded | Charge Connector `actual_gas_used * (1 + connector.margin_percent / 100)`. The full amount is paid to the submitting account. The margin covers the submitter's overhead and provides a profit incentive; there is no separate protocol treasury or owner cut. |
| `CONNECTOR_NOT_FOUND`   | No Connector to slash. Submitting account absorbs execution cost. Failure response enqueued.           |
| `CONNECTOR_UNDERFUNDED` | Slash destination Connector's `locked_stake`. Reimburse submitting account. Failure response enqueued. |

When the failure Response Message arrives back on the **source** ledger:

| `ClprMessageReplyStatus` |                                 Source-Side Action                                  |
|--------------------------|-------------------------------------------------------------------------------------|
| `SUCCESS`                | No penalty. Deliver response to application.                                        |
| `APPLICATION_ERROR`      | No penalty. Deliver error to application.                                           |
| `CONNECTOR_NOT_FOUND`    | Slash source Connector's `locked_stake`. Reimburse source-side submitting account. |
| `CONNECTOR_UNDERFUNDED`  | Slash source Connector's `locked_stake`. Reimburse source-side submitting account. |

Penalties escalate. A single failure results in a fine. Repeated failures MAY result in the Connector being banned
from the Channel and its remaining stake forfeited. Platform-specific specifications MUST define the slashing
schedule (fine amounts, escalation thresholds, ban conditions).

> **Stake-to-exposure invariant.** Each side's Connector bond MUST be sufficient to cover the worst-case endpoint
> losses on that ledger. If a bond is too small, a malicious actor can create a Connector with minimal stake,
> authorize a burst of messages, and drain endpoints of more execution cost than the slash can reimburse. Platform
> specs MUST define minimum Connector bond requirements.

---

# 5. Channel Lifecycle

## 5.1 Channel Registration

Channel creation is **permissionless** and uses a **commit-reveal** scheme to prevent cross-ledger
front-running (channel ID squatting).

### 5.1.1 Anti-Squatting Rationale

A Channel ID is used on both ledgers. Without protection, an attacker who observes a registration on ledger A
could race to claim the same Channel ID on ledger B before the legitimate registrant. The commit-reveal scheme
prevents this: the registrant submits an ownership commitment — `keccak256(channel_id || public_key)` — which
hides both the Channel ID and the public key. The registrant registers this commitment on both ledgers (the
commit phase) without revealing either value. Only after both claims are established does the registrant reveal
the Channel ID and public key (the reveal phase). An attacker who sees the commitment cannot derive the
Channel ID or the public key from the hash, so they cannot complete the reveal phase on either ledger.

### 5.1.2 Phase 1 — Claim (`registerChannel`)

The registrant:

1. Chooses a **Channel ID** — an arbitrary 32-byte value (e.g., randomly generated). The same
   Channel ID will be used on both ledgers.
2. Generates a **keypair** on the local ledger using a signature scheme supported by that platform
   (e.g., ECDSA secp256k1 on Ethereum, Ed25519 on other platforms). Different keypairs may be used
   on different ledgers — the Channel ID (not the key) is what ties the two sides together.
3. Computes the **ownership commitment** as `keccak256(channel_id || public_key)`.
4. Calls `registerChannel` on the local CLPR Service with the ownership commitment.

The CLPR Service:

1. Verifies the ownership commitment is not already claimed.
2. Creates the Channel in PENDING state, keyed internally by the commitment until reveal.

The registrant repeats this claim independently on both ledgers (potentially with different keypairs)
before proceeding to the reveal phase.

### 5.1.3 Phase 2 — Reveal (`completeChannel`)

Once the ownership commitment is registered on both ledgers, the registrant:

1. Deploys a **verifier contract** on the local ledger.
2. Obtains a **configuration proof** and an **endpoint manifest proof** from the peer ledger through off-chain means.
3. Calls `completeChannel` on each ledger with the **Channel ID**, the **public key** (for that
   ledger's keypair), a **signature** proving control of the keypair, the **verifier contract**
   address, the peer's **config proof**, and the peer's **endpoint manifest proof**. The signature scheme
   and key encoding are platform-specific. Different keys may be used on each ledger.

The CLPR Service:

1. Verifies `keccak256(channel_id || public_key)` matches a stored ownership commitment.
2. Verifies the **signature** over `keccak256(channel_id)`. The signature scheme is platform-specific.
3. Calls `verifyConfig(config_proof_bytes, channel_id, endpoint_manifest_proof_bytes)` on `verifier_contract` to obtain
   the peer's verified configuration and initial endpoint manifest (ctx, chain_id,
   peer_config_nanos, throttles, initial_trust_anchor, initial_trust_anchor_id, endpoint_manifest).
4. Stores the returned `ChannelContext` as `Channel.channel_context` (immutable thereafter). Stores
   `initial_trust_anchor` as `Channel.trust_anchor` and `initial_trust_anchor_id` as
   `Channel.trust_anchor_id`. If `initial_trust_anchor` is empty, both are empty bytes.
   For Hiero TSS verifiers, the initial trust anchor is the peer's `ledger_id` extracted from the config
   proof. For verifiers with no rotating-authority concept, `initial_trust_anchor` is empty bytes.
5. Stores the returned `ClprEndpointManifest` as `Channel.endpoint_manifest`, truncated to
   `ClprThrottles.max_peer_endpoints` if the endpoint list exceeds that limit (§2.4.2).
   `Channel.endpoint_manifest.version` MUST be ≥ 1 after this step.
6. Transitions the Channel from PENDING to ACTIVE with the revealed Channel ID, verifier
   contract, `channel_context`, `trust_anchor`, and peer config metadata. Removes the commitment.

A `PENDING` Channel cannot be used for messaging — no `sendMessage`, no bundle submission, no syncing.
It exists solely as a claimed reservation.

### 5.1.4 Admin Cleanup

The CLPR Service admin can clean up abandoned `PENDING` Channels by calling `closeChannel`. When called on
a `PENDING` Channel, the Channel is deleted immediately (no drain phase is needed since no messages have
been exchanged). For `ACTIVE` and later states, `closeChannel` follows the normal graceful drain lifecycle.

### 5.1.5 Verifier Immutability

The verifier contract is fixed at registration time (phase 1) and cannot be changed. If the source
ledger upgrades its proof format, a new Channel must be registered with a new verifier. This simplifies the trust
model: applications can evaluate a Channel's verifier once and know it will not change.

## 5.2 Endpoint Discovery

**Local manifest (on-ledger).** Each CLPR Service maintains an on-ledger local endpoint manifest (§2.4.1) as the
authoritative list of endpoints registered on **this** ledger. On permissionless platforms, endpoints are
admitted through the two-step process described in §6.5: any caller registers via `registerEndpoint` (pending
state), and the CLPR Service Admin confirms via `addEndpoint` (live state, appears in `ClprEndpointManifest`).
On platforms where endpoints are derived from the consensus roster (e.g., Hiero), the CLPR Service maintains
this automatically with no manual administration.

**Peer manifest (on-ledger, per Channel).** Each Channel stores an on-ledger `ClprEndpointManifest`
(§2.4.2) as its authoritative directory of remote endpoints to contact for sync. It is populated at
`completeChannel` from the `ClprEndpointManifest` returned by the extended `verifyConfig` call and
refreshed automatically through manifest-update bundle payloads when the remote manifest advances
(§4.2 Step 1b). The current manifest version is propagated in every sync request via
`current_endpoint_manifest_version`, allowing endpoints to detect staleness without a full
state query. Any party may call `getEndpointManifest()` on the remote CLPR Service to retrieve the
current manifest and construct a state proof for bundle submission.

**Reciprocity-based peer selection.** Endpoints SHOULD prefer to sync with peers that reciprocate by providing
messages in return. An endpoint that only requests messages without providing any will be deprioritized by its
peers. This naturally converges on efficient pairings where both sides do useful work.

## 5.3 Administrative Operations

The CLPR Service admin can:

- **Close** a Channel — initiate graceful shutdown. Valid from `PENDING`
  (immediate deletion — no drain needed), `ACTIVE`, `PAUSED`, and `DRAINED`. From `ACTIVE` or `PAUSED`:
  transitions to `CLOSING` (`ChannelSyncData.status` reflects this and is propagated to the peer on next
  sync). All inbound Data Messages from the peer continue to be dispatched normally and generate Response
  Messages until both sides have fully drained. The Channel transitions to `CLOSED` when an inbound bundle
  carries `sync_data.status ∈ {DRAINED, CLOSED}` AND the local queue is empty (§4.2 Step 5b).
  If closed from `PAUSED` via `CLOSING`, the Channel cannot drain until the peer fixes the response
  ordering. From `DRAINED`: transitions directly to `CLOSED` — the admin recovery path when close-notification
  delivery is not possible (see §3.4).
- **Update the local configuration** — change throttles. Changes are propagated to peers via ConfigUpdate
  Control Messages, lazily enqueued on each Channel at its next interaction (see §1.3).

---

# 6. Pseudo-API Reference

The following pseudo-APIs describe the operations that every CLPR Service implementation MUST support. Platform-
specific specifications map these to native constructs. Parameters marked `[auth]` require the caller to authenticate
(platform-specific mechanism — transaction signature, `msg.sender`, etc.).

## 6.1 Configuration Management

```
// Set or update this ledger's local CLPR configuration.
// Authority: CLPR Service admin only.
// Stores the new configuration. The configuration's consensus_timestamp
// naturally serves as the version marker. ConfigUpdate Control Messages
// are lazily enqueued on each Channel the next time it processes a
// bundle or enqueues a message (see ClprConfigUpdate in §1.3).
setLedgerConfiguration(
  [auth] admin,
  configuration: ClprLedgerConfiguration
) → success | error

// Query this ledger's current CLPR configuration.
// Authority: any caller.
getLedgerConfiguration() → ClprLedgerConfiguration
```

## 6.2 Channel Management

```
// Phase 1 (Commit): Register an ownership commitment on this ledger.
// Authority: any caller (permissionless).
// Creates the Channel in PENDING state, keyed by the (eventual, hidden) Channel ID.
// Storage uses the commitment as the lookup key during the commit phase; the actual
// Channel ID is not revealed until completeChannel.
registerChannel(
  [auth] caller,
  ownership_commitment: bytes(32), // keccak256(channel_id || public_key). Neither the
                                   // channel_id nor the public_key is revealed at this stage.
) → success | error

// Phase 2 (Reveal): Complete registration by revealing the Channel ID, public key,
// verifier contract, peer configuration proof, and peer endpoint manifest proof.
// Authority: any caller (permissionless).
// Preconditions: keccak256(channel_id || public_key) matches a stored commitment;
// verifier_contract is deployed on the local ledger.
// The Service:
//   1. Verifies keccak256(channel_id || public_key) matches a stored commitment.
//   2. Verifies the signature over keccak256(channel_id), proving caller controls the keypair.
//   3. Calls verifyConfig(config_proof_bytes, channel_id, endpoint_manifest_proof_bytes) on
//      verifier_contract to obtain the peer's verified configuration and initial endpoint manifest
//      (ctx, chain_id, peer_config_nanos, throttles, initial_trust_anchor,
//      initial_trust_anchor_id, endpoint_manifest).
//   4. Stores ctx as Channel.channel_context (immutable thereafter). Stores
//      initial_trust_anchor as Channel.trust_anchor and initial_trust_anchor_id as
//      Channel.trust_anchor_id. For Hiero TSS verifiers this
//      is the peer's ledger_id. For verifiers with no rotating authority, both are
//      empty bytes. This is the only path other than verifyBundle that may write
//      Channel.trust_anchor.
//   5. Stores the returned ClprEndpointManifest as Channel.endpoint_manifest (§2.4.2).
//      Channel.endpoint_manifest.version MUST be >= 1 after this step.
//   6. Transitions the Channel from PENDING to ACTIVE with the revealed Channel ID,
//      verifier contract, channel_context, trust_anchor, and peer config metadata. Removes the commitment.
// The signature scheme and key encoding are platform-specific. The channel_id ties the
// two sides together — different keys may be used on each ledger.
completeChannel(
  [auth] caller,
  channel_id: bytes(32),       // arbitrary 32-byte identifier, same on both ledgers.
  public_key: bytes,              // public key used to compute the ownership commitment.
  signature: bytes,               // signature over keccak256(channel_id).
  verifier_contract: bytes,       // address of locally deployed verifier (immutable after this call).
  config_proof_bytes: bytes,      // opaque proof from peer ledger; passed to verifyConfig() to
                                  // obtain verified peer configuration (ctx, chain_id,
                                  // throttles, timestamp).
  endpoint_manifest_proof_bytes: bytes, // opaque proof from peer ledger; passed alongside
                                  // config_proof_bytes to the extended verifyConfig() call to
                                  // obtain the verified initial ClprEndpointManifest.
) → success | error

// Close a Channel (graceful drain, then terminal).
// Authority: CLPR Service admin only.
// From PENDING: deletes the Channel immediately (no drain phase needed).
// From ACTIVE or PAUSED: transitions to CLOSING. ChannelSyncData.status reflects
//   this and is propagated to the peer on next sync. All inbound Data Messages
//   continue to be dispatched normally. Transitions to DRAINED when
//   acked_message_id ≥ next_message_id − 1 (§4.2 Step 5b); if the queue is already
//   empty at close time, CLOSING and DRAINED are reached in a single step. Transitions
//   to CLOSED when DRAINED, peer sync_data.status ∈ {DRAINED, CLOSED}, AND
//   acked_message_id ≥ next_message_id − 1 (§4.2 Step 5b). If closed from PAUSED
//   via CLOSING, cannot drain until the peer fixes response ordering.
// From DRAINED: transitions directly to CLOSED. The outbound queue is already fully
//   acknowledged; nothing is abandoned. Use as the recovery path when the remote
//   endpoint cannot deliver its close-notification (see §3.4).
// MUST reject if Channel is CLOSING or CLOSED.
closeChannel(
  [auth] admin,
  channel_id: bytes(32)
) → success | error

// Query a Channel's current outbound queue depth.
// Authority: any caller.
getQueueDepth(
  channel_id: bytes(32)
) → { depth: uint64, max: uint32 } | error
```

## 6.3 Connector Management

Connector registration uses commit-reveal, mirroring the Channel registration scheme. The
commit phase reserves the Connector ID; the reveal phase verifies the operator's keypair and
creates the Connector in active state. The same `(public_key, salt)` produces the same
Connector ID on every ledger, allowing the operator to register the Connector independently
on source and destination without any cross-chain coordination.

```
// Phase 1 (Commit): Register a Connector ownership commitment on this ledger.
// Authority: any caller (permissionless).
// Stores keccak256(connector_id || public_key) under (channel_id, connector_id).
// No Connector state is created yet.
registerConnector(
  [auth] caller,
  channel_id: bytes(32),
  ownership_commitment: bytes(32),  // keccak256(connector_id || public_key)
) → success | error

// Phase 2 (Reveal): Complete Connector registration by revealing key and salt.
// Authority: any caller (permissionless, but requires initial funds).
// Preconditions: a matching commitment exists for (channel_id, derived_connector_id).
// The Service:
//   1. Re-derives connector_id = keccak256(channel_id || public_key || salt).
//   2. Verifies keccak256(connector_id || public_key) matches the stored commitment.
//   3. Verifies signature over keccak256(connector_id || service_address) — proving control
//      of the keypair and binding to this CLPR Service deployment.
//   4. Creates the Connector keyed by (channel_id, connector_id) in active state.
// The caller becomes the Connector admin.
// Platform specs MUST define minimum stake requirements.
completeConnector(
  [auth] caller,
  channel_id: bytes(32),
  public_key: bytes,                // platform-specific encoding
  salt: bytes(32),                  // 32-byte label; bytes32(0) for the default Connector
  signature: bytes,                 // signature over keccak256(connector_id || service_address)
  connector_contract: bytes,        // address of the Connector's authorization contract on this ledger
  initial_balance: uint,            // funds for message execution (native tokens)
  stake: uint                       // stake locked against misbehavior
) → { connector_id: bytes(32) } | error

// Add funds to a Connector's balance.
// Authority: Connector admin only.
topUpConnector(
  [auth] admin,
  channel_id: bytes(32),
  connector_id: bytes(32),
  amount: uint
) → success | error

// Withdraw surplus funds from a Connector's balance (not locked stake).
// Authority: Connector admin only.
withdrawConnectorBalance(
  [auth] admin,
  channel_id: bytes(32),
  connector_id: bytes(32),
  amount: uint
) → success | error

// Remove a Connector and return remaining funds and stake.
// Authority: Connector admin only.
// MUST NOT remove if the Connector has unresolved in-flight messages.
removeConnector(
  [auth] admin,
  channel_id: bytes(32),
  connector_id: bytes(32)
) → success | error

// Query a Connector's current state.
// Authority: any caller.
getConnector(
  channel_id: bytes(32),
  connector_id: bytes(32)
) → Connector | error

// Helper: derive a Connector ID off-chain (or via a view call). Pure function.
deriveConnectorId(
  channel_id: bytes(32),
  public_key: bytes,
  salt: bytes(32)
) → bytes(32)
```

> 💡 **Hiero:** The signature in `completeConnector` is over
> `keccak256(connector_id || 0x000000000000000000000000000000000000016e)` where `0x16e` is the
> EVM address of the CLPR system contract — the Hiero equivalent of `address(this)` on EVM.

## 6.4 Messaging

```
// Send a cross-ledger message via a Connector.
// Authority: any caller.
// Returns the assigned message_id on success, which the caller can use
// to correlate with the eventual Response Message.
sendMessage(
  [auth] caller,
  channel_id: bytes(32),
  connector_id: bytes(32),        // 32-byte derived Connector ID (see §2.2)
  target_application: bytes,      // destination app address
  message_data: bytes             // opaque application payload
) → { message_id: uint64 } | error

// Submit a bundle received from a peer endpoint for on-chain processing.
// Authority: any caller.
// The channel_id identifies which Channel to process against.
// The bundle_payload is the BundleResponse.bundle_payload received during
// sync. Call the verifier and process the bundle contents (§4.2).
submitBundle(
  [auth] caller,
  channel_id: bytes(32),
  bundle_payload: bytes          // BundleResponse.bundle_payload
) → success | error
```

## 6.5 Endpoint Management

On ledgers where endpoint registration is permissionless (e.g., Ethereum, Solana), the CLPR Service MUST expose
the following operations. On ledgers where endpoints are managed by the platform (e.g., Hiero, where consensus
nodes are the endpoints and the manifest is derived automatically from the consensus roster), these operations
are not needed.

All admitted endpoints are expected to serve all Channels on the CLPR Service in good faith — this is a
service-level commitment, not per-Channel. Bond posting and bundle submission are completely orthogonal:
anyone may submit a bundle that makes progress regardless of manifest membership, and the bond plays no role
in submission reimbursement. Endpoint bonds are never slashed; misbehavior results in eviction with full bond
refund. Slashing applies only to Connectors.

```
// Register as an endpoint for this CLPR Service.
// Authority: any caller, including the CLPR Service admin.
// Stores endpoint_data in pending state; does not appear in the live manifest
// until confirmed by the admin. Bond held in escrow; returned in full on
// rejection or cancellation. Manifest version is not incremented on registration.
// Platform specs MUST define the bond delivery mechanism and minimum bond amount.
// MUST be rejected if endpoint_data.tls_certificate exceeds 512 bytes.
registerEndpoint(
  [auth] caller,                  // stored as registrant_account for this entry
  endpoint_data: ClprEndpoint,    // endpoint data (service_endpoint, tls_certificate)
  bond: uint                      // bond posted on registration
) → success | error
```

```
// Admit an endpoint to the live ClprEndpointManifest.
// Authority: CLPR Service admin only.
// If a pending registration exists for registrant_account, the endpoint is admitted
// using its self-registered ClprEndpoint data (endpoint_data is ignored).
// If no pending registration exists, endpoint_data is added directly with no bond recorded.
// Manifest version incremented by 1.
// MUST be rejected if the live manifest is already at max_local_endpoints (when non-zero; zero means no limit).
// For a direct add, MUST be rejected if endpoint_data.tls_certificate exceeds 512 bytes.
addEndpoint(
  [auth] admin,
  registrant_account: bytes,      // identifies the pending entry to approve, or the new registrant for direct add
  endpoint_data: ClprEndpoint     // used only for direct add; ignored if a pending entry exists
) → success | error
```

```
// Remove an endpoint from the live manifest or cancel a pending registration.
// Authority: the endpoint itself or the CLPR Service admin.
// Bond returned in full. Manifest version incremented only if the endpoint
// was live; canceling a pending registration does not increment the version.
removeEndpoint(
  [auth] admin_or_self,
  registrant_account: bytes
) → success | error
```

```
// Atomically apply a batch of additions and removals to the ClprEndpointManifest.
// Authority: CLPR Service admin only.
// All changes are applied atomically. The manifest version is incremented
// exactly once for the entire batch, regardless of batch size.
//
// For each entry in add:
//   - If a pending registration exists for entry.registrant_account: approve it
//     using the endpoint's self-registered data (same logic as addEndpoint).
//   - If no pending registration exists: add entry.endpoint_data directly (no bond).
//   - If registrant_account is already in the live manifest: silently skip (idempotent).
// For each registrant_account in remove:
//   - Evict the endpoint whether pending or live; return held bond in full.
//   - If registrant_account does not exist: silently skip (idempotent).
//
// MUST be rejected if the post-add live count would exceed max_local_endpoints
// (after removing endpoints and skipping already-live entries; zero means no limit).
// MUST be rejected if any added entry's endpoint_data.tls_certificate exceeds 512 bytes.
updateEndpointManifest(
  [auth] admin,
  add: repeated { registrant_account: bytes, endpoint_data: ClprEndpoint },
  remove: repeated bytes          // registrant_accounts to remove
) → success | error
```

```
// Return the current ClprEndpointManifest for this CLPR Service.
// Authority: any caller (public read-only).
getEndpointManifest(
) → ClprEndpointManifest
```

### Application Delivery

When the CLPR Service dispatches a Data Message or delivers a Response Message, it invokes the target application.
The mechanism for this invocation is **platform-specific**: on EVM chains, it is a contract call; on Hiero, it is
a system-level dispatch; on Solana, a CPI. Platform-specific specifications MUST define:

1. The **callback interface** that applications implement to receive messages and responses.
2. The **gas/compute budget** allocated to the application callback.
3. The **return value convention** for indicating success vs. application-level failure.
4. Whether the application callback is **synchronous** (completes within the bundle transaction) or
   **asynchronous** (queued for later execution).
5. How **responses are delivered** to originating senders that are externally owned accounts (not contracts).
   EOA senders cannot receive callbacks; responses SHOULD be recorded in the transaction receipt/record
   but no callback is made.

## 6.6 Misbehavior Detection

Misbehavior detection and enforcement are strictly local (see section 1.6). No cross-ledger misbehavior
reports are exchanged.

---

# 7. Configuration Parameters

|      **Parameter**       | **Default** |          **Scope**          |                                                                                                                            **Description**                                                                                                                             |
|--------------------------|-------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `clprEnabled`            | `false`     | Global                      | Master enable switch. When disabled, all pseudo-API calls MUST return an error. Checked dynamically at call time (not at startup), enabling runtime kill-switch.                                                                                                       |
| `maxQueueDepth`          | TBD         | Per-Ledger (per-Channel) | Maximum unacknowledged messages in the outbound queue before new messages are rejected.                                                                                                                                                                                |
| `maxMessagePayloadBytes` | TBD         | Per-Ledger (per-Channel) | Maximum payload size for a single message. Enforced by source (enqueue) and destination (bundle).                                                                                                                                                                      |
| `maxMessagesPerBundle`   | TBD         | Per-Ledger (per-Channel) | Maximum messages a single bundle may contain. Platform specs MUST set this to ensure bundles fit within the platform's transaction size and execution budget limits.                                                                                                   |
| `maxGasPerMessage`       | TBD         | Per-Ledger (per-Channel) | Maximum gas (or ops budget) allocated to processing a single message.                                                                                                                                                                                                  |
| `maxSyncBytes`           | TBD         | Per-Ledger (per-Channel) | Maximum total size of a serialized `ClprSyncPayload`. Endpoints MUST reject messages exceeding this limit.                                                                                                                                                             |

**Scope note.** Parameters marked "Per-Ledger (per-Channel)" are published in the ledger-wide configuration
(`ClprThrottles`) and advertised to all peers. Each Channel independently enforces them against incoming data.

---

# 8. Security Considerations

## 8.1 Protocol Strictness

All limits are published in the configuration so that both sides know the rules. A peer that exceeds any published
limit is committing a measurable, attributable violation. The receiving side MUST reject the offending submission and
MAY count repeated violations toward local misbehavior thresholds (shunning, disconnection).

Implementations MUST NOT silently ignore unrecognized fields, unknown message types, or malformed metadata.
Unrecognized data MUST be rejected outright.

## 8.2 Trust Model

Using a CLPR Channel means trusting:
1. The **verifier contract** on the Channel — that it correctly validates proofs from the peer ledger.
2. The **peer ledger's consensus mechanism** and proof system.
3. The **local and remote CLPR Service** implementations — that they faithfully execute the protocol.

The trust chain is: the channel registrant chose the verifier, the verifier checks peer proofs, and
the peer's consensus/proof system is assumed trustworthy. Since Channel creation is permissionless and there is
no verifier whitelist, applications must independently evaluate the verifier contract before using a Channel.
A malicious verifier could return fabricated data, compromising the Channel.

## 8.3 Reorg Risk

For ledgers without instant finality (e.g., Ethereum), the commitment level at which the verifier accepts proofs
determines the reorg risk. A verifier at `latest` commitment is vulnerable to chain reorganizations. For high-value
operations, only verifiers enforcing `finalized` commitment (or equivalent) should be used. Ledgers with ABFT
finality do not have this concern.

## 8.4 Endpoint Sybil Resistance

On permissionless ledgers, Sybil resistance is provided primarily by admin curation — the two-step admission
model (§6.5) requires explicit admin approval via `addEndpoint`, making it impossible to flood the manifest
without admin complicity. This mechanism is chain-type agnostic: it applies identically across EVM-based,
UTXO-based, and other permissionless ledger types, because it operates at the CLPR Service admin layer rather
than at the chain's consensus layer. The bond provides commitment friction across all chain types — a signal
of intent, not a threshold calibrated against majority-attack costs on any specific chain. Misbehaving
endpoints are evicted by the admin; there is no behavioral scoring or reputation tracking for endpoints in
the CLPR protocol. Stale manifest-update bundles (those where `new_endpoint_manifest` is empty bytes or not
advancing) are deterred by transaction cost — the submitting account pays the transaction fee and the bundle
makes no manifest progress. Admin curation places full trust in the admin's continued good-faith operation,
including not deliberately emptying the manifest. Peer endpoint selection during sync should incorporate
randomization.

## 8.5 Reentrancy

When dispatching messages to target applications, the CLPR Service hands execution control to arbitrary external
code. Implementations MUST use reentrancy guards on all state-modifying functions and MUST follow the
checks-effects-interactions pattern: update all Channel state before dispatching to the application.

## 8.6 Untrusted Payloads

Both messages and responses carry opaque application-layer payloads. A malicious actor could craft payloads
designed to exploit the receiving application. Applications MUST treat all cross-ledger payloads as untrusted
input. CLPR guarantees authenticity and integrity, not semantic correctness.

## 8.7 No Confidentiality

CLPR provides integrity and authenticity but NOT confidentiality. Payloads are stored on-chain in plaintext.
Applications requiring confidentiality MUST encrypt payloads at the application layer.

## 8.8 Upgradeable Verifier Contracts

Although the verifier address is immutable after channel creation, the verifier contract itself MAY be deployed
behind an upgradeable proxy (e.g., EIP-1967). If the verifier is a proxy, the underlying implementation can change
without CLPR's knowledge. The protocol is agnostic to this — upgradeable verifiers are perfectly valid.
Applications evaluating a Channel should verify whether the verifier is a proxy and, if so, who controls the
upgrade authority. This is an application-level trust decision, not a protocol constraint.

## 8.9 Upgradeable CLPR Service Contracts

On platforms where the CLPR Service is an upgradeable contract, endpoint proof construction depends on the
contract's storage layout. Contract upgrades that change the storage layout MUST be coordinated with endpoint
operators and verifier contract updates. Uncoordinated upgrades will break proof construction and halt all
syncs until endpoints are updated.

## 8.10 Endpoint Pre-Funding

Endpoint operators MUST pre-fund their transaction signing accounts with sufficient native tokens to cover
`submitBundle` transaction fees. Connector margin reimbursement occurs post-consensus and cannot cover the initial
transaction cost. Endpoints that run out of funds cannot submit bundles and effectively go offline.

## 8.11 Queue Monopolization

A single Connector could authorize a large volume of messages to fill the queue to `max_queue_depth`, blocking all
other Connectors on the Channel. This is a denial-of-service vector. Platform-specific specifications SHOULD
define mitigations such as per-Connector queue quotas, escrow requirements at send time, or priority pricing as the
queue fills.

---

# 9. Recovery Scenarios

| #  |                  Scenario                   |      Sync connection        | Recovery Path                                                                                                                                                                                                                                                                                                     |                                  Status                                  |
|----|---------------------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| R1 | Endpoints rotated during partition          | Broken                      | Any party calls `getEndpointManifest()` on the remote chain, constructs a state proof, embeds it in a bundle payload, and calls `submitBundle` as an on-chain transaction. The CLPR Service atomically replaces the Channel's peer manifest. No gRPC connectivity to existing endpoints is required.            | Works                                                                    |
| R2 | Source ledger upgrades proof format         | Breaks when source switches | Register new Channel with new verifier. Applications migrate. Admin closes old Channel.                                                                                                                                                                                                                     | Works (requires new Channel)                                          |
| R3 | Endpoints rotated (proof format unchanged)  | Broken                      | Manifest-update bundle path (as R1); sync resumes once the peer manifest is refreshed.                                                                                                                                                                                                                            | Works                                                                    |
| R4 | Endpoints rotated AND proof format changed  | Broken                      | Register new Channel with new verifier. Manifest-update bundle path on new Channel. Admin closes old Channel.                                                                                                                                                                                            | Works (requires new Channel)                                          |
| R5 | Verifier compromised or broken              | Suspect                     | Admin closes Channel (transitions through CLOSING → DRAINED → CLOSED). Since verifier is immutable, register new Channel with correct verifier.                                                                                                                                                             | Works (data loss on close)                                               |
| R6 | Queue state permanently corrupted on peer   | Working                     | Channel pauses (§4.5). Auto-resumes if peer fixes ordering. Admin may close a PAUSED Channel (`closeChannel` transitions to CLOSING); the Channel drains once the peer fixes the ordering. If the peer never fixes it, the Channel stays PAUSED (or CLOSING, if the admin closed it) indefinitely. | Works (PAUSED until peer recovers; admin can close to prepare for drain) |
| R7 | Network partition (endpoints unchanged)     | Temporarily broken          | Syncs resume automatically. Monotonic IDs and running hash verify integrity.                                                                                                                                                                                                                                      | Works                                                                    |
| R8 | Peer ledger down entirely                   | Broken                      | Messages queue up to `max_queue_depth`, then backpressure. Syncs resume when peer returns.                                                                                                                                                                                                                        | Works                                                                    |
| R9 | Both sides' endpoints change simultaneously | Broken on both sides        | Any party on either side may submit a manifest-update bundle directly as an on-chain transaction, breaking the deadlock without gRPC connectivity to old endpoints.                                                                                                                                               | Works                                                                    |

