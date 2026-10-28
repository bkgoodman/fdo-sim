Copyright 2026 FIDO Alliance

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

------------------

> **PLACEHOLDER / DRAFT.** This document reserves the module name and
> records the intended design. It is not a specification yet: the key
> names and encodings below are a starting point for discussion, and
> nothing here is normative. Tracked in [TODO.md](../TODO.md).

This document will define the 'tpm-quote' FDO serviceinfo module (FSIM).
It lets the Owner obtain TPM evidence of the device's boot state
(measured-boot PCR values and the event log) during onboarding, and
binds that evidence to the Device Key that attested in TO2.

## Motivation

In TO2, the device proves possession of its Device Key. With the key in
a TPM, as described in "Securing FDO Credentials in the TPM"
(`securing-fdo-in-tpm.bs`, section "At Onboarding Time: Proving
Provenance"), that proves the TPM took part. It does not prove anything
about the software that asked the TPM to sign. A compromised device, or
one booted into an unexpected environment, produces the same signature.

An Owner that wants to decide based on boot state needs a TPM Quote.
A Quote alone is not enough: the Owner must also know that the quoting
TPM is the TPM that holds the voucher's Device Key. This module carries
the standard TPM evidence chain for both:

1. **EK to AK.** The device creates a restricted Attestation Key (AK).
   The Owner proves that the AK lives in the TPM identified by the EK
   certificate (`TPM2_MakeCredential` / `TPM2_ActivateCredential`).
2. **AK to Device Key.** The device certifies the Device Key with the
   AK (`TPM2_Certify`). The default Device key template in the TPM
   specification permits this through its authPolicy
   (`TPM2_PolicyCommandCode(TPM_CC_Certify)`).
3. **AK to boot state.** The device returns a `TPM2_Quote` by the AK over
   the PCRs that the Owner selects, with an Owner nonce, plus the event
   log.

The Owner decides which measurements are acceptable. The module only
transports evidence; it defines no policy.

The Device Key itself is not usable for quotes: it is an unrestricted
signing key, so any software with TPM access could sign a forged
`TPMS_ATTEST` with it.

## Draft key-value pairs (non-normative)

| Direction | Key Name | Value | Meaning |
|:----------|:---------|:------|:--------|
| o <-> d | `fdo.tpm-quote:active` | `bool` | Activate or deactivate the module |
| o --> d | `fdo.tpm-quote:ak-req` | TBD | Ask the device to create an AK |
| o <-- d | `fdo.tpm-quote:ak-res` | TBD | AK public area, EK certificate (or EK public key) |
| o --> d | `fdo.tpm-quote:credential` | TBD | `TPM2_MakeCredential` output (credentialBlob, secret) |
| o <-- d | `fdo.tpm-quote:activated` | TBD | Secret recovered with `TPM2_ActivateCredential` |
| o --> d | `fdo.tpm-quote:certify-req` | TBD | Nonce for certifying the Device Key |
| o <-- d | `fdo.tpm-quote:certify-res` | TBD | `TPMS_ATTEST` and AK signature over the Device Key |
| o --> d | `fdo.tpm-quote:quote-req` | TBD | PCR selection and nonce |
| o <-- d | `fdo.tpm-quote:quote-res` | TBD | `TPMS_ATTEST`, AK signature, PCR values |
| o <-- d | `fdo.tpm-quote:eventlog` | TBD | Event log (TCG PC Client / IMA), chunked |
| o <-- d | `fdo.tpm-quote:error` | TBD | Error indication |

## Open questions

- [ ] Nonce binding: is an Owner nonce enough (the module runs inside
      the encrypted TO2 session, whose key exchange the Device Key
      signed), or should `qualifyingData` also include a value derived
      from the TO2 session?
- [ ] **Trust anchor for the AK** (anchor order and demo status: see
      "Trust anchor for the AK" below). The Quote is only meaningful if the
      Owner knows the AK is a genuine TPM key. Candidates, to be
      accepted side by side:
      - TPM vendor: EK certificate (TPM NV, or vendor online service for
        firmware TPMs) chaining to the TPM vendor CA. Requires the Owner
        to hold TPM vendor roots.
      - Device manufacturer, whom the Owner already trusts through the
        voucher: EK public key (or its hash) recorded at Device
        Initialization in the Device certificate (`OVDevCertChain`), or
        an IAK certificate (TCG "TPM 2.0 Keys for Device Identity and
        Attestation"), which removes the ActivateCredential round trip.
      With no anchor, report the evidence as unverifiable.
      The DI-time carrier for the manufacturer anchor is the Device
      Manufacturing Info appnote (`fdo-appnote-device-mfg-info.bs`,
      `DMI_OVE_EXTRA_REQUEST` -> `OVEExtraInfo` of the first voucher
      entry, signed by the manufacturer). See "Proposals for the Device
      Manufacturing Info appnote" below.
      Not a solution: re-signing the Quote with the Device Key. It only
      proves "whoever holds the Device Key sent this", which the TO2
      session already proves; software with Device Key access could
      endorse a Quote from a fake software TPM just as well.
- [ ] AK lifetime: created for each session and flushed, or persisted
      for reuse after onboarding.
- [ ] Event log transport: reuse the chunking strategy
      ([chunking-strategy.md](chunking-strategy.md)); which log formats.
- [ ] Devices whose Device Key has an empty authPolicy (TPM2_Certify not
      possible): the error to report, and whether the Owner can fall
      back to `TPM2_CertifyCreation` evidence supplied at manufacture.
- [ ] Hash banks: let the Owner pick a bank, or report every
      allocated bank.
- [ ] Relationship to IETF RATS / EAT evidence formats: carry raw TPM
      structures, or wrap them in a standard evidence format.

## Ideas

- [ ] **Event logs alongside the Quote.** Let the device include the
      logs that are bound to PCRs, selected by the Owner:
      - TCG PC Client firmware event log (measured boot)
      - runtime measurement log (e.g. Linux IMA)

      The Owner replays each log and checks that the result matches the
      quoted PCRs; that is what makes the log trustworthy. Logs that no
      PCR covers are out of scope (the Quote cannot vouch for them).
- [ ] **Shape: a variant of `fdo.payload`.** The logs are bulk data, so
      the module is probably the payload FSIM's transfer mechanism plus
      an attestation exchange in front of it: Owner nonce -> Certify +
      Quote (nonce in `qualifyingData`) -> logs.

## Trust anchor for the AK: demo status and the IAK path

The Quote proves something only if the Owner knows the AK is a genuine
TPM key in the TPM that holds the voucher's Device Key. The Owner gets
there from one of these anchors, in order of preference:

| # | Anchor | Voucher carries | At every TO2 | Status |
|:--|:-------|:----------------|:-------------|:-------|
| 1 | **IAK** (TCG): manufacturer-issued certificate for a persistent, restricted Initial Attestation Key | `PT_TPM_AK_CERT` (15) with the IAK certificate | Certify(Device Key by IAK) + Quote(IAK); no ActivateCredential, no fresh AK | Aspiration (below) |
| 2 | **EK recorded by the manufacturer** at DI | `PT_TPM_EK_CERT` (14), or proposed `PT_TPM_EK_PUB` | fresh AK; MakeCredential to that EK / ActivateCredential; Certify + Quote | Demo works |
| 3 | **EK certificate from a TPM vendor CA** the Owner trusts | nothing | as 2, EK from the device's certificate | Demo works |

With no anchor the Owner reports the evidence as unverifiable (the demo
fails the module). Anchors 1 and 2 put the trust in the manufacturer,
whom the Owner already trusts through the voucher (the entry-0 signature
covers `OVEExtraInfo`); no TPM vendor roots are needed, and they work for
firmware TPMs whose EK certificate is not in NV.

**The factory must check what it records.** FDO DI is one-way (the
device's data goes into `DI.AppStart`, nothing comes back before the
voucher is made), so the EK-to-Device-Key proof (MakeCredential /
ActivateCredential plus Certify of the Device Key by an AK) cannot run
inside DI. The station either runs it beside DI (a station that also
drives the device, as quick-di does) or trusts its line; the appnote
should say which the manufacturer vouches for.

### Demo (2026-10-28, swtpm)

- go-fdo branch `tpm-quote-demo`: `tpm/attest.go` (device steps and
  verifier checks, shared by DI and TO2), example server `-tpm-quote`
  [`-tpm-quote-ca`], example client device module,
  `examples/tpm-quote-demo.sh`.
- go-fdo-quick-di branch `tpm-quote-demo`: `-tpm-record-ek` checks the
  EK-to-Device-Key binding (fresh AK certifies the DAK and activates a
  credential made to the EK) and only then writes `PT_TPM_EK_CERT` into
  `OVEExtraInfo` of the first voucher entry.
- Results (16 checks): DI refuses to record another TPM's EK; the Owner
  verifies the quote with the voucher's EK and no TPM CA; a changed PCR
  shows in the next verified quote; stale nonce, another key certified,
  altered quote and wrong activation secret are rejected at their
  checks; without an EK in the voucher the vendor-CA path verifies, and
  an untrusted CA or another TPM's EK certificate is rejected; a voucher
  recording the wrong EK (factory check skipped) fails ActivateCredential.
- **Real TPM (K800, Intel PTT, Alder Lake firmware TPM), 2026-10-29:**
  quick-di DI refuses another TPM's EK and records the PTT's own; TO2
  verifies with the voucher's EK and no CA, over real measured-boot PCRs
  0-7; stale nonce, other key certified, altered quote, wrong secret and a
  voucher recording the wrong EK are rejected at their checks. The vendor
  path also works with Intel's real chain: EK certificate -> PTT, Kernel
  and ROM CAs (stored in TPM NV at 0x01C00100) -> ODCA issuing and
  intermediate CAs (downloaded via AIA from tsci.intel.com) -> Intel OnDie
  CA root; an untrusted CA is rejected.
- Real-TPM differences found: the EK is not persisted (recreated from the
  TCG default template, which matches the certificate), and the vendor's
  intermediate CAs live in the EK chain NV indices, so the device sends
  them with the EK certificate.
- Demo limits: RSA 2048 EK only, no event log; on swtpm, PCR 16 stands in
  for a measurement.

### Long-term: IAK / IDevID (TCG "TPM 2.0 Keys for Device Identity and Attestation", v1.1, 2025)

This is the standard form of what the demo does at every onboarding,
done once by the OEM:

- **Roles.** IDevID = unrestricted signing key for device identity;
  IAK = restricted signing (attestation) key. These map onto the FDO
  Device Key and the AK. The FDO TPM specification already lists
  `DeviceKeyType` 1 = IDevID.
- **OEM enrollment** (sections 6.1, 6.2) is the same proof: the CA
  validates the EK certificate and the IAK attributes, uses
  MakeCredential / ActivateCredential to confirm the IAK is in the TPM
  with that EK, and the IAK certifies the IDevID (`TPM2_Certify`); the
  OEM then issues IAK and IDevID certificates, which may also be stored
  in TPM NV (7.4.3).
- **Handles** (Table 2): IDevID `0x81020000`, IAK `0x81020001`, next to
  FDO's Device Key `0x81020002` and HMAC key `0x81020003`; no collision.
- **Device key cannot be the IAK:** the IAK is restricted (signs only
  TPM-generated structures); the Device Key must sign FDO messages.

What it would take:

- Spec: the appnote's `PT_TPM_AK_CERT` (15) already allows "a
  certificate or certification blob"; add that a TCG IAK certificate
  goes there (no new identifier). This FSIM: anchor order above (IAK,
  then EK in voucher, then vendor CA). TPM spec: note that a TCG
  IDevID/IAK pair MAY serve as the Device Key and AK.
- DI: create (or find) the persistent IAK, run the check, have the
  manufacturer's CA issue the IAK certificate, record it.
- TO2: skip the fresh AK and the activation round trip.

Open questions for the IAK path:

- [ ] IDevID template vs the FDO Device Key template. TCG recoverable
      templates use `IDevIDKeyPolicy` = PolicyOR(PolicySecret(endorsement),
      PolicyCommandCode(Certify), PolicyCommandCode(ActivateCredential),
      PolicyAuthorizeNV delegation) and let the Privacy Administrator set
      an authValue; FDO's Device Key uses PolicyCommandCode(Certify) and
      empty-auth signing. Can an existing IDevID be the Device Key as is?
- [ ] IAK certificate from the voucher or from TPM NV (or both, with the
      voucher copy as the manufacturer's endorsement).
- [ ] Whether the FDO manufacturer key and the OEM IAK CA are the same
      trust root, and what the Owner checks when they are not.
- [ ] Devices that already carry IDevID/IAK from the OEM before FDO DI:
      reuse them instead of creating a new Device Key.

## Proposals for the Device Manufacturing Info appnote

`fdo-appnote-device-mfg-info.bs` (branch `bgoodman-di-standarization`)
already lets the device put `PT_TPM_EK_CERT` (14), `PT_TPM_AK_CERT`
(15) and `PT_TPM_QUOTE` (16, with PCR values and event log) into
`OVEExtraInfo` at DI, and its "Attestation Freshness" section points to
"an appropriate FSIM" for fresh attestation in TO2; this module is that
FSIM. To make the DI-time data a usable anchor for it:

- [ ] **EK public key without a certificate** (new PT, e.g.
      `PT_TPM_EK_PUB`: marshalled `TPM2B_PUBLIC`). Many TPMs, notably
      firmware TPMs, have no EK certificate in NV. The manufacturer's
      signature on the voucher entry then anchors the EK instead of a
      TPM vendor CA.
- [ ] **Manufacturing station verifies before it signs.** Today the
      station includes device-supplied entries "verbatim", so the
      voucher only proves the device *claimed* them. For EK/AK/IAK
      entries the station SHOULD verify, at DI, that they are in the same
      TPM as the Device Key (ActivateCredential against the EK;
      `TPM2_Certify` of the Device Key by the AK), and the appnote should
      say how a consumer knows this was done.
- [ ] **Device Key binding evidence** (new PT, e.g.
      `PT_TPM_DAK_CERTIFY`: `TPMS_ATTEST` + signature from
      `TPM2_Certify(Device Key, AK)`), so downstream parties with TPM
      vendor roots can check the binding themselves.
- [ ] **IAK:** mention the TCG Initial Attestation Key certificate as a
      form of `PT_TPM_AK_CERT`; quotes signed by an IAK need no
      ActivateCredential round trip. See "Long-term: IAK / IDevID" above.
- [ ] **Baseline = the DI event log, compared by events.** The DI-time
      `PT_TPM_QUOTE` reflects the factory boot; by TO2 the PCRs have been
      extended further (field OS). Final PCR values then differ even when
      the firmware part is unchanged. Replaying the event log recovers
      every intermediate PCR value, and the replay must match the quoted
      final values, so earlier events are authenticated too. An attestor
      can therefore check that the TO2 log starts with the same firmware
      events as the DI log (up to `EV_SEPARATOR` in PCRs 0-7) and judge
      the rest separately; drift shows up as specific events (e.g. a
      db/dbx update). So the DI quote SHOULD carry the event log
      (`TPMQuotePayload` key 5). Reference values can still go in
      `PT_HW_MANIFEST_CORIM` (23). Limits: only PCRs whose extends are all
      logged; some events record only a digest.
- [ ] **`TPMQuotePayload` wording:** key 3 "PCR digest values
      (concatenated)" reads like the single `pcrDigest`; it should say the
      PCR values themselves, in selection order, for the quoted bank (the
      consumer hashes them and compares with `pcrDigest` in the signed
      `TPMS_ATTEST`). Key 2 (PCR selection) duplicates the selection
      inside `TPMS_ATTEST`. The quote is a superset of a plain PCR list: a
      list is only manufacturer-signed device claims, the quote adds "the
      TPM produced these" (given an anchored AK).

## References

- FIDO Device Onboard specification (`FIDO-IoT-spec.bs`)
- Securing FDO Credentials in the TPM (`securing-fdo-in-tpm.bs`)
- TCG TPM 2.0 Library, Part 3: `TPM2_Certify`, `TPM2_Quote`,
  `TPM2_MakeCredential`, `TPM2_ActivateCredential`,
  `TPM2_PolicyCommandCode`
- TCG PC Client Platform Firmware Profile (event log)
- TCG TPM 2.0 Keys for Device Identity and Attestation, v1.1 (2025):
  IDevID/IAK, OEM enrollment, persistent handles
- TCG EK Credential Profile (EK certificate NV indices)
