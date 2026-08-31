# Removing Message Redaction

This ADR removes redaction from the specification.  Because bundles contain messages that are secured by the running 
hash and not directly state proven, CLPR Endpoints have the capability of redacting unredacted messages or restoring 
redacted messages that are not the anchor message.  CLPR Endpoints should not have this ability to determine whether 
or not a message is redacted.  

**Changes in this ADR:**
- The protobuf supporting redacted messages is removed. 
- The running hash computation is simplified. 

---

## 1. Problem

### 1.1 Redacted and non-redacted encodings are hash-equivalent by construction

At enqueue, a message's hash contribution to the chain is `payload_hash = SHA-256(serialized_payload)`.
When a message is later redacted, its slot's payload is replaced with a `ClprRedactedMessage` marker
carrying `message_hash = SHA-256(serialized_original_payload)` — the same hash, over the same bytes,
computed the same way — and the slot's `running_hash_after_processing` is explicitly left unchanged. The
running hash chain therefore carries no information distinguishing "the original payload was hashed
into this slot" from "the redacted marker's hash field was supplied directly." Both produce the exact
same chain value, for anyone who has ever observed the original payload.

Only the bundle's anchor message has a state-proven `anchor_running_hash`; every preceding message is
accepted solely because a hash chain recomputed from submitter-supplied bytes reaches that one proven
value. That trust extension is sound only if each slot's chain contribution uniquely determines which
encoding occupied it. It does not. 

### 1.2 Bundle submission is permissionless, so the ambiguity is directly exploitable

`redactMessage` is gated to the CLPR Service admin. `submitBundle` is not gated at all — any
authenticated caller may submit a bundle, a property the specification relies on elsewhere (e.g., any
party recovering a channel after both endpoints are lost). Because the chain cannot distinguish the two
encodings, this permissionless submitter — not only a compromised admin or a compromised endpoint
implementation — can independently choose, for any not-yet-acknowledged message it has ever observed,
to substitute a redacted marker for a message the admin never redacted, or the original payload for a
message the admin did redact. Both substitutions verify successfully.


---

## 2. Options Considered and Non-Goals

### Options Considered

State proving every message is one option. This would drop the running hash completely, but the costs of state proving 
additional items is costly in computation time and space.  For this reason we are keeping the running hash for its 
efficiency and dropping the ability to redact messages.   

### Non-Goals

Redaction could have functioned as a form of claw-back mechanism to attempt to undo initiating a state transition.
Preserving any kind of alternate uses for redaction is not a goal. 

---

## 3. The Solution

### 3.1 Remove redaction from the message model

`ClprRedactedMessage` is removed. `ClprMessagePayload`'s `redacted_message` variant is removed.

```protobuf
message ClprMessagePayload {
  oneof payload {
    ClprMessage message = 1;             // Data Message — application content
    ClprMessageReply message_reply = 2;  // Response Message — outcome of a Data Message
    ClprControlMessage control = 3;      // Control Message — protocol management
  }
}
```

`ClprMessageReplyStatus.REDACTED` and `ClprMessageReplyStatus.CHANNEL_CLOSED` are removed.  Reserved fields in protobuf
are unnecessary until the final spec is released to the public.  

```protobuf
enum ClprMessageReplyStatus {
  SUCCESS = 0;
  APPLICATION_ERROR = 1;
  CONNECTOR_NOT_FOUND = 2;
  CONNECTOR_UNDERFUNDED = 3;
}
```

`redactMessage` is removed from the pseudo-API surface entirely. The "**Redaction.**" handling in the
Message Lifecycle section is removed: a Data Message now has exactly one path from enqueue to deletion —
delivery and a matched response, or one of the existing failure statuses — with no marker substitution
possible. Response Ordering Verification's fallback to `ClprRedactedMessage.sender` is removed; the
originating application address is always read from `ClprMessage.sender`, since every retained Data
Message slot now always holds its original `ClprMessage`.

### 3.2 Simplify the running hash formula

The intermediate payload-hash step existed to give a redacted slot a value to transmit in place of an
absent payload. With no redacted slots, it serves no purpose:

```
running_hash = SHA-256(previous_running_hash || serialized_payload)
```

`ClprMessageValue.running_hash_after_processing`'s comment simplifies accordingly — it is the cumulative
hash after processing this message, with no note about redacted slots, since none exist. This changes
the running hash algorithm and requires a protocol version bump and Channel renegotiation, as already
specified for hash algorithm upgrades.

---

## 4. Testing

- No operation matching `redactMessage`'s shape (admin-authorized, channel_id + message_id) exists in
  the pseudo-API surface.
- A Data Message's only reachable terminal outcomes are delivery-and-matched-response, or one of
  `APPLICATION_ERROR`, `CONNECTOR_NOT_FOUND`, `CONNECTOR_UNDERFUNDED` — `REDACTED` is never generated.
- Running hash computed via the simplified single-hash formula matches between source and destination
  across a representative message sequence, including Control and Response Messages.
- A Channel negotiated under the old (double-hash) protocol version and one under the new (single-hash)
  version MUST NOT interoperate without a version bump and renegotiation.
- Response Ordering Verification (§4.5) resolves the originating application address from
  `ClprMessage.sender` for every retained Data Message slot, with no fallback branch reachable.

---

## Appendix A: Patching the Spec

### `clpr-service-spec.md`

| Location | Change |
|---|---|
| §1.1 `ClprLedgerConfiguration` | Close the field 6 gap: remove the "Field 6 (endpoints) removed" comment; renumber `initial_trust_anchor` 7→6, `initial_trust_anchor_id` 8→7. |
| §1.1 `ClprThrottles` | Close field gaps at 2 and 7: renumber `max_message_payload_bytes` 3→2, `max_gas_per_message` 4→3, `max_queue_depth` 5→4, `max_sync_bytes` 6→5, `max_local_endpoints` 8→6, `max_peer_endpoints` 9→7. |
| §1.4 `ClprMessageValue` | `payload` comment: remove "or Redacted". `running_hash_after_processing` comment: replace the two-step formula and the redacted-slot note with `SHA-256(previous_running_hash \|\| serialized_payload)`. |
| §1.4 `ClprMessagePayload` | Remove `ClprRedactedMessage redacted_message = 4`. |
| §1.4 `ClprRedactedMessage` | Remove the entire message definition and its block comment. |
| §1.4 `ClprMessageReplyStatus` | Remove `REDACTED = 4` and `CHANNEL_CLOSED = 5` with their comments. |
| §4.1 Running Hash Computation | Replace the two-step formula with `running_hash = SHA-256(previous_running_hash \|\| serialized_payload)`. Remove the sentence explaining the two-step form was deliberate. |
| §4.2 Bundle Verification Step 4 | Replace the conditional (non-redacted / redacted branches) with the single formula `SHA-256(prev_hash \|\| serialized_payload)` applied uniformly to every slot. |
| §4.3 Message Enqueue Step 7 | Replace `payload_hash = SHA-256(serialized_payload)` then `running_hash = SHA-256(sent_running_hash \|\| payload_hash)` with `running_hash = SHA-256(sent_running_hash \|\| serialized_payload)`. |
| §4.4 section title | "Message Lifecycle and Redaction" → "Message Lifecycle". |
| §4.4 Redaction paragraph | Remove the entire **Redaction.** paragraph. |
| §4.5 Step 1 | Remove "treating each redacted slot as if the original Data Message were still present". |
| §4.5 Step 3 | Remove the sentences attributing the sender address to `ClprRedactedMessage.sender` as a fallback. Address is always read from `ClprMessage.sender`. |
| §4.6 Slashing table | Remove the `REDACTED` row from the source-side table. |
| §5.3 Administrative Operations | Remove the **Redact** bullet. |
| §6.4 Messaging | Remove the `redactMessage` pseudo-API block. |

### `clpr-service.md`

| Location | Change |
|---|---|
| §2.4 CLPR Service Admin role summary | "can configure, close, and redact, but cannot…" → "can configure and close, but cannot…" |
| §3.1.4 Endpoint Communication Protocol | Update cross-reference anchor: `#327-message-lifecycle-and-redaction` → `#327-message-lifecycle`. |
| §3.2.2 Message Storage | Running hash formula in prose: `SHA-256(previous_running_hash ‖ SHA-256(message_payload))` → `SHA-256(previous_running_hash ‖ serialized_payload)`. |
| §3.2.7 section title | "Message Lifecycle and Redaction" → "Message Lifecycle". |
| §3.2.7 Redaction paragraph | Remove the paragraph beginning "CLPR also supports **redaction** of Data Message payloads…". |
| §3.3.4 Failure Consequences and Slashing | "`SUCCESS` and `REDACTED` are non-penalized outcomes." → "`SUCCESS` is the non-penalized success outcome." |
| §4.6 CLPR Service Admin section body | "configure, close, and redact" → "configure and close". |
| §4.6 CLPR Service Admin operations list | Remove the **Redact** bullet. |
