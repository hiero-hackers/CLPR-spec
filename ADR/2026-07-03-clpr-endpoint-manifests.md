# CLPR Endpoint Manifests

This document proposes a change to how CLPR manages and communicates the set of endpoints that remote ledgers should contact when initiating sync cycles. It describes the problem with the current approach, the proposed solution, and enumerates every change required to the existing specification files.

---

## 1. Problem

Before a CLPR Endpoint can initiate a sync cycle for a given Channel, it must know the network addresses of endpoints on the remote ledger — specifically which gRPC hosts to contact to exchange bundle proofs. This per-Channel collection of remote endpoint addresses is what this proposal calls the **endpoint manifest**.

In the current specification, the remote ledger's endpoint list is embedded as a field in `ClprLedgerConfiguration` — the same configuration object that carries throttle limits, protocol version, and the ledger's chain identifier. When the remote ledger's CLPR Service Admin updates the endpoint list, the change is bundled into a `ConfigUpdate` Control Message that is lazily enqueued on each active Channel's outbound message queue. The update travels to the peer through the normal bundle sync protocol.

This design has a structural flaw that becomes a hard deadlock when the remote ledger undergoes complete endpoint turnover — that is, when every previously registered endpoint is replaced at once. The sequence of failure is straightforward: the `ConfigUpdate` carrying the new endpoint list enters the outbound queue and waits to be delivered during a future sync cycle. Meanwhile, the local CLPR Endpoints attempt to initiate a sync but find no valid gRPC addresses in their peer manifest — all the old endpoints are gone. Without a reachable peer, no sync can occur, and without a sync, the `ConfigUpdate` cannot be delivered. The update is unreachable precisely because the information needed to reach the remote ledger is contained within the update itself. The Channel stalls indefinitely with no automated recovery path.

It is worth noting that partial endpoint turnover — where some old endpoints remain reachable — does not produce this deadlock. As long as at least one surviving endpoint can carry a sync, the `ConfigUpdate` will eventually arrive and the manifest will be refreshed. The deadlock is specific to complete replacement of the remote endpoint set.

The severity of this problem is sharpened by two characteristics of the CLPR deployment environment. First, the protocol supports two fundamentally different endpoint operator profiles. On Hiero networks, every consensus node is automatically a CLPR Endpoint and is expected to service all Channels on the native CLPR Service without any manual configuration — these operators want the system to manage itself. On other ledger types such as Ethereum or Besu-based networks, CLPR Endpoints are independent of the validator set and are run as separately operated infrastructure, often by parties seeking to earn reimbursement from Connector margin. These independent operators may selectively choose which Channels to participate in. Both operator profiles depend on automated, reliable endpoint discovery: the Hiero consensus node operator because manual manifest management is contrary to their operating model, and the independent operator because they need to find and connect to remote endpoints for the Channels they choose to service. Neither profile has a clean fallback when the endpoint manifest becomes stale and undeliverable through the queue mechanism.

Second, the CLPR protocol permits anyone to submit sync bundles directly to a ledger's CLPR Service as an on-ledger transaction, without being a formally registered endpoint. This permissionless bundle submission was a deliberate design relaxation intended to support free-agent participants and reduce barriers to bundle delivery. However, it does not eliminate the endpoint discovery problem. Even a free-agent bundle submitter must know the gRPC addresses of remote endpoints to participate in the bidirectional sync exchange — the `sync` RPC requires a live gRPC connection to a remote peer. The discovery problem persists regardless of whether bundle submission itself is gated.

These considerations lead to four requirements that any solution must satisfy. First, for each active Channel, the local CLPR Endpoints must have access to a current list of remote endpoint addresses they can contact via gRPC. Second, this list must be updated when remote endpoints change, and the update mechanism must not rely on the existing peer endpoint list being reachable. Third, for newly created Channels, the initial list must be available at the time the Channel is established without requiring manual configuration by the channel creator beyond what they already do to set up the Channel. Fourth, ongoing updates must propagate automatically with as little manual intervention as possible, matching the automation expectations of Hiero consensus node operators while accommodating the selective participation model of independent operators.

---

## 2. Options Considered

Reaching the proposed solution required working through several candidate approaches, each of which addressed some aspects of the requirements but failed on others. This section presents those candidates in the order they were evaluated, identifying the specific failing that prompted the move to the next alternative.

### 2.1 Ad-Hoc Endpoint Self-Advertisement

The simplest conceivable approach is to allow any CLPR Endpoint to announce its own availability directly. An endpoint wanting to receive sync bundles for a given Channel would contact the remote Channel and register itself — providing its gRPC address, its ledger account identifier, and its willingness to serve. No admin curation or prior arrangement would be required. The remote Channel's peer manifest would be populated organically from whoever chose to participate.

This approach fails because it has no trust mechanism. A party wishing to disrupt a Channel could flood the peer manifest with entries pointing to endpoints that are unresponsive, unreliable, or actively adversarial. The receiving side has no way to distinguish legitimate participants from noise without some form of prior vetting or accountability. With a polluted manifest, a legitimate CLPR Endpoint may exhaust its retry budget against bad entries before reaching a functioning peer, degrading or halting progress on the Channel. Without a bond, a stake, or any form of accountability for entries in the manifest, the manifest cannot be trusted by sync participants.

The failure points to a necessary property: the source of the endpoint list must carry some form of accountability that makes adding bad entries costly or impossible without authorization.

### 2.2 Off-Chain Endpoint Registries

The next candidate externalizes accountability to off-chain registry operators. CLPR Endpoints would consult a designated off-chain registry service that maintains a curated list of endpoints per network and CLPR Service. The registry operator is accountable for the quality of its listings. A standardized format and protocol would allow any CLPR Endpoint implementation to query any compliant registry. Multiple registries could exist, operated by different parties, providing redundancy against any single registry going offline.

This approach eliminates the accountability gap from the previous option but introduces its own failures. A single registry is a single point of failure — if it becomes unavailable, every CLPR Endpoint that depends on it loses the ability to discover peers. Multiple registries reduce this availability risk but create a new coordination problem: every CLPR Service Admin across every participating network must push updates to every registry whenever their endpoint set changes. With N registries and M CLPR Services, every manifest update triggers N update operations, and the registries must remain synchronized with each other. In practice this means either a hub-and-spoke architecture (which reintroduces a single point of failure) or a complex multi-party synchronization protocol between the registries themselves.

There is also a trust question that the protocol cannot resolve. Registries are off-chain entities with no protocol-level accountability — a registry that returns stale or fabricated endpoint lists cannot be detected or penalized by the CLPR protocol. Participants must trust registry operators entirely through out-of-band means.

The failure points to a different necessary property: the authoritative source for the endpoint list should be the CLPR Service itself, not an external party, so that the same trust and provability guarantees that apply to other CLPR state apply equally to the endpoint list.

### 2.3 Each CLPR Service as Its Own Registry

The insight from evaluating off-chain registries is that if the registry format is standardized and every CLPR Endpoint can query any compliant registry, there is no fundamental reason the registries need to be operated by external parties. Each CLPR Service could function as the authoritative registry for its own endpoint set. Channel creators and CLPR Endpoints would query the remote CLPR Service directly, eliminating the trusted-third-party problem and the N×M synchronization overhead in a single step.

This approach requires a way to address the remote CLPR Service for the query. The two options are a URL pointing to some hosted API, or a protocol-defined query interface built into each CLPR Service. A URL approach requires someone to own and maintain a server that answers queries — this is hosted infrastructure that must be deployed, kept available, and kept synchronized with actual CLPR Service state. It introduces operational obligations that did not previously exist and creates availability risk similar to the off-chain registry problem.

A protocol-defined query API built into the CLPR Service itself seems cleaner — but it requires specifying a list of server addresses that answer the API. Maintaining that list of server addresses is precisely the problem being solved. To find the endpoints for a Channel, a CLPR Endpoint needs to know where to send the query; to know where to send the query, it needs a list of addresses; to get that list of addresses, it needs to ask somewhere. The recursion has no base case outside of the CLPR Service's own on-ledger state, which does not require any prior address to query — ledger state is already accessible through the ledger's own node network.

The failure points to a final necessary property: the endpoint manifest must be accessible directly from the ledger's own provable state, readable through the same mechanisms used to access any other on-chain data, without depending on any off-chain or separately addressed service.

### 2.4 Endpoint Contract Specified at Channel Creation

With the conclusion that the manifest must live in on-ledger state, a natural question is whether it should be an intrinsic part of the CLPR Service or a separately deployed contract referenced by each Channel, analogous to how verifier contracts work. The channel creator would designate an endpoint contract at `completeChannel` time — a locally deployed smart contract that holds the current manifest for the remote CLPR Service. The verifier contract already understands the remote ledger's proof system; the endpoint contract would be a second locally deployed contract serving a parallel purpose.

This design has a genuine advantage for a specific topology: when multiple Channels from the same local ledger all communicate with the same remote CLPR Service — for example, three Channels to an Ethereum CLPR Service using different commitment levels — all three could reference the same endpoint contract. A manifest update on the remote CLPR Service would require only one update to the shared local endpoint contract, rather than one update flowing through each Channel's message queue independently.

However, the design has two significant problems. First, like verifier contracts, the endpoint contract address would be fixed at channel creation time. If the contract itself is immutable, a substantial change to the remote endpoint set — such as a complete manifest replacement — would require deploying a new endpoint contract and registering a new Channel with it, since an existing Channel cannot reference a different endpoint contract. If the contract is upgradeable through a proxy pattern, the upgrade key holder can alter the endpoint list without the CLPR protocol's knowledge, introducing the same proxy trust concerns that apply to upgradeable verifier contracts.

Second, the advantage of collapsed updates only materializes when multiple Channels share the same remote CLPR Service — a topology that arises only when an application deliberately uses the same CLPR Service at multiple commitment levels. For the typical case of a single Channel between two CLPR Services, the endpoint contract approach adds a new contract type, additional trust evaluation burden at channel creation, and an immutability constraint, with no offsetting benefit.

The immutability problem and the narrowness of the topology benefit prompted a return to the simpler approach: the endpoint manifest as an intrinsic, mutable piece of the CLPR Service's own state.

### 2.5 Endpoint Manifest in CLPR Service State — and the Remaining Update Problem

With the manifest established as intrinsic CLPR Service state, the design aligns with the current specification, which already embeds the remote ledger's endpoint list in `ClprLedgerConfiguration` and stores it on the Channel as the peer manifest. However, as the problem description establishes, the current mechanism for propagating updates — `ConfigUpdate` Control Messages enqueued in the Channel's outbound message queue — creates a deadlock when the remote endpoint set undergoes complete turnover. The update is buried in the queue, and delivering it requires a successful sync cycle with a reachable peer endpoint, which is exactly what becomes unavailable when all known peers have changed.

The resolution is to apply the same pattern used for trust anchor rotations: embed manifest updates directly in the sync bundle payload rather than as messages in the queue. When a CLPR Endpoint detects that the remote manifest has advanced, it includes a state proof of the new manifest in the next bundle payload it constructs. The verifier contract extracts and verifies the new manifest as part of `verifyBundle`, and the CLPR Service atomically replaces the Channel's peer manifest before processing any message in the bundle — no queue delivery required. Because bundle submission is permissionless, a new remote endpoint that is not yet in any peer's cached manifest can construct and submit a valid bundle directly to the local ledger, breaking the deadlock without requiring any prior gRPC connectivity to old endpoints.

This approach also motivates moving the endpoint list out of `ClprLedgerConfiguration` into a dedicated `ClprEndpointManifest` structure in CLPR Service state. Separating it from the ledger configuration allows the manifest to have its own version counter, to be state-proven independently of the configuration, and to be updated without triggering unrelated configuration propagation logic. The full mechanism is described in the following section.

---

## 3. Proposed Solution

### The Endpoint Manifest as Distinct CLPR Service State

This proposal introduces the `ClprEndpointManifest` — a new first-class data structure maintained by the CLPR Service as a distinct piece of on-ledger state, independent of `ClprLedgerConfiguration`. A `ClprEndpointManifest` holds the ordered list of `ClprEndpoint` entries representing the current active endpoints for this ledger's CLPR Service, along with a monotonically increasing version counter that is incremented each time the endpoint set changes, and a `service_address` field identifying the specific CLPR Service instance this manifest belongs to. The version counter starts at 1 when the manifest is first created and has no correlation to any other version numbers in the system. The `service_address` field carries the on-ledger address of the CLPR Service and is included in the state-proven manifest content itself. A verifier MUST reject a manifest proof whose `service_address` does not match the peer's service address in the Channel's `channel_context`. The trust anchor provides implicit chain association (the manifest proof is verified against the chain's trust anchor); `service_address` provides binding to the specific CLPR Service instance on that chain, preventing a malicious channel registrant from supplying a proof for a different CLPR Service's manifest on the same chain. Because the manifest lives in CLPR Service state rather than being embedded in the ledger configuration, it can be updated independently of other configuration parameters, and state proofs over it can be constructed using the same mechanisms used to prove any other ledger state — Merkle paths on Hiero, storage proofs on Ethereum, and so on.

### Manifest Curation and Administration

The process by which the manifest is maintained differs by ledger type. On Hiero networks, the CLPR Service automatically derives the endpoint manifest from the active consensus node roster. Only changes to the composition of the node set trigger a manifest update and version increment — specifically, a node joining, leaving, or rotating its network identity. Roster changes that affect only consensus weight without altering the set of nodes do not change the endpoint manifest and do not increment the version counter. When CLPR is first enabled on a Hiero network, the initial manifest is seeded from the current roster at version 1, regardless of how many prior roster changes have occurred in the network's history. The Hiero manifest is exclusively derived from the consensus roster; no other endpoints are admitted, and no manual administration is required or permitted.

On other ledger types, admission to the manifest is a two-step process requiring both endpoint initiative and admin confirmation.

**Step 1 — Endpoint self-registration (`registerEndpoint`).** An endpoint wishing to be listed calls `registerEndpoint`, providing its `ClprEndpoint` data and posting a bond. Registration is at the service level — there is no per-Channel scoping. The bond may be zero if the CLPR Service does not require one; the CLPR Service MAY reject the registration if the posted amount does not meet its minimum requirement. The bond is held in escrow by the CLPR Service. The endpoint enters a *pending* state: the bond is held, but the endpoint is not yet part of the manifest and is not expected to serve any Channels.

**Step 2 — Admin confirmation (`addEndpoint`).** The CLPR Service Admin reviews pending registrations and calls `addEndpoint` for each endpoint it chooses to admit. This atomically moves the endpoint from pending into the live manifest and increments the version counter. The admin is the sole authority for manifest admission.

**Bond semantics.** The bond is held for as long as the endpoint remains in the pending or live manifest state. The bond is always returned in full when an endpoint exits — whether by self-removal or admin action. There is no slashing of endpoint bonds. The CLPR Service is expected to vet endpoints before admitting them; misbehavior results in eviction (removal from the manifest with full bond refund), not bond forfeiture. Slashing applies only to Connectors.

**Bundle submission and bonds are orthogonal.** The bond and the act of submitting bundles have no relationship. Anyone — including endpoints not on the manifest — may submit a bundle that makes progress to the CLPR Service. The manifest exists solely to advertise which endpoints are available and willing to participate in sync cycles. Endpoints pay for their own transaction submissions from their own accounts and are reimbursed to those accounts; the bond plays no role in this.

**Exits.** An endpoint may remove itself from the live manifest at any time by calling `removeEndpoint`; its bond is returned in full and the version counter is incremented. An endpoint that has registered but not yet been confirmed by the admin may cancel its pending registration by calling `removeEndpoint`; in this case the bond is returned in full and the version counter is not incremented (the manifest was never changed). The admin may remove an endpoint from the live manifest or reject a pending registration at any time; in both cases the bond is returned in full and the version counter is incremented only if the endpoint was in the live manifest.

Once admitted to the manifest, an endpoint is expected to serve all Channels on the CLPR Service in good faith — not only to receive inbound syncs but to proactively send bundles for every active Channel. This is a service-level commitment, not a per-Channel one. An endpoint that selectively participates or fails to send bundles in good faith may be evicted by the admin.

**Empty manifest.** An empty manifest — version ≥ 1, endpoints list empty — is valid. This allows a network to indicate it is not advertising its endpoints; for example, a private or hidden network may choose not to publish endpoint addresses in the on-ledger manifest. A network with an empty manifest must have its own endpoints proactively drive syncing on all Channels, since remote peers cannot initiate gRPC sync without endpoint addresses. An empty manifest is distinct from a manifest that has not yet been created: once created at version 1, the manifest is always present, though it may be empty. `completeChannel` may succeed when the remote CLPR Service has an empty manifest — the Channel is validly established with an empty peer manifest and sync will begin once at least one endpoint is admitted on both sides. The `verifyConfig` call MUST NOT revert solely because the manifest proof contains an empty endpoint list.

This arrangement allows the two operator profiles to coexist without conflict. Hiero consensus node operators receive fully automated manifest management with no manual intervention. Independent endpoint operators on permissionless networks operate within a curated environment where the CLPR Service Admin controls admission and eviction. The expectation of serving all Channels is the same in both cases.

### Querying the Manifest and Constructing State Proofs

The current endpoint manifest for any ledger is publicly readable. Any party — a channel registrant, a CLPR Endpoint seeking peers, an application developer auditing infrastructure, or a monitoring tool — can call `getEndpointManifest()` on the CLPR Service to retrieve the current `ClprEndpointManifest`. On ledgers where CLPR Service state is part of a Merkle state tree (e.g., Hiero), a state proof is constructable directly from the query result using the ledger's existing proof infrastructure. On other ledger types (e.g., Ethereum), the caller constructs the state proof separately using the platform's native mechanism — for example, `eth_getProof` against the CLPR Service contract's storage slots. In both cases, the proof establishes that the manifest content was committed to the ledger's state at a specific point in time and has not been tampered with.

### Channel Creation and Initial Peer Manifest

When establishing a new Channel, the channel creator must supply the remote ledger's current endpoint manifest so the local CLPR Service can populate the Channel's initial peer manifest. The creator calls `getEndpointManifest()` on the remote ledger to retrieve the current manifest, constructs a state proof of that manifest using the remote ledger's proof mechanism, and provides the resulting proof bytes as `endpoint_manifest_proof_bytes` to the `completeChannel` call on the local ledger.

The `verifyConfig` method on the verifier contract is extended to accept a second argument, `endpoint_manifest_proof_bytes: bytes`, alongside the existing `config_proof_bytes`. When called, `verifyConfig` verifies the manifest proof and returns the existing configuration fields plus the initial `ClprEndpointManifest` at version 1 or greater. The version number is part of the manifest data format — it is state-proven content read directly from the manifest, not external metadata. The `endpoint_manifest_proof_bytes` must be verifiable against the initial trust anchor — the same trust anchor that `verifyConfig` establishes for the Channel. The CLPR Service stores the returned manifest as the Channel's initial peer manifest, recording both the manifest contents and its version counter.

**Staleness window.** There is an inherent window between the moment the channel creator obtains the manifest proof and the moment `completeChannel` is processed on-chain, during which the remote manifest may have advanced. This is acceptable: the initial peer manifest is a snapshot that does not need to be current at creation time. As long as at least one endpoint in the initial manifest is reachable, it will detect the version mismatch via `endpoint_manifest_version` in `ClprQueueMetadata` on the first sync and automatically propagate the update through a bundle payload. No freshness check at `completeChannel` time is required or appropriate.

### Propagating Updates Through Bundle Payloads

The central change in this proposal is that endpoint manifest updates travel through the sync bundle payload rather than through the Channel's outbound message queue. The reason follows directly from the deadlock analysis above: any update mechanism that requires a successful sync with a known peer endpoint cannot be the sole recovery mechanism for complete endpoint turnover, because complete turnover leaves no known peer endpoints to sync with.

The CLPR protocol already handles an analogous problem for trust anchor rotations. When the remote ledger's signing authority changes, a transition bundle embeds the rotation evidence directly in the bundle payload — not as a queued message — and the verifier contract's `verifyBundle` method returns the new trust anchor, which the CLPR Service stores atomically before processing any message in the bundle. The same pattern is applied here.

When a CLPR Endpoint detects that the remote ledger's endpoint manifest has advanced — by observing a new version number in the remote chain's state — it embeds a state proof of the updated manifest in the next bundle payload it constructs. The verifier contract's `verifyBundle` method is extended to also return an optional `new_endpoint_manifest` when such a proof is present in the bundle payload. When no manifest proof is present in the bundle, `verifyBundle` returns no manifest (absent/null). If `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, the CLPR Service atomically replaces the Channel's peer manifest and version counter as part of accepting the bundle. The updated manifest is immediately available for subsequent sync cycles on that Channel.

**Ordering of manifest update and trust anchor update.** Everything in the bundle is verified against the current trust anchor in a single `verifyBundle` call. The manifest update (Step 1b) is applied before the trust anchor update (Step 1c).

**Silent skip for absent or stale manifest.** When `new_endpoint_manifest` is absent, or when it is present but `new_endpoint_manifest.version ≤ Channel.endpoint_manifest_version`, the manifest content is silently skipped — the bundle is not rejected for this reason alone. Bundle Progress Criterion 5 is satisfied only when `new_endpoint_manifest` is present and `new_endpoint_manifest.version` is strictly greater than the stored `Channel.endpoint_manifest_version`. If after skipping the manifest content the bundle satisfies no other progress criterion, the existing NoProgress rejection applies normally.

**Channel state and manifest updates.** Manifest-update bundles submitted against PENDING Channels are rejected, consistent with the existing rule that PENDING Channels reject all bundles. CLOSING and DRAINED Channels continue to accept bundles — including manifest updates, trust anchor updates, and queue metadata — for the purpose of maintaining liveness until both sides reach CLOSED. A manifest-update-only bundle satisfying only Criterion 5 is accepted on CLOSING and DRAINED Channels. The updated peer manifest on a DRAINED Channel cannot be used for resumed sync on that Channel (DRAINED is a terminal state on the path to CLOSED), but updating it is harmless and allows the manifest to remain current for administrative purposes.

**Entire manifest replacement.** When a manifest update is applied, the entire manifest is replaced in full. The version number is the sole criterion for whether a manifest update is applied — if `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, the CLPR Service replaces `Channel.endpoint_manifest` with the full returned manifest, including any truncation applied by `max_peer_endpoints`. Partial updates are not defined. When the remote manifest shrinks below the current cached subset, the cached manifest is simply replaced with the smaller manifest — no prior entries are retained.

Because bundle submission is permissionless, manifest recovery does not require the existing peer manifest to be reachable. A new endpoint that is not yet in the local peer manifest can construct a valid bundle — it knows the remote chain's current state, including the updated manifest — and submit it directly as a transaction to the local ledger. The local CLPR Service processes the bundle, the verifier extracts and returns the new manifest, and the CLPR Service updates the peer manifest. No prior gRPC connectivity to old endpoints is required to initiate this recovery.

**Manual recovery mode.** If the entire peer manifest for a Channel is stale — every cached address is unreachable — any party may recover the Channel manually without requiring gRPC connectivity to any existing endpoint and without special authority. The procedure is: (1) call `getEndpointManifest()` on the remote chain to retrieve the current manifest; (2) construct a state proof of that manifest using the remote chain's native proof mechanism; (3) embed the proof in a bundle payload and call `submitBundle` directly as an on-chain transaction on the local ledger. The CLPR Service calls `verifyBundle` on the verifier contract, which returns the verified manifest; the CLPR Service atomically replaces the Channel's peer manifest. This recovery path requires only read access to the remote chain's state — no relationship with any existing endpoint is needed. Note that on a DRAINED Channel, updating the peer manifest via this path does not resume normal sync — DRAINED is a terminal state — but the manifest update still occurs and may be useful for administrative purposes.

### Version Propagation for Staleness Detection

To allow CLPR Endpoints to detect endpoint manifest staleness efficiently — without performing a full state query on the remote chain before every sync — the current `endpoint_manifest_version` of the local ledger's manifest is included in `ClprQueueMetadata` in every sync payload. When an endpoint receives a sync from a remote peer, it reads the `endpoint_manifest_version` field and compares it against the version stored in the Channel's peer manifest. If the versions differ, the endpoint knows the remote manifest has advanced and should embed a manifest state proof in its next outbound bundle. This provides automatic, lightweight detection without requiring out-of-band communication.

### Endpoint Manifest Advancement as a Bundle Progress Criterion

The CLPR protocol rejects any submitted bundle that makes no progress — that is, a bundle that does not advance any of the four defined Bundle Progress Criteria (new messages, trust anchor advancement, acknowledgement progress, or Channel state transition). A bundle carrying only an endpoint manifest update but no application messages is valid and necessary for manifest recovery on otherwise idle Channels. To prevent such bundles from being rejected, endpoint manifest advancement is added as a fifth Bundle Progress Criterion: a bundle satisfies this criterion when `new_endpoint_manifest.version` returned by the verifier is greater than the `endpoint_manifest_version` currently stored on the Channel.

### Bundle Size Interplay

The endpoint manifest proof contributes to the total bundle payload size alongside other variable components — application messages, trust anchor rotation proofs, and bundle overhead. All of these components are optional within a single bundle; a bundle carrying only an endpoint manifest update with no application messages is valid under Bundle Progress Criterion 5. The precise sizing equation accounting for all optional components is deferred to integration time.

If the manifest proof is too large to fit within a peer's `max_sync_bytes`, manifest updates cannot be delivered to that peer and the Channel's peer manifest will not refresh. This is recoverable: the peer's admin can increase `max_sync_bytes` via `setLedgerConfiguration`, which propagates as a `ConfigUpdate` Control Message — a small payload that fits within any bundle satisfying the existing `max_sync_bytes` minimum constraint. Alternatively, the local admin can reduce `max_local_endpoints` and remove endpoints until the manifest proof fits. Once updated throttles arrive via ConfigUpdate, endpoints can deliver manifest-carrying bundles within the new limit. Operators should size `max_local_endpoints` with awareness of the manifest proof size and the `max_sync_bytes` advertised by their expected peer networks.

### Supplemental Discovery Protocols

The on-ledger manifest and `getEndpointManifest()` are the authoritative and sufficient mechanism for endpoint discovery within the CLPR protocol. Supplemental off-chain discovery protocols are possible and may be developed independently of this specification. Because bundle submission is permissionless — anyone may submit a bundle that makes progress to the CLPR Service regardless of whether they appear in the manifest — supplemental registries may list any endpoint willing to participate without conflicting with the on-ledger manifest. Such protocols are out of scope for this specification.

---
---

## 4. Code Artifacts

The following are the normative code blocks to be inserted or replaced in the spec verbatim. Each block is self-contained and corresponds to one change table entry in Section 5.

---

### 4.1 `ClprLedgerConfiguration` — remove `endpoints` field (`clpr-service-spec.md` §1.1)

Replace the existing `ClprLedgerConfiguration` message with:

```protobuf
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

  // Field 6 (endpoints) removed. The endpoint list is now maintained as a
  // separate ClprEndpointManifest in CLPR Service state (see §2.4.2, §6.5).

  // Initial trust anchor for this ledger. For ledgers where the signing authority
  // is a stable property of the ledger configuration (e.g., Hiero TSS ledger_id),
  // this field MUST be populated by the source ledger. verifyConfig() returns it
  // directly as initial_trust_anchor. For ledgers where the signing authority is
  // state external to the configuration (e.g., Ethereum sync committees), this
  // field is empty and the verifier derives the trust anchor from a separate state
  // proof embedded in the same proof_bytes.
  bytes initial_trust_anchor = 7;

  // Opaque identifier for initial_trust_anchor. MUST be non-empty iff
  // initial_trust_anchor is non-empty. Format is verifier-defined.
  bytes initial_trust_anchor_id = 8;
}
```

---

### 4.2 `ClprThrottles` — update fields 8 and 9 comments (`clpr-service-spec.md` §1.1)

Update the comments on `max_local_endpoints` and `max_peer_endpoints`:

```protobuf
  // Maximum number of endpoints that may appear in the live ClprEndpointManifest
  // for this CLPR Service. Excess registrations are rejected.
  uint32 max_local_endpoints = 8;

  // Maximum number of peer endpoints this ledger will cache per Channel
  // from the remote ClprEndpointManifest. Zero means no limit.
  uint32 max_peer_endpoints = 9;
```

---

### 4.3 `ClprEndpointManifest` — new message (`clpr-service-spec.md` §1.2, after `ClprEndpoint`)

Insert this message after the closing `}` of `ClprEndpoint`:

```protobuf
// The authoritative, on-ledger endpoint set for this CLPR Service.
// Maintained as distinct CLPR Service state, separate from ClprLedgerConfiguration.
// Service-scoped: all admitted endpoints serve all Channels on this CLPR Service.
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

---

### 4.4 `ClprEndpoint` — update message comment (`clpr-service-spec.md` §1.2)

Replace the message-level comment on `ClprEndpoint` with:

```protobuf
// An endpoint participating in CLPR syncs for this CLPR Service.
// Used in ClprEndpointManifest.endpoints — the service-level on-ledger endpoint
// manifest (see §2.4.2, §6.5).
```

The fields of `ClprEndpoint` are unchanged.

---

### 4.5 `ClprQueueMetadata` — add field 7 (`clpr-service-spec.md` §1.5)

Add after the `trust_anchor_id = 6` field:

```protobuf
  // The sender's current Channel.endpoint_manifest_version.
  // Allows the receiver to detect whether the remote endpoint manifest has
  // advanced since the last sync. Zero indicates the Channel is PENDING.
  uint64 endpoint_manifest_version = 7;
```

---

### 4.6 `ClprEndpointService` gRPC — remove `discoverEndpoints` (`clpr-service-spec.md` §1.5)

Replace the `ClprEndpointService` service definition and the `DiscoverEndpointsRequest` / `DiscoverEndpointsResponse` message types with:

```protobuf
service ClprEndpointService {
  // Bidirectional sync: exchange pre-computed bundle payloads with a peer endpoint.
  rpc sync(ClprSyncPayload) returns (ClprSyncPayload);

  // discoverEndpoints removed. See §2.4.2, §6.5.
}

// DiscoverEndpointsRequest and DiscoverEndpointsResponse removed.
```

---

### 4.7 `Channel` struct — add peer manifest fields (`clpr-service-spec.md` §2.1)

Add the following two fields to the Channel struct, after `trust_anchor_id` and before `status`:

```
  // --- Peer Endpoint Manifest ---
  endpoint_manifest         : ClprEndpointManifest  // Cached peer endpoint manifest.
  endpoint_manifest_version : uint64                // Version of the cached manifest.
                                                    // 0 = uninitialized (PENDING);
                                                    // >= 1 after completeChannel.
```

The **Initial values** paragraph gains:

> `endpoint_manifest_version` = 0 (uninitialized sentinel; set to the manifest version returned by `verifyConfig` when `completeChannel` completes — MUST be ≥ 1 thereafter). `endpoint_manifest` = empty (populated at the same time).

Replace the **Peer manifest** paragraph with:

> **Peer manifest** is stored as `Channel.endpoint_manifest` and `Channel.endpoint_manifest_version` (see §2.4.2). Initialized at `completeChannel` from the `ClprEndpointManifest` returned by the extended `verifyConfig(config_proof_bytes, channel_id, endpoint_manifest_proof_bytes)` call. Refreshed automatically through manifest-update bundle payloads (§4.2 Step 1b). The `PeerEndpointRosterEntry` concept is superseded.

---

### 4.8 `IClprVerifier.verifyConfig` — extended signature (`clpr-service-spec.md` §3.1)

Replace the existing `verifyConfig` function block with:

```
  // Verify the remote ledger's configuration proof and endpoint manifest proof.
  //
  //   config_proof_bytes            : bytes     — proof of the remote ClprLedgerConfiguration.
  //   channel_id                 : bytes(32) — the Channel identifier. Echoed verbatim
  //                                               in the returned ctx.
  //   endpoint_manifest_proof_bytes : bytes     — proof of the remote ClprEndpointManifest.
  //                                               MUST be verifiable against the initial trust
  //                                               anchor established by this call.
  //
  //   ctx                   : ChannelContext — constant channel-scoped data: channel_id
  //                                   (echoed from input) and service_address (peer CLPR Service
  //                                   address). Stored as Channel.channel_context; passed
  //                                   to every verifyBundle call.
  //   chain_id              : string — CAIP-2 identifier of the remote ledger.
  //   peer_config_nanos     : uint96 — consensus timestamp of the verified config,
  //                                   as nanoseconds since Unix epoch.
  //   throttles             : ClprThrottles — capacity limits of the remote ledger.
  //   initial_trust_anchor  : bytes  — initial signing authority; empty if the verifier
  //                                   has no rotating-authority concept.
  //   initial_trust_anchor_id : bytes — opaque identifier for initial_trust_anchor.
  //                                   MUST be non-empty iff initial_trust_anchor is
  //                                   non-empty. Format is verifier-defined.
  //   endpoint_manifest     : ClprEndpointManifest — verified manifest at version >= 1.
  //                                   MUST NOT revert solely because endpoints is empty.
  //
  // MUST revert if:
  //   - any proof verification fails
  //   - manifest.service_address does not match ctx.service_address
  //   - manifest.version is 0
  function verifyConfig(bytes config_proof_bytes, bytes channel_id, bytes endpoint_manifest_proof_bytes)
    returns (ChannelContext ctx, string chain_id, uint96 peer_config_nanos,
             ClprThrottles throttles, bytes initial_trust_anchor,
             bytes initial_trust_anchor_id, ClprEndpointManifest endpoint_manifest)
```

---

### 4.9 `IClprVerifier.verifyBundle` — extended return values (`clpr-service-spec.md` §3.1)

Replace the existing `verifyBundle` function block with:

```
  // Verify a bundle payload against the Channel's current trust anchor.
  // Returns verified queue metadata, an ordered array of message payloads,
  // a successor trust anchor, and an optional updated endpoint manifest.
  //
  //   bundle_payload   : bytes             — state proof and message data (format is verifier-specific).
  //   trust_anchor     : bytes             — the Channel's current signing-authority material,
  //                                         as stored by the CLPR Service. Empty for verifiers with
  //                                         no rotating-authority concept.
  //   ctx              : ChannelContext — constant channel-scoped data (channel_id,
  //                                         service_address). Same value stored by completeChannel;
  //                                         never mutated for the lifetime of the Channel.
  //
  //   ClprQueueMetadata    — verified queue state. sent_running_hash MUST cover the
  //                          last message in the returned array.
  //   ClprMessagePayload[] — ordered message payloads proven by the bundle.
  //   new_trust_anchor     : bytes — successor signing authority if the bundle contains
  //                          state-proven rotation evidence; empty bytes if none. MUST be
  //                          state-proven against trust_anchor or MUST revert (§3.2, §4.2).
  //   new_trust_anchor_id  : bytes — opaque identifier for new_trust_anchor. MUST be
  //                          non-empty iff new_trust_anchor is non-empty.
  //   new_endpoint_manifest : ClprEndpointManifest (optional) — verified updated manifest
  //                          if the bundle contains a manifest proof; absent if none.
  //
  // Used during:
  //   - submitBundle (on-chain bundle processing)
  //
  // MUST revert if verification fails.
  // SHOULD fail fast on obviously malformed inputs before expensive cryptographic operations.
  function verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)
    returns (ClprQueueMetadata, ClprMessagePayload[],
             bytes new_trust_anchor, bytes new_trust_anchor_id,
             optional ClprEndpointManifest new_endpoint_manifest)
```

---

### 4.10 §6.5 Endpoint Management — full section replacement (`clpr-service-spec.md` §6.5)

Replace the entire §6.5 section (introductory paragraph and all code blocks) with:

```
On ledgers where endpoint registration is permissionless (e.g., Ethereum, Solana), the CLPR
Service MUST expose the following operations. On ledgers where endpoints are managed by the
platform (e.g., Hiero, where consensus nodes are the endpoints and the manifest is derived
automatically from the consensus roster), these operations are not needed.

All admitted endpoints are expected to serve all Channels on the CLPR Service in good
faith — this is a service-level commitment, not per-Channel. Bond posting and bundle
submission are completely orthogonal: anyone may submit a bundle that makes progress
regardless of manifest membership, and the bond plays no role in submission reimbursement.
Endpoint bonds are never slashed; misbehavior results in eviction with full bond refund.
Slashing applies only to Connectors.
```

```
// Register as an endpoint for this CLPR Service.
// Authority: any caller, including the CLPR Service admin.
// Stores endpoint_data in pending state; does not appear in the live manifest
// until confirmed by the admin. Bond held in escrow; returned in full on
// rejection or cancellation. Manifest version is not incremented on registration.
// Platform specs MUST define the bond delivery mechanism and minimum bond amount.
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
// MUST be rejected if the live manifest is already at max_local_endpoints.
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
// (after accounting for skipped already-live entries).
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

---

## 5. Changes to Existing Specification Files

The following tables enumerate every change required to implement this proposal. Line numbers are approximate references to the current file state and are provided to identify the target location; they may shift as surrounding content is edited.

### `clpr-service-spec.md`

| Section | Approx. Lines | Description of Change |
|---|---|---|
| §1.1 `ClprLedgerConfiguration` protobuf | ~65 | Remove the `repeated ClprEndpoint endpoints = 6;` field and its entire comment block. The endpoint list is no longer part of the ledger configuration; it is maintained as a separate `ClprEndpointManifest` in CLPR Service state. |
| §1.1 `ClprThrottles` protobuf | ~135 | Update the comment on `max_local_endpoints` to say "live ClprEndpointManifest for this CLPR Service" instead of "for a single Channel". Update the comment on `max_peer_endpoints` to reference `ClprEndpointManifest` instead of `ClprLedgerConfiguration.endpoints`. See §4.2 for the replacement comment text. |
| §1.1, new section after `ClprEndpoint` | New | Add the `ClprEndpointManifest` protobuf message definition with three fields: `uint64 version` (MUST be strictly positive (≥ 1); a monotonically increasing counter; starts at 1 when the manifest is first created; incremented by 1 on each change to the endpoint set; has no correlation to any other version numbers in the system; on Hiero, incremented only when the node set composition changes — pure consensus weight changes do not increment the version; a composition change is any event that adds or removes a node from the active roster, identified by a change to the set of node identity values; weight-only changes that do not alter the node identity set do not increment the version), `repeated ClprEndpoint endpoints` (the current endpoint set for this CLPR Service; may be empty — an empty manifest at version ≥ 1 is valid and indicates the network is not advertising endpoint addresses), and `bytes service_address` (the on-ledger address of the CLPR Service this manifest belongs to; included in the state-proven manifest content; a verifier MUST reject a manifest proof whose `service_address` does not match `ctx.service_address`). |
| §1.3 `ClprConfigUpdate` message | ~210–212 | Remove any description of endpoint propagation via `ConfigUpdate`. Update the message comment to clarify that `ConfigUpdate` carries only throttle and protocol version changes; endpoint changes are propagated through bundle payloads as described in the new §3.X endpoint manifest section. |
| §1.5 `ClprQueueMetadata` message | after ~380 | Add `uint64 endpoint_manifest_version = 7;` with a comment explaining that this field carries the sender's current `Channel.endpoint_manifest_version`, allowing the receiver to detect whether the remote manifest has advanced since the last sync without performing a full state query. |
| §1.5 gRPC `ClprEndpointService` | ~398–419 | Remove the `discoverEndpoints` RPC and the `DiscoverEndpointsRequest` and `DiscoverEndpointsResponse` protobuf message types entirely. The on-ledger `getEndpointManifest()` query is the authoritative discovery mechanism; a peer-to-peer gRPC discovery protocol is no longer part of the CLPR specification. Add a comment noting that supplemental off-chain discovery protocols are possible independently of this specification, since bundle submission is permissionless and anyone may submit a bundle that makes progress regardless of manifest membership. |
| §2.1 Channel state model | ~454–499 | Add `endpoint_manifest : ClprEndpointManifest` and `endpoint_manifest_version : uint64` fields to the Channel struct with comments describing their role. Note that the peer manifest is now represented by these fields and that the separate `PeerEndpointRosterEntry` concept is superseded. Also: add `channel_context : ChannelContext` to the Channel struct (introduced by the concurrent ChannelContext change, §3.1 — constant data from `verifyConfig`, passed to every `verifyBundle` call; immutable after `completeChannel`); and remove the standalone `service_address : bytes` field from Channel, which is now subsumed by `channel_context.service_address`. |
| §2.1.2 Bundle Progress Criteria table | ~564–574 | Add Criterion 5 to the table: **Endpoint manifest advancement** — holds when `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`. Update the introductory text to reflect five criteria. |
| §2.4.2 Peer Manifest section | ~679–704 | Rewrite this section entirely (retitling it "Peer Manifest"). Replace the `PeerEndpointRosterEntry` struct and its current description with a description of `Channel.endpoint_manifest` as the on-ledger peer manifest. Describe initial population via the extended `verifyConfig(config_proof_bytes, endpoint_manifest_proof_bytes)` call at `completeChannel` — there is no separate `verifyEndpointManifest` call — and ongoing refresh via `verifyBundle` return values. Remove the reference to `discoverEndpoints` as an off-chain complement. |
| §3.1 `IClprVerifier.verifyConfig` | ~773–776 | Replace the `verifyConfig` signature with: `verifyConfig(bytes config_proof_bytes, bytes channel_id, bytes endpoint_manifest_proof_bytes)` returning `(ChannelContext ctx, string chain_id, uint96 peer_config_nanos, ClprThrottles throttles, bytes initial_trust_anchor, bytes initial_trust_anchor_id, ClprEndpointManifest endpoint_manifest)`. The `channel_id` input is echoed verbatim in the returned `ctx`. The returned `ChannelContext` bundles `channel_id` and `service_address` (peer CLPR Service address); it replaces the standalone `service_address` return value. Add `ClprEndpointManifest` (the initial manifest at version ≥ 1) to the return values. Remove `endpoints: ClprEndpoint[]` from the return value list. The verifier MUST revert if the manifest proof's `service_address` does not match `ctx.service_address`, if the returned manifest version is 0, or if any cryptographic verification fails. The verifier MUST NOT revert solely because the manifest's endpoint list is empty. |
| §3.1 `IClprVerifier.verifyBundle` | ~801–802 | Replace the `verifyBundle` signature with: `verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)` returning `(ClprQueueMetadata, ClprMessagePayload[], bytes new_trust_anchor, bytes new_trust_anchor_id, ClprEndpointManifest new_endpoint_manifest)`. The `ctx` input carries constant channel-scoped data (channel_id, service_address) and is the same value stored by `completeChannel`; it is never mutated. Add optional `new_endpoint_manifest: ClprEndpointManifest` to the return value list. When a manifest proof is present in the bundle payload and verification succeeds, the verifier returns the verified manifest (present, version ≥ 1). When no manifest proof is present in the bundle, the verifier returns no manifest (absent/null). When `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, the CLPR Service atomically replaces the Channel's peer manifest before processing any message in the bundle. When `new_endpoint_manifest` is absent, or present but `new_endpoint_manifest.version ≤ Channel.endpoint_manifest_version`, the manifest content is silently skipped. |
| §3.1 `IClprVerifier`, new method | after `verifyBundle` | ~~Remove this row.~~ The separate `verifyEndpointManifest` method is not added. The manifest proof is verified through the extended `verifyConfig` call (see `§3.1 IClprVerifier.verifyConfig` row above). No new method is introduced on `IClprVerifier`. Also add the `ChannelContext` struct definition before `interface IClprVerifier {`. |
| §4.2 Step 1a NoProgress check | ~1012–1017 | Add endpoint manifest advancement as a fifth criterion: a bundle is not rejected for NoProgress if `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version` (strictly greater). An absent manifest or a present manifest with version ≤ the stored version does not satisfy this criterion; the manifest is silently skipped. Update the rejection condition to require that all five criteria are unsatisfied simultaneously. Criterion 5 applies to all Channel states that accept bundles (ACTIVE, CLOSING, DRAINED) — it is not limited to ACTIVE Channels. |
| §4.2 new Step 1b, renumber existing Step 1b to Step 1c | before existing Step 1b (~1020) | Insert new Step 1b (manifest update) before the existing trust anchor step, and renumber the existing Step 1b to Step 1c. Execution order is now: Step 1 (verifier call) → Step 1a (NoProgress check) → Step 1b (manifest update) → Step 1c (trust anchor update). Step 1b: if `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, atomically set `Channel.endpoint_manifest = new_endpoint_manifest` and `Channel.endpoint_manifest_version = new_endpoint_manifest.version`. If `new_endpoint_manifest` is absent, or present but `new_endpoint_manifest.version ≤ Channel.endpoint_manifest_version`, silently skip — do not revert. |
| §2.1 Channel struct, `trust_anchor_id` comment | ~475 | Update the comment on the `trust_anchor_id` field which reads "Updated atomically with each trust_anchor write (see §4.2 Step 1b and §5.1.3)." Change "Step 1b" to "Step 1c" to reflect the renumbering of the trust anchor update step. |
| §5.1.3 `completeChannel` Step 5 | ~1254–1255 | Replace the current step (which stores endpoints from the `verifyConfig` return value) with: the extended `verifyConfig(config_proof_bytes, endpoint_manifest_proof_bytes)` call (already made earlier in `completeChannel`) returns the verified `ClprEndpointManifest`; store this as `Channel.endpoint_manifest`, and store its `ClprEndpointManifest.version` field as `Channel.endpoint_manifest_version`. Remove any mention of a separate `verifyEndpointManifest` call — there is no such call; the manifest is verified within `verifyConfig`. Before `completeChannel` completes, `Channel.endpoint_manifest_version` MUST hold the version from the verified manifest (≥ 1); a value of 0 for `endpoint_manifest_version` at any point after `completeChannel` indicates an uninitialized Channel (still in PENDING state or not yet reached Step 5). |
| §2.3 Endpoint Bond | ~634–640 | Rewrite to reflect that the bond is always returned in full on removal — whether by self-exit or admin eviction. Remove "release conditions" from the list of things platform-specific specs MUST define; the only release condition is removal (which always refunds in full). Misbehavior results in eviction, not bond forfeiture. Add a note that slashing applies only to Connectors, not to endpoint bonds. |
| §8.4 Endpoint Sybil Resistance | ~1644–1648 | Rewrite to reflect that Sybil resistance is provided primarily by admin curation — the two-step admission model requires explicit admin approval, making it impossible to flood the manifest without admin complicity. This mechanism is chain-type agnostic: it applies identically to all permissionless ledger types (EVM-based, UTXO-based, or otherwise) because it operates at the CLPR Service admin layer, not at the chain's consensus or validator layer. The bond provides friction as a commitment signal across all chain types, not as a mechanism calibrated against majority-attack costs on any specific chain. Admin curation is the primary Sybil resistance mechanism regardless of chain type. Remove "endpoint reputation scoring" language entirely — misbehaving endpoints are evicted; there is no reputation tracking or behavioral scoring mechanism for endpoints in the CLPR protocol. Add a note that stale manifest-update bundles (those where `new_endpoint_manifest` is absent or `new_endpoint_manifest.version ≤ Channel.endpoint_manifest_version`) are deterred by transaction cost — the submitting account pays the transaction fee and the bundle makes no manifest progress. Add a note that admin curation places full trust in the admin's continued good faith operation, including not deliberately emptying the manifest. |
| §6.1 `setLedgerConfiguration` | ~1327–1334 | Remove endpoint management from the description of this operation. Add a note directing the reader to §6.5 Endpoint Management for the `registerEndpoint`, `addEndpoint`, and `removeEndpoint` operations. |
| §6.2 `completeChannel` | ~1371–1380 | Add `endpoint_manifest_proof_bytes: bytes` as a new parameter. Update the processing step description: the CLPR Service passes `endpoint_manifest_proof_bytes` to the extended `verifyConfig(config_proof_bytes, endpoint_manifest_proof_bytes)` call on the verifier contract (not a separate `verifyEndpointManifest` call). The returned `ClprEndpointManifest` is stored as the Channel's initial peer manifest. Also remove the stale reference to `endpoints` in the comment block's processing step list — the old step that says "Stores `endpoints` as the initial peer manifest" must be replaced with the description of the extended `verifyConfig` return value. |
| §6.5 Endpoint Management | ~1535–1562 | Rewrite this section to reflect the service-level two-step admission model and clarify the bond/submission orthogonality. Remove `channel_id` from all endpoint management operations — registration is at the CLPR Service level, not per-Channel. Five operations: `registerEndpoint`, `addEndpoint`, `removeEndpoint`, `updateEndpointManifest`, `getEndpointManifest`. See §5 of this proposal for the normative pseudo-API code blocks. Summary of semantics: `registerEndpoint([auth] caller, endpoint_data: ClprEndpoint, bond: uint)` — callable by any caller including the admin; the caller's address is stored as `registrant_account` for this entry; places the endpoint in pending state; bond held in escrow; MUST be rejected if bond does not meet minimum; pending endpoint may cancel via `removeEndpoint` at any time with full bond refund and no version increment. `addEndpoint([auth] admin, registrant_account: bytes, endpoint_data: ClprEndpoint)` — if a pending registration exists for `registrant_account`, approve it using the endpoint's self-registered `ClprEndpoint` data (admin-provided `endpoint_data` is ignored); if no pending registration exists, add `endpoint_data` directly with `registrant_account` as the new registrant, no bond recorded; increments version by 1; MUST be rejected if live manifest already at `max_local_endpoints`. `removeEndpoint([auth] admin_or_self, registrant_account: bytes)` — callable by the endpoint itself or the admin; always returns bond in full; increments version only if endpoint was live (not if pending-only). `updateEndpointManifest([auth] admin, add: repeated { registrant_account: bytes, endpoint_data: ClprEndpoint }, remove: repeated bytes)` — atomic batch; same per-entry resolution logic as `addEndpoint`; entries whose `registrant_account` is already live are silently skipped (idempotent); entries in `remove` that do not exist are silently skipped (idempotent); version incremented exactly once for the entire batch; MUST be rejected if post-add live count would exceed `max_local_endpoints`. `getEndpointManifest() → ClprEndpointManifest` — public, read-only. Add a note that all admitted endpoints are expected to serve all Channels on the CLPR Service — service-level commitment, not per-Channel. Add a note that bond posting and bundle submission are completely orthogonal. Add a note that endpoint bonds are never slashed; misbehavior results in eviction with full bond refund. Add a note that on Hiero, the manifest is derived automatically from the consensus node roster and none of these operations apply. |
| §1.2 `ClprEndpoint` message comment | ~§1.2 | Update the message-level comment on `ClprEndpoint`. The current comment references `ClprLedgerConfiguration.endpoints` as the container and `discoverEndpoints` as a use case. Replace with: `ClprEndpoint` is used in `ClprEndpointManifest.endpoints` (the on-ledger endpoint manifest, maintained as separate CLPR Service state; see §2.4.2 and §6.5). Remove the reference to `ClprLedgerConfiguration.endpoints` and to the `discoverEndpoints` RPC (which is removed). |
| §2.1 Channel state model, Initial values paragraph | ~§2.1 Initial values | Update the Initial values paragraph to add `endpoint_manifest` and `endpoint_manifest_version` to the list of initial Channel field values. Before `completeChannel` is called (while the Channel is in PENDING state), `endpoint_manifest_version` holds 0 — the uninitialized sentinel meaning "not yet populated." All valid manifests have version ≥ 1, so 0 is unambiguously uninitialized. After `completeChannel` completes, `endpoint_manifest_version` MUST hold a value ≥ 1 equal to the `ClprEndpointManifest.version` returned by `verifyConfig`. |
| §2.1, Peer endpoint roster paragraph (immediately after Channel struct) | ~lines 508–511 | Update the "Peer manifest" paragraph that references `PeerEndpointRosterEntry` and the old `endpoints` field. Replace the entire paragraph with: "The peer manifest is maintained as `Channel.endpoint_manifest` (a `ClprEndpointManifest` stored as on-ledger state per Channel; see §2.4.2). It is initially populated from the `ClprEndpointManifest` returned by `verifyConfig` at `completeChannel` and is refreshed automatically through manifest-update bundle payloads as described in §3.X. The `PeerEndpointRosterEntry` concept is superseded by `Channel.endpoint_manifest`. The off-chain `discoverEndpoints` gossip RPC is removed." |
| §2.4.1 Local Endpoint Manifest, `EndpointManifestEntry` struct | ~§2.4.1 | Update the `EndpointManifestEntry` struct definition and its storage key description. Remove `channel_id` from the struct and from the storage key — registration is at the CLPR Service level, not per-Channel. Add `registrant_account: bytes` as a local-only field (not part of `ClprEndpoint`; not included in the manifest sent to peers) — this is the on-ledger account of the registrant, the bond refund recipient, set from the caller at `registerEndpoint` time. The storage key is `registrant_account` (unique per CLPR Service). Add a `status` field to distinguish pending endpoints from live (manifest-admitted) endpoints. The `ClprEndpoint` struct itself (`service_endpoint`, `tls_certificate`) no longer contains the registrant identifier — `registrant_account` lives only in `EndpointManifestEntry`. |
| §4.2 Step 5c lazy config propagation | ~1070–1074 | Narrow the scope of lazy propagation: a `ConfigUpdate` is enqueued only for changes to throttles or protocol version, not for endpoint changes. Remove any mention of endpoints being propagated via this mechanism. Note: no algorithmic change to the trigger condition is required — `ClprConfigUpdate` wraps `ClprLedgerConfiguration`, and since `endpoints` is removed from `ClprLedgerConfiguration`, the propagation mechanism changes automatically. Only the narrative needs to be narrowed to reflect that endpoint changes are no longer carried. |
| §4.3 Message Enqueue Step 1a | ~1093–1097 | Apply the same narrowing as §4.2 Step 5c: lazy config propagation at message enqueue time covers throttle and protocol version changes only, not endpoint changes. As above, no algorithmic change to the trigger condition is needed — narrowing the narrative is sufficient. |
| §9 Recovery Scenarios, rows R1, R3, R4, R9 | ~§9, rows R1, R3, R4, R9 | Rewrite the recovery mechanism description in all four rows. Replace "Endpoints discover new peers via gossip (`discoverEndpoints` RPC) once any connectivity is restored" with the manifest-update bundle mechanism: any party may construct a state proof of the current remote manifest and submit it as a bundle payload via `submitBundle` directly as an on-chain transaction, without gRPC connectivity to any existing endpoint. The CLPR Service calls `verifyBundle`, which returns the verified manifest, and atomically replaces the Channel's peer manifest. The `discoverEndpoints` RPC is removed and must not appear in any recovery scenario. Also update R4, which references gossip-based recovery via R1 — update R4 to reference the manifest-update bundle mechanism instead. |
| §5.2 Endpoint Discovery section | ~1276–1287 | Rewrite this section to describe the full manifest-based discovery model. The rewrite must explicitly replace the entire pre-proposal mechanism description, including: (a) the "Peer manifest" paragraph that references "populated from the `endpoints` returned by `verifyConfig` at `completeChannel` and refreshed whenever a peer `ClprConfigUpdate` CONTROL message arrives" — both mechanisms are replaced; (b) the `discoverEndpoints` RPC paragraph — the RPC is removed entirely. The rewritten section describes: admin curation on permissionless ledgers; automatic population on Hiero; the `getEndpointManifest` query and state proof construction; use at `completeChannel` via the extended `verifyConfig`; bundle-payload-based updates with staleness detection via `endpoint_manifest_version` in `ClprQueueMetadata`; and a note that supplemental off-chain discovery protocols are possible independently of this specification since bundle submission is permissionless. |

### `clpr-service.md`

| Section | Approx. Lines | Description of Change |
|---|---|---|
| §3.1.1 Ledger Identity and Configuration | ~284–288 | Remove `Endpoints` from the list of primary fields in `ClprLedgerConfiguration`. Add a sentence noting that the endpoint manifest is maintained as separate CLPR Service state and is described in §3.1.2. |
| §3.1.2 Endpoint Manifest (entire section) | ~401–459 | Rewrite this section as the unified lifecycle description for endpoint manifests. It should cover: the `ClprEndpointManifest` structure and version counter; admin curation on permissionless ledgers; automatic population on Hiero from the consensus roster; public queryability and state proof construction; use at `completeChannel` via the extended `verifyConfig(config_proof_bytes, endpoint_manifest_proof_bytes)` call (no separate `verifyEndpointManifest` method exists); the deadlock rationale explaining why propagation through the message queue is insufficient; bundle-payload propagation of updates and the atomic replacement guarantee; version propagation via `ClprQueueMetadata`; and Bundle Progress Criterion 5. Remove the reference to `discoverEndpoints` as an off-chain complement; note instead that supplemental off-chain discovery protocols are possible independently of this specification since bundle submission is permissionless. This section should serve as the single reference for the full endpoint manifest lifecycle. |
| §3.1.2 Hiero, Ethereum, and Sybil resistance callouts | ~439–459 | The §3.1.2 rewrite above subsumes these callouts, but the following removals must be explicit. In the Hiero callout: remove "a misbehaving endpoint node's account can be slashed" and "No separate CLPR-specific bond is required because Hiero consensus nodes are permissioned, but this may change in the future for more open deployments" — CLPR imposes no penalties on Hiero nodes; misbehaving nodes are removed from the consensus roster by Hiero governance, which automatically removes them from the endpoint manifest. In the Ethereum callout: remove the description of validators self-registering; replace with the two-step model (any party may call `registerEndpoint`, admin confirms via `addEndpoint`). In the Sybil resistance callout: rewrite to reflect that Sybil resistance is provided by admin curation rather than bond calibration against majority-attack costs; remove "endpoint reputation scoring" — misbehavior results in eviction with no behavioral scoring or tracking. |
| §2.1 Role table | ~182 | In the Endpoint Operator row of the role table, remove "subject to slashing". Endpoint bonds are never slashed; misbehaving endpoints are evicted with full bond refund. |
| §2.2 Endpoint bond, locked-funds bullet | ~269–271 | Remove "or slashing" from the sentence describing the admin as "the sole authority for releasing or slashing" endpoint bonds. Bonds are always returned in full; slashing does not apply to endpoint bonds. |
| §4.5 Endpoint Operator, Risks paragraph | ~1613 | Remove "Misbehaving endpoints are subject to slashing." Replace with: "Misbehaving endpoints are evicted by the admin; the bond is returned in full." |
| §3.1.3 `completeChannel` description and sequence diagram | ~467–493 | Update the `completeChannel` description and sequence diagram to show `endpoint_manifest_proof_bytes` as an additional input parameter alongside `config_proof_bytes`. Show this parameter being passed to the extended `verifyConfig(config_proof_bytes, endpoint_manifest_proof_bytes)` call — not to a separate `verifyEndpointManifest` call. The sequence diagram must show a single `verifyConfig` call that returns both the configuration and the initial `ClprEndpointManifest`. |
| §3.1.5 `verifyConfig` return values | ~706–719 | Remove `endpoints` from the return value list and its accompanying description. Add `ClprEndpointManifest` (the initial manifest at version ≥ 1) to the return values — the manifest is now returned from `verifyConfig` directly via the extended signature, not proven via a separate method. Remove any reference to `verifyEndpointManifest` from this entry. |
| §3.1.5 `verifyBundle` description | ~720–728 | Add `new_endpoint_manifest: ClprEndpointManifest` to the description of what `verifyBundle` may return. Describe the behavior: if `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, the CLPR Service atomically replaces the peer manifest before processing any message in the bundle, mirroring the trust anchor update guarantee. When no manifest proof is present, an empty `ClprEndpointManifest` is returned and `new_endpoint_manifest.version` will be 0 by proto3 default. |
| §3.1.5 Verifier Contracts, `verifyConfig` description | existing entry | The separate `verifyEndpointManifest` method is not added — no new method is introduced on the verifier contract. Instead, update the `verifyConfig` description entry to cover the merged signature: `verifyConfig(bytes config_proof_bytes, bytes channel_id, bytes endpoint_manifest_proof_bytes)` returning `(ChannelContext ctx, chain_id, ...)`. The returned `ChannelContext` bundles `channel_id` and `service_address`; it replaces the standalone `service_address` return. Both the config proof and the manifest proof are verified in the same call. The verifier MUST revert if the manifest proof's `service_address` does not match `ctx.service_address`, if the returned manifest version is 0 (MUST be strictly positive), or if any cryptographic verification fails. The verifier MUST NOT revert solely because the manifest's endpoint list is empty. |
| §3.2.2 `ConfigUpdate` description | ~985–991 | Remove any statement or implication that endpoint changes propagate via `ConfigUpdate`. Clarify that `ConfigUpdate` carries throttle and protocol version changes only; endpoint changes travel through bundle payloads as described in §3.1.2. |
| §3.2.4 Bundle Progress Criteria table | ~1041–1053 | Add Criterion 5 (endpoint manifest advancement) to the table and update the accompanying text to reference five criteria. |
| §3.2.5 Bundle Verification description | ~1121–1153 | Add a step for endpoint manifest update: if the verifier returns a `new_endpoint_manifest` that is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, the CLPR Service atomically replaces `Channel.endpoint_manifest` and its version counter as part of accepting the bundle. |
| §3.1.1 Ledger Identity and Configuration, Timestamp subsection | ~Timestamp paragraph | Remove the word "endpoints" from the sentence "Any configuration update — including changes to throttles or endpoints — advances the timestamp." After the proposal, endpoints are no longer part of `ClprLedgerConfiguration`, so endpoint changes do not advance the configuration timestamp. The sentence should read "Any configuration update — including changes to throttles or protocol version — advances the timestamp." |
| §4.5 Endpoint Operator role | ~§4.5 | Update the Endpoint Operator description to reflect the two-step admission model. Replace the current text implying `registerEndpoint` is sufficient for participation with: calling `registerEndpoint` places the endpoint in pending state and posts a bond; the endpoint is not active until the CLPR Service Admin confirms via `addEndpoint`. Describe both steps as required for the endpoint to appear in the live manifest. Add a note that a pending endpoint may cancel its pending registration at any time by calling `removeEndpoint`; the bond is returned in full. |
| §4.6 CLPR Service Admin role | ~§4.6 | Update the Admin role description to reflect the new admission authority. Add to the admin's listed powers: confirming endpoint admissions via `addEndpoint` (the exclusive authority for moving an endpoint from pending to live manifest state) and rejecting pending registrations via `removeEndpoint`. |
| §5 Recovery Scenarios, rows R1, R3, R4, R9 | ~R1, R3, R4, R9 | Rewrite the recovery mechanism description in all four rows. Replace "Endpoints discover new peers via gossip (`discoverEndpoints` RPC) once any connectivity is restored" with the manifest-update bundle mechanism: any party may construct a state proof of the current remote manifest and submit it as a bundle payload via `submitBundle` directly as an on-chain transaction. The CLPR Service calls `verifyBundle`, which returns the verified manifest, and atomically replaces the Channel's peer manifest. No gRPC connectivity to old endpoints is required. Also update R4, which references gossip-based recovery via R1 — update R4 to reference the manifest-update bundle mechanism instead. |

### `clpr-test-spec.md`

| Section | Approx. Lines | Description of Change |
|---|---|---|
| §3.1.2 `getLedgerConfiguration` | ~§3.1.2 | Remove `endpoints` from the list of mutable fields returned by `getLedgerConfiguration`. After the proposal, `endpoints` is no longer a field of `ClprLedgerConfiguration`. Update the test assertion to reflect that ledger configuration contains throttles and protocol version but not endpoint data. |
| §3.14.1 Endpoints in Configuration | ~§3.14.1 | Remove this test entirely or rewrite it. The current assertion — "The ledger configuration includes an `endpoints` list used for initial peer discovery... Endpoints are propagated to peers via ConfigUpdate Control Messages" — is fully invalidated by the proposal. Replace with tests covering the `ClprEndpointManifest` structure, the `getEndpointManifest()` query, and the bundle-payload propagation mechanism. |
| §3.15.1 Registration with Bond | ~§3.15.1 | Rewrite to cover the two-step admission model: (1) `registerEndpoint` → endpoint enters pending state, bond held; (2) `addEndpoint` → endpoint enters live manifest, version increments; (3) `removeEndpoint` from pending → no version increment, bond returned in full; (4) admin eviction from live → version increments, bond returned in full. Verify that the endpoint does not appear in the live manifest between steps 1 and 2. |
| §3.10.4 `MaxLocalEndpoints` and §3.10.5 `MaxPeerEndpoints` | ~§3.10.4–§3.10.5 | Update §3.10.5 to remove the assertion that truncation applies "on every `ClprConfigUpdate`" — after the proposal, `ClprConfigUpdate` no longer carries endpoint data. The truncation behavior now applies at `completeChannel` (via the `ClprEndpointManifest` returned by `verifyConfig`) and on manifest-update bundle processing (via the `ClprEndpointManifest` returned by `verifyBundle`). Update §3.10.4 to reflect that `addEndpoint` MUST be rejected when the live manifest is already at `max_local_endpoints`, not `registerEndpoint`. |
| §7.5.3 Peer Discovery Convergence | ~§7.5.3 | Remove this test. It describes gossip-based peer discovery convergence via gossip, which is removed by the proposal. Replace with a test covering manifest-version staleness detection via `endpoint_manifest_version` in `ClprQueueMetadata` and the automatic bundle-payload update path: when a remote manifest advances, the receiving peer detects the version mismatch and the next inbound bundle carrying the manifest proof causes `Channel.endpoint_manifest` to be atomically updated. |
| §8.1.1, §8.1.2, §8.1.3 Endpoint Rotation | ~§8.1.x | Rewrite all three tests. The current descriptions cover recovery via gossip (`discoverEndpoints`). After the proposal, recovery is via manifest-update bundles: any party constructs a state proof of the current remote manifest, embeds it in a bundle payload, and calls `submitBundle` directly as an on-chain transaction. Verify that `Channel.endpoint_manifest` is atomically replaced and that the updated manifest is available for subsequent sync cycles. |
| §9 Cross-reference table, Endpoint Manifest and Endpoint Protocol rows | ~§9 | Update the `§3.1.2 Endpoint Roster` row and the `§3.1.4 Endpoint Protocol` row to remove references to `§7.5.3 Peer Discovery Convergence` (which is removed) and to `discoverEndpoints`. Replace with references to the new manifest-based tests described above. |
| New tests — `verifyConfig` extended signature | New | Add tests for `verifyConfig` with three inputs (`config_proof_bytes`, `channel_id`, `endpoint_manifest_proof_bytes`): basic success (returns `ChannelContext ctx` and `ClprEndpointManifest` at version ≥ 1); failure on invalid proof bytes; failure when manifest `service_address` does not match `ctx.service_address`; failure when returned manifest version is 0; success when returned manifest endpoint list is empty (empty manifest is valid). |
| New tests — Criterion 5 NoProgress check | New | Add tests: (1) manifest-advance-only bundle (no application messages, only a manifest proof with advancing version) is accepted and satisfies Criterion 5; (2) bundle where `new_endpoint_manifest` is absent, or present with `new_endpoint_manifest.version ≤ Channel.endpoint_manifest_version`, does not satisfy Criterion 5 — if no other criterion is satisfied, the NoProgress rejection applies; (3) Criterion 5 applies on CLOSING and DRAINED Channels as well as ACTIVE; (4) manifest-update bundle submitted against a PENDING Channel is rejected. |
| New tests — manifest presence and ordering | New | Add tests: (1) when `verifyBundle` returns no manifest (absent), no manifest update is applied and the bundle proceeds normally; (2) when a bundle carries both a trust anchor update and a manifest update, the manifest (Step 1b) is applied before the trust anchor (Step 1c); (3) when `new_endpoint_manifest` is present and `new_endpoint_manifest.version > Channel.endpoint_manifest_version`, `Channel.endpoint_manifest` is atomically replaced with the full returned manifest. |
| New tests — empty manifest | New | Add tests: `completeChannel` succeeds when the remote CLPR Service has an empty manifest (version ≥ 1, no endpoints); `Channel.endpoint_manifest` stores the empty manifest correctly; `Channel.endpoint_manifest_version` is set to the manifest version (≥ 1). |
| New tests — manual recovery mode | New | Add a test for the manual recovery path: all known peer endpoints are stale; any party calls `getEndpointManifest()` on the remote chain, constructs a state proof, embeds it in a bundle payload, and calls `submitBundle` as an on-chain transaction; verify that `Channel.endpoint_manifest` is updated without gRPC connectivity to any existing endpoint. |
| New tests — Hiero automatic manifest derivation | New | Add tests: (1) a consensus roster composition change (node added or removed) triggers a version increment in the Hiero manifest; (2) a weight-only roster change (stake adjustment, no node identity change) does not increment the version. |

---

## Editorial Notes on Review Findings

The following review findings are noted but require no additional changes beyond what is addressed above:

**Finding 31** — No `setEndpointManifest` reference found in the proposal. No action needed.

**Finding 32** — `discoverEndpoints` is correctly listed as removed in the change table entries for §1.5 and §5.2. The additional locations (§9 Recovery Scenarios, §7.5.3 in the test spec) are addressed by the new rows above. No additional action needed on this finding.

**Finding 33** — The `ClprEndpointManifest` protobuf field comment now reads "MUST be strictly positive (≥ 1)" in the updated §1.1 table row above, providing explicit RFC 2119 normative force rather than merely descriptive language.

**Finding 34** — The §5.1.3 `completeChannel` Step 5 table entry now explicitly states "store its `ClprEndpointManifest.version` field as `Channel.endpoint_manifest_version`" rather than the implicit "with its `endpoint_manifest_version`."

**Finding 35** — Resolved by removing `new_endpoint_manifest_version` as a separate return value entirely, and by using absence (null/absent) rather than a version-0 sentinel to indicate "no manifest in this bundle." `verifyBundle` returns `optional ClprEndpointManifest new_endpoint_manifest`; when no manifest proof is present, the value is absent. When present, `new_endpoint_manifest.version` is always ≥ 1 (the verifier MUST NOT return a manifest with version 0). The CLPR Service checks presence first, then `new_endpoint_manifest.version > Channel.endpoint_manifest_version` to determine whether to apply the update.
