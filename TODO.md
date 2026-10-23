# TODO

## Active

### Q62: Decide which document wins where they disagree
- Owner: David (policy owner)
- Added: 22 October 2026
- Source: OPEN-QUESTIONS.md Q62
- Decisions needed:
  1. NEAR-002: the requirement says presence carries the GPS fix time; the spec sends the fix age in seconds. Decide which is right.
  2. NEAR-004: the requirement says every presence signal reaches the app; the spec filters some out. Decide whether filtering is acceptable.
  3. POLICY §10 and §18: POLICY says to build and test duty-cycle limits; the spec uses a 30% channel-busy limit instead, with SOS exempt. Decide whether POLICY should change. First confirm with the spec team whether Trail Buddy may be sold outside the US. See also Q64 (no check-before-sending rule).
  4. Encryption: POLICY §12 says the encryption choices are decided; the spec says they go to the security review before freeze. Decide which is true.
  5. Precedence: write a rule for which document wins when they disagree, and add it to POLICY.
- Then: record each decision with date and reason, release updated POLICY and REQUIREMENTS, send spec changes to the spec team for v0.4, and log everything in the provenance log.

### Q63: Approve the standards the spec relies on
- Owner: David (policy owner), with the spec team
- Added: 22 October 2026
- Source: OPEN-QUESTIONS.md Q63
- Problem: POLICY §14 permits only Bluetooth and FCC Part 15. The spec relies on others, and SOURCE-REGISTER.md, where approvals must be logged, doesn't exist.
- Standards needing approval: HKDF with SHA-256, SHA-256, AES-128-CCM, Ed25519, CRC-16/CCITT-FALSE, Base64url, Base32 alphabet (short ID), UTF-8, the QR code format, Protocol Buffers proto3, and UTC/GPS time.
- Steps:
  1. Decide where SOURCE-REGISTER.md lives, then create it.
  2. Ask the spec team to name the defining document for each standard and approve it.
  3. Decide whether the clean team may use only the published standard, or also reference code or a library.
  4. If legal counsel is involved, have them confirm the list.
  5. Log each approval in SOURCE-REGISTER.md.
  6. Ask the security reviewer whether the Nordic/Zephyr crypto library supports all the required algorithms.

## On hold
- Pocket V3 board profile, buzzer and power button — waiting for the Pocket V3 to ship and its specs to arrive.

### Define "approaching" for the app
- Status: On hold, as of 22 October 2026
- Why on hold: product decision deferred by the owner. It doesn't block firmware.
- Source: spec v0.3, Section 14, open item f
