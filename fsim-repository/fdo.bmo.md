# FDO Service Info Module: fdo.bmo

**Version:** 1.0 (Draft)
**Status:** Specification Draft

Copyright &copy; 2026 Dell Technologies and FIDO Alliance
Author: Brad Goodman, Dell Technologies

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

---

## Overview

**Module Name**: `fdo.bmo`
**Version**: 1.0
**Status**: Draft

The `fdo.bmo` (Bare Metal Onboarding) FSIM enables delivery of bootable images and BIOS configuration to device firmware. It combines payload delivery (like `fdo.payload`) with firmware settings (like `fdo.sysconfig`), but is designed exclusively for pre-OS firmware environments.

## Multi-Asset Handling and NAK Fallback

### Multiple Boot Assets Strategy

BMO FSIM can receive **multiple boot assets** in a single session. This enables sophisticated fallback strategies where the server offers different boot options and the firmware selects the first one it can handle.

#### NAK-Based Selection Process

When the server presents multiple boot assets:

1. **Server presents first asset** (most preferred option)
2. **Firmware checks MIME type compatibility**
   - If supported: Firmware sends `image-ack [true]` and receives the asset
   - If unsupported: Firmware sends `image-ack [false, 1, "Image type not supported"]`
3. **Server presents next asset** (next preferred option)
4. **Process repeats** until firmware accepts an asset
5. **First accepted asset terminates BMO phase** - firmware boots it immediately

#### Server Presentation Order

**Servers SHOULD present boot assets in order of preference:**

| Preference | Typical Asset Type | Rationale |
|------------|-------------------|-----------|
| 1 (Best) | `application/efi` | UEFI applications are lightweight, fast to boot |
| 2 | `application/x-iso9660-image` | Standard ISO boot images |
| 3 (Fallback) | `application/x-raw-disk-image` | Raw disk images (most cumbersome) |

**Example preference hierarchy:**

```text
1. UEFI App (application/efi)
   ↓ NAK if not supported
2. ISO Image (application/x-iso9660-image)
   ↓ NAK if not supported
3. Raw Disk (application/x-raw-disk-image)
   ↓ NAK if not supported
4. PXE/iPXE (last resort)
```

#### BMO Phase Termination

**Critical behavior**: The **first successfully received and executed boot asset terminates the BMO phase**:

- Firmware accepts asset via `image-ack [true]`
- Asset is transferred and verified
- Firmware sends `image-result [0, "Booting..."]`
- **Firmware immediately chainloads/boots the asset**
- **FDO session ends** - no further FSIM processing occurs

This ensures that:

- **Only one boot asset is ever executed** per FDO session
- **The best available option is used** (first in preference order)
- **No unnecessary transfers occur** after successful boot

#### No-Op Completion

The converse of termination-by-boot is the case where TO2 completes successfully but **nothing was installed, booted, or configured**. This occurs when the Owner holds the device's voucher but has no provisioning content to deliver — no boot asset, no BIOS parameters — and therefore activates no module that performs work.

By the letter of the FDO protocol this is a *successful* TO2: mutual authentication succeeded, the ServiceInfo exchange completed, and `Done`/`Done2` were exchanged. Treating that as "onboarding complete" is wrong, because the device is in **exactly the state it started in**.

**Normative rule**: A device MUST NOT record onboarding as complete on the basis of TO2 protocol success alone. Completion is a function of **work performed**, not protocol outcome.

Specifically, an implementation of this module:

- MUST track whether any BMO operation actually took effect during the session — an image accepted and chainloaded, or at least one `set` parameter successfully applied.
- MUST NOT set any persistent "onboarding complete" indicator if no such operation occurred.
- SHOULD treat the session as a **no-op** and re-attempt onboarding, subject to the platform's normal retry policy.

This rule is decidable locally: the firmware always knows whether it chainloaded an asset or mutated a BIOS setting.

**Why this belongs in `fdo.bmo` specifically.** This module is the pre-OS, firmware-resident case. A device in this state typically has no OS installed and has FDO boot as its only viable boot target, so the platform's ordinary boot path leads back into FDO on the next power cycle. Making the no-op rule explicit turns that from accidental behavior into specified behavior, and prevents an implementation from recording spurious completion and thereby stranding the machine with no bootable target.

**Relationship to deferred onboarding.** Where the Owner knows it will have content *later*, it should say so explicitly via the [`fdo.defer`](fdo.defer.md) module rather than relying on this fallback. The no-op rule remains the required safety net for devices that do not implement `fdo.defer`, for Owners that send no directive at all, and for deferrals that are never resolved — in all three cases the device falls back to "nothing happened, try again."

#### Implementation Benefits

This multi-asset approach provides:

1. **Graceful degradation**: If firmware doesn't support the preferred format, it falls back automatically
2. **Broad compatibility**: Single server can support diverse firmware capabilities
3. **Optimal selection**: Firmware gets the best boot method it supports
4. **Efficient transfers**: Only one asset transferred per successful session

#### Protocol Example

```
Owner → Device: image-begin { -1: "application/x-iso9660-image", 3: true }
Device → Owner: image-ack [false, 1, "ISO not supported"]

Owner → Device: image-begin { -1: "application/efi", 3: true }
Device → Owner: image-ack [true]

[Transfer EFI application...]

Device → Owner: image-result [0, "Booting EFI app..."]
[Firmware chainloads EFI app, FDO session ends]
```

### Delivery Mode Fallback

The NAK fallback mechanism applies to **delivery modes** just as it does to MIME types. Owners MAY offer the **same image** via different delivery modes, and devices accept the first supported option.

#### Same Image, Multiple Delivery Methods

- Devices **SHOULD support all delivery modes** (inline, url, meta-url)
- Devices **MAY support only a subset** of delivery modes
- Owners present options in **preference order**; device accepts first supported option

#### Example: Owner Prefers Inline (Cached Image)

If an Onboarding Service has an image cached locally, it may prefer inline delivery (faster, no external dependency). If the device doesn't support inline for large images, the owner falls back to URL:

```text
1. Owner → Device: image-begin {
     -1: "application/x-raw-disk-image",
     -6: 0,                              // inline
     0: 524288000,                       // 500MB
     3: true
   }
   Device → Owner: image-ack [false, 3, "Image too large for inline transfer"]

2. Owner → Device: image-begin {
     -1: "application/x-raw-disk-image",
     -6: 1,                              // url (same image!)
     -7: "https://images.example.com/rhel9.dd",
     3: true
   }
   Device → Owner: image-ack [true]
   ; Device downloads from URL
```

#### Example: Owner Prefers URL (Bandwidth Savings)

Conversely, if the owner prefers URL delivery but the device (e.g., firmware without network stack) doesn't support it:

```text
1. Owner → Device: image-begin {
     -1: "application/efi",
     -6: 1,                              // url preferred
     -7: "https://images.example.com/boot.efi",
     3: true
   }
   Device → Owner: image-ack [false, 14, "URL delivery not supported"]

2. Owner → Device: image-begin {
     -1: "application/efi",
     -6: 0,                              // inline fallback
     3: true
   }
   Device → Owner: image-ack [true]
   ; Owner sends image-data-* chunks
```

#### Combined MIME Type and Delivery Mode Fallback

The full preference hierarchy can combine both MIME types and delivery modes:

```text
1. Try inline EFI (fastest, most compatible)
   → NAK "EFI not supported"

2. Try URL-referenced ISO (avoids large inline transfer)
   → NAK "URL delivery not supported"

3. Try inline ISO (fallback)
   → ACK, transfer begins
```

### CDN and Cloud Scaling Use Cases

#### Why URL Delivery for Scale-Out

While inline delivery is optimal for local installations (avoiding over-the-top network traffic), URL-based delivery enables **cloud and CDN scale-out**:

- **CDN Distribution**: Large images hosted on CDNs (Akamai, CloudFront, Azure CDN) can serve thousands of devices simultaneously without overloading the Onboarding Service
- **Geographic Optimization**: CDNs route devices to nearest edge nodes, reducing latency
- **Bandwidth Offload**: Onboarding Service only sends small `image-begin` messages; heavy lifting is done by CDN infrastructure
- **Cost Efficiency**: CDN bandwidth is often cheaper than direct server egress at scale

**Typical deployment pattern:**

```text
Owner (Onboarding Service)          CDN / Cloud Storage
         |                                   |
         | image-begin { url: CDN }          |
         |---------------------------------->| Device
         |                                   |
         |                                   |<-- Device fetches from CDN
         |                                   |
         | image-result [0, "success"]       |
         |<----------------------------------|
```

#### Runtime Network Accessibility Fallback

**Critical concept**: Devices may **accept** a URL-based delivery mode but **fail at runtime** when attempting to fetch. This is a normal error case, not a protocol violation.

**Common scenarios:**

- Device is on an isolated network without public internet access
- Firewall blocks outbound HTTPS to CDN domains
- DNS resolution fails for external URLs
- Network timeout due to congestion or routing issues

**Protocol behavior:**

When a device accepts URL delivery (`image-ack [true]`) but subsequently fails to fetch:

1. Device attempts to download from URL after receiving `image-end`
2. Download fails (timeout, DNS failure, connection refused, etc.)
3. Device sends `image-result` with error code 9 (URL Fetch Failed)
4. Owner MAY present an alternative delivery method (e.g., inline fallback)
5. Process continues until device successfully receives image or all options exhausted

**Example: CDN Preferred, Inline Fallback**

```text
; Owner prefers CDN for bandwidth efficiency
Owner → Device: image-begin {
  -1: "application/x-iso9660-image",
  -6: 1,                              // url
  -7: "https://cdn.example.com/rhel9.iso",
  -9: h'abc123...',
  3: true
}
Device → Owner: image-ack [true]      // Device accepts URL mode

Owner → Device: image-end {}          // Signal to fetch

; Device attempts download but fails (no internet access)
Device → Owner: image-result [9, "URL fetch failed: connection timeout"]

; Owner falls back to inline delivery (same image)
Owner → Device: image-begin {
  -1: "application/x-iso9660-image",
  -6: 0,                              // inline fallback
  3: true
}
Device → Owner: image-ack [true]

; Owner sends image-data-* chunks directly
Owner → Device: image-data-0..N
Owner → Device: image-end { 2: h'abc123...' }

Device → Owner: image-result [0, "Image received, booting"]
```

**Key points:**

- **Error code 9** (URL Fetch Failed) after `image-ack [true]` indicates runtime failure, not capability rejection
- **Owner SHOULD be prepared** to fall back to inline when URL fails
- **Device MAY retry** URL fetch before reporting failure (implementation-defined)
- **This is expected behavior** in mixed-network environments where some devices have internet access and others don't

#### Deployment Recommendations

**For large-scale deployments:**

1. **Primary**: URL delivery via CDN (scales to thousands of devices)
2. **Fallback**: Inline delivery for devices without internet access

**For isolated/air-gapped networks:**

1. **Primary**: Inline delivery (no external dependencies)
2. **Alternative**: URL to internal image server (if available)

**For mixed environments:**

1. **Primary**: URL to CDN (optimistic - most devices have internet)
2. **Fallback**: Inline (handles devices without internet access)

This approach maximizes efficiency for the common case while gracefully handling edge cases.

## ServiceInfo Module Key-Value Pairs

### Module Activation

| Key | Direction | Type | Description |
| --- | --------- | ---- | ----------- |
| `fdo.bmo:active` | Bidirectional | Boolean | Module activation status |

### Image Transfer (Boot Images, Certificates)

| Key | Direction | Body on the wire | Purpose |
| --- | --------- | ---------------- | ------- |
| `fdo.bmo:image-begin` | Owner → Device | Signed envelope, or bare `ImageBegin` map (see below) | Announces an image or certificate transfer. Because accepting this message causes the device to install a bootable image or enrol a UEFI DB/DBX entry, it is subject to [Authorization of Provisioning Messages](#authorization-of-provisioning-messages): normally a COSE_Sign1 envelope, or a bare map where the device permits channel authority. |
| `fdo.bmo:image-ack` | Device → Owner | CBOR array, unsigned | Accept or reject the transfer (only sent when `require_ack` was set in `image-begin`). |
| `fdo.bmo:image-data-<n>` | Owner → Device | CBOR byte string, unsigned | Data chunk number `n` (0-based). Integrity of the reassembled image is bound back to the signed `image-begin` through its `expected_hash` field (key `-9`). |
| `fdo.bmo:image-end` | Owner → Device | CBOR map, unsigned | Signals that all chunks have been sent (inline delivery) or that the device should now fetch from the URL (URL / meta-URL delivery). |
| `fdo.bmo:image-result` | Device → Owner | CBOR array `[status, ?message]`, unsigned | Final outcome reported by the device. |

#### Wire body of `fdo.bmo:image-begin` — CDDL

On the wire, the body of `fdo.bmo:image-begin` is **a single CBOR item**. Under artifact authority — the normal case, and the only one that works through a conduit the Owner does not fully trust — that item is a `COSE_Sign1` (RFC 9052, §4) wrapped in CBOR tag 18, as specified below. Under channel authority it is instead the bare `ImageBegin` map, which a device accepts only if configured to and only from a TO2 peer holding provisioning authority. The CBOR tag is the discriminator: tag 18 means the envelope form, and a device that has entered the envelope path MUST NOT fall back to the bare form. See [Authorization of Provisioning Messages](#authorization-of-provisioning-messages).

```cddl
; === Wire body (full CBOR item) ===
fdo.bmo.image-begin-body = #6.18(COSE_Sign1_ImageBegin)

; === COSE_Sign1 envelope ===
COSE_Sign1_ImageBegin = [
    protected   : bstr .cbor ImageBeginProtectedHeader,  ; header map, serialised
    unprotected : ImageBeginUnprotectedHeader,           ; header map, inline
    payload     : bstr .cbor ImageBegin,                 ; the INNER payload
    signature   : bstr                                   ; alg-defined sig bytes
]

; === Protected header (serialised as a bstr in position 0 above) ===
ImageBeginProtectedHeader = {
    1 => int,                                           ; alg  (ES256=-7, ES384=-35, RS256=-257, ...)
    3 => "application/cbor+fdo.bmo.image-begin",        ; content_type (MUST be this exact string)
    ? "fdo.bmo.scope" => BmoScope                       ; device / validity binding -- see §Scope Constraints
}

; === Unprotected header ===
ImageBeginUnprotectedHeader = {
    ? 33 => [+ bstr]     ; x5chain (RFC 9360):
                         ;   ABSENT  => signer is the Owner (Owner-direct).
                         ;   PRESENT => signer is a Delegate; bstr values are
                         ;              DER X.509 certs, leaf first, chaining
                         ;              up to (but not including) the Owner key.
}

; === Inner payload — what the signature actually covers ===
; This is the SAME ImageBegin map defined in §ImageBegin — see there for the
; full key table. Excerpted here for clarity:
ImageBegin = {
    ? 0  => uint,                    ; total_size (bytes)
    ? 1  => tstr,                    ; hash alg   (e.g. "sha256")
      -1 => tstr,                    ; image_type (REQUIRED, MIME type)
    ? -2 => tstr,                    ; boot_args
    ? -3 => tstr,                    ; name
    ? -4 => tstr,                    ; version
    ? -5 => tstr,                    ; description
    ? -6 => uint,                    ; delivery_mode (0=inline,1=url,2=meta-url)
    ? -7 => tstr,                    ; url
    ? -8 => bstr,                    ; tls_ca (single DER cert)
    ? -9 => bstr,                    ; expected_hash
    ? -10 => bstr                    ; meta_signer (COSE_Key)
}

; === External AAD (NOT on the wire; MUST be fed into sign/verify) ===
; Per §External AAD, the Sig_structure's `external_aad` field is the CBOR
; encoding of the following array. Verifiers that use a different value MUST
; reject the signature.
FdoBmoProvisionAAD = ["FDO-FSIM-BmoProvision-v1"]
```

**Worked example — Owner-direct signature over a minimal `image-begin`.**

Inner payload (`ImageBegin`) as a diagnostic notation:

```cbor-diag
{ -1: "application/efi", 3: true }
```

The full wire body would be laid out as (annotated):

```text
D2                                  # CBOR tag 18  (== COSE_Sign1)
   84                               # array(4) -- COSE_Sign1 has 4 elements
      43                            # bstr, length 3  (protected header bytes)
         A2                         #   map(2)
            01 26                   #     1 => -7          ; alg = ES256
            03 78 26 "application/cbor+fdo.bmo.image-begin"
                                    #     3 => content_type
      A0                            # map(0)              ; unprotected header
                                    #   (empty => Owner-direct; no x5chain)
      58 19                         # bstr, length 25     ; payload bytes
         A2                         #   map(2)            ; ImageBegin inner
            20 70 "application/efi" #     -1 => image_type
            03 F5                   #      3 => true      ; require_ack
      58 40 <64 bytes>              # bstr, length 64     ; ES256 signature
```

For a Delegate signature, the unprotected header would be `A1 18 21 82 <leaf-der> <intermediate-der>` (`{33: [leaf, intermediate]}`) instead of `A0`, and the signature would be computed with the Delegate's private key. Everything else is identical.

**Signing / verification input.** The `COSE_Sign1` signature is computed and verified over the CBOR encoding of `Sig_structure` per RFC 9052 §4.4:

```cddl
Sig_structure = [
    context       : "Signature1",
    body_protected: bstr,                               ; == protected header bstr above
    external_aad  : bstr .cbor FdoBmoProvisionAAD,      ; see §External AAD
    payload       : bstr                                ; == payload bstr above
]
```

The device MUST use `FdoBmoProvisionAAD` (not an empty bstr, not some other tag) when computing the `Sig_structure`; a mismatch causes verification to fail and the device returns error 15. See [§Authorization of Provisioning Messages](#authorization-of-provisioning-messages) for the complete verification algorithm.

**Rejection rules (short form):**

- First byte not `0xD2` (not a tag-18 item), and either the device does not permit channel authority or the TO2 peer lacks provisioning authority ⇒ error 15.
- `content_type` in protected header ≠ `"application/cbor+fdo.bmo.image-begin"` ⇒ error 15.
- `x5chain` present but chain does not root at the TO2-proven Owner key, or leaf lacks `fdo-ekt-permit-provision` ⇒ error 15.
- Signature verification fails ⇒ error 15.
- `payload` bstr does not decode as an `ImageBegin` map ⇒ error 15.
- `fdo.bmo.scope` contains a field the device cannot evaluate ⇒ error 17 if the obstacle is the clock, otherwise error 15.
- `fdo.bmo.scope.guid` present and not matching the voucher GUID ⇒ error 16.
- `fdo.bmo.scope` validity window present and current trusted time outside it ⇒ error 17.
- `fdo.bmo.scope.generation` present and below the device's recorded high-water mark ⇒ error 18.

In no case does a failure in the envelope path permit the message to be reconsidered as a bare, channel-authorised `ImageBegin`.

### BIOS/Firmware Configuration

| Key | Direction | Body on the wire | Purpose |
| --- | --------- | ---------------- | ------- |
| `fdo.bmo:set` | Owner → Device | Signed envelope, or bare `BiosParam` array (see below) | Instructs the device to apply one or more BIOS / firmware parameter changes (e.g. enable Secure Boot, set a BIOS password). Because accepting this message mutates firmware state, it is subject to [Authorization of Provisioning Messages](#authorization-of-provisioning-messages): normally a COSE_Sign1 envelope, or a bare array where the device permits channel authority. |
| `fdo.bmo:response` | Device → Owner | CBOR array `[status, ?message]` (one per parameter), unsigned | Result of applying each parameter from the preceding `set`. |

#### Wire body of `fdo.bmo:set` — CDDL

On the wire, the body of `fdo.bmo:set` is a single CBOR item: a `COSE_Sign1` wrapped in CBOR tag 18, exactly as defined for `image-begin` above, differing only in the `content_type` and the inner payload schema.

```cddl
; === Wire body (full CBOR item) ===
fdo.bmo.set-body = #6.18(COSE_Sign1_Set)

; === COSE_Sign1 envelope ===
COSE_Sign1_Set = [
    protected   : bstr .cbor SetProtectedHeader,
    unprotected : SetUnprotectedHeader,
    payload     : bstr .cbor BiosParam,                 ; the INNER payload
    signature   : bstr
]

SetProtectedHeader = {
    1 => int,                                           ; alg (as for image-begin)
    3 => "application/cbor+fdo.bmo.set",                ; content_type
    ? "fdo.bmo.scope" => BmoScope                       ; device / validity binding -- see §Scope Constraints
}

SetUnprotectedHeader = {
    ? 33 => [+ bstr]       ; x5chain: absent for Owner-direct; present for Delegate
}

; === Inner payload ===
BiosParam = [ + [ name: tstr, value: any ] ]
```

**Worked example — Owner-direct signature over `[["secure-boot", true]]`.**

```text
D2                                  # CBOR tag 18 (COSE_Sign1)
   84                               # array(4)
      4A                            # bstr, length 10    ; protected header
         A2                         #   map(2)
            01 26                   #     1 => -7 (ES256)
            03 78 1E "application/cbor+fdo.bmo.set"
                                    #     3 => content_type
      A0                            # map(0)              ; unprotected: Owner-direct
      52                            # bstr, length 18     ; payload
         81                         #   array(1)          ; one param pair
            82                      #     array(2)
               6B "secure-boot"     #       name
               F5                   #       value = true
      58 40 <64 bytes>              # bstr, length 64     ; ES256 signature
```

**Signing / verification input** uses the same `Sig_structure` construction as `image-begin`, with `external_aad = FdoBmoProvisionAAD` and `content_type = "application/cbor+fdo.bmo.set"`. A `set` whose `content_type` is wrong, whose `x5chain` does not validate, whose signature does not verify, or whose `fdo.bmo.scope` is unsatisfied MUST be rejected **and the device MUST NOT apply any parameter** from the rejected message (not even those that parse successfully). A `set` whose body is not a tagged `COSE_Sign1` is rejected with error 15 unless the device permits channel authority and the TO2 peer holds provisioning authority. See [§Authorization of Provisioning Messages](#authorization-of-provisioning-messages).

### Error Handling

| Key | Direction | Type | Description |
| --- | --------- | ---- | ----------- |
| `fdo.bmo:error` | Device → Owner | Object | Error during any operation |

## Data Structures

### ImageBegin

Boot image transfers use the generic chunking strategy. The inner payload is a CBOR map using the keys defined in the schema below; `fdo.bmo` reserves the listed negative keys.

The wire-level `fdo.bmo:image-begin` message body is normally a tagged `COSE_Sign1` (CBOR tag 18) whose payload is the CBOR-encoded `ImageBegin` map shown below; where the device permits channel authority it may instead be the bare map. Signing, scope evaluation and verification are normative and are defined in [Authorization of Provisioning Messages](#authorization-of-provisioning-messages). A device that receives an `image-begin` it cannot authorise under either mode MUST respond with `error` code 15 (Provisioning Not Authorized) — or 16/17/18 where a scope constraint is the specific cause — and abort the transfer.

**Inner payload (pre-signing, before COSE wrapping):**

```
{
  0: 524288000,                    / total_size: 500MB ISO /
  1: "sha256",                     / hash algorithm /
  4: 3600,                         / estimated_duration (seconds, advisory) /
  -1: "application/x-iso9660-image", / image_type (required) /
  -2: "inst.ks=http://... quiet",  / boot_args (optional) /
  -3: "rhel-9.3-installer.iso",   / name (optional) /
  -4: "9.3",                       / version (optional) /
  -5: "RHEL 9.3 Installer"         / description (optional) /
}
```

#### ImageBegin Schema Extensions

| Key | Name | Type | Requirement | Description |
| --- | ---- | ---- | ----------- | ----------- |
| `-1` | image_type | tstr | **Required** | MIME type of the boot image |
| `-2` | boot_args | tstr | Optional | Kernel/boot arguments to pass when booting the image |
| `-3` | name | tstr | Optional | Descriptive name for the image (informational) |
| `-4` | version | tstr | Optional | Version string (informational) |
| `-5` | description | tstr | Optional | Human-readable description (informational) |
| `-6` | delivery_mode | uint | Optional | 0=inline (default), 1=url, 2=meta-url. See [Delivery Modes](#delivery-modes). |
| `-7` | url | tstr | Conditional | URL to fetch image or meta-payload. Required when `delivery_mode` ≠ 0. |
| `-8` | tls_ca | bstr | Optional | Single DER-encoded CA certificate for TLS validation of URL. |
| `-9` | expected_hash | bstr | Optional | Expected hash of final image (algorithm specified in key `1`). |
| `-10` | meta_signer | bstr | Optional | COSE_Key for meta-payload signature verification. If present, meta-payload MUST be COSE Sign1. |

**Notes:**

- Only `image_type` is required; all other fields are optional
- `boot_args` is the most commonly used optional field - it passes kernel command line arguments (e.g., kickstart URLs, installer options)
- `name`, `version`, and `description` are informational only - implementations may log them but are not required to act on them
- `estimated_duration` (key `4`, from the generic chunking spec) is especially relevant for `fdo.bmo` because firmware-stage transfers often involve large boot images (multi-GiB ISOs) over constrained links, and UEFI watchdog timers are typically more aggressive than OS-level timeouts. Owners SHOULD include this field for any image expected to take longer than a few minutes to transfer and apply. See `chunking-strategy.md` [Estimated Duration](chunking-strategy.md#estimated-duration).
- `tls_ca` is a **single certificate** (root or intermediate CA), not a chain. This mirrors UEFI Secure Boot DB behavior where individual certificates are enrolled. Chain validation occurs at TLS handshake time using the provided CA as trust anchor.
- When `delivery_mode` is 0 (inline) or omitted, the existing chunked transfer behavior applies
- When `delivery_mode` is 1 or 2, no `image-data-*` chunks are sent; the device fetches from the URL after `image-end`

### ImageAck

When `require_ack` (key 3) is set to `true` in `image-begin`, firmware MUST respond with `image-ack` before data transfer begins. This uses the standard acknowledgment gate format from `chunking-strategy.md`:

```cddl
ImageAck = [
    accepted: bool,        ; true = proceed, false = rejected
    ? reason_code: uint,   ; Rejection reason (see Error Codes)
    ? message: tstr        ; Human-readable explanation
]
```

**Recommendation**: Owners SHOULD always set `require_ack: true` for BMO transfers since boot images are typically large and firmware capabilities vary significantly.

### ImageResult

```
[
  0,                              / status_code: 0=success /
  "Image received, booting..."    / optional message /
]
```

### BiosParam (set message)

The wire-level `fdo.bmo:set` message body is normally a tagged `COSE_Sign1` (CBOR tag 18) whose payload is a CBOR array of parameter name/value pairs for BIOS configuration; where the device permits channel authority it may instead be the bare array. Signing, scope evaluation and verification rules are normative and are defined in [Authorization of Provisioning Messages](#authorization-of-provisioning-messages); the inner (pre-signing) payload is shown here.

**Inner payload (pre-signing, before COSE wrapping):**

```cbor
[
  ["secure-boot", true],
  ["bios-password", "EnterpriseKey"]
]
```

Each pair is exactly two CBOR elements: parameter name (tstr) and parameter value (type depends on parameter). A device that receives a `set` whose body is **not** a tagged `COSE_Sign1`, or whose signature does not validate, MUST respond with `error` code 15 (Provisioning Not Authorized) and MUST NOT apply any parameter from the rejected message.

### BiosResponse (response message)

One CBOR response per parameter in the corresponding `set` message:

```cbor
[
  0,                    / status_code: 0=success, 1=warning, 2=error /
  "Secure Boot enabled" / optional message /
]
```

### Atomicity and Error Handling

When a `set` message contains multiple parameters, firmware SHOULD apply them atomically (all-or-nothing):

- If **any** parameter fails validation or application, **all** parameters in that message SHOULD be rolled back
- This ensures the device is not left in a partially-configured state

Because atomic behavior may be difficult to guarantee in all firmware implementations, **owners SHOULD issue single key-value commands** for critical settings. This allows:

- Clear disambiguation of which parameter failed
- Simpler error handling and retry logic
- More predictable behavior across diverse firmware implementations

**Recommended pattern for critical settings:**

```
fdo.bmo:set = [["secure-boot", true]]
fdo.bmo:response = [0, "Secure Boot enabled"]

fdo.bmo:set = [["bios-password", "EnterpriseKey"]]
fdo.bmo:response = [0, "Password set"]
```

Rather than combining them in a single message.

## Authorization of Provisioning Messages

> **For a high-level, user-oriented introduction** to the authorization model -- why it exists, when to use channel vs. artifact authority, and how to build and deploy signed artifacts with the `fdo-meta-tool` -- see [Provisioning Security: Authorizing What Gets Installed on Your Devices](../../go-fdo/provisioning-security.md). That document is written for people who build and operate onboarding infrastructure. This section is the normative specification.
>
> The common chunking transport layer also references this section: see [chunking-strategy.md, "Authorization of Begin Messages"](chunking-strategy.md#authorization-of-begin-messages).

### Rationale

`fdo.bmo` conveys **security-sensitive** operations: installation of bootable images, enrollment of UEFI Secure Boot DB/DBX certificates, and mutation of BIOS configuration. The question this section answers is not "is the peer authentic" — TO2 already settles that — but **"where does the authority to install this particular thing come from?"**

There are two defensible answers, and this specification supports both because they serve genuinely different deployments:

- **The channel.** Authority is established once, during TO2, by the peer proving it holds the Owner key or by presenting an Owner-signed Delegate chain stating which permissions the Owner granted it. Everything the peer subsequently says is said with that authority, bounded by those permissions. This covers the Owner running its own onboarding service against its own fleet — the simple case, which should stay simple — and equally an Owner who has deliberately granted a third party `fdo-ekt-permit-provision` and is content for it to decide.

- **The artifact.** The Owner orchestrates provisioning *through* a party it does not fully trust — a managed onboarding service, a CDN, a systems integrator, a regional operator. That party is authorised to *onboard*, but the Owner intends to decide *what gets installed* itself. The authorisation must therefore be a self-contained object that survives passage through the conduit: it is minted by the Owner, it names the image and the device it applies to, and the conduit can neither forge it nor alter it nor retarget it.

These are labelled **channel authority** and **artifact authority** throughout this section. Neither subsumes the other — see [§Both modes are scoped](#both-modes-are-scoped-but-they-scope-different-things) — and both are required of a conforming device; see [§Conformance](#conformance).

#### Why permitting channel authority is not a weakening

It is tempting to mandate signatures unconditionally on the grounds that "signed is more secure." For the peers eligible to use channel authority, it is not:

- A peer that proved possession of the **Owner key** during TO2 did so *by signing `TO2.ProveOVHdr` with it*. That key is demonstrably online in the session. Requiring it to also sign `image-begin` asks it to demonstrate a capability it has just demonstrated, and grants the device no information it did not already have.
- A peer holding a Delegate certificate bearing `fdo-ekt-permit-provision` can mint a valid provisioning artifact over arbitrary bytes at will. Letting it instead assert the same thing over the authenticated channel changes nothing about the set of images it can cause to be installed.

In both cases the reachable set of outcomes is identical, so channel authority is a convenience, not a privilege escalation. Conversely, a peer that is **not** eligible — most importantly a Delegate holding only `fdo-ekt-permit-onboard-*` — gains nothing from this allowance and must still present a valid artifact. That is precisely the separation the Owner wanted when it issued a narrow certificate.

#### Both modes are scoped, but they scope different things

It would be a mistake to read channel authority as "unscoped." The channel is scoped, and scoped cryptographically: during TO2 the peer either proves possession of the Owner key, or presents an Owner-signed Delegate chain whose permission OIDs state exactly what the Owner authorised it to do. A peer granted `fdo-ekt-permit-onboard-new-cred` but not `fdo-ekt-permit-provision` is scoped *out* of provisioning by that chain, and the device enforces it. That is a real, Owner-authored constraint carried by the channel.

The distinction is **what** each mode scopes:

- **The channel scopes the actor.** "This party may provision." The scope is a property of the peer, established once, at the start of the session.
- **The artifact scopes the act.** "This image, on this device, until this date." The scope is a property of the individual authorisation, fixed at signing time by whoever minted it.

A live channel also supplies, for free and without anyone having to state them, two properties the artifact must reconstruct explicitly:

- **Device binding.** The session *is* the session with this device — mutually authenticated against this device's credential. There is nothing to bind, because there is nowhere else the message could be going.
- **Freshness.** The session is live and nonce-protected. A channel-authorised instruction cannot be recorded and replayed later, because it has no existence outside the session that carried it.

An artifact, by contrast, is a detached bearer object. It has been separated from any session precisely so that it can travel through a conduit, and in being separated it loses both of the above. `fdo.bmo.scope` exists to put them back: `guid` restores device binding, `not_after` and `generation` restore freshness. The constraints in [§Scope Constraints](#scope-constraints) are not extra power that artifacts have and channels lack — they are compensation for context that detachment threw away.

What artifact authority genuinely adds, and the only thing it adds, is this: **the decision about what to install was made by a party other than the one on the wire, at a time other than now.** That is the entire point, and it is exactly what a deployment orchestrating through a semi-trusted conduit needs.

#### What the two modes cost

| | Channel authority | Artifact authority |
| --- | --- | --- |
| Scopes the actor's permissions | Yes — PERM OIDs proven in TO2 | Yes — PERM.7 on the signing certificate |
| Device binding | Implicit in the session | Explicit (`guid`), or absent |
| Freshness / replay resistance | Implicit in the session | Explicit (`not_after`, `generation`), or absent |
| Content fixed by the authorising party | No — peer chooses freely within its permissions | Yes — fixed at signing time |
| Deciding party must be online at onboarding | Yes | No — may be minted offline, on an HSM, ahead of time |
| Conduit must be trusted to choose content | Yes | **No** |
| Onboarding party may hold only `fdo-ekt-permit-onboard-*` | No | Yes |
| Device may retain a durable record of what authorised the install | No | Yes |

Read down the column: channel authority is *stronger* on binding and freshness and *weaker* on who decides. That is the whole trade, and it is why neither mode subsumes the other.

### Conformance

Implementation requirements for devices, in descending order of obligation. Note throughout the distinction between **implementing** a capability and **enabling** it by policy: a device may be required to implement something it is configured not to use.

| Capability | Requirement | Notes |
| ---------- | ----------- | ----- |
| Determining the TO2 peer's provisioning authority | **MUST** | Falls out of TO2 processing the device performs regardless. |
| Channel authority | **MUST** implement, **MUST** be enabled in the default configuration | The cheapest mode and the one that makes a locally-hosted deployment workable without per-device signing. An Owner relies on it working out of the box. |
| Artifact authority, Owner-direct signature | **MUST** | One COSE_Sign1 verification against the TO2-proven Owner key. No certificate handling. |
| Artifact authority, Delegate `x5chain` | **SHOULD** | Requires X.509 parsing and chain validation, which is a substantial cost in a constrained firmware stack. A device that does not implement it MUST reject `x5chain`-bearing artifacts with error 15 and MUST document the limitation. |
| `fdo.bmo.scope.guid` evaluation | **MUST** if artifact authority is implemented | Comparing 16 bytes against a value the device already holds. |
| `fdo.bmo.scope.not_before` / `not_after` evaluation | **SHOULD** | Requires a clock the device can justify trusting. See [§Device Clocks in Firmware](#device-clocks-in-firmware). |
| `fdo.bmo.scope.generation` evaluation | **MAY** | Requires rollback-protected non-volatile storage, which many platforms do not have. See [§`generation`](#generation--clock-free-supersession). |
| Strict provisioning policy (require artifacts even from an authorised peer) | **MAY** | Operator-enabled tightening. MUST default to off. |
| Rejecting rather than ignoring a constraint it cannot evaluate | **MUST** | See [§Unevaluable constraints](#unevaluable-constraints). This applies to every OPTIONAL row above. |

The optionality in the lower rows is deliberate: a constrained device is permitted not to implement time or generation checking, but it is **not** permitted to pretend it did. The combination — optional to implement, mandatory to fail closed — is what makes it safe for an Owner to use a constraint without knowing the capabilities of every device in the fleet, at the cost of a visible failure rather than a silent one.

### Messages Requiring Authorization

The following messages carry authority and are subject to this section:

| Message | Inner Payload | Rationale |
| ------- | ------------- | --------- |
| `fdo.bmo:image-begin` | `ImageBegin` map (§ImageBegin) | Authorizes installation of a bootable image or UEFI DB/DBX entry and binds the image identity (`image_type`, `expected_hash`, `url`, `meta_signer`) to an Owner-authorised decision. |
| `fdo.bmo:set` | `BiosParam` array (§BiosParam) | Authorizes mutation of firmware configuration (`secure-boot`, `bios-password`, boot order, …). |

Each of these messages is transmitted either as a tagged `COSE_Sign1` (CBOR tag 18) carrying artifact authority, or as its bare inner payload (a CBOR map for `image-begin`, a CBOR array for `set`) relying on channel authority. The CBOR tag is the unambiguous discriminator between the two forms; see [§Selecting the Authorization Mode](#selecting-the-authorization-mode).

All other `fdo.bmo:*` messages (`active`, `image-ack`, `image-data-<n>`, `image-end`, `image-result`, `response`, `error`) carry only bookkeeping; they grant no new authority to the device and are always transmitted unsigned. Bulk image bytes (`image-data-<n>`) are not signed individually — their integrity is bound to the authorizing `image-begin` via `expected_hash` (key `-9`, REQUIRED unless the Owner explicitly chooses to delegate chunk integrity to the transport). It follows that an `image-begin` which omits `expected_hash` conveys no binding to the image bytes at all, and signing such a message authorises only the *metadata*; Owners using artifact authority SHOULD always populate `expected_hash`.

### Trust Anchor

The trust anchor for every `fdo.bmo` provisioning signature is the **Owner public key proven to the device during TO2** — specifically, the public key present in the final entry of the Ownership Voucher, which the device cryptographically verifies as part of `TO2.ProveOVHdr` processing. This key is already established as the ultimate authority over the device in the FDO protocol; BMO reuses it rather than introducing a separate trust root.

An implementation MUST NOT use the TO2 transport peer's public key, the TO2 session key, or any other channel-bound material as the trust anchor when verifying a provisioning **signature**. A Delegate that signed `TO2.ProveOVHdr` on the Owner's behalf is not thereby a provisioning authority, and treating its key as the anchor would collapse the two roles this section exists to separate.

This is distinct from — and must not be confused with — channel authority, which does not involve a signature at all and is evaluated per [§Channel Authority](#channel-authority) below.

### Signer

A provisioning message carrying artifact authority MUST be signed by **exactly one** of the following:

1. **Owner-direct** — the Owner private key itself. In this case the `COSE_Sign1` carries **no** `x5chain` header; the device verifies the signature directly against the TO2-proven Owner public key.

2. **Delegated** — a Delegate private key distinct from the Owner key. In this case the `COSE_Sign1` MUST carry an `x5chain` unprotected header (COSE header label `33`, RFC 9360) listing a DER-encoded X.509 certificate chain, **leaf first**, whose root certificate is issued by (or is) the Owner key. The leaf certificate MUST carry the permission OID

    ```
    fdo-ekt-permit-provision    OID 1.3.6.1.4.1.45724.3.1.7    (PERM.7)
    ```

    A Delegate certificate that is valid for other FDO purposes (e.g. `fdo-ekt-permit-onboard-new-cred`, `fdo-ekt-permit-redirect`) but lacks `fdo-ekt-permit-provision` MUST NOT be accepted as a BMO provisioning signer. This is how Owners grant narrow "may provision BMO" authority without granting full Owner authority.

Allowing Owner-direct signatures unconditionally is intentional: a deployment whose TO2 frontend happens to be the Owner itself should not be forced to mint a self-signed delegate certificate. Allowing Delegate signatures via `x5chain` is also intentional: an Owner whose TO2 frontend is an orchestrator or third party can issue one or more short-lived `fdo-ekt-permit-provision` certificates without giving out its Owner key.

The signing party need not be the party that transmits the message, and in the CDN / managed-service deployments this section is chiefly aimed at, it is not. An Owner MAY mint artifacts entirely offline and hand them to a conduit for delivery.

### Channel Authority

A device accepts a provisioning message that is **not** wrapped in a `COSE_Sign1` on the authority established during TO2. To do so it MUST determine the **provisioning authority of the TO2 peer**, as a single decision made during TO2 processing and retained for the session:

- If `TO2.ProveOVHdr` carried no `DelegateChain` and its signature verified directly against the TO2-proven Owner public key, the peer **has** provisioning authority (it holds the Owner key).
- If `TO2.ProveOVHdr` carried a `DelegateChain` that validated to the Owner key, the peer has provisioning authority **if and only if** `fdo-ekt-permit-provision` is granted by that chain, evaluated under the usual FDO rule that a permission is granted only when present in every certificate in the chain.
- In all other cases the peer does **not** have provisioning authority.

A device whose policy disables channel authority, and any device facing a peer that does **not** hold provisioning authority, MUST treat an unsigned provisioning message as unauthorised and reject it with error 15.

**Support for channel authority is REQUIRED, and a device MUST accept it in its default configuration.** An unsigned provisioning message from a peer that has proven provisioning authority is the simple, baseline case, and a conforming device installs it. It is also the cheapest mode to implement — the peer's authority is already known from TO2 processing the device must perform anyway, and no COSE verification, certificate parsing or scope evaluation is involved — and it is what lets an operator stand up a local onboarding service and provision a fleet without minting a signed artifact per device. An Owner MUST be able to rely on this working against any conforming device out of the box.

Crucially, the security of this mode does not rest on the policy switch. It rests on the permission model: a peer that does not hold the Owner key and was not granted `fdo-ekt-permit-provision` cannot use channel authority at all, whatever the device's policy says. An Owner who has not delegated provisioning authority is protected by that fact alone. Conversely, an Owner who *has* granted `fdo-ekt-permit-provision` to a party has already said that party may decide what to install, and it would be strange for the device to then refuse to let it.

A device MAY additionally offer a **strict provisioning** policy that an operator can switch on to require artifact authority even from a peer holding provisioning authority. This serves the Owner who delegates provisioning broadly but still wants each individual installation pinned to a device and bounded in time — the constraints of [§Scope Constraints](#scope-constraints) are only available in the artifact form. Such a policy is a deliberate local tightening of the baseline, not an alternative reading of it: it MUST be off unless explicitly enabled, MUST NOT be implied by any other setting, and MUST be documented.

#### No downgrade

If the message body **is** a tagged `COSE_Sign1`, the device MUST complete artifact verification and MUST NOT, on any failure, fall back to channel authority — not when the signature is invalid, not when the `x5chain` fails to validate, not when a scope constraint is unmet, and not when the signature algorithm is unsupported. A failed artifact is a rejected message, never an unsigned one.

Without this rule an Owner's decision to constrain an authorisation could be undone by the conduit simply corrupting it, and the narrower the Owner made the artifact the easier it would be to strip. Note also that a conduit cannot usefully *remove* an envelope: doing so converts the message to a channel-authority assertion, which succeeds only if the conduit itself holds provisioning authority — in which case it could have minted its own artifact regardless.

### Selecting the Authorization Mode

The device distinguishes the two forms by the leading CBOR item:

| First byte | Interpretation |
| ---------- | -------------- |
| `0xD2` (tag 18) | Artifact authority. Verify per [§Device Verification Algorithm](#device-verification-algorithm). |
| Any other valid CBOR head for the expected inner type (`map` for `image-begin`, `array` for `set`) | Channel authority. Accept only if policy permits **and** the TO2 peer has provisioning authority. |
| Anything else | Malformed; error 15. |

Owners SHOULD send the artifact form whenever they are able to; it is a superset of the channel form in expressiveness and the only form that survives an untrusted conduit.

### `COSE_Sign1` Structure

```cddl
BmoProvisioningEnvelope<InnerPayload> = #6.18(COSE_Sign1)

protected_header = {
    1: alg,                   ; Signature alg (ES256, ES384, ES512, RS256, PS256, …)
    3: content_type,          ; MUST match the inner payload semantic
    ? "fdo.bmo.scope": BmoScope   ; Scope constraints -- see §Scope Constraints
}

unprotected_header = {
    ? 33: [+ bstr]            ; x5chain (RFC 9360): DER certs, leaf first.
                              ;   Present IFF signer is a Delegate.
                              ;   Owner-direct signatures MUST NOT include x5chain.
}

payload   = bstr .cbor InnerPayload
signature = bstr
```

`fdo.bmo.scope` is carried in the **protected** header so that it is covered by the signature, and as a header rather than a payload field so that it applies uniformly to `image-begin` (whose inner payload is a map) and `set` (whose inner payload is an array) without perturbing either schema. A text-string label is used deliberately: it is self-describing and cannot collide with a future IANA COSE header registration.

The protected `content_type` MUST be one of the values registered below. The `content_type` carried in the protected header MUST match the wire-level message key (per the §Content-Type Registry); a device that observes a mismatch MUST reject the message with error 15.

#### Content-Type Registry

| Content-Type                           | Inner Payload Type | Wire-Level Message |
| -------------------------------------- | ------------------ | ------------------ |
| `application/cbor+fdo.bmo.image-begin` | `ImageBegin` map   | `fdo.bmo:image-begin` |
| `application/cbor+fdo.bmo.set`         | `BiosParam` array  | `fdo.bmo:set` |

### External AAD (Domain Separation)

All `fdo.bmo` provisioning signatures MUST be computed with the `COSE_Sign1` `external_aad` set to the CBOR encoding of:

```cddl
FdoBmoProvisionAAD = ["FDO-FSIM-BmoProvision-v1"]
```

This domain-separation tag ensures that a signature minted for BMO provisioning cannot be replayed against any other FDO structure that uses `COSE_Sign1` (e.g. `TO0.OwnerSign`, `TO2.ProveOVHdr`, BMO meta-payloads). Devices MUST use the identical tag when verifying; any other value MUST cause verification to fail.

### Scope Constraints

A signed provisioning artifact is, by construction, a **bearer object**: anything holding a copy can present it. Domain separation stops it being replayed into a different protocol context, but on its own nothing stops it being replayed to a *different device*, or to the same device *at a later date* long after the Owner intended the authorisation to lapse. In the deployments this section targets — where the conduit is a managed service the Owner does not fully trust — both are live concerns, and both are concerns about a party that is otherwise behaving legitimately.

As noted in [§Both modes are scoped](#both-modes-are-scoped-but-they-scope-different-things), a channel-authorised instruction needs neither of these constraints: the session supplies device binding and freshness inherently. The fields below exist to restore those two properties to an authorisation that has been deliberately detached from any session so that it can be minted offline and carried by a third party. An Owner that states no constraints has minted an authorisation valid on every device the conduit can reach, for as long as the conduit keeps a copy — which is sometimes exactly right (one golden image, one fleet) and sometimes a serious over-grant.

The `fdo.bmo.scope` protected header lets the Owner narrow an authorisation along three axes:

```cddl
BmoScope = {
    ? "guid"       => bstr / [+ bstr],   ; device binding: 16-byte FDO GUID(s)
    ? "not_before" => uint,              ; seconds since 1970-01-01T00:00:00Z (UTC)
    ? "not_after"  => uint,              ; seconds since 1970-01-01T00:00:00Z (UTC)
    ? "generation" => uint               ; monotonic supersession counter
}
```

All four fields are OPTIONAL and the header itself is OPTIONAL. An absent field is an absent constraint: an artifact with no `guid` is valid for any device that can verify its signature, which is exactly what an Owner wants when authorising one golden image across a whole fleet. Constraints are cumulative — every field that is present MUST be satisfied.

#### `guid` — device binding

The value is compared against **the GUID in the Ownership Voucher header proven to the device during TO2** — equivalently, the GUID the device itself presented in `TO2.HelloDeviceProbe`. It is emphatically *not* compared against any replacement GUID delivered in `TO2.SetupDevice`: `SetupDevice` precedes the ServiceInfo exchange, so by the time BMO runs the device may already hold a replacement, and an Owner minting artifacts offline cannot know what that replacement will be. Implementations MUST retain the voucher GUID for the lifetime of the TO2 session for this comparison.

If the value is an array, the constraint is satisfied when the device's GUID equals any member. This supports minting one artifact for a small named batch; it is not intended as a fleet-management mechanism, and Owners SHOULD prefer per-device artifacts or an absent `guid` over very large arrays.

A mismatch MUST be rejected with error 16.

Per-device artifacts are inexpensive in practice. The envelope is on the order of a kilobyte, the Owner already knows its fleet's GUIDs (it holds the vouchers), and the large object — the image itself — remains shared and unduplicated. Minting one artifact per device at scale is a signing-throughput question, not a storage or bandwidth one.

#### `not_before` / `not_after` — validity window

Both are seconds since the Unix epoch, UTC. `not_before` bounds the artifact below and `not_after` above; the artifact is valid when `not_before <= now <= not_after`. A violation MUST be rejected with error 17.

The purpose is supersession: it bounds how long a conduit may keep re-presenting an authorisation the Owner has moved on from. Without it, an `image-begin` authorising a build later found to be vulnerable remains installable forever by anyone who retained a copy.

**This mechanism is only as trustworthy as the device's clock, which in firmware is frequently not trustworthy at all.** See [§Device Clocks in Firmware](#device-clocks-in-firmware) — that discussion is normative in its requirements and Owners should not deploy a time gate without reading it.

#### `generation` — clock-free supersession

A monotonically increasing counter chosen by the Owner, scoped to the (Owner key, device) pair. A device implementing this constraint records the highest value it has accepted and MUST reject any artifact whose `generation` is lower than the recorded value, with error 18. On acceptance it updates the stored value before acting on the payload.

`generation` exists because it is the one supersession mechanism that does not depend on a clock, and is therefore the only one that holds up against an attacker with physical access. Owners requiring assured supersession SHOULD use `generation`, optionally alongside `not_after`. Note that it is strictly ordering-based: it cannot express "expires in 30 days," only "this supersedes everything before it."

**Implementing `generation` is OPTIONAL.** It requires a facility many platforms genuinely lack, and this specification does not require a device to invent one.

**Storage requirements.** Where implemented, the high-water mark MUST be held in storage that is both non-volatile and **rollback-protected** — a TPM NV index under an appropriate policy, a UEFI authenticated variable, a monotonic counter in secure storage, or equivalent. It MUST NOT be held in storage that can be cleared or rewound by any means available to an attacker who can reach the counter's threat surface. In particular, battery-backed RAM, ordinary UEFI variables, an unauthenticated NV index, and a file in the ESP are all unsuitable.

The reason this is stated as a hard requirement rather than a recommendation is a specific and nasty failure mode. A counter that resets to zero does not fail visibly — it fails *permissively*. The device continues to believe it is enforcing supersession while in fact accepting every artifact ever minted, and the Owner, seeing no errors, continues to believe its revocations are effective. A rewindable counter is therefore materially worse than no counter at all, because no counter at least fails loudly per the rule below.

Accordingly: a device that cannot guarantee rollback-protected storage MUST NOT report or behave as though it implements `generation`. It MUST instead treat `generation` as a constraint it cannot evaluate and reject artifacts carrying it, per [§Unevaluable constraints](#unevaluable-constraints).

**When the device cannot evaluate it.** An artifact carrying `generation` presented to a device that does not implement it — whether because the platform lacks suitable storage, or because the implementation simply omits the feature — MUST be rejected with error 18. The device does not install, and does not silently drop the constraint.

The remedy is on the Owner's side, and it is the obvious one: **mint the artifact without a `generation` field.** An Owner that does not know the capabilities of every device in its fleet has three workable strategies, in rough order of preference:

1. Omit `generation` and use `not_after` for supersession, accepting that expiry is weaker against a physical attacker (see [§Device Clocks in Firmware](#device-clocks-in-firmware)).
2. Omit both and rely on `guid` alone, accepting that the authorisation does not expire.
3. Mint per-device artifacts, including `generation` only for the devices known to support it.

A conforming device MUST NOT be "helpful" here by accepting an artifact whose `generation` it cannot check. The failure is intended to be visible, and the resulting error 18 tells the operator exactly which constraint to drop.

#### Unevaluable constraints

A device MUST NOT silently ignore a scope constraint it does not understand or cannot evaluate. If `fdo.bmo.scope` contains a field the device does not implement, or a field it implements but cannot evaluate in its current state, the device MUST reject the message rather than proceed. The two common cases are `not_before`/`not_after` on a device with no trustworthy clock (error 17) and `generation` on a device with no rollback-protected storage (error 18); an entirely unrecognised field is error 15.

Several scope capabilities are OPTIONAL to implement — see [§Conformance](#conformance) — and this rule is what makes that optionality safe. An Owner may attach a constraint without surveying its fleet, knowing that a device unable to honour it will say so rather than quietly install anyway.

This is the single most important requirement in this subsection. An Owner that adds a constraint has, by doing so, declared the authorisation conditional; a device that accepts the artifact while discarding the condition has granted strictly more authority than the Owner granted, and has done so invisibly. Failing closed converts a silent policy violation into a visible, diagnosable error.

**Open issue — capability discovery.** This version of the specification provides no wire mechanism by which a device tells the Owner which scope constraints it is able to evaluate, so an Owner minting artifacts must know its fleet's capabilities out of band. Where it guesses wrong the failure is safe but late: the device rejects with error 16/17/18 partway through TO2, and the operator must correlate the code back to the offending field and re-mint. A future revision should let the device declare its evaluable constraint set before the first `image-begin`, so that an Owner can select constraints it knows will be honoured. Until then, Owners provisioning heterogeneous fleets SHOULD prefer the lowest-common-denominator constraint set — in practice `guid` alone, which every device implementing artifact authority is required to support.

### Device Clocks in Firmware

Time gating is specified above because it is genuinely useful against the threat this section is about. It is documented at length here because implementers routinely overestimate what it provides.

#### What a firmware-stage clock actually is

`fdo.bmo` runs pre-OS, which removes most of the machinery that normally makes system time dependable:

- **UEFI `GetTime()`** reads a platform RTC. On a system whose coin cell has died — extremely common in returned, warehoused, or long-shelved hardware, which is exactly the population that gets bare-metal onboarded — it typically returns the platform's epoch default, often the firmware build year or 1970.
- **Many platforms have no RTC at all.** A large fraction of ARM SoCs and embedded designs have no battery-backed timekeeping; `GetTime()` either fails or returns a fixed constant every boot.
- **Virtual machines** inherit host time when configured to, and a fabricated constant when not. QEMU defaults vary by invocation.
- **Manufacturing state.** A device performing its first onboarding may never have had its clock set by anyone, because setting it is something an OS normally does.
- **No network time.** NTP is an OS-stage service. A firmware implementation could speak NTP over the network it already has, but unauthenticated NTP is trivially controlled by exactly the network-positioned attacker a time gate is meant to resist, and NTS is not realistically available in a pre-OS stack. Fetching time over the TO2 channel is no better: it would mean asking the conduit what time it is, which defeats the purpose.

#### Attacks on the clock

The practical question is who can move the clock backwards, because backwards is the dangerous direction — it resurrects an expired authorisation.

- **A physical attacker can always do it.** Pulling the RTC battery, entering BIOS setup, or reflashing NVRAM resets the clock. A `not_after` gate therefore provides **no protection whatsoever** against an attacker with hands on the machine. Specifications that treat expiry as a security boundary against physical access are simply wrong, and implementers should not be misled by the fact that X.509 has the same weakness.
- **A remote conduit generally cannot.** The managed service, CDN, or integrator that delivers the artifact has no path to the device's RTC. Against *this* adversary — the one the whole section is written for — a validity window is meaningful and is the cheapest available supersession control.
- **A network attacker can, if the platform trusts network time.** This is a good reason for firmware not to set its clock from unauthenticated network sources, and for a device to prefer "no trustworthy clock" over "a clock an attacker supplied."

The correct summary is therefore: **a validity window is a supersession control against a remote conduit, not a tamper-resistance control against a local attacker.** Deployments needing the latter MUST use `generation`, whose rollback protection lives in hardware rather than in a battery.

#### Recommended implementation

1. **Decide explicitly whether the clock is trustworthy, and record why.** A device SHOULD classify its clock as trustworthy only if it has a functioning battery-backed RTC (or platform equivalent) *and* the value read is plausible — at minimum, not earlier than the build timestamp of the firmware performing the check, which is a free sanity bound that catches dead batteries and epoch defaults.

2. **Maintain a monotonic floor.** A device SHOULD keep a high-water mark of the greatest timestamp it has ever accepted, in the same rollback-protected storage used for `generation`, and treat any clock reading earlier than that mark as untrustworthy. This converts a resettable clock into a ratchet, and mirrors the timestamp semantics UEFI authenticated variables already use for the same reason. It does not stop a physical attacker who can also clear that storage, but it does stop the trivial battery-pull.

3. **Fail closed, per [§Unevaluable constraints](#unevaluable-constraints).** If the clock is not trustworthy and an artifact carries `not_before` or `not_after`, reject with error 17. Do not accept the artifact with the window ignored, and do not substitute a guess.

4. **Give the Owner a way to avoid the problem.** Because a device with no clock cannot honour a validity window, Owners provisioning such fleets must be able to mint artifacts without one. Owners SHOULD reach for `generation` in that case rather than omitting supersession control altogether.

5. **Do not use clock state to make any other security decision.** In particular, the validity dates in a Delegate `x5chain` face precisely this problem, and a device that cannot evaluate `not_after` on an artifact cannot evaluate `notAfter` on a certificate either. Implementations SHOULD apply one consistent policy to both rather than enforcing certificate validity while ignoring artifact validity, or vice versa.

### Device Verification Algorithm

On receipt of `fdo.bmo:image-begin` or `fdo.bmo:set`, the device MUST execute the following algorithm before acting on the inner payload:

1. Decode the message body as CBOR.
    1. If the outer item is CBOR tag 18 (`COSE_Sign1`), this is an artifact-authority message. Continue at step 2. Having entered this branch the device MUST NOT leave it except by acceptance at step 8 or by returning an error; falling back to channel authority is forbidden (see [§No downgrade](#no-downgrade)).
    2. Otherwise, if the item is the bare inner payload type registered for the message (`map` for `image-begin`, `array` for `set`), this is a channel-authority message. Accept it only if device policy permits channel authority **and** the TO2 peer was determined to hold provisioning authority per [§Channel Authority](#channel-authority); otherwise return error 15. If accepted, skip to step 8.
    3. Otherwise return error 15.
2. Parse the `COSE_Sign1`. Extract the protected header's `content_type` (label 3). If absent, or if it does not equal the content-type registered for the incoming message key (see §Content-Type Registry), return error 15.
3. Inspect the unprotected header's `x5chain` (label 33).
    1. **Absent** — the signer is claiming to be the Owner. Set the verification key to the TO2-proven Owner public key. Skip to step 5.
    2. **Present** — the signer is a Delegate. Parse each DER certificate in `x5chain` (leaf first). Validate the chain such that the topmost certificate is signed by the TO2-proven Owner public key (equivalently: treat the Owner key as an implicit self-signed root). If chain construction or signature validation fails, return error 15.
4. Verify that the leaf certificate of the delegate chain carries `fdo-ekt-permit-provision` (OID `1.3.6.1.4.1.45724.3.1.7`) as an extended key usage. If missing, return error 15. Set the verification key to the leaf's public key.
5. Verify the `COSE_Sign1` signature using the verification key chosen above and the external AAD `FdoBmoProvisionAAD`. On failure, return error 15.
6. Evaluate the `fdo.bmo.scope` protected header, if present, per [§Scope Constraints](#scope-constraints). This step happens **after** signature verification, so that constraint evaluation never depends on unauthenticated data.
    1. Any field the device does not implement, or cannot evaluate in its current state, MUST cause rejection — error 17 if the obstacle is an untrustworthy clock, otherwise error 15.
    2. `guid` present and not matching the voucher GUID ⇒ error 16.
    3. `not_before` / `not_after` present and the current trusted time outside the window ⇒ error 17.
    4. `generation` present and lower than the device's recorded high-water mark ⇒ error 18.
7. Decode the verified `payload` bstr as the inner payload type registered for the message (`ImageBegin` map or `BiosParam` array). If decoding fails, return error 15.
8. Commit any monotonic state the accepted message requires (`generation` high-water mark, accepted-timestamp floor) **before** acting on the payload, then proceed with normal BMO processing (e.g., NAK with `image-ack`, begin chunked transfer, apply BIOS parameters, …).

Devices MUST NOT process the inner payload until step 8 is reached. A device that "reads ahead" into the payload (e.g., to pre-allocate buffers) MUST discard any such state if verification fails.

Ordering in this algorithm is normative, not incidental. Signature verification precedes scope evaluation so that no constraint decision rests on unauthenticated bytes; monotonic state is committed before the payload is acted upon so that a device interrupted mid-install cannot be induced to replay a superseded artifact on the next boot.

### Owner / Delegate Requirements

An Owner (or a Delegate acting on its behalf) that produces `fdo.bmo:image-begin` or `fdo.bmo:set` using **artifact authority**:

- MUST wrap the inner payload in a tagged `COSE_Sign1` per the structure above.
- MUST populate the protected `content_type` to match the wire-level message.
- MUST compute the signature with `external_aad = FdoBmoProvisionAAD`.
- MUST, when signing as a Delegate, include the leaf certificate (and any intermediates chaining up to the Owner key) in the unprotected `x5chain` header, leaf first.
- MUST NOT, when signing as the Owner directly, include an `x5chain` header.
- MUST NOT include a certificate for the Owner key itself in `x5chain`; the Owner key is the implicit trust root and is transported out-of-band via the Ownership Voucher.
- MUST place any scope constraints in the **protected** header under `fdo.bmo.scope`; scope constraints in the unprotected header are unsigned and MUST be ignored by devices.
- SHOULD populate `expected_hash` (key `-9`) in a signed `image-begin`, since without it the signature authorises only the transfer's metadata and not its content.
- SHOULD NOT emit a validity window to a fleet known to lack trustworthy clocks; such devices will fail closed. Use `generation` instead.

An Owner delivering **through a conduit it does not fully trust** MUST use artifact authority, and SHOULD constrain the artifact with `guid` and with `not_after` and/or `generation`. Handing an unconstrained artifact to such a conduit authorises that image on every device the conduit can reach, for as long as the conduit retains a copy.

An Owner relying on **channel authority** MUST hold either the Owner key or a Delegate certificate bearing `fdo-ekt-permit-provision`. No signing, scope authoring or per-device artifact minting is involved; this is the intended path for an operator running its own onboarding service. Be aware that a device MAY be configured in strict provisioning mode, in which case unsigned messages are rejected even from an authorised peer.

## BIOS/Firmware Configuration

The BMO FSIM includes BIOS configuration capabilities alongside image transfer. This allows a single FSIM to handle the complete bare-metal onboarding flow: certificate enrollment, Secure Boot configuration, and boot image delivery.

**Note:** All BMO messages use CBOR encoding, consistent with the FDO protocol.

### Standard BIOS Parameters

| Parameter | Value Type | Purpose |
| --------- | ---------- | ------- |
| `secure-boot` | bool | Enable (`true`) or disable (`false`) UEFI Secure Boot |
| `bios-password` | tstr / null | Set password (string) or clear it (`null`) |
| `boot-order` | array | Set boot device priority order |

### secure-boot

Enables or disables UEFI Secure Boot.

- **Parameter name**: `secure-boot`
- **Value type**: Boolean
- **Values**: `true` (enable) or `false` (disable)

**Example:**

```
fdo.bmo:set = [["secure-boot", true]]
fdo.bmo:response = [0, "Secure Boot enabled"]
```

**Implementation notes:**

- Firmware MUST verify that enabling Secure Boot will not render the system unbootable
- If no valid boot path exists with Secure Boot enabled, firmware SHOULD reject with error
- Enabling Secure Boot typically requires valid certificates in the DB first

### bios-password

Sets or clears the BIOS/UEFI setup password.

- **Parameter name**: `bios-password`
- **Value type**: Text string or null
- **Values**: `"password-string"` (set) or `null` (clear/unlock)

**Example:**

```
fdo.bmo:set = [["bios-password", "SecureP@ss123"]]
fdo.bmo:response = [0, "Password set"]
```

**Implementation notes:**

- Password is transmitted over the already-encrypted FDO channel
- Setting a password "locks" the BIOS - users cannot modify settings without it
- Clearing the password (`null`) "unlocks" the BIOS for user modification
- Owner SHOULD set BIOS password as final step after all other configuration

### boot-order

Sets the boot device priority order.

- **Parameter name**: `boot-order`
- **Value type**: Array of strings (device identifiers)

**Example:**

```
fdo.bmo:set = [["boot-order", ["NVMe0", "PXE", "USB"]]]
fdo.bmo:response = [0, "Boot order set"]
```

### Vendor-Specific Parameters

Vendor-specific BIOS parameters use reverse-DNS notation:

- `com.dell.asset-tag` → `"ASSET12345"`
- `com.hp.virtualization` → `true`

Unknown parameters MUST be rejected with an error response.

## Delivery Modes

The `delivery_mode` field (`-6`) controls how the boot image is delivered to the device. This enables flexible deployment strategies where owners can choose between inline transfer, direct URL download, or meta-payload indirection.

### Mode 0: Inline (Default)

When `delivery_mode` is 0 or omitted, the existing chunked transfer behavior applies:

- Owner sends `image-begin` with metadata
- Owner sends `image-data-0` through `image-data-N` chunks
- Owner sends `image-end` with optional hash
- Device verifies and boots the image

This is the traditional BMO flow and remains the default for backward compatibility.

### Mode 1: Direct URL Reference

When `delivery_mode` is 1, the device fetches the image from a URL instead of receiving inline chunks:

```text
Owner → Device: image-begin {
  -1: "application/x-raw-disk-image",  // MIME type of FINAL image
  -6: 1,                                // delivery_mode = url
  -7: "https://images.example.com/rhel9.dd",
  -8: h'3082...',                       // optional: custom CA cert (DER)
  -9: h'a1b2c3...',                     // optional: expected SHA-256 hash
  1: "sha256",                          // hash algorithm (if -9 provided)
  3: true                               // require_ack
}
Device → Owner: image-ack [true]

; No image-data-* chunks sent!

Owner → Device: image-end {}            // signals "go fetch it"
Device → Owner: image-result [0, "Downloaded and verified"]
```

**Key Points:**

- **MIME type (`-1`)** describes the **final image**, not the URL
- **No new MIME types needed** - same types work for inline or URL delivery
- **TLS CA (`-8`)** is optional - if omitted, device uses system trust store
- **Hash (`-9`)** is optional - if provided, device MUST verify after download
- **Device can still NAK** based on MIME type, size concerns, or policy

### Mode 2: Meta-Payload Indirection

When `delivery_mode` is 2, the device fetches a CBOR meta-payload from the URL, which then defines the actual image location. This enables **third-party delegation** of image selection.

#### Design Rationale: Why Meta-Payloads?

Meta-payload indirection serves two key purposes:

1. **Delegate image selection to a third party** (e.g., OS vendor, image repository)
2. **Provide cryptographic integrity for unsigned image formats** (e.g., raw disk images, ISOs)

The owner specifies:

- The meta-payload URL
- The signing key for verification (optional but recommended)

The **entity controlling the meta-payload** determines which image version devices receive. This decouples fleet management from image versioning and provides a single point of control for updates.

#### Use Case 1: Vendor-Managed Images

An OS vendor (e.g., Red Hat, Canonical, Microsoft) hosts the meta-payload and controls image selection:

- Owner doesn't need to update every device's configuration when a new OS version is released
- Vendor can update the meta-payload to point to newer images
- Devices always get the "current" image as determined by the vendor

**Example**: Owner configures 10,000 devices with Red Hat's meta-URL and signing key. Red Hat updates the meta-payload when RHEL 9.4 releases. All devices automatically get the new version without owner intervention.

#### Use Case 2: Fleet Operator-Managed Images

An individual end-user or fleet operator hosts their own meta-payload to control image versions across their fleet:

**Example**: A data center operator manages 500 servers running a custom Linux image:

1. **Initial deployment**: Operator creates a meta-payload pointing to `image-v1.dd` with its SHA-256 hash, signs it with their private key, and hosts it at `https://images.mycompany.com/datacenter/meta.cbor`
2. **Fleet configuration**: All devices are configured with the meta-URL and the operator's public signing key
3. **Upgrade to v2**: When ready to upgrade, the operator:
   - Uploads `image-v2.dd` to their image server
   - Generates a new meta-payload with the v2 URL and hash
   - Signs and replaces the meta-payload at the same URL
4. **Automatic rollout**: All devices onboarding after the update automatically receive v2

This provides a **single point of control** for fleet-wide image updates without modifying device configurations or Onboarding Service settings.

#### Use Case 3: Signing Unsigned Image Formats

Many boot image formats—such as raw disk images (`dd`), ISO images, and legacy BIOS images—have **no well-defined mechanism for cryptographic signing**. The meta-payload solves this problem by providing an **external signature envelope**:

1. **Hash as signature proxy**: The meta-payload includes the expected hash of the image. Since the meta-payload itself is signed (COSE Sign1), the hash is cryptographically bound to the signer's key.
2. **Verification chain**: Device verifies meta-payload signature → extracts trusted hash → downloads image → verifies image hash matches. This effectively "signs" an image that cannot be signed internally.
3. **Easy updates**: When the image is updated (breaking the old hash), the operator simply generates a new signed meta-payload with the new hash. No changes to the image format or device configuration required.

**Example**: A raw disk image (`rhel9.dd`) cannot be signed directly. The operator:

1. Computes `sha256sum rhel9.dd` → `a1b2c3...`
2. Creates a meta-payload with `url: https://images.example.com/rhel9.dd` and `expected_hash: a1b2c3...`
3. Signs the meta-payload with their private key
4. Devices verify the signature, then verify the downloaded image matches the trusted hash

This provides **end-to-end integrity** for image formats that lack native signing support.

#### Meta-Payload Construction

Meta-payloads are constructed using a tool (TBD) that:

1. Takes the image URL, MIME type, and optional metadata as input
2. Computes the image hash (if integrity verification is desired)
3. Encodes the meta-payload as CBOR
4. Optionally wraps the payload in a COSE Sign1 structure using the operator's signing key
5. Outputs the final meta-payload for hosting at the configured URL

The meta-payload URL configured in devices may also reference the expected signing key, ensuring devices only accept meta-payloads signed by the authorized party.

#### Meta-Payload Structure (CBOR)

```cddl
MetaPayload = {
  0: tstr,           ; mime_type - MIME type of actual image
  1: tstr,           ; url - URL to fetch actual image
  ? 2: bstr,         ; tls_ca - CA cert for image URL (DER)
  ? 3: tstr,         ; hash_alg - hash algorithm
  ? 4: bstr,         ; expected_hash - hash of actual image
  ? 5: tstr,         ; boot_args - kernel arguments
  ? 6: tstr,         ; name
  ? 7: tstr,         ; version
  ? 8: tstr          ; description
}
```

#### Meta-Payload Signing (Optional COSE Sign1)

Signing is **controlled by the presence of `-10` (meta_signer)** in `image-begin`:

| `-10` Present? | Meta-Payload Format | Device Behavior |
|----------------|---------------------|-----------------|
| No | Raw CBOR `MetaPayload` | Parse directly, no signature check |
| Yes | COSE Sign1 wrapping `MetaPayload` | Verify signature, then parse payload |

**COSE Sign1 Structure** (when `-10` is present):

```cddl
COSE_Sign1 = [
  protected: bstr,    ; { 1: -7 } = ES256 (or other alg)
  unprotected: {},
  payload: bstr,      ; CBOR-encoded MetaPayload
  signature: bstr
]
```

**Device behavior when `-10` is present:**

1. Fetch meta-payload from URL
2. Parse as COSE Sign1
3. Verify signature using public key from `-10`
4. Reject if signature invalid (error code 12)
5. Extract and parse inner payload as `MetaPayload`
6. Proceed to fetch actual image

#### Protocol Flow (Meta-URL, Signed)

```text
Owner → Device: image-begin {
  -1: "application/x-bmo-meta",         // indicates meta-payload
  -6: 2,                                // delivery_mode = meta-url
  -7: "https://vendor.example.com/fleet-image.cbor",
  -10: h'a401...',                      // COSE_Key - signature required
  3: true
}
Device → Owner: image-ack [true]

Owner → Device: image-end {}

; Device fetches COSE Sign1 meta-payload, verifies signature, extracts:
; {
;   0: "application/x-raw-disk-image",
;   1: "https://images.vendor.com/rhel9-v2.dd.gz",
;   4: h'deadbeef...'
; }
; Device then fetches actual image, verifies hash, boots

Device → Owner: image-result [0, "Meta resolved, image downloaded, booting"]
```

#### Protocol Flow (Meta-URL, Unsigned)

```text
Owner → Device: image-begin {
  -1: "application/x-bmo-meta",
  -6: 2,
  -7: "https://internal.example.com/image-config.cbor",
  // No -10 = no signature verification
  3: true
}
Device → Owner: image-ack [true]

Owner → Device: image-end {}

; Device fetches raw CBOR MetaPayload (no signature wrapper)
; Parses and proceeds to fetch actual image

Device → Owner: image-result [0, "Meta resolved, image downloaded, booting"]
```

## Supported Image Types

### Boot Images

| MIME Type | Description |
|-----------|-------------|
| `application/efi` | UEFI executable application (.efi) |
| `application/vnd.efi` | Vendor-specific EFI application |
| `application/x-iso9660-image` | Bootable ISO image |
| `application/x-raw-disk-image` | Raw disk image |
| `application/x-pxe` | PXE boot image |
| `application/x-ipxe-script` | iPXE boot script |

### UEFI Secure Boot Database Operations

These image types enable enrollment of certificates into UEFI Secure Boot databases. Firmware that does not support database modification SHOULD NAK these with error code 7 (DB Modification Not Supported).

| MIME Type | Description |
|-----------|-------------|
| `application/x-uefi-db-cert` | Enroll certificate into Secure Boot DB (allowed signatures) |
| `application/x-uefi-dbx-hash` | Enroll hash into Secure Boot DBX (forbidden signatures) |
| `application/x-uefi-dbx-cert` | Enroll certificate into Secure Boot DBX (revoked certificates) |

#### DB vs DBX

| Database | Purpose | Effect | Use Case |
|----------|---------|--------|----------|
| **DB** | Allowed Signature Database | Certificates/hashes that ARE trusted for boot | Enroll enterprise signing cert to allow custom EFI apps |
| **DBX** | Forbidden Signature Database | Certificates/hashes that are REVOKED/blocked | Revoke compromised bootloaders, block known-bad hashes |

#### Certificate Enrollment Payload Format

For `application/x-uefi-db-cert` and `application/x-uefi-dbx-cert`, the payload is a DER-encoded X.509 certificate.

For `application/x-uefi-dbx-hash`, the payload is a raw SHA-256 hash (32 bytes) of the image to be blocked.

#### Security Considerations for DB/DBX Modification

**DB Enrollment** (`application/x-uefi-db-cert`):

- Adds a trusted signing certificate
- Images signed by this certificate will be allowed to boot
- **Risk**: Enrolling an untrusted cert allows arbitrary code execution
- **Mitigation**: FDO channel is authenticated; only legitimate owner can enroll

**DBX Enrollment** (`application/x-uefi-dbx-hash`, `application/x-uefi-dbx-cert`):

- Blocks specific hashes or revokes certificates
- Prevents boot of images matching the hash or signed by the revoked cert
- **Risk**: Incorrect DBX entry could brick the device (block legitimate bootloader)
- **Mitigation**: Firmware SHOULD validate that at least one valid boot path remains

#### Protocol Example: Certificate Enrollment

```
Owner → Device: fdo.bmo:image-begin {
  0: 1245,                           / cert size /
  -1: "application/x-uefi-db-cert"   / enroll to DB /
}
Device → Owner: fdo.bmo:image-ack [true]

[Transfer DER certificate...]

Device → Owner: fdo.bmo:image-result [0, "Certificate enrolled in DB"]
```

#### NAK for Unsupported DB Modification

```
Owner → Device: fdo.bmo:image-begin {
  -1: "application/x-uefi-db-cert"
}
Device → Owner: fdo.bmo:image-ack [false, 7, "DB modification not supported"]
```

## Error Codes

### Image Operation Error Codes

| Code | Name | Description |
| ---- | ---- | ----------- |
| 1 | Unknown Image Type | Firmware does not support the image type |
| 2 | Invalid Format | Image format is invalid or corrupted |
| 3 | Size Exceeded | Image exceeds available memory/storage |
| 4 | Boot Failed | Chainload/boot attempt failed |
| 5 | Transfer Error | Error during data transfer |
| 6 | Secure Boot Violation | Image fails Secure Boot verification |
| 7 | DB Modification Not Supported | Firmware cannot modify Secure Boot DB/DBX |
| 8 | DB Modification Failed | DB/DBX enrollment failed (e.g., invalid cert, policy violation) |
| 9 | URL Fetch Failed | Could not download from URL (network error, timeout, 404, etc.) |
| 10 | TLS Validation Failed | TLS certificate validation failed for URL |
| 11 | Hash Mismatch | Downloaded image hash doesn't match expected hash |
| 12 | Meta Signature Invalid | COSE Sign1 signature verification failed for meta-payload |
| 13 | Meta Parse Error | Meta-payload CBOR is malformed or missing required fields |
| 14 | Delivery Mode Not Supported | Firmware does not support the requested delivery mode (url or meta-url) |
| 15 | Provisioning Not Authorized | `image-begin` or `set` failed the verification algorithm in [Authorization of Provisioning Messages](#authorization-of-provisioning-messages). Concretely: body is not tagged `COSE_Sign1` and channel authority is unavailable (either disabled by policy or the TO2 peer lacks provisioning authority), `content_type` mismatch, `x5chain` does not chain to the TO2-proven Owner key, delegate leaf lacks `fdo-ekt-permit-provision`, signature invalid, external AAD mismatch, unrecognised `fdo.bmo.scope` field, or inner payload malformed. |
| 16 | Provisioning Scope Mismatch | The artifact carried `fdo.bmo.scope.guid` and it did not match the GUID in the Ownership Voucher header proven during TO2. The authorisation was minted for a different device. |
| 17 | Provisioning Validity Failed | The artifact carried `not_before` / `not_after` and either the current trusted time falls outside the window, or the device has no clock it considers trustworthy and therefore cannot evaluate the constraint. See [Device Clocks in Firmware](#device-clocks-in-firmware). |
| 18 | Provisioning Superseded | The artifact carried `generation` lower than the highest value the device has already accepted, or the device cannot evaluate `generation` because it has no rollback-protected storage. |

### BIOS Parameter Error Codes

| Code | Name | Description |
| ---- | ---- | ----------- |
| 0 | Success | Parameter set successfully |
| 1 | Unknown Parameter | Parameter name not recognized |
| 2 | Invalid Value | Parameter value is invalid |
| 3 | Permission Denied | Insufficient permissions |
| 4 | Operation Failed | Generic failure |
| 5 | Not Supported | Parameter not supported by firmware |

**Note**: BIOS parameter error codes are mapped to the basic BMO response status codes (0=success, 1=warning, 2=error) in the protocol. Specific error details should be provided in the optional message field.

## Protocol Flow

> **Note on signatures in the diagrams below.** For readability, the ASCII flow diagrams that follow show the *logical* exchange and elide the `COSE_Sign1` envelope. In all diagrams, every arrow labelled `fdo.bmo:image-begin` or `fdo.bmo:set` carries on the wire a tagged `COSE_Sign1` (CBOR tag 18) whose `payload` is the CBOR shown in the diagram, signed and verified per [Authorization of Provisioning Messages](#authorization-of-provisioning-messages). No other `fdo.bmo:*` arrow is signed.

### Successful Boot Image Delivery (with acknowledgment)

```
Owner                           Device (Firmware)
  |                               |
  | fdo.bmo:active = true         |
  |<------------------------------|
  |                               |
  | fdo.bmo:image-begin           |
  | { 3: true, -1: "app/efi" }    |
  |------------------------------>|
  |                               | Validate image type
  |                               |
  | fdo.bmo:image-ack [true]      |
  |<------------------------------|
  |                               |
  | fdo.bmo:image-data-0          |
  |------------------------------>|
  |         ...                   |
  | fdo.bmo:image-data-N          |
  |------------------------------>|
  |                               |
  | fdo.bmo:image-end             |
  |------------------------------>|
  |                               | Verify hash, prepare boot
  |                               |
  | fdo.bmo:image-result          |
  |<------------------------------|
  |                               |
  |         [FDO session ends]    |
  |                               |
  |         [Firmware chainloads] |
```

### Multi-Asset NAK Fallback Flow

```
Owner                           Device (Firmware)
  |                               |
  | fdo.bmo:image-begin           |
  | { 3: true, -1: "app/x-iso" }  |
  |------------------------------>|
  |                               | Check: ISO not supported
  | fdo.bmo:image-ack [false, 1, "ISO not supported"] |
  |<------------------------------|
  |                               |
  | fdo.bmo:image-begin           |
  | { 3: true, -1: "application/efi" } |
  |------------------------------>|
  |                               | Check: EFI supported!
  | fdo.bmo:image-ack [true]      |
  |<------------------------------|
  |                               |
  | fdo.bmo:image-data-0          |
  |------------------------------>|
  |         ...                   |
  | fdo.bmo:image-data-N          |
  |------------------------------>|
  |                               |
  | fdo.bmo:image-end             |
  |------------------------------>|
  |                               | Verify hash, prepare boot
  |                               |
  | fdo.bmo:image-result          |
  |<------------------------------|
  |                               |
  |         [FDO session ends]    |
  |                               |
  |         [Firmware boots EFI] |
```

### Single Asset Rejection

```
Owner → Device: fdo.bmo:image-begin {
  3: true,
  -1: "application/x-unsupported"
}
Device → Owner: fdo.bmo:image-ack [false, 1, "Image type not supported"]
                ; Transfer cancelled - no data sent
```

Owner MAY then attempt a different image type if firmware supports alternatives.

### Complete Onboarding Flow (BIOS + Boot Image)

This example shows a complete bare-metal onboarding using a single BMO FSIM session:

```
Owner                           Device (Firmware)
  |                               |
  | fdo.bmo:active = true         |
  |<------------------------------|
  |                               |
  |  Step 1: Enroll certificate to Secure Boot DB
  |                               |
  | fdo.bmo:image-begin           |
  | { -1: "application/x-uefi-db-cert" } |
  |------------------------------>|
  | fdo.bmo:image-ack [true]      |
  |<------------------------------|
  | fdo.bmo:image-data-0 (cert)   |
  |------------------------------>|
  | fdo.bmo:image-end             |
  |------------------------------>|
  | fdo.bmo:image-result [0]      |
  |<------------------------------|
  |                               |
  |  Step 2: Enable Secure Boot
  |                               |
  | fdo.bmo:set [["secure-boot", true]] |
  |------------------------------>|
  | fdo.bmo:response [0, "Secure Boot enabled"] |
  |<------------------------------|
  |                               |
  |  Step 3: Set BIOS password (lock config)
  |                               |
  | fdo.bmo:set [["bios-password", "EnterpriseKey"]] |
  |------------------------------>|
  | fdo.bmo:response [0, "Password set"] |
  |<------------------------------|
  |                               |
  |  Step 4: Deliver signed boot image
  |                               |
  | fdo.bmo:image-begin           |
  | { -1: "application/efi" }     |
  |------------------------------>|
  | fdo.bmo:image-ack [true]      |
  |<------------------------------|
  | fdo.bmo:image-data-0..N       |
  |------------------------------>|
  | fdo.bmo:image-end             |
  |------------------------------>|
  | fdo.bmo:image-result [0, "Booting..."] |
  |<------------------------------|
  |                               |
  |         [FDO session ends]    |
  |         [Firmware boots EFI]  |
```

## Implementation Requirements

### Device (Firmware) Requirements

**MUST**:

- Advertise `fdo.bmo:active = true` only if capable of booting received images
- **Verify `image-begin` and `set` as tagged `COSE_Sign1` per [Authorization of Provisioning Messages](#authorization-of-provisioning-messages), rooted in the TO2-proven Owner public key**
- **Reject unsigned `image-begin` / `set` with error 15 (Provisioning Not Authorized); MUST NOT install an image or mutate BIOS state on an unsigned message**
- **When a delegate `x5chain` is present, verify that the leaf carries `fdo-ekt-permit-provision` (OID 1.3.6.1.4.1.45724.3.1.7); reject with error 15 otherwise**
- Validate image type before accepting data
- Verify hash when provided in `image-end` or `expected_hash` (`-9`)
- Report errors with appropriate codes
- Validate BIOS parameter names and values before applying
- Return appropriate response codes for each BIOS parameter
- NAK with error code 14 if `delivery_mode` is not supported
- **Treat a TO2 session in which no image was booted and no BIOS parameter was applied as a no-op; MUST NOT record onboarding as complete on the basis of TO2 protocol success alone (see [No-Op Completion](#no-op-completion))**

**SHOULD**:

- Support at least `application/efi` and `application/x-iso9660-image`
- Support at least `secure-boot` and `bios-password` BIOS parameters
- Support all delivery modes (inline, url, meta-url) when network stack is available
- Validate Secure Boot signatures when Secure Boot is enabled
- Verify Secure Boot enablement won't brick the device
- Provide meaningful error messages
- Verify COSE Sign1 signatures when `meta_signer` (`-10`) is provided
- Use provided `tls_ca` (`-8`) for TLS validation when fetching from URLs

**MAY**:

- Support additional image types
- Support additional BIOS parameters (boot-order, vendor-specific)
- Provide boot progress indication
- Support URL and meta-url delivery modes (firmware without network stack may only support inline)

### Owner (Server) Requirements

**MUST**:

- Only send boot images to clients advertising `fdo.bmo`
- **Wrap every `image-begin` and `set` in a tagged `COSE_Sign1` envelope with `external_aad = ["FDO-FSIM-BmoProvision-v1"]`, per [Authorization of Provisioning Messages](#authorization-of-provisioning-messages)**
- **Sign either with the Owner key directly (no `x5chain`) or with a Delegate key whose certificate chain roots in the Owner key and whose leaf carries `fdo-ekt-permit-provision`**
- Specify valid image type in the inner `ImageBegin` payload
- Send data in appropriate chunk sizes for firmware memory constraints (inline mode)
- Set `require_ack: true` for all image-begin messages to enable NAK fallback
- Handle BIOS response codes appropriately (especially errors)
- Enroll required certificates before enabling Secure Boot
- Provide `url` (`-7`) when `delivery_mode` is 1 or 2
- Send `image-end` (with no data chunks) to signal "go fetch" for URL modes

**SHOULD**:

- Provide hash for integrity verification (`expected_hash` for URL modes, `image-end` hash for inline)
- Include descriptive metadata (name, version)
- **Present multiple boot assets in preference order** (EFI → ISO → Raw disk)
- **Implement NAK fallback** - if firmware rejects first asset, try next preferred option
- **Implement delivery mode fallback** - if firmware rejects URL mode, fall back to inline
- Set BIOS password as final configuration step
- Log all BIOS configuration changes for audit
- Present preferred delivery mode first (based on caching, bandwidth, latency considerations)
- Be prepared to fall back to alternative delivery modes for the same image

**Multi-Asset and Delivery Mode Strategy**:

When offering multiple boot assets, servers SHOULD:

1. **Start with most preferred format and delivery mode** (e.g., inline EFI if cached locally)
2. **Use NAK feedback** to determine firmware capabilities (both MIME type and delivery mode)
3. **Progress through preference hierarchy** until firmware accepts
4. **Terminate after first successful transfer** (BMO phase ends)

This ensures firmware receives the **best boot method it supports** while maintaining broad compatibility across diverse firmware implementations. The same image MAY be offered via different delivery modes (e.g., inline first, then URL fallback) to accommodate varying device capabilities.

## Security Considerations

### Provisioning Authority

The normative rules for authorizing `image-begin` and `set` are defined in [Authorization of Provisioning Messages](#authorization-of-provisioning-messages). The central property that section establishes is **trust-the-message, not the-messenger**: the device's decision to install an image or mutate BIOS state is gated on a `COSE_Sign1` signature whose trust anchor is the same Ownership-Voucher-proven Owner key used by TO2, **not** on the identity of the TO2 transport peer. Compromise of a narrowly-scoped Delegate (or of a short-lived `fdo-ekt-permit-provision` certificate) therefore does not confer blanket provisioning authority on the attacker; it bounds the damage to the lifetime and scope of the issued certificate. Deployments MUST NOT accept unsigned `image-begin` / `set` outside of an explicitly-marked debug configuration.

### Secure Boot Integration

When Secure Boot is enabled, firmware MUST validate that received EFI images are signed by trusted keys before execution. The `fdo.bmo` module does not bypass Secure Boot - it only delivers the image; firmware enforces signature verification.

### Image Source Trust

The image is delivered over the FDO TO2 encrypted channel from an authenticated owner. However, the image content itself may come from various sources. Firmware implementations SHOULD:

- Log image metadata for audit purposes
- Verify image signatures when applicable
- Reject images that fail integrity checks

### URL Delivery Security

When using URL-based delivery modes (1 or 2), additional security considerations apply:

**TLS Validation:**

- Devices MUST validate TLS certificates when fetching from HTTPS URLs
- If `tls_ca` (`-8`) is provided, device SHOULD use it as the trust anchor
- If `tls_ca` is not provided, device SHOULD use system trust store
- Devices MUST reject connections with invalid or expired certificates (error code 10)

**Hash Verification:**

- When `expected_hash` (`-9`) is provided, device MUST verify the downloaded image matches
- Hash verification provides end-to-end integrity even if TLS is compromised
- Devices MUST reject images with hash mismatch (error code 11)

**Meta-Payload Signing:**

- When `meta_signer` (`-10`) is provided, device MUST verify the COSE Sign1 signature
- This protects against compromised meta-payload URLs or man-in-the-middle attacks
- The signing key is delivered over the authenticated FDO channel, establishing trust
- Devices MUST reject meta-payloads with invalid signatures (error code 12)

**Network Exposure:**

- URL delivery exposes the device to external network traffic outside the FDO channel
- Firmware SHOULD minimize attack surface by:
  - Using HTTPS only (reject HTTP URLs)
  - Validating URL format before fetching
  - Implementing timeouts to prevent resource exhaustion
  - Limiting redirect following

**Third-Party Delegation Trust Model:**

When using meta-url mode with third-party vendors:

- The owner trusts the vendor by including their signing key (`-10`)
- The vendor controls image selection but cannot modify the trust relationship
- Devices verify the vendor's signature, ensuring image authenticity
- This model enables fleet-wide updates without owner intervention while maintaining security

## Tooling

### CLI: `fdo meta`

The `fdo meta` CLI subcommand provides tools for creating, signing, and verifying meta-payloads. See [CLI_COMMANDS.md](CLI_COMMANDS.md#meta-commands-bmo-meta-payload-tooling) for full documentation.

```bash
fdo meta create         # Create unsigned meta-payload CBOR
fdo meta sign           # Sign with ECDSA private key (COSE Sign1)
fdo meta verify         # Verify signature + optionally print contents
fdo meta create-signed  # Create + sign in one step
fdo meta export-pubkey  # Export public key as COSE_Key CBOR
```

### Server Flag: `-bmo-meta-url`

Configures the example server to use meta-URL delivery mode:

```bash
# Unsigned
fdo server -bmo-meta-url http://cdn.example.com/meta.cbor

# Signed (PEM key auto-converted to COSE_Key)
fdo server -bmo-meta-url "http://cdn.example.com/meta-signed.cbor:signer-key.pem"
```

### Library API: `fsim` Package

The `fsim` package provides the building blocks used by the CLI:

| Function | Description |
|----------|-------------|
| `CreateMetaPayload()` | Build CBOR `MetaPayload` with functional options |
| `SignMetaPayload()` | Wrap CBOR in COSE Sign1 envelope |
| `MarshalSignerPublicKey()` | Convert `crypto.PublicKey` → COSE_Key CBOR |
| `ComputeSHA256()` | Compute SHA-256 hash of data |
| `CoseSign1Verifier.Verify()` | Verify COSE Sign1 signature, return inner payload |

**Functional options for `CreateMetaPayload()`:** `WithBootArgs()`, `WithVersion()`, `WithDescription()`, `WithTLSCA()`.

### Integration Tests

| Test | Command | Description |
|------|---------|-------------|
| `bmo-meta-url` | `./test_examples.sh bmo-meta-url` | Unsigned meta-payload via CLI + HTTP server |
| `bmo-meta-signed` | `./test_examples.sh bmo-meta-signed` | Signed meta-payload + tampered-signature negative test |
| Scenario 8 | `bash tests/supertest/scenario-8-bmo-meta-url.sh` | Full supertest: inline + unsigned + signed + negative |
