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

The NAK-based selection above also covers **delivery modes**: an Owner MAY offer the same image inline and by URL, in preference order, and fall back when the device refuses a mode (`image-ack [false, 14]`) or accepts a URL but cannot fetch it (`image-result` error 9). The generic rules are in [chunking-strategy.md, Offering Alternatives](chunking-strategy.md#offering-alternatives-fallback). For firmware, inline is the dependable fallback; see [Delivery Modes](#delivery-modes).

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
| `fdo.bmo:image-data-<n>` | Owner → Device | CBOR byte string, unsigned | Data chunk number `n` (0-based). Integrity of the reassembled image is bound back to the authorizing `image-begin` through its `expected_hash` field (key `8`). |
| `fdo.bmo:image-end` | Owner → Device | CBOR map, unsigned | Signals that all chunks have been sent (inline delivery) or that the device should now fetch from the URL (URL / meta-URL delivery). |
| `fdo.bmo:image-result` | Device → Owner | CBOR array `[status, ?message]`, unsigned | Final outcome reported by the device. |

#### Wire body of `fdo.bmo:image-begin` — CDDL

On the wire, the body of `fdo.bmo:image-begin` is **a single CBOR item**. Under artifact authority — the only form that works through a conduit the Owner does not fully trust — that item is a `COSE_Sign1` (RFC 9052, §4) wrapped in CBOR tag 18, as specified below. Under channel authority it is instead the bare `ImageBegin` map, accepted only from a TO2 peer holding provisioning authority. The CBOR tag is the discriminator: tag 18 means the envelope form, and a device that has entered the envelope path MUST NOT fall back to the bare form. The generic structure is defined in [chunking-strategy.md, `COSE_Sign1` Structure](chunking-strategy.md#cose_sign1-structure); the CDDL below is its `fdo.bmo` instantiation.

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
    ? "fdo.scope" => Scope                              ; device / validity binding -- see chunking-strategy.md, Scope Constraints
                                                        ;   (legacy label "fdo.bmo.scope" also accepted)
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
    ; --- generic keys (chunking-strategy.md) ---
    ? 0  => uint,                    ; total_size (bytes)
    ? 1  => tstr,                    ; hash alg   (e.g. "sha256")
    ? 3  => bool,                    ; require_ack
    ? 4  => uint,                    ; estimated_duration
    ? 5  => uint,                    ; delivery_mode (0=inline,1=url,2=meta-url)
    ? 6  => tstr,                    ; url
    ? 7  => bstr,                    ; tls_ca (single DER cert)
    ? 8  => bstr,                    ; expected_hash
    ? 9  => bstr,                    ; meta_signer (COSE_Key)
    ; --- fdo.bmo keys ---
      -1 => tstr,                    ; image_type (REQUIRED, MIME type)
    ? -2 => tstr,                    ; boot_args
    ? -3 => tstr,                    ; name
    ? -4 => tstr,                    ; version
    ? -5 => tstr                     ; description
    ; Legacy aliases -6..-10 for keys 5..9: accepted by devices, never emitted
    ; (see chunking-strategy.md, Reserved Key Policy).
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

The device MUST use `FdoBmoProvisionAAD` (not an empty bstr, not some other tag) when computing the `Sig_structure`; a mismatch causes verification to fail and the device returns error 15.

**Rejection rules (short form; normative version: [Verification Algorithm](chunking-strategy.md#verification-algorithm)):**

- Not a tag-18 item, and the device does not permit channel authority or the TO2 peer lacks provisioning authority ⇒ error 15.
- `content_type` ≠ `"application/cbor+fdo.bmo.image-begin"` ⇒ error 15.
- `x5chain` present but does not validate to the TO2-proven Owner key, or does not grant `fdo-ekt-permit-provision` ⇒ error 15.
- Signature verification fails ⇒ error 15.
- `payload` does not decode as an `ImageBegin` map ⇒ error 15.
- Scope field the device cannot evaluate ⇒ error 17 if the obstacle is the clock, 18 if it is `generation`, otherwise 15.
- `guid` present and not matching the voucher GUID ⇒ error 16.
- Validity window present and current trusted time outside it ⇒ error 17.
- `generation` below the device's recorded high-water mark ⇒ error 18.
- Inline delivery and no `expected_hash` (key `8`) in a signed `image-begin` ⇒ error 15.

In no case does a failure in the envelope path permit the message to be reconsidered as a bare, channel-authorised `ImageBegin`.

### BIOS/Firmware Configuration

| Key | Direction | Body on the wire | Purpose |
| --- | --------- | ---------------- | ------- |
| `fdo.bmo:set` | Owner → Device | Signed envelope, or bare `BiosParam` array (see below) | Instructs the device to apply one or more BIOS / firmware parameter changes (e.g. enable Secure Boot, set a BIOS password). Because accepting this message mutates firmware state, it is subject to [Authorization of Provisioning Messages](#authorization-of-provisioning-messages): normally a COSE_Sign1 envelope, or a bare array where the device permits channel authority. |
| `fdo.bmo:response` | Device → Owner | CBOR array `[status, ?message]`, unsigned — exactly **one per `set`** | Result of applying the preceding `set` as a whole. A `set` is atomic: see [Atomicity and Error Handling](#atomicity-and-error-handling). |

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
    ? "fdo.scope" => Scope                              ; device / validity binding (legacy "fdo.bmo.scope" accepted)
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

**Signing / verification input** uses the same `Sig_structure` construction as `image-begin`, with `external_aad = FdoBmoProvisionAAD` and `content_type = "application/cbor+fdo.bmo.set"`. A `set` whose `content_type` is wrong, whose `x5chain` does not validate, whose signature does not verify, or whose scope is unsatisfied MUST be rejected **and the device MUST NOT apply any parameter** from the rejected message (not even those that parse successfully). A `set` whose body is not a tagged `COSE_Sign1` is rejected with error 15 unless the device permits channel authority and the TO2 peer holds provisioning authority. See [§Authorization of Provisioning Messages](#authorization-of-provisioning-messages).

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
| `-6`…`-10` | *(legacy aliases)* | | Deprecated | Former BMO-local numbering of the generic delivery keys below. Devices SHOULD accept them; senders MUST NOT emit them. |

Delivery and integrity fields are **generic** keys defined in [chunking-strategy.md, Begin Message Fields](chunking-strategy.md#table-begin-message-fields):

| Key | Name | Legacy alias | Description |
| --- | ---- | ------------ | ----------- |
| `5` | delivery_mode | `-6` | 0=inline (default), 1=url, 2=meta-url. See [Delivery Modes](#delivery-modes). |
| `6` | url | `-7` | URL of the image (mode 1) or meta-payload (mode 2). Required when `delivery_mode` ≠ 0. |
| `7` | tls_ca | `-8` | Single DER-encoded CA certificate used as TLS trust anchor for `url`. |
| `8` | expected_hash | `-9` | Hash of the final image (algorithm in key `1`). SHOULD always be present; REQUIRED for a signed inline `image-begin`. |
| `9` | meta_signer | `-10` | `COSE_Key` of a third-party meta-payload publisher. If present, the meta-payload MUST be signed by this key. |

**Notes:**

- Only `image_type` is required; all other fields are optional
- `boot_args` is the most commonly used optional field - it passes kernel command line arguments (e.g., kickstart URLs, installer options)
- `name`, `version`, and `description` are informational only - implementations may log them but are not required to act on them
- `estimated_duration` (key `4`, from the generic chunking spec) is especially relevant for `fdo.bmo` because firmware-stage transfers often involve large boot images (multi-GiB ISOs) over constrained links, and UEFI watchdog timers are typically more aggressive than OS-level timeouts. Owners SHOULD include this field for any image expected to take longer than a few minutes to transfer and apply. See `chunking-strategy.md` [Estimated Duration](chunking-strategy.md#estimated-duration).
- `tls_ca` is a **single certificate** (root or intermediate CA), not a chain. This mirrors UEFI Secure Boot DB behavior where individual certificates are enrolled. Chain validation occurs at TLS handshake time using the provided CA as trust anchor.
- When `delivery_mode` is 0 (inline) or omitted, the existing chunked transfer behavior applies
- When `delivery_mode` is 1 or 2, no `image-data-*` chunks are sent; the device fetches from the URL after `image-end`, and MUST authenticate what it fetches per [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content)

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

Exactly one CBOR response for each `set` message, describing the outcome of the whole message:

```cbor
[
  0,                    / status_code: 0=success, 1=warning, 2=error /
  "Applied 2 parameters" / optional message /
]
```

| Status | Meaning |
| ------ | ------- |
| `0` success | Every parameter in the `set` was applied. |
| `1` warning | Every parameter was applied; the message explains the caveat (for example, a reboot is required). |
| `2` error | **No** parameter was applied. The message SHOULD identify the parameter or condition that caused the rejection. |

### Atomicity and Error Handling

A `set` message is **atomic**: either every parameter in it takes effect, or none does. The device MUST NOT leave firmware in a state where only some of the parameters of one `set` have been applied.

- The device MUST validate every parameter (name, value type and value) before applying any of them. A malformed or unsupported parameter rejects the whole message with status `2`.
- If applying a parameter fails after others have been applied, the device MUST roll back the parameters it has applied, and respond with status `2`.
- A device that cannot apply a multi-parameter `set` atomically MUST reject it with status `2`, without applying any parameter. Such a device may still accept a `set` containing a single parameter.
- A device without a BIOS configuration interface MUST still respond, with status `2` (for example `"BIOS configuration not supported"`).
- The device MUST send exactly one `fdo.bmo:response` for each `set` it receives, including when it rejects the message. (A `set` refused under [Authorization of Provisioning Messages](#authorization-of-provisioning-messages) is answered with `error` code 15 instead.)
- The Owner MUST wait for the response to a `set` before considering the module complete, so that the result is not lost (see chunking-strategy.md "Completion Ordering").

Grouping settings that depend on each other into one `set` is the intended use: for example, setting a BIOS password together with the Secure Boot state it protects, so the device is never left with one but not the other. Settings that are independent MAY be sent as separate `set` messages, which makes it easier to tell which one a device rejected:

```
fdo.bmo:set = [["secure-boot", true], ["bios-password", "EnterpriseKey"]]
fdo.bmo:response = [0, "Applied 2 parameters"]

fdo.bmo:set = [["boot-order", "pxe,disk"]]
fdo.bmo:response = [2, "boot-order: unsupported value; no parameters applied"]
```

## Authorization of Provisioning Messages

> **The authorization model is specified normatively in [chunking-strategy.md, "Authorization of Begin Messages"](chunking-strategy.md#authorization-of-begin-messages)**, which applies to every FSIM that transfers security-sensitive content. That section defines channel vs. artifact authority, the trust anchor, signers (Owner-direct or Delegate `x5chain` granting `fdo-ekt-permit-provision`), the no-downgrade rule, the `COSE_Sign1` structure, scope constraints, device clocks, the verification algorithm, and Owner/Delegate requirements. Fetched content (URL and meta-URL delivery) is governed by [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content).
>
> This section contains only what is specific to `fdo.bmo`: its declarations under that model, and BMO-specific rules.
>
> For a high-level introduction, see [Provisioning Security: Authorizing What Gets Installed on Your Devices](../../go-fdo/provisioning-security.md) and the article *Trust the Message, or Trust the Messenger? Security and Authority in FDO Bare-Metal Onboarding*.

### Why `fdo.bmo` Is Authorization-Gated

`fdo.bmo` conveys operations that amount to full control of the device: installing a bootable image, enrolling UEFI Secure Boot DB/DBX certificates, and mutating firmware configuration. Every one of these MUST be authorized by a party holding provisioning authority — the Owner, or a Delegate whose chain grants `fdo-ekt-permit-provision` (PERM.7) — either as the TO2 peer (channel authority) or as the signer of the message (artifact authority). A peer holding only `fdo-ekt-permit-onboard-*` permissions may relay a signed message but can never originate one.

### FSIM Declarations

Per [FSIM Declarations](chunking-strategy.md#fsim-declarations), `fdo.bmo` declares:

| Message | Inner payload (channel-authority form) | `content_type` | `external_aad` tag |
| ------- | -------------------------------------- | -------------- | ------------------ |
| `fdo.bmo:image-begin` | `ImageBegin` map ([§ImageBegin](#imagebegin)) | `application/cbor+fdo.bmo.image-begin` | `"FDO-FSIM-BmoProvision-v1"` |
| `fdo.bmo:set` | `BiosParam` array ([§BiosParam](#biosparam-set-message)) | `application/cbor+fdo.bmo.set` | `"FDO-FSIM-BmoProvision-v1"` |

Meta-payload key classification ([Meta-Payload Structure](chunking-strategy.md#meta-payload-structure)):

| Meta-payload key | Field | Class |
| ---------------- | ----- | ----- |
| `5` | `boot_args` — kernel / boot command line | **Instruction** |

All other `fdo.bmo:*` messages (`active`, `image-ack`, `image-data-<n>`, `image-end`, `image-result`, `response`, `error`) carry only bookkeeping, grant no authority, and are always transmitted unsigned.

### BMO-Specific Rules

- **Image integrity.** Image bytes (`image-data-<n>`, or content fetched by URL) are never signed individually. They are bound to the authorizing `image-begin` per [Hash Handling](chunking-strategy.md#hash-handling) and [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content). In particular, a signed inline `image-begin` without `expected_hash` (key `8`) authorizes only metadata and MUST be rejected.
- **`set` is all-or-nothing.** A `set` that fails authorization MUST be rejected and the device MUST NOT apply **any** parameter from it, not even those that parse successfully.
- **Scope header label.** The generic label is `"fdo.scope"`. For backward compatibility, `fdo.bmo` devices MUST also accept the legacy label `"fdo.bmo.scope"` with identical meaning, and MUST reject an artifact carrying both. Senders SHOULD emit `"fdo.scope"`.
- **`boot_args` from an unauthenticated meta-payload.** Because key `5` is an instruction field, an unauthenticated meta-payload carrying `boot_args` MUST be rejected with error 15. A kernel command line such as `init=/bin/sh` takes over the device without any change to the image, so a pinned image hash does not make it safe.
- **Secure Boot is separate.** Authorization decides whether the device *accepts* an image. When Secure Boot is enabled, firmware still enforces its own signature policy at boot (see [Secure Boot Integration](#secure-boot-integration)).

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

Delivery modes — inline, URL and meta-URL — are generic. Their semantics, the meta-payload format and signing, fallback between modes, and the rules for authenticating fetched content are specified in [chunking-strategy.md, Delivery Modes](chunking-strategy.md#delivery-modes) and [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content). `fdo.bmo` uses them unchanged, through generic keys `5`–`9` (legacy aliases `-6`…`-10`, see [§ImageBegin](#imagebegin)). Only the following is specific to boot images in firmware.

### Firmware considerations

- **Inline is the dependable mode.** Modes 1 and 2 are OPTIONAL, and much firmware has no usable network stack for them. Owners SHOULD always be able to fall back to inline delivery.
- **Do not rely on HTTPS in firmware.** Firmware TLS support (e.g. EDK2 `TlsDxe`) is a platform build option and is often absent; where present it provides only one-way server authentication and has **no general root CA store** — trust anchors must be provisioned separately (e.g. the `TlsCaCertificate` variable or BIOS setup), and an application may not be able to supply its own. Some firmware HTTP stacks also refuse plain `http://` by default. Devices therefore SHOULD assume no validated TLS, in which case TLS is never evidence and **every image fetched by URL MUST be pinned by a hash** (`expected_hash`, key `8`, or the hash in a signed meta-payload); otherwise it is refused with error 19. Owners SHOULD always include `expected_hash`.
- **Memory.** Firmware typically buffers the whole image in RAM before verifying and booting it; `total_size` (key `0`) lets the device refuse an image it cannot hold (`image-ack [false, 2]`) before anything is transferred.

### Mode 2: Meta-Payload Indirection

Meta-URL delivery lets an image publisher — an OS vendor naming its current release, or the Owner's own release process — change which image is installed without changing any `image-begin` that points at it. A signed meta-payload is also how an image format that cannot be signed itself (raw disk images, ISOs) gets a signature: the signature covers the image's hash. See [chunking-strategy.md, Meta-Payload](chunking-strategy.md#meta-payload).

BMO-specific points:

- `image_type` (`-1`) in a meta-URL `image-begin` is `application/x-bmo-meta`; the type of the image actually booted is the meta-payload's `mime_type`.
- Meta-payload key `5` is `boot_args`, an **instruction** field: it is honoured only from an authenticated (signed) meta-payload, and an unauthenticated meta-payload carrying it is rejected with error 15. A kernel command line such as `init=/bin/sh` takes over the device without any change to the image, so a pinned image hash does not make it safe.
- Meta-payloads are produced and verified with the `fdo meta` tool (see [Tooling](#tooling)).

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
| 9–19 | *(generic)* | Delivery and authorization errors — URL fetch, TLS validation, hash mismatch, meta-payload signature/parse, unsupported delivery mode, provisioning not authorized, scope mismatch, validity failed, superseded, unauthenticated source. Defined in [chunking-strategy.md, Transfer Error Codes](chunking-strategy.md#transfer-error-codes). |

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
- **Authorize every `image-begin` and `set` per [Authorization of Provisioning Messages](#authorization-of-provisioning-messages)**: accept a signed (`COSE_Sign1`) message only if its signer is the Owner or a Delegate whose `x5chain` grants `fdo-ekt-permit-provision`; accept an unsigned message only if the TO2 peer holds provisioning authority. Reject otherwise with error 15
- **Determine the TO2 peer's provisioning authority from how `TO2.ProveOVHdr` was verified** (Owner key directly, or a Delegate chain and its permissions) — never from the mere availability of the Owner key
- **Authenticate all content fetched by URL or meta-URL** per [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content); refuse unauthenticated content with error 19
- Validate image type before accepting data
- Verify every hash provided, in `image-end` or `expected_hash` (key `8`)
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
- Accept the legacy key aliases `-6`…`-10` and the legacy scope label `"fdo.bmo.scope"`
- Use provided `tls_ca` (key `7`) for TLS validation when fetching from URLs

**MAY**:

- Support additional image types
- Support additional BIOS parameters (boot-order, vendor-specific)
- Provide boot progress indication
- Support URL and meta-url delivery modes (firmware without network stack may only support inline)

### Owner (Server) Requirements

**MUST**:

- Only send boot images to clients advertising `fdo.bmo`
- **Hold provisioning authority for every `image-begin` and `set` it originates**: either send it unsigned while holding the Owner key or a Delegate chain granting `fdo-ekt-permit-provision` (channel authority), or relay a `COSE_Sign1` signed by such a party with `external_aad = ["FDO-FSIM-BmoProvision-v1"]` (artifact authority). An onboarding service holding only `fdo-ekt-permit-onboard-*` permissions MUST use artifact authority
- **Use artifact authority when delivering through a conduit it does not fully trust**
- Emit the generic keys `5`–`9` and the scope label `"fdo.scope"`; MUST NOT emit the legacy aliases
- Specify valid image type in the inner `ImageBegin` payload
- Send data in appropriate chunk sizes for firmware memory constraints (inline mode)
- Set `require_ack: true` for all image-begin messages to enable NAK fallback
- Handle BIOS response codes appropriately (especially errors)
- Enroll required certificates before enabling Secure Boot
- Provide `url` (key `6`) when `delivery_mode` is 1 or 2
- Send `image-end` (with no data chunks) to signal "go fetch" for URL modes

**SHOULD**:

- Provide `expected_hash` (key `8`) in every `image-begin` (REQUIRED for signed inline transfers)
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

The normative rules for authorizing `image-begin` and `set` are defined in [chunking-strategy.md, Authorization of Begin Messages](chunking-strategy.md#authorization-of-begin-messages), with `fdo.bmo`'s declarations in [Authorization of Provisioning Messages](#authorization-of-provisioning-messages). The model establishes two modes, and every provisioning decision is gated on one of them, rooted in the same Ownership-Voucher-proven Owner key used by TO2:

- **Channel authority** — trust the messenger. An unsigned message is accepted only when the TO2 peer itself holds provisioning authority: it is the Owner, or it presented a Delegate chain granting `fdo-ekt-permit-provision`.
- **Artifact authority** — trust the message. A `COSE_Sign1` message is accepted on the strength of its signer (the Owner, or a Delegate whose `x5chain` grants `fdo-ekt-permit-provision`), regardless of who delivered it.

The security property is that **a party holding only onboarding permissions can never originate a provisioning decision**: it cannot use channel authority, and it cannot mint a valid artifact. It can only relay artifacts that a provisioning authority produced. Compromise of such a Delegate therefore does not confer provisioning authority on the attacker. Compromise of a party that *does* hold `fdo-ekt-permit-provision` is bounded by the lifetime and scope of the certificate that granted it.

Requiring artifact authority for every message is permitted as an opt-in strict policy (see [Channel Authority](chunking-strategy.md#channel-authority)), but is not the baseline: an Owner that runs its own onboarding service, or has deliberately granted `fdo-ekt-permit-provision` to the party that does, gains nothing from signing each message.

### Secure Boot Integration

When Secure Boot is enabled, firmware MUST validate that received EFI images are signed by trusted keys before execution. The `fdo.bmo` module does not bypass Secure Boot - it only delivers the image; firmware enforces signature verification.

### Image Source Trust

The image is delivered over the FDO TO2 encrypted channel from an authenticated owner. However, the image content itself may come from various sources. Firmware implementations SHOULD:

- Log image metadata for audit purposes
- Verify image signatures when applicable
- Reject images that fail integrity checks

### URL Delivery Security

URL and meta-URL delivery fetch content from outside the authenticated FDO channel. The normative rules are in [chunking-strategy.md, Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content). In summary:

- **Every fetched object must be authenticated** — by a hash pinned in an authorized parent, by a signature (named `meta_signer`, the Owner, or a PERM.7 Delegate `x5chain`), or by TLS the device actually validated. Otherwise the device refuses (error 19).
- **A hash SHOULD always be provided.** It is end-to-end, independent of TLS, and the only evidence that pins exact content.
- **TLS counts only if validated.** Devices MUST validate certificates when fetching over HTTPS (error 10 on failure), using `tls_ca` (key `7`) or a trusted system store. Firmware without a TLS stack or trust anchor cannot use TLS as evidence.
- **An unauthenticated meta-payload is only a pointer**, and MUST be rejected if it carries `boot_args` or `tls_ca`.
- Firmware SHOULD minimize network exposure: prefer HTTPS, validate URL format, bound timeouts and redirects.

**Third-party publisher trust model.** Naming a publisher's key in `meta_signer` (key `9`) hands that publisher the choice of image for as long as the `image-begin` is honoured. The publisher cannot widen that trust — it can only sign meta-payloads the device verifies against the named key — but Owners SHOULD name only publishers they would trust with `fdo-ekt-permit-provision`.

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

> **Note:** Under [Authenticating Fetched Content](chunking-strategy.md#authenticating-fetched-content), an unsigned meta-payload fetched over plain HTTP is accepted only if `image-begin` also pins the image hash (key `8`), and only as a pointer (no `boot_args` / `tls_ca`). The unsigned form above is refused (error 19) unless the server also sends that hash.

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
