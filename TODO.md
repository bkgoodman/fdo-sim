# TODO — FSIM specifications

Open spec items carried over from `go-fdo/fsim-todo.md` (2026-01, removed
2026-10-08 — everything else in it was done: payload signing/integrity,
security sections, error codes, WiFi fast roaming / Hotspot 2.0 / CA
certificates, and the FDO 2.0 owner module system in go-fdo).

## fdo.wifi-setup

- [ ] Hidden SSIDs: no field tells the device to probe for a non-broadcast SSID.
- [ ] PSK rules: state the WPA2/WPA3-PSK passphrase length (8-63 chars) or
      64-hex-digit raw PSK, and what the device does with an invalid one.
- [ ] WPA3-Enterprise 192-bit (Suite B) mode: not expressible today.
