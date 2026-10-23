# Trail Buddy radio: clean-room policy

Version 0.4 · 23 October 2026 · Issued together with message spec Draft v0.4 (`docs/SPEC-v0.4.md`) and `REQUIREMENTS.md` v0.4. Section numbers are kept from earlier versions; Section 9 (product requirements) is in `REQUIREMENTS.md`.

**Which document wins.** REQUIREMENTS says what the product must do, POLICY says how the clean team works, and the SPEC says exactly how the radio behaves. If they disagree, none of them wins: write the conflict in OPEN-QUESTIONS.md and build nothing that depends on it until the spec team answers. Until then, for anything affecting SOS, the reading that is safer for the rider applies.

This document is written for the implementation team. It deliberately does not name or describe any other project.

## 7. What we are building

Firmware for the Trail Buddy radio. A group of off-road riders each carry a phone running the Trail Buddy app and a small LoRa radio. The phone connects to its radio over Bluetooth Low Energy. The radios form a mesh: each one relays the others' messages, so riders can see each other on a map, message each other and send an SOS with no mobile signal. Any Trail Buddy radio can also hear a short presence signal from any other, so riders can see nearby Trail Buddy riders who are not in their ride.

## 8. Hardware

| Board | Notes |
| --- | --- |
| RAKwireless RAK19026 VC base board with RAK4631 core | **Version 1, lead board.** Reset button, one user button (SW1), two user LEDs, battery disconnection switch, integrated GNSS. Zephyr has a RAK4631 board definition. |
| Heltec Mesh Node T114 V2 | **Version 1.** V2 only; V1 is excluded. One user button; GPS module fitted on every Trail Buddy unit. Zephyr has a board definition. |
| Elecrow ThinkNode M1 | **Version 1.** Function, page and reset buttons; power switch; GPS switch; status LED; buzzer; e-paper screen; integrated GNSS. |
| RAKwireless WisMesh Pocket V3 | IP67 waterproof; RAK19026 baseboard; GPS; notification buzzer. Added when it ships. |

- All of them use a Nordic nRF52840 microcontroller (Bluetooth 5.0) and a Semtech SX1262 LoRa radio.
- Region: United States, 902–928 MHz.
- One firmware serves every board, with a separate board profile for each: pins, GNSS, user button, status light and battery measurement.
- Version 1 runs on three boards: the **RAK19026 VC base board with the RAK4631 core** (lead board), the **Heltec T114 V2** and the **Elecrow ThinkNode M1**. Each build step is done on the lead board first; the radio bring-up and the hardware tests are run on all three (Sections 17, 18 and 20). Each has a written hardware profile in `docs/hardware/`. The Pocket V3 is added when it ships.
- Zephyr's RAK4631 and Heltec T114 V2 board definitions may be used as starting points. They are part of the open-source Zephyr project, not another mesh project's board files. The ThinkNode M1's definition is written by the clean team from Elecrow's documents.
- Version 1 uses each board's user button, GNSS and one status light, as mapped in spec Section 9. The status light shows power-on, normal running, update reboot and factory reset only. Buzzers and screens are not used in version 1. SOS alerts are shown by the app.
- Board profiles come only from the manufacturer: datasheets, schematics and the manufacturer's own board support. Never from any other mesh project's board files.
- All board-specific GPIO, SPI, interrupt, power and radio-register logic stays in the board support package (BSP) and radio HAL. Mesh and application code never contains it.

## 10. Firmware architecture

| Layer | Responsibility |
| --- | --- |
| Phone link (BLE) | Commands, events, settings with read-back, version check, diagnostics. |
| Trail Buddy protocol | Message types, serialization, identity, addressing, acknowledgements, encryption envelope. |
| Mesh | Relay, duplicate suppression, hop limit, rebroadcast delay, queues and priority (SOS first). |
| Channel access | Channel activity detection before every transmission, randomized backoff (spec 4.7), channel-busy measurement and the 30% airtime budget with its slow-down steps (spec 6.10), receive/transmit state machine. Version 1 serves the United States only, where no duty-cycle limit applies. |
| Settings and storage | Persistent settings, position history for Fill on Return, safe defaults. |
| Radio HAL | SX1262: frequency, spreading factor, bandwidth, coding rate, power, receive, transmit, channel activity detection. |
| BSP | nRF52840, SPI, GPIO, interrupts, timers, flash, power, GNSS, user button, status light, per board profile. |

## 11. Packets and messages

The spec team designs the formats fresh; nothing is inherited from any existing project.

- **Over the air:** the spec defines every frame byte for byte (spec Sections 4 and 5): envelope, identities, encryption, counters, priorities and relay rules. The clean team implements those layouts exactly and does not change them.
- **Phone link:** the spec defines every message's fields, types, units and limits (spec Section 7). The clean team writes the protobuf schema from those tables, following spec 7.1.
- Anything the spec leaves to the implementation is recorded in DECISIONS.md with its rationale. Anything that would change a frame or message goes to OPEN-QUESTIONS.md instead.

## 12. Security

These are decided for building; the spec gives the detail (spec 4.3 to 4.8 and 7.2). Only the independent security review before freeze can change them, and any change it makes comes as a new spec version.

- ride messages use AES-128-CCM with an 8-byte authentication tag; keys and tags are derived from the ride key with HKDF-SHA256
- a per-ride key generated on the phone and shared in the ride QR code
- a frame counter that never repeats, even across power loss, and a replay window
- presence frames carry no ride key, no names and only a rotating anonymous ID; fake presence is limited as in spec 6.6
- the phone link is paired and encrypted before any message except HELLO and WELCOME
- firmware images are signed with Ed25519 (spec 7.4, Updates); the signing key never enters the repository
- an independent review of the design before the spec is frozen
- established, licensed crypto libraries only; no home-made primitives. The product owner confirms, before build step 2, that the chosen library supports AES-128-CCM, HKDF-SHA256 and Ed25519.

Robustness:

- Validate every length, ID, hop count, message type and field.
- A malformed packet must never cause a reboot, a hang or memory corruption.
- Treat queue exhaustion and floods of packets as denial-of-service cases and test them.

## 13. Phone ↔ radio boundary

The phone app never needs to know how the mesh routes packets. The interface is small, and is defined message by message in the message spec:

- connect, disconnect and protocol-version check
- read radio identity, firmware version, battery and link status
- write ride settings (ride key, tier) with read-back
- read ride state and SOS state again after reconnecting
- send and receive group messages and acknowledgements
- send and receive positions
- send SOS, "Copy that" and "I'm OK"
- Fill on Return requests and answers
- receive nearby riders' presence
- read the channel-busy measure
- start a firmware update

The app never depends on routing tables, radio registers, queue internals or board details.

## 14. Allowed and forbidden sources

| Allowed? | Source | Examples |
| --- | --- | --- |
| Yes | The clean repository | `docs/`, `spec/`, tests, answered questions |
| Yes | Hardware and vendor documentation | Nordic nRF52840 product specification; nRF Connect SDK and Zephyr documentation, including Zephyr's RAK4631 and Heltec T114 V2 board definitions; Semtech SX1262 datasheet and application notes; Heltec T114 V2 schematic; Elecrow ThinkNode M1 documentation; RAKwireless RAK4631, RAK4630 and RAK19026 documentation; GNSS module manuals |
| Yes | Approved library documentation | RadioLib, nanopb, the UF2 bootloader |
| Yes | Standards and regulations | Bluetooth Core Specification, FCC Part 15, and the standards approved in spec Section 13: RFC 5869, FIPS 180-4, FIPS 197, NIST SP 800-38C, RFC 3610, RFC 8032, RFC 4648, RFC 3629, Google's proto3 language specification, ITU-R TF.460, IS-GPS-200, Microsoft's UF2 format and, for the app team only, ISO/IEC 18004. Read only the copies in `docs/standards/`. |
| Yes | Published literature | Papers on flooding and broadcast suppression, cited in DECISIONS.md |
| No | Other mesh messaging projects | Their code, documents, board files, protocol definitions, issue trackers, forums and blog posts, however obtained, including through web search or an AI chat |
| No | Anything outside the clean repository not on this list | Unless the spec team approves it and it is logged in SOURCE-REGISTER.md first |

Vendor documents and standards are added to the repository by the product owner, and each is logged in SOURCE-REGISTER.md, so sessions don't need web access.

## 15. Libraries

| Purpose | Library | Licence |
| --- | --- | --- |
| RTOS and BLE | Zephyr / nRF Connect SDK | Apache-2.0 (verify) |
| LoRa driver | RadioLib, or Semtech's SX126x driver | MIT for RadioLib (verify); check Semtech's |
| Phone-link serialization | nanopb, from our own protobuf schema | zlib (verify) |
| Over-the-air frames | Hand-written from the spec's byte layouts | Ours |
| Cryptography | The crypto library supplied with nRF Connect SDK / Zephyr | Apache-2.0 (verify) |
| Updates | A two-slot bootloader that checks signatures and rolls back, chosen by the clean team from the Zephyr ecosystem | Record in SOURCE-REGISTER.md (verify) |
| Recovery | Each board's UF2 bootloader, over USB | MIT (verify) |

- Record every dependency's licence and purpose in SOURCE-REGISTER.md before using it.
- Do not use any library that contains code taken from another mesh messaging project.

## 16. Rules for clean Claude Code sessions

**Environment**

- A separate macOS user account with no copy of earlier Trail Buddy code, a separate Claude login and a separate GitHub account.
- A fresh `~/.claude` folder: no memory, instructions or skills from earlier Trail Buddy work.
- The repository's `CLAUDE.md` holds the session instructions.
- Web search and web reading are off.
- One session per build step, starting from the spec and the previous step's tests.
- Every session transcript is saved to `logs/sessions/`.

**Working rules**

- Implement only from the sources allowed in Section 14.
- Do not search for, read, copy, translate, port or imitate code from other mesh messaging projects.
- If something is missing or unclear, stop and write it in OPEN-QUESTIONS.md. Do not guess, and do not fill gaps from memory of how other systems work.
- Answers come back only as a new spec version from the spec team, written as behaviour, never code.
- Every change maps to a requirement ID and a test.
- Names, APIs, constants, state machines, comments and code structure are Trail Buddy's own, unless a vendor API or public standard requires otherwise.
- A clean human reviewer approves code before merge.

**Diagnostics**

- Logging must never crash the radio: no arithmetic inside log statements, and widen unsigned values before subtracting.
- Log what was actually sent and received, not that a function ran.
- Detailed logging is compiled out of release builds. The diagnostic counters required by DIAG-001 stay in release builds.

## 17. Build order

Hardest first, so failures show up early. Unit tests come before hardware for every step. All steps run on the lead board first.

1. **Radio bring-up:** raw send and receive between two lead boards, then between each pair of the three version 1 boards.
2. **Frame format and crypto:** all test vectors in spec Section 11 pass. Conformance tests run in CI from here on.
3. **BLE phone link** with a minimal test app.
4. **GPS and position** broadcast by tier.
5. **Mesh simulator** (1–30 virtual nodes), then relay and duplicate suppression. The simulator confirms the airtime figures in spec 2.5.
6. **Repeater mode.**
7. **SOS:** priority and repeat.
8. **Nearby riders:** presence send and receive.
9. **Fill on Return.**
10. **Settings persistence** with read-back and power-loss safety.
11. **Firmware updates:** signed images and rollback.
12. **T114 V2 and ThinkNode M1:** complete each board profile (user button, status light, GNSS, battery) and re-run the hardware-in-the-loop and field tests on each.
13. **WisMesh Pocket V3,** when it ships.

## 18. Testing

| Layer | Tests | Passes when |
| --- | --- | --- |
| Unit | Encode/decode, IDs, hop count, duplicates, queues, backoff | Deterministic automated pass |
| Conformance | Every test vector, encrypted and decrypted | All vectors pass on every change |
| Protocol | Ride broadcast, unicast, presence, unknown sender, malformed and duplicate packets, replays | Expected state transitions; malformed input rejected |
| Mesh simulation | 1–30 virtual nodes; loss, duplicates, congestion, partitions, rejoining; channel busy per tier | No relaying beyond the hop limit; SOS delivered first under load; presence never delays SOS |
| Radio / channel access | Channel busy and clear, collisions, listen before talking, backoff, receive/transmit switching, slow-down steps | No deadlocks; normal traffic held at or below 30% busy; SOS never waits for a clear channel |
| Hardware-in-the-loop | 3–5 boards, including at least one of each version 1 board: cold boot, sleep/wake, power loss during settings write, failed update, sustained traffic | Settings intact; rollback works; stable operation |
| Field | Range, Fill on Return timing, SOS, approaching-rider warnings, full-day battery, a 4-rider ride at each tier | Meets NFR-001 targets; with 4 riders at 10 s updates, the channel stays usable for an SOS |
| Phone | Connect, version check, send, receive, tracking, nearby riders, reconnect, read-back | App sees only defined protocol events |

## 19. Repository layout and records

| Path | Contents |
| --- | --- |
| `CLAUDE.md` | Session instructions |
| `docs/` | Message spec drafts, as handed over |
| `docs/clean-room/` | POLICY.md, REQUIREMENTS.md |
| `docs/hardware/` | One hardware profile per board, with datasheet references |
| `docs/standards/` | Official copies of the approved standards (spec Section 13), added by the product owner |
| `spec/` | Frozen protocol spec (versioned) |
| `spec/vectors/` | Test vectors: the spec's own (spec Section 11) and the recorded phone-link vectors |
| `firmware/bsp/`, `radio/`, `channel/`, `mesh/`, `protocol/`, `storage/`, `ble/`, `app/` | Firmware by layer (Section 10) |
| `tests/unit/`, `conformance/`, `simulation/`, `hardware/` | Tests by layer |
| `tools/` | Test and debug utilities |
| `logs/sessions/` | Every clean session transcript |

| Record | Purpose | Kept by |
| --- | --- | --- |
| SOURCE-REGISTER.md | Every external source and dependency, its licence and what it contributed | Implementers |
| DECISIONS.md | Implementation decisions and rationale, with literature citations | Implementers |
| OPEN-QUESTIONS.md | Questions for the spec team; never guessed | Implementers |
| TEST-MATRIX.md | Requirement-to-test traceability | Implementers |
| ATTESTATIONS/ | Signed statements from every human implementer and reviewer | Product owner |
| Clean-room log | Who saw what and when, spec hand-overs, questions and answers. Kept outside the clean repo | Product owner |

## 20. Definition of done

- Builds reproducibly from a clean checkout for all three version 1 boards; only approved board targets are enabled.
- Hardware-in-the-loop tests pass on the RAK19026 VC, the T114 V2 and the ThinkNode M1, and a mixed ride using all three works.
- All test vectors and conformance tests pass.
- SOS goes first, and an SOS or "I'm OK" is never dropped, under simulated and bench load. An active SOS resumes after a reboot.
- Duplicates are suppressed and the hop limit stops flooding.
- Presence is sent and received, and never delays an SOS.
- DIAG-001 counters are present in the release build.
- Settings survive power loss, and every write is read back.
- Firmware updates roll back cleanly after a failure.
- The phone app connects and exchanges every message in the message spec.
- Every requirement has a passing test.
- Records in Section 19 are complete, and a clean reviewer has approved the code.
- Regulatory and release sign-off from the product owner before any release.

## Changes from v0.3

| Section | Change |
| --- | --- |
| Header | "Which document wins" rule added. |
| 8 | Version 1 boards are the RAK19026 VC base board with the RAK4631 core (lead), the Heltec T114 V2 and the Elecrow ThinkNode M1; user button mapped per board in spec 9; one status light for power and mode; no buzzer or screen in version 1. |
| 17, 18, 20 | Bring-up, hardware tests and done-criteria cover all three boards. |
| 10, 18 | Channel access: listen before talking and the 30% airtime budget; no duty-cycle limit in the United States. |
| 12 | Decided for building; only the security review can change it. Ed25519 named. Crypto library to be confirmed before build step 2. |
| 14, 19 | Approved standards listed; copies kept in `docs/standards/`. |
| 20 | SOS done-criterion matches SOS-002 and SOS-003. |
