# TODO — FSIM specifications

Open spec items carried over from `go-fdo/fsim-todo.md` (2026-01, removed
2026-10-08 — everything else in it was done: payload signing/integrity,
security sections, error codes, WiFi fast roaming / Hotspot 2.0 / CA
certificates, and the FDO 2.0 owner module system in go-fdo).

## fdo.tpm-quote (new)

Placeholder: [fsim-repository/fdo.tpm-quote.md](fsim-repository/fdo.tpm-quote.md).
TPM evidence of boot state (EK -> AK -> Certify(Device Key) -> Quote +
event log), so the Owner can check what the device booted, not only that
its TPM signed. Depends on the TPM spec's default Device key authPolicy
`PolicyCommandCode(TPM_CC_Certify)` (securing-fdo-in-tpm.bs, Table 12).
Workstream: `fdo-tpm-quote`.

- [ ] Write the spec: key names, CBOR encodings, error codes, example.
- [ ] Settle the open questions listed in the placeholder (nonce binding,
      EK certificate source, AK lifetime, event log transport, keys
      without the Certify policy, hash banks, RATS/EAT format).
- [ ] Idea: include the PCR-bound event logs with the Quote data (TCG
      measured-boot log, IMA runtime log); probably a variant of
      `fdo.payload` plus a nonce/Certify/Quote exchange for freshness. See
      "Ideas" in the placeholder.
- [ ] Propose DI-time anchor data for the Device Manufacturing Info
      appnote (`~/internet-of-things-specs/fdo-appnote-device-mfg-info.bs`,
      branch `bgoodman-di-standarization`): EK public key without a cert,
      station verifies EK/AK are in the Device Key's TPM before signing,
      `TPM2_Certify(Device Key, AK)` evidence, IAK, PCR baseline + CoRIM
      reference values. Detail: "Proposals for the Device Manufacturing Info
      appnote" in the placeholder.
- [ ] Reference implementation in go-fdo (owner + device) and
      fdo-uefi-rs (device), tested against swtpm and a real TPM.

## fdo.wifi-setup

- [ ] Hidden SSIDs: no field tells the device to probe for a non-broadcast SSID.
- [ ] PSK rules: state the WPA2/WPA3-PSK passphrase length (8-63 chars) or
      64-hex-digit raw PSK, and what the device does with an invalid one.
- [ ] WPA3-Enterprise 192-bit (Suite B) mode: not expressible today.
