# FDO Service Info Module: fdo.defer

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

**Module Name**: `fdo.defer`
**Version**: 1.0
**Status**: Draft

The `fdo.defer` (Deferred Onboarding) FSIM allows an Owner to tell a Device *"I know you, I own you, but I am not ready for you yet."*

A Device may complete TO2 authentication against an Owner that legitimately holds its Ownership Voucher, but which has no provisioning content to deliver — no boot asset, no autoinstall configuration, no payload — because a provisioning profile has not yet been assigned to that Device. Without this module the Owner has only two unsatisfactory options:

- **Reject TO2** — indistinguishable, from the Device's perspective, from "you are not authorized."
- **Complete TO2 with no ServiceInfo** — the Device sees protocol success but received nothing, and is left in an undefined state.

This module makes the third case explicit and machine-readable.

### Scope

This module conveys **session disposition** only. It transfers no provisioning content, mutates no Device state, and is independent of the [chunking strategy](chunking-strategy.md) — its payloads are a few bytes.

## Design Model

Provisioning configuration is treated as **atomic**: an Owner either has a complete configuration for a Device or it has none. This yields a strictly binary branch at the start of the ServiceInfo exchange:

| Owner state | Behavior |
| ----------- | -------- |
| Configuration available | Activate the content-bearing modules (`fdo.bmo`, `fdo.payload`, …) and deliver. `fdo.defer` is not used. |
| No configuration, none expected | Activate nothing. Session is a no-op (see [Interaction with `fdo.bmo`](#interaction-with-fdobmo)). |
| No configuration, expected later | Activate **only** `fdo.defer` and issue a directive. |

Because the Owner never publishes a module manifest — it simply activates modules as it uses them — an Owner that defers names no other module on the wire. There is no ordering problem to resolve and no need to advertise modules for which no content exists.

### Capability Negotiation and Graceful Degradation

Support is negotiated by the existing FDO mechanism: the Device lists `fdo.defer` in `devmod:modules`, and the Owner activates it with `fdo.defer:active = true`.

A Device that does **not** support this module never sees it. Combined with the binary branch above, such a Device completes TO2 having done nothing — which is precisely the pre-existing behavior this module improves upon. Non-supporting Devices are therefore no worse off than before, and the fallback is the no-op completion rule defined in [`fdo.bmo`](fdo.bmo.md#no-op-completion): nothing happened, so try again.

This same property makes the module safely extensible. A Device that does not recognize a future `action` value degrades to the same path.

## ServiceInfo Module Key-Value Pairs

### Module Activation

| Key | Direction | Type | Description |
| --- | --------- | ---- | ----------- |
| `fdo.defer:active` | Bidirectional | `bool` | Module activation status |

### Directive

| Key | Direction | Body on the wire | Purpose |
| --- | --------- | ---------------- | ------- |
| `fdo.defer:directive` | Owner → Device | CBOR map | Instructs the Device to defer onboarding. |
| `fdo.defer:ack` | Device → Owner | CBOR array | Reports whether the directive was understood and the delay the Device will actually apply. |

Unlike the provisioning messages of [`fdo.bmo`](fdo.bmo.md#authorization-of-provisioning-messages), `fdo.defer` payloads are **not** carried in a `COSE_Sign1` envelope. Signing is required there because accepting a message installs a bootable image or mutates firmware state. A defer directive mutates nothing; it only affects the Device's own scheduling, and it is already delivered inside the authenticated, encrypted, replay-protected TO2 channel. See [Security Considerations](#security-considerations).

#### Directive Structure

The `fdo.defer:directive` value is a CBOR map using small unsigned integer keys, following the key conventions of the [chunking strategy](chunking-strategy.md): non-negative keys are reserved to this specification, negative keys are available for vendor extensions.

| Key | Field | Type | Required | Description |
| --- | ----- | ---- | -------- | ----------- |
| 0 | `action` | `uint` | Yes | Disposition. See [Actions](#actions). |
| 1 | `delay` | `uint` | Yes | Seconds to wait before the next attempt. Subject to Device clamping. |
| 2 | `jitter` | `uint` | No | Maximum additional random delay, in seconds. The Device selects a uniform random value in `[0, jitter]` and adds it to `delay`. |
| 3 | `reason_code` | `uint` | No | Machine-readable reason. See [Reason Codes](#reason-codes). |
| 4 | `message` | `tstr` | No | Human-readable explanation, for logging. |

```cddl
defer-directive = {
    0: uint,        ; action
    1: uint,        ; delay (seconds)
  ? 2: uint,        ; jitter (seconds)
  ? 3: uint,        ; reason_code
  ? 4: tstr         ; message
}
```

#### Ack Structure

```cddl
defer-ack = [
    understood      : bool,   ; false if the action value was not recognized
    effective_delay : uint,   ; seconds the Device will actually wait, post-clamping
  ? message         : tstr
]
```

`effective_delay` reports the value **after** the Device has applied its own clamping and jitter. This lets the Owner know when to expect the Device back — which it needs in order to satisfy its session-lifetime obligation for `wait` (see [Owner Requirements](#owner-server-requirements)).

### Actions

| Value | Action | Meaning |
| ----- | ------ | ------- |
| 0 | `retry` | Complete this TO2 session normally, then re-attempt onboarding from TO1 after the delay. |
| 1 | `wait` | Remain in this TO2 session. Poll again after the delay. |
| 2–127 | — | Reserved for future versions of this specification. |

Values ≥ 128 are reserved for vendor use.

#### Action 0 — `retry`

The Device completes the TO2 protocol **normally** — it does not abort, and does not emit an error message. `Done` / `Done2` are exchanged as usual and all session state is released on both sides. After the effective delay elapses, the Device re-enters the onboarding sequence at TO1.

This is the mandatory-to-implement action, and the correct choice for deferrals on human timescales (minutes to days) — for example, waiting for an administrator to assign a provisioning profile. It holds no state on either side.

#### Action 1 — `wait`

The Device remains within the ServiceInfo exchange. After the effective delay elapses it sends a `DeviceServiceInfo` message with no content, to which the Owner responds with one of:

- another `fdo.defer:directive` (either `wait` or `retry`);
- content, by activating a content-bearing module and proceeding normally;
- session completion.

Each poll is a discrete, individually authorized exchange: the Device polls again **only** because the Owner explicitly told it to. The Device performs no autonomous looping, and an Owner that becomes unreachable terminates the session as any transport failure would.

The Owner need not deactivate `fdo.defer` before delivering content; it may simply stop sending directives and activate the module it intends to use.

`wait` avoids a full TO1 + TO2 re-handshake and is appropriate for short deferrals — an image being built, a backend workflow in flight. It is **optional to implement for Owners** and, per [Device Requirements](#device-requirements), MUST degrade to `retry` on Devices that do not implement it.

### Reason Codes

| Code | Meaning |
| ---- | ------- |
| 0 | Unspecified |
| 1 | No provisioning configuration assigned to this device |
| 2 | Configuration is being prepared |
| 3 | Awaiting administrative approval |
| 4 | Owner service temporarily at capacity |

Reason codes are advisory. A Device MUST NOT alter its deferral behavior based on the reason code; it exists for logging and operator diagnosis.

## Protocol Flow

### Retry

```text
Owner → Device: fdo.defer:active = true
Device → Owner: fdo.defer:active = true
Owner → Device: fdo.defer:directive { 0: 0, 1: 300, 2: 60, 3: 1,
                                      4: "No provisioning profile assigned" }
Device → Owner: fdo.defer:ack [true, 342]
                ; 300 + 42s jitter
[TO2 completes normally: Done / Done2]
[Device waits 342 s, then re-attempts from TO1]
```

### Wait, Then Content

```text
Owner → Device: fdo.defer:active = true
Device → Owner: fdo.defer:active = true
Owner → Device: fdo.defer:directive { 0: 1, 1: 30, 3: 2, 4: "Building image" }
Device → Owner: fdo.defer:ack [true, 30]

[Device waits 30 s]
Device → Owner: (empty ServiceInfo)
Owner → Device: fdo.defer:directive { 0: 1, 1: 30, 3: 2 }
Device → Owner: fdo.defer:ack [true, 30]

[Device waits 30 s]
Device → Owner: (empty ServiceInfo)
Owner → Device: fdo.bmo:active = true
                ; configuration resolved; proceed with BMO normally
```

### Wait Not Supported — Degradation to Retry

```text
Owner → Device: fdo.defer:directive { 0: 1, 1: 30 }
Device → Owner: fdo.defer:ack [false, 600]
                ; Device does not implement `wait`; treats as `retry`
                ; with its own default delay
[TO2 completes normally; Device re-attempts from TO1 after 600 s]
```

## Interaction with `fdo.bmo`

`fdo.bmo` is terminal: the first successfully received boot asset is chainloaded and the FDO session ends. A deferral and a successful boot are therefore mutually exclusive.

- An Owner MUST NOT issue a defer directive in a session in which a boot asset has been accepted.
- If a Device has performed a terminal action, it MUST ignore any subsequent defer directive.
- A Device whose deferral is never resolved — including one that abandons a `wait` under its own policy — falls back to the [no-op completion](fdo.bmo.md#no-op-completion) rule.

Firmware-resident implementations warrant particular care. A Device deferring from within a chainloaded UKI holds RAM and prevents normal boot for the duration. Such implementations SHOULD prefer `retry` over `wait`, and SHOULD apply a shorter deferral budget than a Device deferring from firmware.

## Credential State

**An Owner that may issue a defer directive MUST have selected the Credential Reuse Protocol in `TO2.SetupDevice`.**

If the Owner instead rotates credentials and then defers, the Device returns under a new GUID, any Rendezvous registration must be redone, and the retry path becomes needlessly fragile.

This requirement is trivially satisfiable despite `TO2.SetupDevice` preceding the ServiceInfo exchange. The reuse decision is made from the Ownership Voucher, which carries the GUID, and "do I have configuration for this GUID?" is exactly the question the deferral branch turns on. The Owner evaluates the same lookup one message earlier.

More generally, credential rotation exists to sever the reachability of the manufacturing and distribution chain, which is a concern of *first* onboarding, and to effect genuine ownership transfer, which is a concern of *offboarding*. Neither applies to a deferred session, in which no ownership change occurs at all.

## Implementation Requirements

### Device Requirements

**MUST**:

- Advertise `fdo.defer` in `devmod:modules` only if able to honor at least action 0 (`retry`)
- Implement action 0 (`retry`)
- Treat an unrecognized `action` value as `retry`, using the Device's own default delay, and report `understood: false` in `fdo.defer:ack`
- Clamp `delay` to the Device's own minimum polling interval. A Device MUST NOT poll more frequently than once per second regardless of the value received, and MUST apply its default when `delay` is absent, zero, or below its floor
- Report the post-clamping, post-jitter value in `fdo.defer:ack`
- Complete TO2 normally on `retry` — not abort, and not emit a protocol error
- Ignore a defer directive if a terminal action has already been performed in the session

**SHOULD**:

- Use a minimum polling floor of 10 seconds
- Apply its own cumulative deferral budget, independent of any value supplied by the Owner, and fall back to no-op completion on expiry. This is Device policy rather than a protocol safeguard — each poll is individually Owner-authorized, so there is no runaway-loop hazard — but a healthy Owner that defers indefinitely will otherwise prevent the Device from ever booting
- Log `reason_code` and `message` for operator diagnosis
- Support action 1 (`wait`)

**MAY**:

- Apply additional jitter beyond that requested
- Abandon a `wait` and fall back to `retry` at any time under local policy

### Owner (Server) Requirements

**MUST**:

- Select the Credential Reuse Protocol in `TO2.SetupDevice` for any Device that may be deferred (see [Credential State](#credential-state))
- Ensure that the remaining TO2 session lifetime exceeds the `wait` delay it advertises, so that a returning Device does not present an expired session token. An Owner chaining multiple `wait` directives MUST issue a `retry` before the session would otherwise expire
- Not activate content-bearing modules for which it has no content
- Not issue a defer directive after a terminal action has been accepted by the Device

**SHOULD**:

- Supply `reason_code`, and `message` where an operator-facing explanation is useful
- Supply `jitter` for `retry` at fleet scale, to avoid synchronized re-attempts after a correlated event such as a site-wide power restoration
- Prefer `retry` where the expected deferral exceeds a few minutes
- Re-evaluate provisioning availability on each `wait` poll

**MAY**:

- Issue `retry` to a Device that has reported `understood: false`, rather than continuing to attempt `wait`

## Security Considerations

### Deferral Extends the Supply-Chain Exposure Window

Deferring on a Device's *first* contact necessarily implies credential reuse, which means the Device continues to hold credentials reachable by its manufacturing and distribution chain for the full duration of the deferral. Rotation at first onboarding is what ordinarily severs that reachability.

The exposure is negligible for short deferrals and material for long ones. Owners SHOULD bound total deferral time for Devices that have not yet completed a first successful onboarding, and SHOULD prefer resolving configuration before first contact where the deployment model allows it.

### No Signature Requirement

Defer directives are unsigned. They are delivered inside the TO2 channel, which is authenticated, encrypted and replay-protected, and they cause no Device state mutation — the worst outcome an attacker who could inject one might achieve is delayed onboarding, which is equally achievable by dropping packets.

This is a deliberate contrast with `fdo.bmo:image-begin` and `fdo.bmo:set`, which require `COSE_Sign1` precisely because accepting them installs a bootable image or mutates firmware state.

### Denial of Service

A compromised or malfunctioning Owner could defer a Device indefinitely. Device-side clamping of the polling interval bounds the request rate, and the Device's own cumulative deferral budget bounds total time before fallback. Neither protects against an Owner that is simply never ready — but such an Owner could equally withhold content altogether, so this module introduces no new exposure.

## Related Specifications

- [`fdo.bmo`](fdo.bmo.md) — Bare Metal Onboarding; defines the [no-op completion](fdo.bmo.md#no-op-completion) fallback that terminates an unresolved deferral
- [Chunking Strategy](chunking-strategy.md) — source of the integer key-space conventions used by `fdo.defer:directive`
