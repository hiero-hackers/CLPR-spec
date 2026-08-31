# CLPR Specification Options to Consider

This document captures possible changes or expansions to the CLPR specification that have emerged from design
review discussions. Each section presents a self-contained proposal: the problem it solves, the proposed
solution, edge cases and limitations, and the value of implementing it. These are presented for evaluation
by the specification authors and reviewers — none are committed changes.

These proposals are ordered independently. They may be adopted individually or in combination.

---

# 1. Application-Level Message Redaction (Clawback / Undo)

## 1.1 Problem

Once an application calls `sendMessage` and the message is enqueued on the Channel's outbound queue, there
is no mechanism for the application to cancel it. The only entity that can redact a queued message is the CLPR
Service Admin. This creates two gaps:

- **No undo for applications.** An application that detects an error (wrong amount, wrong destination,
  fraudulent transaction) immediately after sending has no recourse. The message will be delivered and
  executed on the remote ledger regardless.
- **No clawback for regulated institutions.** Financial institutions subject to regulatory requirements
  (AML holds, fraud detection, compliance reviews) often need the ability to halt or reverse an outbound
  transfer within a short window after initiation. Currently this must be built entirely at the application
  layer — the application would need to implement its own internal queue with a delay window before calling
  `sendMessage`, which adds complexity and latency.
- **No clean unwinding on Channel closure.** When a Channel enters CLOSING or is closed, messages
  sitting in the outbound queue that have not yet been synced to the destination have an ambiguous fate. If
  the application could redact its own undelivered messages, it could cleanly reverse escrowed assets rather
  than waiting for an ambiguous outcome.

## 1.2 Proposed Solution

Allow the original sending application (identified by the `sender` field stamped at enqueue time) to redact
its own messages, subject to the constraint that the message has not yet participated in a sync.

**New operation:** `redactMessage` is extended to accept calls from the message's `sender` (in addition to
the CLPR Service Admin). The protocol behavior is identical to admin redaction: the message payload is
removed, the message slot is retained (carrying `SHA-256(serialized_original_payload)` as `message_hash`), and a `REDACTED` response is generated for the
application.

**Safety constraint:** The message must not have been included in a bundle that has been submitted to the
destination. The most conservative check: the message ID must be greater than `acked_message_id` (the peer
has not confirmed receipt). A stricter check could also verify that no `submitBundle` transaction referencing
this message has been submitted locally, but tracking this is difficult since endpoints operate off-chain.

## 1.3 Edge Cases and Limitations

**Race with delivery.** The fundamental limitation is that redaction is a local operation, but delivery is a
distributed one. Between the moment the message is enqueued and the moment the application redacts it, an
endpoint may have:
1. Read the message from state
2. Constructed a proof over it (including the running hash)
3. Transmitted it to a peer endpoint
4. The peer endpoint may have submitted it to the destination ledger

If step 4 has occurred, the destination processes the original message and generates a response. The source
has redacted the message and the application believes it was cancelled, but the destination has executed it.
The application receives both a `REDACTED` response (from the local redaction) and — eventually — the actual
response from the destination (which arrives as a Response Message in the inbound queue). The application
must handle the case where a `SUCCESS` or other response arrives for a message it believes was redacted.

This race window is proportional to the sync frequency plus consensus time on the destination. On fast
networks (Hiero-to-Hiero), this may be sub-second. On slower networks (involving Ethereum), it could be
12+ seconds.

**Connector accounting.** The Connector's `authorizeMessage()` was called and returned true at enqueue time.
The Connector may have updated internal state (reserved funds, adjusted exposure tracking). Redaction voids
this commitment without the Connector's explicit involvement. In practice this is benign — the Connector is
not charged (the message was never delivered) — but Connectors with sophisticated accounting may need to
handle a "message redacted" callback or event.

**Running hash preservation.** The running hash was computed over the original payload at enqueue time and
cannot be retroactively changed without breaking the chain for all subsequent messages. The redacted message
slot retains its `running_hash_after_processing` and carries `message_hash = SHA-256(serialized_original_payload)`
so the destination can recompute the chain step without the original payload. When the destination receives
this slot in a bundle, it computes `SHA-256(prev || message_hash)` — a valid, unbroken chain step. This is
identical to admin redaction and is already handled by the protocol.

## 1.4 Value

- **Immediate practical value for regulated institutions.** Clawback and compliance holds are real
  requirements for financial applications. Providing this at the protocol layer is simpler and more reliable
  than requiring every application to build its own pre-send staging queue.
- **Clean Channel wind-down.** Applications can proactively redact their undelivered messages when a
  Channel is closing, recovering escrowed assets cleanly rather than waiting for ambiguous outcomes.
- **Small specification change.** The redaction mechanism already exists for the admin. Extending it to the
  sending application is a permission change, not a new mechanism. The safety constraints and running hash
  behavior are identical.
- **The race condition is manageable.** The race with delivery produces the same kind of ambiguity that
  already exists when a Channel is closed with in-flight messages. Applications that are designed to
  handle Channel closure (as they must be) can handle the redaction race with the same reconciliation
  logic.

---

# 2. Message Migration Between Channels

## 2.1 Problem

Several recovery scenarios in the specification (R2, R4, R5 in §4 of the design document) require closing an
old Channel and registering a new one. When the old Channel is closed, messages in the outbound queue
that have not yet been delivered — and messages that were delivered but whose responses have not yet
returned — have an ambiguous fate. The specification currently states that these messages require
"application-level out-of-band reconciliation," but provides no tooling or protocol support for this.

For applications with significant in-flight message volume, this means that every proof format upgrade, every
verifier replacement, and every Channel migration causes a burst of ambiguous messages that each
application must individually reconcile. This is operationally expensive, error-prone, and creates a strong
disincentive to perform necessary upgrades.

The core question: can undelivered messages on a closing Channel be automatically migrated to an active
Channel that serves the same peer ledger?

## 2.2 Analysis: What Can and Cannot Be Migrated

Messages are bound to their Channel through three mechanisms: message ID sequencing, running hash chains,
and Connector mappings. These bindings mean a message cannot be moved verbatim from one Channel's queue to
another. However, the *payload content* (the application data, target application address, Connector
reference, and sender address) is not intrinsically tied to any Channel — it is transport metadata that
could be re-enqueued on a different Channel.

There are two distinct categories of messages:

**Undelivered outbound messages (source side).** These sit in the source ledger's outbound queue and have
not been synced to the destination. The payload is in local state. Migration means extracting the payload
and re-enqueuing it on the new Channel with a fresh message ID and running hash. This is entirely local
to the source ledger — the destination is not involved.

**Delivered-but-unresponded messages (both sides).** These were delivered to the destination and the
application executed, but the response has not made it back to the source. The response sits in the
destination's outbound queue on the old Channel. Migrating these requires the destination to extract the
response payload and re-enqueue it on the new Channel — which requires coordinated action between both
ledgers' CLPR Services.

## 2.3 Approach A: Source-Side Migration (Undelivered Messages Only)

### Solution

A new CLPR Service admin operation: `migrateMessages(source_channel_id, dest_channel_id,
message_id_range)`. For each message in the range:

1. Verify the message has not been delivered (message ID > `acked_message_id` plus a safety margin to
   account for in-flight bundles).
2. Extract the payload (Data Message content: `connector_id`, `target_application`, `sender`,
   `message_data`).
3. Re-enqueue the payload on the destination Channel via the normal enqueue path — assigning a new
   message ID, computing a new running hash, and calling the Connector's `authorizeMessage()` on the new
   Channel.
4. Mark the original message slot on the old Channel as migrated (a new disposition alongside
   delivered/redacted).
5. Emit a migration record mapping old `(channel_id, message_id)` to new `(channel_id, message_id)`
   so applications can update their correlation state.

### Edge Cases

- **Connector must be registered on the new Channel.** If the same Connector operates on both
  Channels, migration proceeds naturally. If not, the Connector's `authorizeMessage()` will fail and the
  message cannot be migrated. The admin would need to coordinate with the Connector Operator to register on
  the new Channel first.
- **Connector may reject re-authorization.** The Connector's policy may have changed, or its balance on the
  new Channel may be insufficient. Messages that fail re-authorization remain on the old Channel and
  become ambiguous when it closes.
- **Control Messages cannot be migrated.** ConfigUpdate messages are Channel-specific. They must be
  skipped during migration.
- **Response Messages cannot be migrated via this approach.** Responses reference inbound message IDs from
  the old Channel that have no meaning on the new Channel. Source-side migration handles only outbound
  Data Messages.
- **In-flight bundles.** A message may have been read by an endpoint and included in a bundle that is in
  transit to the destination but not yet acknowledged. Migrating this message creates a duplicate — the
  original arrives on the destination via the old Channel, and the migrated copy arrives via the new
  Channel. The application on the destination must handle this idempotently, or the safety margin on
  the message ID range must be wide enough to exclude any message that could be in flight.

### Value

- Eliminates ambiguity for the most common case: messages that were enqueued but never left the source
  ledger.
- Makes proof format upgrades (R2) and verifier replacements (R5) significantly less disruptive —
  applications retain their message ordering and do not need to reconcile undelivered messages.
- The mechanism is local to one ledger. No cross-ledger coordination is required for this approach.

## 2.4 Approach B: Coordinated Migration via Control Messages

### Solution

Extend the protocol to support coordinated migration of both undelivered messages and pending responses.
This uses new Control Message types on the new Channel to synchronize the migration between both ledgers.

**Phase 1 — Source-side migration (same as Approach A):**
The source admin migrates undelivered outbound Data Messages to the new Channel. A
`MigrationManifest` Control Message is enqueued on the new Channel's outbound queue, containing:
- The old Channel ID
- A mapping of old message IDs to new message IDs for all migrated messages
- A list of old message IDs that were delivered on the old Channel and are still awaiting responses

**Phase 2 — Destination processes the manifest:**
When the destination's CLPR Service processes the `MigrationManifest` (received via a normal bundle on the
new Channel), it:
1. Looks up each "awaiting response" message ID in the old Channel's state.
2. For messages whose responses are still in the destination's outbound queue on the old Channel:
extract the response payload and re-enqueue it on the new Channel's outbound queue with a new
message ID. The `original_message_id` field in the response is remapped using the manifest's ID mapping.
3. For messages whose responses have already been sent (but not yet acknowledged by the source): the
response may be in flight on the old Channel. These are flagged as "migration-pending" and handled
when the old Channel's remaining syncs complete.

**Phase 3 — Source receives migrated responses:**
Responses arriving on the new Channel carry remapped `original_message_id` values that correspond to the
new message IDs. The source CLPR Service matches them normally. Applications see responses correlated to the
new message IDs (which they already know about from the migration record emitted in Phase 1).

### Edge Cases

- **Responses already in flight on the old Channel.** If a response was included in a bundle on the old
  Channel that is in transit, it may arrive on the source after migration. The source has already
  remapped the message to the new Channel. It must recognize the old-Channel response and correlate
  it to the migrated message. This requires the source to retain migration state (old message ID → new
  message ID) until the old Channel is fully drained.
- **Partial migration.** Some messages may be migratable and others not (e.g., Connector not registered on
  the new Channel). The manifest must indicate which messages were successfully migrated and which were
  left behind.
- **Ordering guarantees across Channels.** Messages migrated from the old Channel receive new IDs on
  the new Channel. If the new Channel already has messages of its own, the migrated messages are
  interleaved with them. The relative ordering of migrated messages is preserved (they are re-enqueued in
  original ID order), but their position relative to new Channel's native messages depends on when the
  migration occurs. Applications that depend on strict ordering across the old and new Channels must
  be aware of this.
- **Both admins must act.** The source admin initiates migration by migrating messages and sending the
  manifest. The destination admin must have the new Channel active and its CLPR Service must support
  manifest processing. If the destination's CLPR Service does not recognize `MigrationManifest` (e.g.,
  running an older version), the manifest is rejected as an unknown Control Message type and the
  migration fails.
- **Running hash divergence.** The old Channel's running hash chain continues to reflect the original
  messages (including those that were migrated). The new Channel has its own independent running hash
  chain. There is no cryptographic linkage between the two — the migration is a semantic operation (payload
  transfer), not a cryptographic continuation.

### Value

- Provides end-to-end migration: both undelivered messages and pending responses are moved to the new
  Channel.
- Eliminates the "ambiguous outcome" problem for proof format upgrades and verifier replacements. Messages
  are not lost — they are relocated.
- Uses the existing Control Message mechanism, extending it with a new subtype rather than introducing a
  separate migration protocol.

### Cost

- Significant protocol complexity. The `MigrationManifest` control message, the destination-side manifest
  processing, the response remapping, and the in-flight handling all add substantial specification and
  implementation surface.
- Both CLPR Services must support the manifest. This creates a version dependency: migration only works if
  both sides are running a version that understands `MigrationManifest`. This may be acceptable for
  planned migrations (proof format upgrades) but not for emergency situations (verifier compromise) where
  the peer may not have upgraded.

## 2.5 Approach C: Application-Layer Migration with Protocol Assistance

### Solution

Rather than migrating messages at the protocol level, provide protocol primitives that make application-layer
migration reliable. The CLPR Service does not move messages itself — it provides the information applications
need to do it.

**New query:** `getUndeliveredMessages(channel_id, message_id_range)` — returns the payload content of
undelivered outbound Data Messages on a Channel. Available to any caller (the payloads are on-chain and
already readable, but this provides a convenient batched interface).

**New event/record:** When a Channel transitions to CLOSING or CLOSED, the CLPR Service emits a
structured record listing all undelivered outbound Data Message IDs and all delivered-but-unresponded Data
Message IDs. Applications subscribe to this event and perform their own migration logic: re-sending
messages on the new Channel using `sendMessage`, handling Connector selection, and managing their own
ID correlation.

### Edge Cases

- **Same race conditions as Approach A** for messages that may be in flight at the time of closure.
- **Each application migrates independently.** If 10 applications have messages on the closing Channel,
  each one must perform its own migration. There is no batched admin operation.
- **Connector selection is the application's responsibility.** The application must choose which Connector
  to use on the new Channel, which may be different from the original.
- **No response migration.** Responses stuck on the destination are not addressed — the application must
  still reconcile these out-of-band. This approach only helps with the source-side undelivered messages.

### Value

- Minimal protocol change — a new query and a new event, no new state transitions or Control Message types.
- Applications retain full control over the migration logic, including Connector selection, message
  filtering, and retry behavior.
- No cross-ledger coordination required at the protocol level.

### Cost

- Migration burden falls on every application individually. A protocol-level migration (Approach A or B)
  does it once for all applications.
- No help with response migration — the hardest part of the problem is not addressed.

## 2.6 Comparison

|           Dimension           | Approach A (Source-Side) | Approach B (Coordinated) |  Approach C (App-Layer)   |
|-------------------------------|--------------------------|--------------------------|---------------------------|
| Undelivered message migration | Yes                      | Yes                      | Yes (application does it) |
| Pending response migration    | No                       | Yes                      | No                        |
| Cross-ledger coordination     | None                     | Required (both admins)   | None                      |
| Protocol complexity           | Low                      | High                     | Minimal                   |
| Application developer burden  | Low (automatic)          | Low (automatic)          | High (per-application)    |
| Connector re-authorization    | Required                 | Required                 | Application handles       |
| Version dependency            | Single ledger            | Both ledgers             | None                      |
| In-flight message handling    | Safety margin            | Manifest + drain         | Application handles       |

For a first implementation, **Approach A** provides the highest value-to-complexity ratio. It handles the
most common case (undelivered outbound messages), requires no cross-ledger coordination, and is a single
admin operation. The "ambiguous response" problem for delivered-but-unresponded messages remains, but this
is a smaller set of messages and the same ambiguity already exists with Channel closure today.

**Approach B** is the complete solution but carries significant specification and implementation cost. It
may be appropriate as a future enhancement once the protocol is mature and migration patterns are well
understood from operational experience with Approach A.

**Approach C** is the lowest-cost option and may be sufficient as an interim measure, but it places the
migration burden on every application and does not address response migration.

---

# 3. Connector Graceful Wind-Down

## 3.1 Problem

Under the current specification, a Connector cannot deregister while it has unresolved in-flight messages.
The `deregisterConnector` operation is rejected until every Data Message the Connector authorized has
received a response. On an active, healthy Channel this is a manageable wait. On a PAUSED Channel —
particularly one paused due to a response ordering violation — messages may never resolve. The Connector's
entire `locked_stake` and `balance` are frozen indefinitely, with no mechanism to reduce exposure.

This creates a disproportionate penalty for Connectors caught in circumstances outside their control. A
response ordering violation is a peer-side bug, not the Connector's fault. Yet the Connector bears the full
economic consequences: their funds are locked, they cannot redeploy capital to other Channels, and they
have no timeline for resolution. The longer the Channel stays PAUSED, the worse the Connector's position
becomes — and the Connector has no power to fix the situation. The admin may close a PAUSED Channel
(transitions to CLOSING), but the Channel cannot drain until the peer fixes the ordering.

## 3.2 Proposed Solution

Introduce a **wind-down** state for Connectors that immediately stops new message authorization while
allowing partial withdrawal of funds, retaining only a reserve sufficient to cover the worst-case slashing
exposure from remaining in-flight messages.

**New operation:** `windDownConnector(channel_id, source_connector_address)`. Callable by the Connector
admin. The CLPR Service:

1. **Marks the Connector as winding down.** From this point, the CLPR Service auto-denies any
   `authorizeMessage()` call for this Connector. No new messages can enter the pipeline. This is
   instantaneous and has no race condition — authorization is checked by the CLPR Service itself at
   enqueue time.

2. **Computes the minimum reserve.** The CLPR Service counts the number of unresolved Data Messages
   this Connector authorized (messages that have been enqueued but have not yet received a response).
   The reserve is:

   ```
   minimum_reserve = unresolved_message_count × max_slash_per_message
   ```

   Where `max_slash_per_message` is the worst-case escalated slash amount from the platform's slashing
   schedule. This must use the escalated amount (not the base fine) to account for the possibility that
   all remaining messages trigger slashing.

3. **Releases excess funds.** The Connector's `balance` is fully withdrawable (no new messages need
   funding). The Connector's `locked_stake` is withdrawable down to `minimum_reserve`. The Connector
   calls `withdrawConnectorBalance` and receives funds immediately.

4. **Reserve decrements as messages resolve.** Each time a response arrives for one of the Connector's
   in-flight messages:

   - If the response is `SUCCESS`, `APPLICATION_ERROR`, or `REDACTED`: no slash. The reserve for that
     message is freed. The Connector can withdraw the freed amount.
   - If the response is `CONNECTOR_NOT_FOUND` or `CONNECTOR_UNDERFUNDED`: the reserve for that message
     is slashed and paid to the submitting endpoint. The reserve decreases by the slashed amount.
5. **Final withdrawal and deregistration.** As in-flight messages resolve, the reserve shrinks. The
   Connector Operator can return at any time to withdraw newly freed funds. When the unresolved count
   reaches zero (all messages have received responses), or when the Channel is closed (no further
   responses will arrive), the Connector calls `deregisterConnector` to reclaim any remaining reserve
   in full and remove the Connector from the Channel.

## 3.3 Scenarios

**PAUSED Channel, peer eventually fixes ordering.** The Connector winds down, withdraws excess funds, and
waits with minimal locked exposure. When the peer fixes the response ordering violation and the Channel
auto-resumes to ACTIVE, responses flow normally. Each response either frees reserve (no slash) or consumes
it (slash). Eventually all messages resolve and the Connector deregisters cleanly.

**PAUSED Channel, peer never fixes ordering.** This is the most important scenario. Without wind-down, the
Connector's entire stake is locked indefinitely — possibly forever if the bug is never fixed. With wind-down:
the Connector immediately recovers everything above the reserve. The reserve sits locked until the peer fixes
the ordering (Channel auto-resumes, responses flow, slashing occurs or doesn't). The Connector's locked
exposure drops from their entire `locked_stake` to just the reserve covering in-flight messages. Note: the
admin may close a PAUSED Channel (transitions to CLOSING), but it cannot drain until the peer fixes
the ordering.

**CLOSING Channel.** When the admin calls `closeChannel`, the Channel transitions to CLOSING: queued
messages receive `CHANNEL_CLOSED` responses (no slashing), and the Channel drains to CLOSED. Each
`CHANNEL_CLOSED` response frees reserve (no slash). The Connector's remaining reserve is returned in full
at deregistration. CLOSING resolves the indefinite lock-up problem for admin-initiated exits — the Connector
is no longer blocked waiting for responses that may never come.

**Active Channel, normal exit.** The Connector winds down on a healthy Channel. Authorization stops
immediately. Responses flow normally and the reserve decrements with each one. This is strictly better than
the current model where the Connector must wait with their full stake locked until all messages settle.

**Multiple Connectors, one winds down.** Other Connectors on the same Channel are unaffected. They
continue to authorize messages and operate normally. Applications that were using the winding-down Connector
see their `authorizeMessage()` calls rejected and must switch to an alternative Connector.

## 3.4 Edge Cases and Limitations

**Reserve calculation must use worst-case escalated slash amounts.** The slashing schedule escalates with
repeated failures. If the reserve is computed using the base fine and all remaining messages trigger
slashing, the reserve is insufficient. The reserve formula must use the maximum possible slash per message
assuming all remaining messages fail. Platform-specific specifications define the slashing schedule and must
provide this worst-case value.

**Destination-side wind-down.** The Connector exists on both ledgers. On the destination side, the
Connector's `balance` pays for execution of incoming messages, and `locked_stake` covers destination-side
slashing. The same wind-down logic applies: count in-flight messages that haven't been processed, compute
reserve, allow partial withdrawal. The destination's estimate of in-flight messages is less precise (it
depends on what the source has sent but not yet arrived), so the destination must use a conservative
estimate derived from the Channel's queue metadata — the gap between `next_message_id` on the source
(learned via the most recent sync) and `received_message_id` on the destination.

**Wind-down is irreversible.** Once a Connector enters wind-down, it should not be able to return to active
status on the same Channel. If the Connector wants to resume serving the Channel later, it must
deregister (after messages resolve) and re-register. This prevents gaming where a Connector winds down to
withdraw funds, then reactivates with reduced stake.

**No new attack surface.** A Connector in wind-down is strictly less capable than an active Connector — it
cannot authorize new messages, cannot increase exposure, and can only withdraw funds. There is no mechanism
for a winding-down Connector to harm endpoints or applications beyond the exposure that already existed
from its previously authorized messages.

## 3.5 Value

- **Directly solves the Connector deregistration timing problem.** This is an identified open issue in the
  specification. The current model can lock Connector funds indefinitely on a PAUSED Channel with no
  resolution timeline (the peer may never fix the ordering violation, and even if the admin closes the
  Channel it cannot drain until the peer fixes the ordering). Wind-down provides an immediate,
  quantifiable reduction in exposure. For CLOSING Channels,
  `CHANNEL_CLOSED` responses resolve messages without slashing, so the Connector drains naturally —
  wind-down is still useful to release excess funds faster during the drain period.
- **Fair to Connectors.** A response ordering violation is not the Connector's fault. Wind-down lets them
  limit their exposure to the actual risk (in-flight messages) rather than bearing the full cost of a
  system-level problem.
- **Encourages Connector participation.** Connector Operators evaluating the risk of serving a Channel
  will be more willing to participate if they know they can limit their downside in adverse scenarios.
  Without wind-down, serving a Channel is an open-ended commitment with unbounded lock-up risk on
  PAUSED Channels.
- **Modest specification change.** One new operation (`windDownConnector`), one new Connector state
  (winding down), and a reserve computation that uses data the CLPR Service already tracks (unresolved
  message count per Connector, slashing schedule). No new Control Message types, no cross-ledger
  coordination, no changes to the bundle verification algorithm.

---

# 4. Reconciliation Tooling for `UNRESOLVED` Messages

## 4.1 Problem

When a Channel is closed with in-flight messages — or when recovery scenarios such as verifier
replacement or proof-format upgrades require closing the old Channel — applications can receive an
`UNRESOLVED` outcome for messages that were sent but never definitively delivered or responded to. The
specification today says these require "application-level out-of-band reconciliation," but provides no
tooling or standard interfaces for actually performing that reconciliation. Each application is left to
build its own ad-hoc mechanism for figuring out whether a particular message landed on the destination,
was executed, or was lost.

This is operationally painful: every application must independently solve the same problem, and any
mistake in their reconciliation logic can lead to double-spends, missed deliveries, or stuck escrow.

## 4.2 Discussion

What tooling or standard interfaces should CLPR provide to make `UNRESOLVED` reconciliation easier?
Possibilities include:

- A standard query against the remote ledger's CLPR Service to check the status of a specific message
  by ID — returning whether the message was received, executed, what response was generated, etc.
- A standard event/record format that applications can subscribe to when a Channel enters CLOSING
  or CLOSED, listing all in-flight messages and their last-known state.
- A reconciliation gRPC method on `ClprEndpointService` that lets a source ledger query a destination
  ledger directly (off-chain) for the disposition of a message.
- A standardized application-side library that consumes these primitives and exposes a higher-level
  reconciliation API.

This needs to be designed in concert with the Message Migration proposals (§2 above), since both
address overlapping concerns about what happens to in-flight messages when a Channel ends.

---

# 5. Multiple Proof Formats per Channel

## 5.1 Problem

A Channel today verifies bundles using a single proof format — the verifier contract registered at
`completeChannel` time is immutable, and all bundles flowing on the Channel MUST be verifiable
against it. The endpoint roster may carry multiple entries with different platform labels, but every
entry verifies the **same proof format** (the source ledger generates one kind of proof); the label
distinguishes implementations for different target platforms, not different proof formats.

This raises the question: do we need to support multiple proof formats per Channel? Scenarios that
might motivate this include staged migration to a new proof format, supporting heterogeneous
destination platforms with different verifier capabilities, or hedging against a vulnerability in one
format by accepting an alternate.

## 5.2 Discussion

If multiple proof formats per Channel are needed, the protocol would have to specify:

- How a bundle declares which format it uses, and how the receiver dispatches to the correct verifier.
- Whether the formats are interchangeable on a per-bundle basis, or whether one is a "primary" and
  others are fallbacks.
- How `ConfigUpdate` carries multiple verifier contracts and how peers coordinate when a new format is
  added or retired.
- How the running hash treats payloads that may have been signed by different proof systems.

The simpler alternative is to require a new Channel (with a new verifier) for any change of proof
format, leaning on the Message Migration proposals (§2) to move in-flight state across. The trade-off
is operational disruption versus protocol complexity.

---

# 6. CLPR Service Admin Economic Incentive Gap

## 6.1 Problem

The CLPR Service Admin has significant responsibility — monitoring Channels, responding to
incidents, coordinating Channel closures with peer admins, deciding when to close a misbehaving
Channel, redacting application messages when required by compliance, and so on. The Admin role is
therefore a load-bearing component of the protocol's operational integrity.

However, the protocol has no in-protocol economic participation for the Admin. There is no admin fee,
no slashing, no staking, and no protocol-level reward mechanism. The Admin's compensation, if any,
must come from outside the protocol — for example, from the deploying organization or from
application-level fees layered on top.

This creates a gap: the role is critical but unrewarded, which may lead to under-investment in
incident response, slow reactions to misbehavior, or admins who treat the role as purely ceremonial.

## 6.2 Discussion

Should the protocol include an admin fee or incentive mechanism? Options include:

- A small per-message or per-bundle fee routed to the admin account, configurable in the
  `ClprLedgerConfiguration`.
- An admin staking requirement, where the admin posts a bond that can be slashed by governance for
  failure to act on incidents.
- A no-protocol-level approach: rely entirely on out-of-protocol compensation and accept the resulting
  agency risk.

Each option has trade-offs. Per-message fees can disincentivize use of the Channel. Staking
requires a governance mechanism to slash. The status quo accepts that incentives are external.

---

# 7. Verifier Developer Sustainability

## 7.1 Problem

Verifier contracts are critical security components — every bundle on a Channel is validated by the
verifier registered at `completeChannel`. Verifier Developers must implement, audit, maintain, and
upgrade these contracts as proof formats evolve and bugs are found. This is highly skilled,
cryptographically sensitive work.

The protocol has no in-protocol compensation for Verifier Developers. They are typically funded by
ecosystem grants, foundation budgets, or the deploying organization's own engineering team. If
external funding dries up — a foundation cuts its grants, an organization deprioritizes the work —
verifiers may go unmaintained. Unmaintained verifiers become a systemic risk for every Channel that
depends on them: bugs go unfixed, proof-format upgrades are not supported, and emergency response
slows.

## 7.2 Discussion

How should the protocol ensure long-term Verifier Developer sustainability? Options include:

- A verifier-developer fee routed via the Channel (e.g., a small portion of each bundle's verification
  cost goes to a developer-designated address recorded in the verifier contract).
- A bounty/maintenance fund collectively funded by Channel participants, governed off-chain.
- Treating verifiers as public infrastructure funded by foundations or by the platforms that benefit
  most.

The economic-incentive question for verifiers is parallel to the one for the CLPR Service Admin (§6
above) and could potentially be addressed by a single, more general mechanism for routing protocol
fees to load-bearing roles.

---

# 8. Channel Creation Anti-Griefing

## 8.1 Problem

Channel creation is permissionless and effectively free beyond the transaction fees of the
underlying ledger. Anyone can call `registerChannel` and then `completeChannel`, creating a
Channel slot that consumes on-ledger state. Today, no minimum stake, bond, or deposit is required.

This opens the door to griefing: an attacker can spam Channel registrations, bloating state and
imposing costs on the CLPR Service deployment. While transaction fees provide some friction, on
high-throughput permissionless ledgers (or ones with cheap fees) this friction may be insufficient.

## 8.2 Discussion

Requiring a bonded Connector at registration time would provide economic friction without requiring a
new mechanism — Connectors already have stake. Specifically: at `completeChannel` time the
registrant could be required to provide an initial Connector with a non-zero bond, refundable on
clean Channel closure but slashed if the Channel is abandoned without proper drain.

Alternatives include:

- A flat protocol-level Channel registration deposit, refundable on `closeChannel` reaching
  CLOSED state.
- A platform-defined minimum stake that the CLPR Service deployment can configure independently of the
  per-Connector bond.
- Rate-limiting per registrant address (more complex on permissionless ledgers).

This is not yet part of the specification and the trade-offs (friction vs. accessibility, refund
mechanics, interaction with admin closure) need careful analysis.

---

# 9. Connector Deregistration Timing on PAUSED Channels

## 9.1 Problem

The protocol prevents Connector deregistration while messages are in-flight: `deregisterConnector`
fails until every Data Message the Connector authorized has received a response. CLOSING resolves the
indefinite blocking problem for admin-initiated shutdowns — queued messages receive `CHANNEL_CLOSED`
responses (no slashing) and queues drain, unblocking deregistration.

However, a PAUSED Channel (caused by a response ordering violation by the peer) has no admin
escape. The admin may call `closeChannel`, transitioning the Channel to CLOSING, but it cannot
drain until the peer fixes the ordering. If the peer never fixes it, the Connector's funds and stake
remain locked indefinitely. This is the one scenario where deregistration can be blocked forever — and
it is not the Connector's fault.

## 9.2 Discussion

What should the protocol provide for Connectors caught on indefinitely-PAUSED Channels? Options
include:

- A wind-down mechanism that allows partial withdrawal while preserving a slashing reserve (see §3,
  Connector Graceful Wind-Down — this is the most fully-developed proposal addressing exactly this
  case).
- An admin escape that, after a sufficiently long PAUSED interval, allows the Channel to drain with
  `CHANNEL_CLOSED` responses regardless of peer ordering — at the cost of weaker correctness
  guarantees.
- A protocol-level timeout that auto-resolves PAUSED Channels after a configured period, possibly
  with slashing applied to ambiguous messages.

This question is closely tied to the indefinitely-PAUSED Channel question (§10 below) and to the
Connector Graceful Wind-Down proposal (§3 above).

---

# 10. Indefinitely PAUSED Channel

## 10.1 Problem

If a peer ledger's queue state is permanently corrupted and it can never produce correctly ordered
responses, the Channel stays PAUSED indefinitely (§4.5 of the spec). The admin may close it
(`closeChannel` transitions to CLOSING), but the Channel cannot drain until the peer fixes the
ordering. If the peer never fixes it, the Channel stays in CLOSING indefinitely — no
`CHANNEL_CLOSED` responses are generated while bundles are being rejected for out-of-order
responses. Once the peer fixes the ordering, bundles process with `CHANNEL_CLOSED` responses and
queues drain normally. Applications on a permanently broken peer may still need out-of-band
reconciliation.

In short: when a peer ledger's queue state is permanently corrupted, the Channel stays PAUSED
indefinitely; admin closure transitions it to CLOSING but cannot drain. Connectors and applications
remain economically and operationally entangled with a peer that cannot recover.

## 10.2 Discussion

What tooling or admin escape hatch should the protocol provide for the indefinitely-PAUSED case?
Possibilities include:

- A "force drain" admin operation that, after a long PAUSED interval and explicit acknowledgement of
  the consequences, allows the Channel to drain with `CHANNEL_CLOSED` responses regardless of
  the peer's ordering state — accepting that the peer's state is permanently divergent.
- A standardized reconciliation tooling layer (see §4 above) that helps applications recover whatever
  off-chain state they need before the admin force-drains.
- Coordination with the Connector Graceful Wind-Down proposal (§3) so Connectors can at least limit
  their economic exposure to the corrupted peer.
- A protocol-level "peer-broken" assertion that the Channel's local admin can declare unilaterally,
  causing CLOSING to drain locally with explicit `PEER_UNRECOVERABLE` responses to applications.

The trade-off is between liveness (letting the local side recover from a broken peer) and correctness
(never dropping messages that the peer might eventually process).

---

# 11. Post-Quantum Endpoint Signing Key

## 11.1 Problem

ECDSA secp256k1 is the current endpoint signing key. It is not quantum-safe. CLPR must support
post-quantum schemes such as Falcon and CRYSTALS-Dilithium before quantum capabilities reach the
threat threshold. Without a planned migration, every Channel's endpoint authentication becomes
vulnerable simultaneously when sufficiently capable quantum hardware appears.

## 11.2 Discussion

Migration path: use the existing `ConfigUpdate` mechanism to rotate endpoint keys across all
Channels once mainline ledgers add native PQ verification support. The verifier-recovery model in
`submitBundle` (recovering the public key from the signature and matching it against the on-chain
roster) generalizes naturally — the signature scheme is platform-defined, and any scheme that supports
a recover-and-match flow fits the existing mechanism.

Open question: which PQ algorithm(s) to standardize on, and how to coordinate the migration across
heterogeneous platforms? Specifically:

- **EVM ecosystem.** Lacks a PQ-friendly precompile equivalent to `ecrecover`. Either wait for a
  precompile, accept very high gas costs for in-bytecode verification, or use a hybrid scheme that
  checks a classical signature on-chain plus a PQ signature off-chain.
- **Hiero native.** Can add new signature schemes via system-contract extensions; this is the most
  straightforward target.
- **Solana-style platforms.** Have syscall-style verification primitives that can be extended for PQ
  schemes given enough validator coordination.

The migration also has to handle the period during which different Channels support different
schemes — the `ClprLedgerConfiguration` may need to enumerate the algorithm explicitly per endpoint,
and the peer roster representation must accommodate variable-size keys.

---

# 12. SHA-384 for Running Hash

## 12.1 Problem

The running hash currently uses SHA-256, chosen for universal platform availability and consistency
with Hiero infrastructure. SHA-384 provides 192-bit preimage resistance under Grover's algorithm
(versus SHA-256's 128-bit), giving stronger long-term security against quantum-assisted attacks on
hash collisions and preimages.

The trade-off is gas cost on Ethereum: SHA-256 has a precompile and is cheap; SHA-384 requires
bytecode implementation and is significantly more expensive per message. Since the running hash is
computed for every message processed on every Channel on every ledger, a per-message gas overhead
multiplies into a substantial protocol-wide cost.

## 12.2 Discussion

Open question: is the additional security worth the per-message gas overhead on EVM chains,
especially given that the hash algorithm can be upgraded later via a protocol version bump and
Channel renegotiation? Considerations:

- The post-quantum hash threat is more gradual than the post-quantum signature threat; SHA-256 is
  expected to retain ~128-bit preimage resistance against Grover's algorithm, which remains
  operationally hard.
- A protocol-version-bump migration path exists: when the threat materializes, peers can renegotiate
  to a stronger hash algorithm. However, retroactively re-hashing existing running-hash chains is not
  possible — only new Channels (or new ranges within Channels, after a `ConfigUpdate`) would use
  the new algorithm.
- An intermediate option: standardize on a hash with a precompile on EVM but stronger than SHA-256
  (e.g., Keccak-512 truncated, BLAKE2). The precompile landscape varies.
- A future proof-friendly hash (e.g., one chosen for SNARK efficiency) might dominate either choice in
  the long run.

This decision should be made before locking the running-hash format into the 1.0 spec, since changing
it later requires either a protocol version bump or accepting heterogeneous hash chains across
Channels.

---

# 13. Cross-Ledger Peer Endpoint Signing Key Propagation

## 13.1 Problem

Each peer's `ClprLedgerConfiguration.endpoints` list supplies the on-chain set of peer signing keys
against which `submitBundle` signatures are verified. The size of this list is governed by the
receiving ledger's `ClprThrottles.max_peer_endpoints` — a configurable limit that accommodates
production deployments with tens of peer nodes (e.g., 40–50). For deployments with hundreds or
thousands of peer endpoints, only a subset can be stored on-chain without prohibitive cost, meaning
the remainder cannot directly participate in cross-ledger sync.

The peer endpoint roster is intentionally not propagated on-chain in full, because the O(N × M) cost
of broadcasting endpoint roster changes across N endpoints and M Channels is prohibitive. But
without some on-chain knowledge of peer signing keys, the receiving ledger cannot attribute a
`submitBundle` signature to a registered peer endpoint, and therefore cannot enforce per-endpoint
throttles or detect duplicate submission misbehavior.

## 13.2 Discussion

A richer mechanism is needed for production-scale deployments. Possible approaches:

- A new Control Message variant that carries peer signing-key roster updates (additions, removals,
  rotations), propagated lazily through the existing ConfigUpdate-style mechanism.
- Storing only an aggregate commitment (e.g., a Merkle root over the peer's signing-key set) on-chain,
  with individual keys proven on demand at `submitBundle` time. This trades larger per-bundle proof
  overhead for O(1) on-chain state per Channel.
- A hybrid model where a small "active" set of signing keys is stored on-chain (similar to the
  on-chain endpoints list today) and rotated periodically, with the remaining keys verified out-of-band.

Trade-offs include on-chain storage cost, propagation latency for endpoint changes, the size of the
verifiable peer set, and the verifier complexity required for membership proofs. The migration path
from the on-chain-endpoints-only model to a richer mechanism must be specified — likely behind a
protocol version bump so both sides agree on the verification rules before such roster updates can
flow.
