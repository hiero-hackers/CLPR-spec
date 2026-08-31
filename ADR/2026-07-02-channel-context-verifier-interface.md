# CLPR ChannelContext Verifier Interface

This document proposes the introduction of `ChannelContext` to the CLPR verifier interface. It describes the gap in the current interface, the alternatives considered, the proposed solution, and enumerates every change required to the existing specification files.

---

## 1. Problem

The CLPR verifier interface defines two cryptographic operations that a verifier contract must implement: `verifyConfig`, called once when a Channel is established, and `verifyBundle`, called on every bundle submission for the lifetime of the Channel.

In the current specification, `verifyBundle` accepts two arguments: `bundle_payload` (the opaque state proof from the source ledger) and `trust_anchor` (the current signing authority material). This is sufficient for the verifier to authenticate that the proof was produced by the expected signing authority. It is not sufficient for the verifier to target the correct on-chain state or to authenticate that a proof belongs to the correct Channel and peer CLPR Service.

The Channel identifier and the peer CLPR Service address are fixed at `completeChannel` time and never change. The CLPR Service holds them on the `Channel` struct. Neither is passed to the verifier — `verifyBundle` has no argument that carries either value. This creates two related but distinct problems.

**EVM and QBFT verifiers cannot perform correct storage slot derivation.** These verifiers validate bundle proofs against Merkle-Patricia-Trie state. The storage slot keys and account proof paths they must inspect are derived from `channel_id` and `service_address`. Channel identity is not embedded in QBFT bundle proofs at all — the proof contains signed block headers and Merkle paths but no explicit channel identifier. Without `channel_id` and `service_address` as explicit inputs, a QBFT verifier has no way to derive the correct storage locations. This is not a security gap alone; it is a functional gap. A reusable verifier contract that services multiple channels cannot correctly target per-channel state without being told which channel it is verifying.

**Bundle provenance can be silently misattributed on Hiero, with a narrow residual gap.** On Hiero, `channel_id` is embedded in the storage keys of the state proof. However, those keys are dropped when the verifier extracts and returns only the message values. A caller could submit a bundle claiming to be for Channel A while the `bundle_payload` is a valid Hiero state proof for Channel B's queue. If both channels share the same trust anchor — which is the case for any two channels to the same remote Hiero network, since the trust anchor is the `ledger_id` — the verifier passes cryptographic verification and returns Channel B's messages without detecting the substitution.

The CLPR Service does have an independent backstop: after `verifyBundle` returns, it validates the `sent_running_hash` in the returned `ClprQueueMetadata` against its own stored running hash for Channel A. In practice, this catches most substitution attempts, because Channel A and Channel B accumulate divergent queue histories quickly. The residual gap is narrow but real: at the start of a channel's lifetime, when both queues are empty and the running hash is at its zero state, a crafted substitution that exploits that window could pass both the verifier and the running hash check. The interface provides no explicit mechanism for the verifier to assert provenance, and the responsibility for catching cross-channel substitution is implicitly left to the running hash — a data-integrity check not designed to serve as a provenance defense.

---

## 2. Options Considered

### 2.1 Embed Channel Identity in the Trust Anchor

The opening question in the design discussion was whether `channel_id` and `service_address` should be added into the trust anchor itself. The trust anchor is already mutable state passed through the verifier interface; baking channel identity into it would give the verifier everything it needs without changing the method signatures.

This was rejected on conceptual grounds. The trust anchor is the signing authority — the material that authenticates who produced a proof. Channel identity is a different kind of information: it identifies the relationship, not the authority. Mixing the two conflates mutable signing-authority material with immutable channel-scoped identity. It also creates a practical problem: the trust anchor can rotate during the lifetime of a Channel, but the channel identity cannot. Encoding identity inside a field that is designed to change makes it impossible to distinguish a legitimate rotation from an attempted identity substitution. The right answer, as identified during the discussion, was to extract channel identity into a separate, explicitly immutable field rather than baking it into a field whose mutation semantics serve a different purpose.

### 2.2 Opaque Platform-Dependent Context Bytes

Once a separate context field was agreed upon in principle, the question became whether it should be opaque bytes — platform-dependent, analogous to how the trust anchor itself is opaque — or a named struct with specified fields. The opaque approach would allow each verifier implementation to define its own context format.

This was set aside because `channel_id` and `service_address` are the same across all verifier types. The values the CLPR Service assembles are not platform-specific: they come from the Channel record using the same mechanism regardless of ledger type. Making the context opaque would add complexity with no benefit — there is nothing platform-dependent to encode. If the fields are uniform across the system, specifying them explicitly is strictly clearer.

### 2.3 Dynamic Assembly by the CLPR Service on Each Call

A related question was whether the context should be stored statically on the Channel and retrieved at call time, or assembled dynamically by the CLPR Service each time `verifyBundle` is invoked. Under the dynamic approach, the CLPR Service would read the relevant fields from the Channel record on each bundle submission and pass them freshly constructed to the verifier, without a dedicated stored field.

The static approach was preferred for a straightforward reason: the trust anchor is mutable and can change between calls; the channel identity is not and cannot. Storing them differently reflects their different semantics. If the CLPR Service assembles context dynamically, it is doing the same work on every bundle submission for no gain — the values never change. Storing `channel_context` as an immutable field on the Channel makes the invariant explicit in the data model and avoids re-derivation on each call.

### 2.4 Named `ChannelContext` Struct with Specified Fields

The chosen approach. A `ChannelContext` struct carries `channel_id` and `service_address` as its two fields. It is returned by `verifyConfig` as part of its output, stored on the `Channel` struct as an immutable field (`channel_context`), and passed to `verifyBundle` as a third argument on every subsequent call.

This names the concept — constant, channel-scoped identity data, distinct from mutable signing-authority material — and makes it a first-class part of the interface. `verifyConfig` gains `channel_id` as an input so the verifier can authenticate that the configuration proof belongs to the specific Channel being established; it returns `ChannelContext` bundling that echoed `channel_id` with the `service_address` it extracts from the verified proof. The CLPR Service stores this once at `completeChannel` and passes it unchanged to every `verifyBundle` call thereafter.

The interface is uniform across all verifier implementations. EVM/QBFT verifiers use both fields to derive storage slot keys and account proof paths. Hiero verifiers authenticate `service_address` directly from the `ClprChannel` leaf in the state proof and may use the context fields for additional validation. Neither implementation requires a bifurcated interface.

---

## 3. Proposed Solution

### `ChannelContext` — a Named Struct for Constant Channel Identity

`ChannelContext` is a new struct carrying the two pieces of data that identify a Channel and its peer CLPR Service:

```
struct ChannelContext {
  bytes channel_id    // 32-byte Channel identifier, same on both ledgers
  bytes service_address  // on-ledger address of the peer's CLPR Service
}
```

Both fields are fixed at `completeChannel` time and never change for the lifetime of the Channel. They are not trust material — they carry no signing authority and cannot be rotated — which is why they are kept separate from the `trust_anchor` / `trust_anchor_id` fields. `ChannelContext` is the constant half of the verifier's inputs; the trust anchor is the mutable half.

### `verifyConfig` — Input and Output Changes

`verifyConfig` gains `channel_id` as a second input argument. This allows the verifier to authenticate that the configuration proof belongs to the specific Channel being established — the channel_id is not derivable from the configuration proof itself, which proves the peer's ledger configuration, not the local channel request.

`service_address`, which was previously returned as a bare `bytes` field, is now returned as part of `ChannelContext`. The verifier echoes `channel_id` from input into the returned `ctx.channel_id` and populates `ctx.service_address` from the verified proof. The CLPR Service stores the returned `ChannelContext` as `Channel.channel_context`, which is immutable thereafter.

The signature changes from:

```
function verifyConfig(bytes proof_bytes)
  returns (string chain_id, bytes service_address, uint96 peer_config_nanos,
           ClprThrottles throttles, bytes initial_trust_anchor,
           bytes initial_trust_anchor_id, ClprEndpoint[] endpoints)
```

to:

```
function verifyConfig(bytes proof_bytes, bytes channel_id)
  returns (ChannelContext ctx, string chain_id, uint96 peer_config_nanos,
           ClprThrottles throttles, bytes initial_trust_anchor,
           bytes initial_trust_anchor_id, ClprEndpoint[] endpoints)
```

### `verifyBundle` — Third Argument

`verifyBundle` gains `ChannelContext ctx` as a third argument. The CLPR Service passes `Channel.channel_context` — the same value stored at `completeChannel` — on every call. The value is never mutated.

The signature changes from:

```
function verifyBundle(bytes bundle_payload, bytes trust_anchor)
  returns (ClprQueueMetadata, ClprMessagePayload[], bytes new_trust_anchor, bytes new_trust_anchor_id)
```

to:

```
function verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)
  returns (ClprQueueMetadata, ClprMessagePayload[], bytes new_trust_anchor, bytes new_trust_anchor_id)
```

### `Channel` Struct — New Field

The `Channel` struct gains:

```
channel_context : ChannelContext  // constant data returned by verifyConfig; passed
                                        // to every verifyBundle call; immutable after
                                        // completeChannel. See §3.1.
```

### Responsibility Clarification

With `ctx` available as an explicit input, the verifier is now solely responsible for validating that the bundle data belongs to the (`trust_anchor`, `ctx.service_address`, `ctx.channel_id`) triple. The CLPR Service does not re-validate this after the call returns. All other data-integrity checks — running-hash chain, message ordering, replay defense — remain the CLPR Service's responsibility and are performed after `verifyBundle` returns.

### Per-Implementation Behavior

The `ChannelContext` is passed uniformly. Each verifier consumes only the fields its proof system requires:

- **EVM / QBFT verifiers** use both `channel_id` and `service_address` to derive the storage slot keys and account proof paths required to validate Merkle-Patricia-Trie state proofs against the peer CLPR Service contract.
- **Hiero verifiers** authenticate `service_address` directly from the `ClprChannel` leaf in the Hiero state proof. The proof itself asserts the correct leaf; `ctx` fields may be used for additional validation but are not the primary authentication mechanism.

---

## 4. Code Artifacts

The following are the normative definitions to be inserted or referenced in the spec verbatim. Each block corresponds to one change table entry in Section 5.

---

### 4.1 `ChannelContext` struct — new definition (`clpr-service-spec.md` §3.1)

Insert before the `IClprVerifier` interface block:

```
`ChannelContext` carries the constant, channel-scoped data that identifies which Channel a
bundle belongs to. It is assembled by the CLPR Service from fields fixed at `completeChannel` time
and is passed to every verifier call unchanged for the lifetime of the Channel. The interface is
uniform across all verifier implementations; individual verifiers consume only the fields their proof
system requires. EVM/QBFT verifiers use `channel_id` and `service_address` to derive storage slot
keys and account proof paths when validating Merkle-Patricia-Trie state proofs. Hiero verifiers
authenticate via state proof: the proof directly authenticates the specific `ClprChannel` leaf's
`service_address` field rather than deriving it from `ChannelContext`.

struct ChannelContext {
  bytes channel_id    // 32-byte Channel identifier, same on both ledgers
  bytes service_address  // on-ledger address of the peer's CLPR Service
}
```

---

### 4.2 `IClprVerifier.verifyConfig` — updated signature (`clpr-service-spec.md` §3.1)

Replace the existing `verifyConfig` function block with:

```
  // Verify a configuration proof and return the peer's verified configuration.
  //
  // Inputs:
  //   proof_bytes    : bytes      — opaque proof produced by the source ledger's proof system
  //                                 (state roots, Merkle paths, etc.).
  //   channel_id  : bytes(32)  — the Channel identifier. Included verbatim in the returned
  //                                 ChannelContext so verifiers can authenticate that the proof
  //                                 belongs to this specific Channel.
  //
  // Returns:
  //   ctx               : ChannelContext — constant channel-scoped data assembled from
  //                                 the verified proof: channel_id (echoed from input) and
  //                                 service_address (on-ledger address of the peer CLPR Service).
  //                                 Stored on the Channel and passed to verifyBundle on every
  //                                 subsequent bundle submission.
  //   chain_id          : string  — CAIP-2 identifier of the peer ledger.
  //   peer_config_nanos : uint96  — config version timestamp in nanoseconds since Unix epoch.
  //   throttles         : ClprThrottles — capacity limits of the peer ledger.
  //   initial_trust_anchor     : bytes — initial signing authority; empty if the verifier has
  //                                 no rotating-authority concept.
  //   initial_trust_anchor_id  : bytes — opaque identifier for initial_trust_anchor. MUST be
  //                                 non-empty iff initial_trust_anchor is non-empty.
  //   endpoints         : ClprEndpoint[] — initial peer endpoint roster.
  //
  // MUST revert if verification fails.
  function verifyConfig(bytes proof_bytes, bytes channel_id)
    returns (ChannelContext ctx, string chain_id, uint96 peer_config_nanos,
             ClprThrottles throttles, bytes initial_trust_anchor,
             bytes initial_trust_anchor_id, ClprEndpoint[] endpoints)
```

---

### 4.3 `IClprVerifier.verifyBundle` — updated signature (`clpr-service-spec.md` §3.1)

Replace the existing `verifyBundle` function block with:

```
  // Verify a bundle payload against the Channel's current trust anchor and context.
  //
  //   bundle_payload   : bytes             — state proof and message data (format is verifier-specific).
  //   trust_anchor     : bytes             — the Channel's current signing-authority material.
  //                                         Mutable: updated whenever a prior verifyBundle call
  //                                         returned a non-empty new_trust_anchor. Empty for
  //                                         verifiers with no rotating-authority concept.
  //   ctx              : ChannelContext — constant channel-scoped data (channel_id,
  //                                         service_address). The same value stored by
  //                                         completeChannel; never mutated. Each verifier
  //                                         consumes only the fields its proof system requires.
  //
  // The verifier is responsible for validating that the bundle data belongs to the
  // (trust_anchor, ctx.service_address, ctx.channel_id) triple. Other data-integrity
  // checking (running-hash chain, message ordering, replay defense) is performed by the
  // CLPR Service after this call returns and is out of scope for the verifier.
  //
  //   ClprQueueMetadata    — verified queue state.
  //   ClprMessagePayload[] — ordered message payloads proven by the bundle.
  //   new_trust_anchor     : bytes — successor signing authority if the bundle contains rotation
  //                          evidence; empty bytes if none.
  //   new_trust_anchor_id  : bytes — opaque identifier for new_trust_anchor. MUST be non-empty
  //                          iff new_trust_anchor is non-empty.
  //
  // MUST revert if verification fails.
  // SHOULD fail fast on obviously malformed inputs before expensive cryptographic operations.
  function verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)
    returns (ClprQueueMetadata, ClprMessagePayload[], bytes new_trust_anchor, bytes new_trust_anchor_id)
```

---

## 5. Changes to Existing Specification Files

The following tables enumerate every change required to implement this proposal. Line numbers are approximate references to the current file state and are provided to identify the target location; they may shift as surrounding content is edited.

### `clpr-service-spec.md`

| Section | Approx. Lines | Description of Change |
|---|---|---|
| §2.1 `Channel` struct | ~491 | Add `channel_context : ChannelContext` field after `trust_anchor_id`. Comment: constant data returned by `verifyConfig` and passed to every `verifyBundle` call; immutable after `completeChannel`. |
| §3.1 `IClprVerifier` preamble | ~732–740 | Update the paragraph describing mutable vs. constant data passed to verifiers. Replace "per-channel trust material... is passed in and returned via `trust_anchor` / `new_trust_anchor` / `new_trust_anchor_id`" with language distinguishing mutable trust material (passed via trust anchor fields) from constant channel data (passed via `ChannelContext`). Insert the `ChannelContext` struct definition and its explanatory paragraph before the `IClprVerifier` interface block. |
| §3.1 `IClprVerifier.verifyConfig` signature | ~815–816 | Replace `verifyConfig(bytes proof_bytes)` with `verifyConfig(bytes proof_bytes, bytes channel_id)`. Replace the bare `bytes service_address` return value with `ChannelContext ctx` as the first return value. Update the method comment block to document the `channel_id` input and the `ctx` return value. |
| §3.1 `IClprVerifier.verifyBundle` signature | ~860 | Replace `verifyBundle(bytes bundle_payload, bytes trust_anchor)` with `verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)`. Update the method comment block to document `ctx`, clarify that `trust_anchor` is mutable while `ctx` is constant, and state that the verifier is responsible for validating bundle provenance against the (trust_anchor, ctx.service_address, ctx.channel_id) triple. |
| §4.2 Step 1 (bundle processing — verifier call) | ~1066–1069 | Update the verifier call description to pass `channel_context` as the third argument: `verifyBundle(bundle_payload, trust_anchor, channel_context)`. Add a note that `channel_context` is constant for the lifetime of the Channel; `trust_anchor` is mutable. |
| §5.1.3 `completeChannel` Step 3 | ~1315 | Update the `verifyConfig` call to `verifyConfig(config_proof_bytes, channel_id)`. Update the return value description: replace `(chain_id, service_address, ...)` with `(ctx, chain_id, ...)`. |
| §5.1.3 `completeChannel` Step 4 | ~1318–1325 | Add storage of the returned `ChannelContext` as `Channel.channel_context` (immutable thereafter). Keep existing text for `initial_trust_anchor` and `initial_trust_anchor_id` storage. |
| §5.1.3 `completeChannel` Step 6 | ~1325 | Add `channel_context` to the list of fields set when the Channel transitions from PENDING to ACTIVE. |
| §6.2 `completeChannel` pseudocode — Step 3 | ~1428 | Update `verifyConfig(config_proof_bytes)` to `verifyConfig(config_proof_bytes, channel_id)`. Update the return value description in the comment from `(chain_id, service_address, ...)` to `(ctx, chain_id, ...)`. |
| §6.2 `completeChannel` pseudocode — Step 4 | ~1430–1433 | Add `Stores ctx as Channel.channel_context (immutable thereafter).` before the existing trust anchor storage sentence. Update the comment to note that both trust anchor fields are empty bytes for verifiers with no rotating-authority concept. |
| §6.2 `completeChannel` pseudocode — Step 6 | ~1435 | Add `channel_context` to the PENDING → ACTIVE transition description. |
| §6.2 `completeChannel` parameter comment | ~1448 | Update the `config_proof_bytes` comment to say it is passed to `verifyConfig()` to obtain `(ctx, chain_id, throttles, timestamp, endpoints)` — replacing the old `(chain_id, service_address, ...)` description. |

### `clpr-service.md`

| Section | Approx. Lines | Description of Change |
|---|---|---|
| §3.1.5 Verifier Contracts — `verifyConfig` entry | ~703–719 | Update the `verifyConfig` signature to `verifyConfig(bytes proof_bytes, bytes channel_id) → (ChannelContext ctx, chain_id, ...)`. Replace the description of `service_address` as a bare return value with a description of `ChannelContext` as the structured return value carrying both `channel_id` (echoed) and `service_address` (proven). Describe `completeChannel` storing this as `Channel.channel_context` (immutable thereafter) and passing it to every `verifyBundle` call. |
| §3.1.5 Verifier Contracts — `verifyBundle` entry | ~720–728 | Update the `verifyBundle` signature to `verifyBundle(bytes bundle_payload, bytes trust_anchor, ChannelContext ctx)`. Add description of `ctx` as the constant channel-scoped data, distinguish it from the mutable `trust_anchor`, and state that the verifier is responsible for validating bundle provenance against the (trust_anchor, ctx.service_address, ctx.channel_id) triple. Note that EVM/QBFT verifiers use both fields; Hiero verifiers authenticate via state proof. |
| §3.2.4 Trust anchor rotation description | ~767 | Update `verifyBundle(bundle_payload, trust_anchor)` to `verifyBundle(bundle_payload, trust_anchor, channel_context)` in the in-band rotation description. |
| §3.2.5 Bundle verification sequence diagram | ~1107 | Update the `verifyBundle` call in the sequence diagram to `verifyBundle(proof_bytes, trust_anchor, channel_context)`. |
| §3.2.5 Bundle verification prose | ~1129–1135 | Update the description of the verifier call to include `channel_context` as a third argument. Add a sentence distinguishing the constant `channel_context` from the mutable `trust_anchor`. Update the description of what the verifier does: uses context to locate the correct on-chain state and trust anchor to authenticate it. |
