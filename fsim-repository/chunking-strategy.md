# FSIM Chunking Strategy

This document defines a common pattern for transmitting large payloads inside FDO ServiceInfo Modules (FSIMs). The goal is to keep the transport rules consistent across all modules so that devices and owners can share code and expectations.

It covers three layers that any content-transferring FSIM can use:

- **Transport** — how bytes move: begin/data/end, acknowledgment, results, diagnostics.
- **Delivery** — where bytes come from: inline, by URL, or via a meta-payload, and how fetched content is authenticated. See [Delivery Modes](#delivery-modes).
- **Authorization** — who may decide what is transferred: channel vs. artifact authority, Owner vs. Delegate, scope. See [Authorization of Begin Messages](#authorization-of-begin-messages).

## Key Naming Pattern

Given a logical payload named `payload` in FSIM `fdo.example`, the chunking keys follow this pattern:

- `fdo.example:payload-begin` – announces the start of a transfer and provides metadata
- `fdo.example:payload-data-&lt;n&gt;` – payload chunks (0-based index embedded in key name)
- `fdo.example:payload-end` – signals completion of the transfer and may carry final metadata

All chunk keys carry CBOR values. `*-data-<n>` values MUST be CBOR byte strings (`bstr`).

FSIMs MAY also define a `*-result` key that the receiver uses to acknowledge completion of the transfer. Result messages follow a common structure described in [Result Messages](#result-messages).

## Begin Message Structure

The `*-begin` value is a CBOR map that uses small unsigned integer keys for compactness. Each optional field is addressable by its integer key:

### Table: Begin Message Fields

| Key | Field | Type | Description |
| --- | ----- | ---- | ----------- |
| 0 | `total_size` | `uint` | Total bytes that will be transmitted. If omitted, receivers treat the transfer as open-ended until `*-end` arrives. |
| 1 | `hash_alg` | `tstr` | Hash algorithm identifier (e.g., `"sha256"`, `"sha384"`). |
| 2 | `metadata` | `map` | Optional FSIM-specific metadata (format undefined at this layer). |
| 3 | `require_ack` | `bool` | If true, sender waits for `*-ack` before sending data chunks. See [Acknowledgment Gate](#acknowledgment-gate). |
| 4 | `estimated_duration` | `uint` | Advisory: estimated seconds for the complete transfer and application of this payload. See [Estimated Duration](#estimated-duration). |
| 5 | `delivery_mode` | `uint` | `0` = inline (default), `1` = URL, `2` = meta-URL. See [Delivery Modes](#delivery-modes). |
| 6 | `url` | `tstr` | URL of the content (mode 1) or of the meta-payload (mode 2). Required when `delivery_mode` ≠ 0. |
| 7 | `tls_ca` | `bstr` | DER X.509 CA certificate to use as the TLS trust anchor when fetching `url`. See [Authenticating Fetched Content](#authenticating-fetched-content). |
| 8 | `expected_hash` | `bstr` | Hash of the final content, computed with `hash_alg` (key 1; default `"sha256"`). See [Hash Handling](#hash-handling). |
| 9 | `meta_signer` | `bstr` | `COSE_Key` of a third-party publisher whose signature authenticates the meta-payload (mode 2 only). See [Meta-Payload Signing](#meta-payload-signing). |

Reserved Key Policy:

- Keys `0..127` (non-negative) are reserved for this generic chunking spec and future extensions.
- Individual FSIMs MAY define negative integer keys (e.g., `-1`) for their own metadata fields without risking future collisions.
- **Legacy aliases.** `fdo.bmo` historically defined keys `5..9` as negative FSIM keys (`-6` delivery_mode, `-7` url, `-8` tls_ca, `-9` expected_hash, `-10` meta_signer). Receivers implementing `fdo.bmo` SHOULD accept those aliases with identical meaning; senders MUST NOT emit them in new implementations. A `*-begin` carrying both a generic key and its alias with different values MUST be rejected.

#### CDDL Example

```cddl
payload-begin = {
  ? 0: uint,        ; total_size
  ? 1: tstr,        ; hash_alg
  ? 2: any,         ; metadata map (FSIM-defined)
  ? 3: bool,        ; require_ack
  ? 4: uint,        ; estimated_duration (seconds, advisory)
  ? 5: uint,        ; delivery_mode (0=inline, 1=url, 2=meta-url)
  ? 6: tstr,        ; url
  ? 7: bstr,        ; tls_ca (DER X.509 CA certificate)
  ? 8: bstr,        ; expected_hash
  ? 9: bstr         ; meta_signer (COSE_Key)
}
```

## Data Messages

- Each `*-data-&lt;n&gt;` key conveys a consecutive slice of the payload.
- `&lt;n&gt;` MUST be a decimal integer starting at 0 and incrementing by 1 for each subsequent chunk.
- Value MUST be a CBOR `bstr` containing the raw bytes for that slice.
- Chunks SHOULD be ≤ 1014 bytes to align with typical FDO MTU limits, but smaller/larger slices are allowed if both sides agree.

Receivers MUST buffer chunks until all bytes have arrived. The expected byte count is determined by:

1. `total_size` (if provided) – once the sum of chunk lengths equals `total_size`, the payload is complete even if `*-end` has not arrived yet.
2. `*-end` without `total_size` – the transfer completes when the end message is received.

### CDDL Example

```cddl
payload-data = {
  ; key is encoded in ServiceInfo name (payload-data-n)
  payload-chunk: bstr
}
```

## End Message Structure

The `*-end` value is also a CBOR map with unsigned integer keys:

| Key | Field | Type | Description |
| --- | ----- | ---- | ----------- |
| 0 | `status` | `int` | FSIM-specific status code (e.g., 0 = success). |
| 1 | `hash_value` | `bstr` | Hash of the full payload, using the algorithm advertised in `*-begin`. |
| 2 | `message` | `tstr` | Optional human-readable note or error string. |

Reserved Key Policy mirrors the `*-begin` map: non-negative keys are owned by this spec, negative keys may be defined by FSIMs for additional metadata.

An empty map (or even an entirely absent `*-end` body) implies completion with no additional metadata, which most FSIMs interpret as success.

### CDDL Example

```cddl
payload-end = {
  ? 0: int,   ; status
  ? 1: bstr,  ; hash_value
  ? 2: tstr   ; message
}
```

### Hash Handling

A hash can be carried in two places: `expected_hash` in `*-begin` (key 8), or `hash_value` in `*-end` (key 1).

- If `hash_alg` was provided in `*-begin`, the sender MAY defer `hash_value` until `*-end`. This accommodates cases where the hash can only be computed after the final chunk is generated.
- Receivers MUST verify every hash that is present. If both `expected_hash` and `hash_value` are present, both MUST match.
- **`*-end` is never signed.** Under artifact authority a `hash_value` in `*-end` therefore binds nothing to the authorisation. For an **inline** transfer under artifact authority, the hash MUST be `expected_hash` in the signed `*-begin`; a signed inline `*-begin` without it authorises only metadata, not content, and MUST be rejected for any transfer the FSIM declares authorization-gated (see [Authorization of Begin Messages](#authorization-of-begin-messages)). Under channel authority, `hash_value` in `*-end` is acceptable: it arrives over the same authenticated session. Content fetched by reference (modes 1 and 2) is governed instead by [Authenticating Fetched Content](#authenticating-fetched-content).
- For inline transfers, any hash mismatch constitutes a **protocol-level error**. Implementations MUST surface these as TO2 ServiceInfo failures (terminating the FSIM exchange) rather than as FSIM-specific result codes. For content fetched by reference (delivery modes 1 and 2) a mismatch is reported as a transfer result instead — error 11 (see [Transfer Error Codes](#transfer-error-codes)) — because the session itself is intact.

### Length Handling

- `total_size` is optional. Senders that cannot determine the payload size upfront may omit it.
- If `total_size` is provided and the receiver observes more bytes than announced, it MUST treat the transfer as invalid.
- When `total_size` is omitted, receivers rely solely on `*-end` to determine completion.
- If the byte count at completion does not match the declared `total_size`, the discrepancy MUST be treated as the same protocol-level TO2 error described above.

## Delivery Modes

The `delivery_mode` field (key `5`) decides how the content reaches the receiver. Separating *what* to transfer (decided by whoever authorised the `*-begin`) from *where the bytes come from* lets a modest onboarding service direct a large fleet to content hosted on a CDN or published by a third party, without relaying every byte itself.

| Mode | Name | `*-begin` carries | Data chunks | Receiver behaviour |
| ---- | ---- | ----------------- | ----------- | ------------------ |
| 0 | Inline (default) | Transfer metadata | `*-data-<n>` as defined above | Reassemble, verify hash, apply |
| 1 | URL | `url` (6); normally `expected_hash` (8); optionally `tls_ca` (7) | **None** | After `*-end`, fetch `url`, authenticate it, apply |
| 2 | Meta-URL | `url` (6) of a [meta-payload](#meta-payload); optionally `meta_signer` (9), `expected_hash` (8), `tls_ca` (7) | **None** | After `*-end`, fetch and authenticate the meta-payload, then fetch and authenticate the content it names, apply |

Rules common to modes 1 and 2:

- The sender MUST NOT send any `*-data-<n>` chunks. A receiver that receives one MUST abort the transfer.
- `*-end` signals "fetch now." `total_size` (0), if present, describes the fetched content; a size mismatch is reported as error 11.
- `require_ack` (3) behaves as for inline transfers: the receiver may decline before fetching anything, e.g. because it does not support the mode (error 14).
- Every fetched object MUST be authenticated per [Authenticating Fetched Content](#authenticating-fetched-content) before it is applied. A receiver that cannot authenticate it MUST refuse it.
- Receivers SHOULD reject non-`https` URLs unless the object is authenticated by a hash or signature, SHOULD bound redirects and timeouts, and MUST NOT treat a redirect target as more trusted than the original URL.

Inline delivery is REQUIRED for any receiver implementing this chunking strategy. Modes 1 and 2 are OPTIONAL; a receiver that does not implement a requested mode MUST reject with error 14 (or `*-ack [false, 14]` when the gate is in use).

### Why Deliver by Reference

Inline delivery pushes every byte through the onboarding service. URL and meta-URL delivery let the service send only a small `*-begin` while the content comes from a CDN or a publisher's server: the service stays small, devices fetch from a nearby edge, and egress is cheaper at scale. The cost is that the content arrives outside the authenticated session, which is why [Authenticating Fetched Content](#authenticating-fetched-content) exists.

### Offering Alternatives (Fallback)

Receivers differ in which modes they support and in what they can reach at runtime, so a sender MAY offer the **same content** more than once, in preference order:

- **Capability refusal.** With `require_ack` set, a receiver that does not support the offered mode answers `*-ack [false, 14]`. The sender MAY then offer the same content in another mode (typically inline). Senders SHOULD set `require_ack` whenever a fallback exists, so the refusal comes before any transfer.
- **Runtime failure.** A receiver may accept a URL-based offer and then fail to fetch it — no route to the CDN, DNS failure, firewall, timeout. This is an ordinary outcome, not a protocol violation: the receiver reports `*-result` with error 9 (URL Fetch Failed), and the sender MAY offer the content again in another mode. Receivers MAY retry a fetch before reporting failure.
- **Authentication refusal is not a reason to downgrade.** A refusal with error 11, 12, 15 or 19 means the offer could not be trusted. A sender MAY re-offer the content in a form the receiver can authenticate (for example inline, or with `expected_hash` added), but MUST NOT treat the refusal as a capability problem to be worked around with a weaker offer.

A typical ordering is URL or meta-URL first (it scales), inline second (it works everywhere, including air-gapped networks).

## Meta-Payload

In mode 2, `url` points not at the content but at a **meta-payload**: a small manifest that names the content. This lets whoever publishes the manifest change *which* content is installed — a new release, a new mirror — without any change to the `*-begin` messages that point at it.

### Meta-Payload Structure

```cddl
MetaPayload = {
    0  => tstr,     ; mime_type  - MIME type of the content
    1  => tstr,     ; url        - URL of the content
  ? 2  => bstr,     ; tls_ca     - DER CA certificate for fetching `url`
  ? 3  => tstr,     ; hash_alg   - default "sha256"
  ? 4  => bstr,     ; expected_hash - hash of the content
  ? 5  => any,      ; reserved: defined by fdo.bmo as boot_args (legacy numbering)
  ? 6  => tstr,     ; name        - informational
  ? 7  => tstr,     ; version     - informational
  ? 8  => tstr,     ; description - informational
  * int => any      ; negative keys: FSIM-defined
}
```

Keys `0..127` are owned by this specification (key `5` is assigned to `fdo.bmo` for historical reasons); negative keys are FSIM-defined.

For the purposes of [Authenticating Fetched Content](#authenticating-fetched-content), fields fall into three classes:

| Class | Fields | Meaning |
| ----- | ------ | ------- |
| **Pointer** | `mime_type` (0), `url` (1) | Where the content is and what it is |
| **Constraint** | `hash_alg` (3), `expected_hash` (4) | Can only make acceptance *stricter*. Always enforced when present; counts as evidence only when the meta-payload itself is authenticated |
| **Informational** | `name` (6), `version` (7), `description` (8) | Logged or displayed only; MUST NOT influence any decision |
| **Instruction** | `tls_ca` (2), key `5`, and every FSIM-defined (negative) key unless that FSIM explicitly classifies it otherwise | Changes what the receiver trusts or does. A kernel command line, for example, is code execution in its own right. |

### Meta-Payload Signing

A meta-payload is transmitted in one of two forms, distinguished by the leading CBOR item:

- **Unsigned:** the bare `MetaPayload` map.
- **Signed:** a `COSE_Sign1` (CBOR tag 18) whose payload is the CBOR-encoded `MetaPayload`, computed with `external_aad` = CBOR encoding of `["FDO-FSIM-MetaPayload-v1"]`.

A signed meta-payload is verified against exactly one key, chosen as follows:

| `*-begin` carries `meta_signer` (9)? | Meta-payload `x5chain` (label 33)? | Verification key | Intended for |
| --- | --- | --- | --- |
| Yes | Ignored | The `meta_signer` `COSE_Key` | **Third-party publishers.** One manifest, one URL, served to every customer; the authorised `*-begin` names the publisher's key. The publisher needs no FDO certificate. |
| No | Present | Leaf of the chain, which MUST validate to the TO2-proven Owner key and grant `fdo-ekt-permit-provision` (validated as in [Signer](#signer)) | **The Owner's own release process**, signing as a provisioning Delegate |
| No | Absent | The TO2-proven Owner public key | The Owner signing directly |

Rules:

- If `meta_signer` is present, the meta-payload MUST be signed and MUST verify against that key. An unsigned meta-payload, or one that fails verification, MUST be rejected with error 12. There is no fallback to any other key or to the unsigned form.
- If `meta_signer` is absent and the meta-payload is signed, it MUST verify as above or be rejected with error 12. A signed meta-payload is never re-interpreted as unsigned.
- A `meta_signer` named in a `*-begin` is a grant of authority over which content is installed, for as long as that `*-begin` is honoured. It derives that authority entirely from the authority of the `*-begin` that names it. Owners SHOULD name only publishers they would trust with `fdo-ekt-permit-provision`.

## Authenticating Fetched Content

**Every object a receiver fetches by reference — the content in mode 1, and both the meta-payload and the content in mode 2 — MUST be authenticated before it is applied or relied on.** An object is authenticated if at least one of the following holds:

| # | Evidence | What it trusts |
| - | -------- | -------------- |
| 1 | A **hash** of the object, carried in an authorized `*-begin` (`expected_hash`, key 8) or in an *authenticated* meta-payload (key 4), matches the fetched bytes | Exactly these bytes, and nothing else |
| 2 | The object is a meta-payload **signed by the `meta_signer`** named in the authorized `*-begin` | The named publisher |
| 3 | The object is a meta-payload **signed by the Owner, or by a Delegate whose chain grants `fdo-ekt-permit-provision`** | The Owner's chain of authority |
| 4 | The object was fetched over **TLS the receiver actually validated**: certificate chain validated to `tls_ca` (key 7 for the `*-begin` URL; meta-payload key 2 for the content URL, honoured only from an authenticated meta-payload) or to a trusted system store, and the server identity matched the URL host | Whoever operates that server |

If none holds, the receiver MUST refuse the object with error 19 (Unauthenticated Source) and MUST NOT apply it.

Further rules:

- **A hash SHOULD always be provided.** It is the only evidence that pins exact content and the only one that does not extend trust to another party. Omitting it is permitted so that a publisher can update content at a stable URL without every `*-begin` that points there being rewritten, but Owners should understand the cost: evidence #4 delegates the choice of content to the server's operator, depends on a TLS trust anchor that pre-OS environments often lack, and provides no rollback protection. Where flexible updates are the goal, a signed meta-payload (evidence #2 or #3) offers the same flexibility while still pinning each release by hash.
- **TLS counts only if it was really validated.** A receiver that has no TLS implementation, cannot validate certificates, or was given an `http` URL MUST NOT treat the fetch as satisfying evidence #4. Implementations SHOULD demonstrate this with a negative test (wrong CA ⇒ refuse).
- **All present hashes are enforced.** If both the `*-begin` and an authenticated meta-payload carry a hash, the content MUST match both. A meta-payload therefore cannot redirect to content the `*-begin` did not authorise.
- **An unauthenticated meta-payload is only a pointer.** A meta-payload authenticated by none of #2–#4 may still be used when the *content* it points to is authenticated by a hash in the `*-begin` (evidence #1). In that case the receiver uses its **pointer** fields; it MUST still enforce any **constraint** fields (the content must match every hash present), but they do not count as evidence; it MUST reject the meta-payload with error 15 if it carries any **instruction** field (see [Meta-Payload Structure](#meta-payload-structure)); and it MUST NOT let informational fields influence any decision.
- **Decide before downloading.** A receiver SHOULD determine which evidence will apply before fetching the content, so that an unverifiable configuration is refused without downloading content that could never be trusted.

### Decision Summary

| Mode | Begin hash (8) | Meta-payload | Result |
| ---- | -------------- | ------------ | ------ |
| 1 | Present | — | Fetch; content MUST match the hash |
| 1 | Absent | — | Accept only over validated TLS (#4); otherwise error 19 |
| 2 | Any | Signed (#2 or #3) | Meta authenticated; its instruction fields usable. Content MUST match every present hash; with no hash anywhere, content accepted only over validated TLS |
| 2 | Any | Unsigned, over validated TLS (#4) | As above: meta authenticated |
| 2 | Present | Unsigned, not over validated TLS | Pointer only: use `url`/`mime_type`; reject if it carries instruction fields; content MUST match the begin hash (and the meta's hash, if present) |
| 2 | Absent | Unsigned, not over validated TLS | Error 19 |

## Estimated Duration

The `estimated_duration` field (key `4`) is an **advisory** hint from the sender indicating approximately how many seconds the complete operation — transfer *plus* application — is expected to take.

### Motivation

Devices often run internal watchdog timers during onboarding to recover from hangs. A hardcoded watchdog value that works for small payloads can cause spurious resets during large transfers (e.g., a 2.8 GiB ISO image over a slow link), producing silent failures that look like hardware or network problems. The `estimated_duration` field lets the sender communicate its best estimate so the receiver can adjust its watchdog or progress-tracking accordingly.

### Semantics

- **Advisory only.** Receivers MAY ignore this field entirely.
- A value of `0`, or omission of the key, means "no estimate provided." Receivers MUST fall back to their own default timeout policy.
- The value represents the sender's best guess at **total wall-clock seconds** from `*-begin` to completion of any post-transfer processing (e.g., running an installer). It accounts for both transfer time (payload size / expected link speed) and application time.
- Receivers that choose to use this field SHOULD apply a safety margin (e.g., double the value) before using it to set a watchdog or timeout.
- The estimate has two components that different parties may know best:
  - **Application time**: how long the device takes to process the payload after receiving it (known by the payload author).
  - **Transfer time**: how long it takes to deliver the payload over the wire (known by the operator who understands the network conditions).
  - The sender SHOULD combine both into a single conservative estimate.

### Example

A 2.8 GiB ISO image on a 100 Mbit/s link takes approximately 240 seconds to transfer. The Ubuntu installer takes approximately 300 seconds to run. The sender might set `estimated_duration` to `600` (10 minutes). A device with a default 1800-second watchdog would keep its existing timeout. A device with a 300-second watchdog would extend it to `1200` (600 x 2 safety margin).

## Error Handling

- If a chunk is missing or corrupted, the receiver SHOULD discard the entire transfer and report an error using the FSIM's normal error/result key.
- Senders MAY restart a transfer by reissuing `*-begin` with a new sequence of chunks.
- Re-sending a specific `*-data-<n>` chunk overwrites the previously received slice with the same index.
- Hash or length mismatches fall outside FSIM semantics and MUST abort the TO2 ServiceInfo exchange; FSIM-level result codes are reserved for application semantics after a well-formed payload is received.

## Result Messages

Many FSIM payloads expect the receiver to emit a follow-up status once the payload is applied. Chunked payloads SHOULD use a `*-result` key whose value is a CBOR array:

```cbor
[
  status_code,   // int: 0=success, 1=warning, 2=error (FSIM MAY define additional values)
  ? message      // optional tstr: human-readable description or error detail
]
```

## CDDL Example

```cddl
payload-result = [
  status-code: int,
  ? message: tstr
]
```

- The sender of the result (receiver of the payload) MUST set `status_code` appropriately.
- `message` MAY be omitted for success cases.
- FSIMs can extend this structure (e.g., add more array items) but SHOULD preserve the leading status/message order for consistency.
- Interpretation of `status_code`/`message` is FSIM-specific; e.g., "setting not applied" or "certificate rejected" are defined by that FSIM's spec.
- These `*-result` errors MUST NOT be confused with the generic FDO TO2 ServiceInfoModule error mechanism, which is reserved for protocol-level failures (timeouts, transport errors, etc.). Use TO2 errors only when the entire ServiceInfo exchange is compromised, not when a specific FSIM payload fails validation.

### Result Is Terminal

`*-result` MUST be the last message of a transfer. Any supplementary messages the receiver wishes to send — notably [diagnostic payloads](#diagnostic-payloads) — MUST be sent *before* it.

### Completion Ordering

**A sender MUST NOT signal module completion until the receiver's `*-result` has been received, or a local timeout has expired.**

This rule exists because FDO routes an incoming ServiceInfo key to the module that is currently active on the receiving side. A sender that declares itself finished as soon as it has transmitted `*-end` may cause the peer's subsequent `*-result` to arrive after the module has been torn down, where it is discarded — silently, in most implementations. The failure is timing-dependent and most likely precisely when the result is most valuable, since a receiver that must apply a payload before reporting on it takes longest to respond when something has gone wrong.

Senders SHOULD apply a bounded timeout while awaiting `*-result` rather than waiting indefinitely, and SHOULD log its expiry.

## Acknowledgment Gate

Some transfers benefit from explicit acceptance before data transmission begins. This is particularly useful when:

- The payload is large and the receiver may not support the content type
- The receiver needs to validate metadata (MIME type, size, permissions) before accepting data
- Multi-stage onboarding scenarios where payloads intended for one stage should not be sent to another

### Enabling the Gate

When `require_ack` (key 3) is set to `true` in the `*-begin` message, the sender MUST wait for a `*-ack` message before sending any `*-data-<n>` chunks.

### Ack Message Structure

The `*-ack` message uses a CBOR array format:

```cddl
payload-ack = [
  accepted: bool,       ; true = proceed with transfer, false = rejected
  ? reason_code: uint,  ; FSIM-specific rejection reason (when accepted=false)
  ? message: tstr       ; Human-readable explanation
]
```

| Index | Field | Type | Description |
| ----- | ----- | ---- | ----------- |
| 0 | `accepted` | `bool` | `true` to proceed, `false` to reject |
| 1 | `reason_code` | `uint` | Optional FSIM-specific code explaining rejection |
| 2 | `message` | `tstr` | Optional human-readable explanation |

### Protocol Flow

**With acknowledgment (accepted):**

```
Sender → Receiver: payload-begin { 3: true, ... }
Receiver → Sender: payload-ack [true]
Sender → Receiver: payload-data-0
Sender → Receiver: payload-data-1
...
Sender → Receiver: payload-end
Receiver → Sender: payload-result
```

**With acknowledgment (rejected):**

```
Sender → Receiver: payload-begin { 3: true, -1: "application/x-iso9660-image" }
Receiver → Sender: payload-ack [false, 1, "Unsupported MIME type"]
                   ; Transfer cancelled - no data chunks sent
```

**Without acknowledgment (backward compatible):**

```
Sender → Receiver: payload-begin { ... }  ; require_ack absent or false
Sender → Receiver: payload-data-0         ; proceeds immediately
...
```

### Requirements

- Senders MUST NOT send `*-data-<n>` chunks until `*-ack` is received when `require_ack` is true
- Receivers MUST send `*-ack` promptly after receiving a `*-begin` with `require_ack: true`
- If `*-ack` contains `accepted: false`, the sender MUST NOT send any data chunks
- The sender MAY attempt a different payload (new `*-begin`) after rejection
- Reason codes are FSIM-specific; common codes should be documented in each FSIM spec

### Implementation Notes

For library/framework implementations, this feature implies an **accept/reject callback** that applications can use to validate incoming transfers before data arrives:

```
// Pseudocode for device-side FSIM handler
type PayloadHandler interface {
    // Called when begin message arrives with require_ack=true
    // Return (true, 0, "") to accept, or (false, code, msg) to reject
    OnPayloadBeginAck(metadata BeginMessage) (accept bool, reasonCode uint, message string)
    
    // Called after transfer completes (existing callback)
    OnPayloadComplete(data []byte) error
}
```

This allows application code to inspect MIME types, sizes, or other metadata and reject transfers that don't apply to the current execution context.

## Diagnostic Payloads

`*-result` carries a status code and an optional `message` string, and that message is bounded by the negotiated ServiceInfo MTU. The MTU floor is small, and implementations generally treat an oversized ServiceInfo array as a hard error rather than truncating it — so an overlong diagnostic does not merely get clipped, it fails the exchange. That makes `*-result` unsuitable for anything larger than a single-line summary.

Where a receiver needs to return substantial diagnostic output — installer logs, interpreter tracebacks, validation reports, command output — FSIMs SHOULD define a `*-log` payload transferred using the **same chunking mechanism in the reverse direction**.

Nothing in this document is direction-specific: `*-begin` / `*-data-<n>` / `*-end` are defined in terms of *sender* and *receiver*, not owner and device. A diagnostic transfer simply inverts those roles — the receiver of the primary payload becomes the sender of the log.

### Requirements

- The `*-log` transfer MUST follow the ordinary begin/data/end rules defined above.
- It MUST be sent **before** `*-result`, which remains the terminal message of the transfer (see [Result Is Terminal](#result-is-terminal)).
- The log sender SHOULD set `require_ack` in `*-log-begin`, allowing the peer to decline the transfer before any data is sent. Diagnostic output can be far larger than the payload that produced it, and a peer that will not retain it should not pay to receive it.
- A peer that does not wish to receive the log responds `*-log-ack [false, 5]` (Diagnostics Not Requested). The log sender MUST then proceed directly to `*-result`.
- Logs are supplementary. A device that cannot afford a chunked upload MUST still report `*-result` correctly. Implementations MUST NOT depend on the log transfer to determine success or failure.

### Begin Metadata

The `*-log-begin` message uses only the generic keys defined in [Begin Message Structure](#begin-message-structure). Log-specific attributes travel in the `metadata` map (key 2):

| Metadata key | Type | Description |
| ------------ | ---- | ----------- |
| `content_type` | `tstr` | MIME type of the log, e.g. `"text/plain"`, `"application/json"`. Defaults to `"text/plain"` when absent. |
| `truncated` | `bool` | True if the sender truncated the output to fit a local size cap. |
| `source` | `tstr` | Origin of the output, e.g. `"stderr"`, `"journal"`, `"installer"`. |

### Rejection Reason Code

This specification reserves one additional `*-ack` reason code for use with diagnostic transfers:

| Code | Meaning |
| ---- | ------- |
| 5 | Diagnostics not requested — the peer does not want the log |

### Protocol Flow

```text
Owner → Device: payload-begin { ... }
Owner → Device: payload-data-0 .. payload-data-N
Owner → Device: payload-end { 1: h'...' }

; Device applied the payload and failed; it has 40 KB of installer output
Device → Owner: payload-log-begin { 0: 40960, 1: "sha256", 3: true,
                                    2: { "content_type": "text/plain",
                                         "source": "installer" } }
Owner → Device: payload-log-ack [true]
Device → Owner: payload-log-data-0 .. payload-log-data-N
Device → Owner: payload-log-end { 1: h'...' }

; Result last, and terminal
Device → Owner: payload-result [2, "autoinstall failed; see log"]
```

Declined:

```text
Device → Owner: payload-log-begin { 0: 40960, 3: true, ... }
Owner → Device: payload-log-ack [false, 5, "Diagnostics not collected"]
Device → Owner: payload-result [2, "autoinstall failed"]
```

## Authorization of Begin Messages

The `*-begin` message is the point at which a transfer is authorized. For FSIMs that deliver security-sensitive content — bootable images (`fdo.bmo`), OS configuration bundles (`fdo.payload`), firmware parameters (`fdo.bmo:set`) — the message is subject to **authorization** before the receiver accepts and acts on it.

This section is the **normative** authorization model for every FSIM that uses it. Individual FSIMs declare which of their messages are authorization-gated (see [FSIM Declarations](#fsim-declarations)) and otherwise reference this section. The model applies equally to a gated message that is not itself a `*-begin` (e.g. `fdo.bmo:set`).

> For a high-level explanation of why these modes exist and how they combine with delegation and delivery modes, see the article *Trust the Message, or Trust the Messenger? Security and Authority in FDO Bare-Metal Onboarding*, and [Provisioning Security: Authorizing What Gets Installed on Your Devices](../../go-fdo/provisioning-security.md).

### Rationale

TO2 already settles whether the peer is authentic. The question here is different: **where does the authority to install this particular thing come from?** There are two defensible answers, and both are supported because they serve different deployments:

- **Channel authority — trust the messenger.** Authority is established once, during TO2, by the peer proving it holds the Owner key, or by presenting an Owner-signed Delegate chain stating which permissions the Owner granted it. Everything the peer subsequently says carries that authority, bounded by those permissions. This covers the Owner running its own onboarding service — the simple case, which should stay simple — and an Owner that has deliberately granted a third party `fdo-ekt-permit-provision`.
- **Artifact authority — trust the message.** The Owner provisions *through* a party it does not fully trust: a managed onboarding service, a CDN, an integrator. That party is authorised to *onboard*, but the Owner decides *what is installed*. The authorisation is therefore a self-contained signed object that survives passage through the conduit: minted by a provisioning authority, naming the content (and optionally the device), and impossible for the conduit to forge, alter or retarget.

#### Why permitting channel authority is not a weakening

For the peers eligible to use it, channel authority grants nothing a signature would not:

- A peer that proved possession of the **Owner key** did so by signing `TO2.ProveOVHdr` with it. Requiring it to sign the `*-begin` as well asks it to demonstrate a capability it has just demonstrated.
- A peer holding a Delegate certificate granting `fdo-ekt-permit-provision` could mint a valid artifact over arbitrary content at will. Letting it assert the same thing over the authenticated channel changes nothing about what it can cause to be installed.

Conversely, a peer that is **not** eligible — most importantly a Delegate holding only `fdo-ekt-permit-onboard-*` permissions — cannot use channel authority at all, and must relay an artifact minted by someone who can. That is precisely the separation the Owner wanted when it issued a narrow certificate.

#### Both modes are scoped, but they scope different things

- **The channel scopes the actor:** "this party may provision." It is a property of the peer, established once, at the start of the session. It also supplies two properties for free: **device binding** (the session is with this device) and **freshness** (the session is live and nonce-protected; nothing can be replayed from it).
- **The artifact scopes the act:** "this content, on this device, until this date." It is a detached bearer object, so it loses device binding and freshness; [Scope Constraints](#scope-constraints) exist to restore them.

| | Channel authority | Artifact authority |
| --- | --- | --- |
| Scopes the actor's permissions | Yes — PERM OIDs proven in TO2 | Yes — PERM.7 on the signing certificate |
| Device binding | Implicit in the session | Explicit (`guid`), or absent |
| Freshness / replay resistance | Implicit in the session | Explicit (`not_after`, `generation`), or absent |
| Content fixed by the authorising party | No — peer chooses within its permissions | Yes — fixed at signing time |
| Deciding party must be online at onboarding | Yes | No — may be minted offline, ahead of time |
| Conduit must be trusted to choose content | Yes | **No** |
| Onboarding party may hold only `fdo-ekt-permit-onboard-*` | No | Yes |
| Durable record of what authorised the install | No | Yes |

### Conformance

Requirements for receivers implementing any authorization-gated message. Note the distinction between **implementing** a capability and **enabling** it by policy.

| Capability | Requirement | Notes |
| ---------- | ----------- | ----- |
| Determining the TO2 peer's provisioning authority | **MUST** | Falls out of TO2 processing. MUST be recorded explicitly — see [Channel Authority](#channel-authority). |
| Channel authority | **MUST** implement, **MUST** be enabled by default | An Owner relies on it working out of the box. |
| Artifact authority, Owner-direct signature | **MUST** | One `COSE_Sign1` verification against the TO2-proven Owner key. |
| Artifact authority, Delegate `x5chain` | **SHOULD** | Requires X.509 chain validation. A receiver that does not implement it MUST reject `x5chain`-bearing artifacts with error 15 and MUST document the limitation. |
| `guid` scope evaluation | **MUST** if artifact authority is implemented | |
| `not_before` / `not_after` evaluation | **SHOULD** | Requires a trustworthy clock. See [Device Clocks](#device-clocks). |
| `generation` evaluation | **MAY** | Requires rollback-protected storage. |
| Strict policy (require artifacts even from an authorised peer) | **MAY** | Operator-enabled tightening. MUST default to off and MUST be documented. |
| Rejecting rather than ignoring a constraint it cannot evaluate | **MUST** | Applies to every OPTIONAL row above. |

A constrained receiver may decline to implement time or generation checking, but it may **not** pretend it did. Optional to implement, mandatory to fail closed: that combination is what lets an Owner attach a constraint without surveying its fleet.

### Trust Anchor

The trust anchor for every authorization signature is the **Owner public key proven during TO2**: the key in the final entry of the Ownership Voucher, verified as part of `TO2.ProveOVHdr` processing. No separate trust root is introduced.

An implementation MUST NOT use the TO2 peer's public key, the TO2 session key, or any other channel-bound material as the trust anchor when verifying a **signature**. A Delegate that signed `TO2.ProveOVHdr` on the Owner's behalf is not thereby a provisioning authority.

### Signer

An artifact MUST be signed by exactly one of:

1. **Owner-direct** — the Owner private key. The `COSE_Sign1` carries **no** `x5chain`; the receiver verifies against the TO2-proven Owner key.
2. **Delegated** — a Delegate key. The `COSE_Sign1` MUST carry an `x5chain` unprotected header (label `33`, RFC 9360) of DER certificates, **leaf first**, whose topmost certificate is issued by the Owner key. The chain MUST grant `fdo-ekt-permit-provision` (OID `1.3.6.1.4.1.45724.3.1.7`, PERM.7) under the usual FDO rule: a permission is granted only when present in **every** certificate of the chain. Chain validation follows the FDO delegate rules (every issuing certificate is a CA; the leaf is not). A chain valid for other FDO purposes but not granting PERM.7 MUST NOT be accepted.

The signer need not be the party that transmits the message, and in the conduit deployments this mode exists for, it is not. A PERM.7 signing key need not hold any onboard permission; an offline signing key that never runs TO2 is a recommended arrangement.

### Channel Authority

A receiver accepts an unsigned gated message on the authority established during TO2. It MUST determine the **provisioning authority of the TO2 peer** as a single decision made during TO2 processing and retained for the session:

- `TO2.ProveOVHdr` carried no Delegate chain and verified directly against the Owner key ⇒ the peer **has** provisioning authority.
- `TO2.ProveOVHdr` carried a Delegate chain that validated to the Owner key ⇒ the peer has provisioning authority **if and only if** that chain grants `fdo-ekt-permit-provision`.
- Otherwise ⇒ the peer does **not** have provisioning authority.

This determination MUST be recorded from **how `ProveOVHdr` was verified**. It MUST NOT be inferred from the availability of the Owner key: the receiver knows the Owner key from the Ownership Voucher whether the peer is the Owner or a Delegate, and inferring authority from it silently promotes every onboard-only Delegate to Owner authority.

A receiver whose policy disables channel authority, or facing a peer without provisioning authority, MUST reject an unsigned gated message with error 15.

A receiver MAY offer a **strict** policy requiring artifact authority even from a peer that holds provisioning authority, for an Owner that delegates broadly but wants every installation pinned to a device and bounded in time. It MUST be off unless explicitly enabled, MUST NOT be implied by any other setting, and MUST be documented.

#### No downgrade

If a gated message **is** a tagged `COSE_Sign1`, the receiver MUST complete artifact verification and MUST NOT, on any failure, fall back to channel authority — not when the signature is invalid, the `x5chain` fails to validate, a scope constraint is unmet, or the algorithm is unsupported. A failed artifact is a rejected message, never an unsigned one. (A conduit cannot usefully *remove* an envelope either: doing so turns the message into a channel-authority assertion, which succeeds only if the conduit itself holds provisioning authority — in which case it could have minted its own artifact.)

### Selecting the Authorization Mode

| Leading CBOR item | Interpretation |
| ----------------- | -------------- |
| Tag 18 (`0xD2`) | Artifact authority. Verify per [Verification Algorithm](#verification-algorithm). |
| A valid head of the inner payload type the FSIM declared for the message (e.g. `map`, `array`) | Channel authority. Accept only if policy permits **and** the peer has provisioning authority. |
| Anything else | Malformed; error 15. |

Senders SHOULD use the artifact form whenever they are able; it is a superset of the channel form in expressiveness and the only form that survives an untrusted conduit.

### `COSE_Sign1` Structure

```cddl
COSE_Sign1_Tagged = #6.18([
    protected:   bstr .cbor protected_header,
    unprotected: unprotected_header,
    payload:     bstr .cbor InnerPayload,
    signature:   bstr
])

protected_header = {
    1: alg,                      ; ES256, ES384, ...
    3: content_type,             ; MUST equal the content type the FSIM registered for this message
    ? "fdo.scope": Scope         ; see Scope Constraints
}

unprotected_header = {
    ? 33: bstr / [+ bstr]        ; x5chain, leaf first. Present IFF the signer is a Delegate.
}
```

Scope is carried in the **protected** header so it is signed, and as a header rather than a payload field so it applies uniformly whatever the inner payload type. Scope in the unprotected header is unsigned and MUST be ignored. Receivers implementing `fdo.bmo` MUST also accept the legacy label `"fdo.bmo.scope"` with identical meaning; an artifact carrying both labels MUST be rejected.

### External AAD (Domain Separation)

Every artifact signature MUST be computed with `external_aad` set to the CBOR encoding of `[tag]`, where `tag` is the domain-separation string the FSIM registered for its authorized messages (e.g. `"FDO-FSIM-BmoProvision-v1"`). Signed meta-payloads use `"FDO-FSIM-MetaPayload-v1"`. Distinct tags ensure that a signature minted for one purpose cannot be replayed as another — as a different FSIM's authorization, as a meta-payload, or as any FDO protocol structure (`TO0.OwnerSign`, `TO2.ProveOVHdr`, …). Receivers MUST verify with the identical tag.

### Scope Constraints

An artifact is a bearer object: anything holding a copy can present it. Scope narrows where and when it is valid:

```cddl
Scope = {
    ? "guid"       => bstr / [+ bstr],   ; device binding: 16-byte FDO GUID(s)
    ? "not_before" => uint,              ; seconds since 1970-01-01T00:00:00Z (UTC)
    ? "not_after"  => uint,              ; seconds since 1970-01-01T00:00:00Z (UTC)
    ? "generation" => uint               ; monotonic supersession counter
}
```

All fields are OPTIONAL; constraints are cumulative. An absent field is an absent constraint — an artifact without `guid` is valid fleet-wide, which is exactly right for one instruction relayed to every device (e.g. "install whatever this publisher's manifest says"), and a serious over-grant when the Owner meant one device.

- **`guid`** is compared against the GUID in the Ownership Voucher header proven during TO2 — *not* any replacement GUID from `TO2.SetupDevice`, which an Owner minting offline cannot know. Receivers MUST retain the voucher GUID for the session. An array matches if any member matches. Mismatch ⇒ error 16.
- **`not_before` / `not_after`** bound validity in time; outside the window ⇒ error 17. Only as trustworthy as the receiver's clock — see [Device Clocks](#device-clocks).
- **`generation`** is a counter scoped to (Owner key, device). A receiver implementing it records the highest accepted value in **non-volatile, rollback-protected** storage (TPM NV under policy, authenticated UEFI variable, secure monotonic counter) and rejects any lower value with error 18, updating the stored value before acting on the payload. Battery-backed RAM, ordinary UEFI variables, unauthenticated NV, and files are unsuitable: a counter that can be reset fails *permissively*, which is worse than no counter. A receiver without suitable storage MUST NOT claim to implement `generation`.

#### Unevaluable constraints

A receiver MUST NOT silently ignore a scope field it does not implement or cannot evaluate in its current state; it MUST reject the message — error 17 for a time constraint without a trustworthy clock, error 18 for `generation` without rollback-protected storage, error 15 for an unrecognised field. An Owner that adds a constraint has made the authorisation conditional; accepting it while discarding the condition grants more than the Owner granted, invisibly. The remedy is on the Owner's side: mint the artifact without constraints the fleet cannot evaluate (in practice, `guid` alone is the universally supported set).

*Open issue:* there is no wire mechanism for a receiver to advertise which constraints it can evaluate. A future revision should add one.

### Device Clocks

Pre-OS environments rarely have a clock worth trusting: dead RTC batteries return epoch defaults, many platforms have no RTC at all, and network time is controlled by exactly the attacker a time gate is meant to resist. A physical attacker can always move the clock backwards. A validity window is therefore **a supersession control against a remote conduit, not a tamper-resistance control against a local attacker**; deployments needing the latter MUST use `generation`.

A receiver SHOULD classify its clock as trustworthy only if it has a functioning RTC *and* the reading is plausible (at minimum, not earlier than the build time of the code performing the check), and SHOULD keep a monotonic floor of the greatest accepted timestamp in rollback-protected storage. If the clock is untrustworthy and an artifact carries a time constraint, it MUST reject with error 17 — never accept with the window ignored. It SHOULD apply one consistent policy to artifact validity windows and to certificate validity dates in an `x5chain`.

### Verification Algorithm

On receipt of a gated message, the receiver MUST execute the following before acting on the inner payload:

1. Decode the body.
    1. Tag 18 ⇒ artifact authority; continue at step 2. Having entered this branch, the receiver MUST NOT leave it except by acceptance at step 8 or by an error (no downgrade).
    2. The declared inner payload type ⇒ channel authority. Accept only if policy permits **and** the peer has provisioning authority; otherwise error 15. If accepted, skip to step 8.
    3. Otherwise ⇒ error 15.
2. Parse the `COSE_Sign1`. The protected `content_type` (label 3) MUST be present and equal the content type registered for the incoming message key; otherwise error 15.
3. Inspect `x5chain` (label 33).
    1. **Absent** ⇒ verification key = TO2-proven Owner key. Skip to step 5.
    2. **Present** ⇒ validate the chain to the Owner key per [Signer](#signer); on failure, error 15.
4. The chain MUST grant `fdo-ekt-permit-provision`; otherwise error 15. Verification key = leaf public key.
5. Verify the signature with that key and the FSIM's registered `external_aad`; on failure, error 15.
6. Evaluate scope, if present — **after** the signature, so no decision rests on unauthenticated bytes: unevaluable ⇒ 17/18/15 as above; `guid` mismatch ⇒ 16; outside time window ⇒ 17; `generation` below high-water mark ⇒ 18.
7. Decode the verified payload as the declared inner payload type; on failure, error 15.
8. Commit any monotonic state (`generation`, timestamp floor) **before** acting on the payload, then proceed. For delivery modes 1 and 2, continue with [Authenticating Fetched Content](#authenticating-fetched-content).

Receivers MUST NOT process the inner payload before step 8; any read-ahead state MUST be discarded on failure.

### Owner / Delegate Requirements

A sender using **artifact authority**:

- MUST wrap the inner payload in a tagged `COSE_Sign1` as above, with the registered `content_type` and `external_aad`.
- MUST, when signing as a Delegate, include the `x5chain` (leaf first, up to but not including the Owner key); MUST NOT include `x5chain` when signing as the Owner directly.
- MUST place scope in the protected header.
- MUST, for inline transfers, include `expected_hash` (key 8); SHOULD include it in every mode (see [Authenticating Fetched Content](#authenticating-fetched-content)).
- SHOULD NOT emit a validity window to a fleet known to lack trustworthy clocks; use `generation`.

A sender delivering **through a conduit it does not fully trust** MUST use artifact authority, and SHOULD constrain device-specific artifacts with `guid`, and every artifact with `not_after` and/or `generation` so it can be retired.

A sender relying on **channel authority** MUST hold the Owner key or a Delegate chain granting `fdo-ekt-permit-provision`.

### FSIM Declarations

An FSIM that uses this model MUST declare, for each authorization-gated message:

| Item | Example (`fdo.bmo`) |
| ---- | ------------------- |
| Message key | `fdo.bmo:image-begin` |
| Inner payload type (the channel-authority form) | `map` (`ImageBegin`) |
| Registered `content_type` | `application/cbor+fdo.bmo.image-begin` |
| `external_aad` domain-separation tag | `"FDO-FSIM-BmoProvision-v1"` |
| Classification of any FSIM-defined meta-payload keys | key `5` (`boot_args`): instruction |

FSIMs whose messages change device state — install software, alter configuration, provision credentials — SHOULD declare those messages authorization-gated. An FSIM that accepts state-changing content without authorization MUST say so explicitly, because in that configuration an onboard-only peer is effectively a provisioning authority.

## Transfer Error Codes

FSIMs that use delivery modes 1–2 or authorization-gated messages MUST reserve the following codes, with these meanings, in their result / error / `*-ack` reason codes. (FSIM-specific codes occupy `1..8`.)

| Code | Name | Meaning |
| ---- | ---- | ------- |
| 9 | URL Fetch Failed | Network error, timeout, HTTP error |
| 10 | TLS Validation Failed | TLS certificate validation failed |
| 11 | Hash Mismatch | Fetched content does not match a required hash, or its size does not match `total_size` |
| 12 | Meta Signature Invalid | Meta-payload signature missing (when `meta_signer` named) or invalid |
| 13 | Meta Parse Error | Meta-payload malformed or missing required fields |
| 14 | Delivery Mode Not Supported | Requested delivery mode not implemented |
| 15 | Provisioning Not Authorized | Failed the [Verification Algorithm](#verification-algorithm): unsigned without channel authority, `content_type` mismatch, `x5chain` invalid or lacking PERM.7, bad signature, unrecognised scope field, malformed payload; or an unauthenticated meta-payload carrying instruction fields |
| 16 | Provisioning Scope Mismatch | `guid` does not match the voucher GUID |
| 17 | Provisioning Validity Failed | Outside `not_before`/`not_after`, or no trustworthy clock to evaluate them |
| 18 | Provisioning Superseded | `generation` below the recorded high-water mark, or `generation` not evaluable |
| 19 | Unauthenticated Source | A fetched object is authenticated by no hash, signature, or validated TLS |

## Integration Notes

- This strategy mirrors the patterns already proposed in `fdo.sysconfig` and other FSIM drafts but centralizes the rules so future modules stay consistent.
- FSIM specifications should reference this document instead of redefining chunk semantics. They only need to specify the logical payload names (e.g., `cert-res`, `payload`, `config-file`) and the meaning of optional metadata or result structures.
- Devices SHOULD implement generic helpers that accept a namespace/payload name and assemble chunks automatically based on the key naming convention.
