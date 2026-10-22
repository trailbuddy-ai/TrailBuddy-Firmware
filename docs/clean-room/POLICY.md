# Trail Buddy radio: clean-room policy

Version 0.2 · 22 October 2026 · Source: Clean-Room Firmware Development Specification, Part 2. Section numbers match that document; Section 9 (product requirements) is in `REQUIREMENTS.md`.

This document is written for the implementation team. It deliberately does not name or describe any other project.

## 7. What we are building

Firmware for the Trail Buddy radio. A group of off-road riders each carry a phone running the Trail Buddy app and a small LoRa radio. The phone connects to its radio over Bluetooth Low Energy. The radios form a mesh: each one relays the others' messages, so riders can see each other on a map, message each other and send an SOS with no mobile signal. Any Trail Buddy radio can also hear a short presence signal from any other, so riders can see nearby Trail Buddy riders who are not in their ride.

## 8. Hardware

| Board | Notes |
| --- | --- |
| Heltec Mesh Node T114 V2 | V2 only; V1 is excluded |
| Elecrow ThinkNode M1 | E-paper screen, several buttons |
| RAKwireless RAK4631 (WisBlock) | Has a board definition in Zephyr |
| RAKwireless WisMesh Pocket V3 | IP67 waterproof; RAK19026 baseboard; GPS; notification buzzer |

- All four use a Nordic nRF52840 microcontroller (Bluetooth 5.0) and a Semtech SX1262 LoRa radio.
- Region: United States, 902–928 MHz.
- One firmware serves every board, with a separate board profile for each: pins, GNSS, display, buttons, battery measurement and buzzer.
- Version 1 is built and tested on one lead board first. The others are added only after the core tests pass on it, each with a written hardware profile in `docs/hardware/`. The product owner names the lead board.
- Board profiles come only from the manufacturer: datasheets, schematics and the manufacturer's own board support. Never from any other mesh project's board files.
- All board-specific GPIO, SPI, interrupt, power and radio-register logic stays in the board support package (BSP) and radio HAL. Mesh and application code never contains it.

## 10. Firmware architecture

| Layer | Responsibility |
| --- | --- |
| Phone link (BLE) | Commands, events, settings with read-back, version check, diagnostics. |
| Trail Buddy protocol | Message types, serialization, identity, addressing, acknowledgements, encryption envelope. |
| Mesh | Relay, duplicate suppression, hop limit, rebroadcast delay, queues and priority (SOS first). |
| Channel access | Channel activity detection before transmit, randomized backoff, airtime and duty-cycle limits, receive/transmit state machine. |
| Settings and storage | Persistent settings, position history for Fill on Return, safe defaults. |
| Radio HAL | SX1262: frequency, spreading factor, bandwidth, coding rate, power, receive, transmit, channel activity detection. |
| BSP | nRF52840, SPI, GPIO, interrupts, timers, flash, power, GNSS, display, buttons, buzzer, per board profile. |

## 11. Packets and messages

The spec team designs the packet format fresh. Nothing is inherited from any existing project. The spec must define, with a rationale for each size and position:

- a protocol version
- a packet identity that supports duplicate suppression
- sender identity, and how a radio's ID is created and kept
- addressing: the whole ride, one rider, and any Trail Buddy radio (presence)
- hops remaining
- a priority class
- the authenticated encryption envelope for ride messages

Every message type is budgeted in bytes against the LoRa payload limit at the chosen radio settings. Field sizes, reserved values and defaults are chosen from Trail Buddy's needs and recorded in DECISIONS.md.

## 12. Security

These are decided; the spec gives the detail:

- authenticated encryption for ride messages: AES-CCM (hardware-accelerated on the nRF52840) or ChaCha20-Poly1305
- a per-ride key generated on the phone and shared in the ride QR code
- a nonce scheme that never repeats a nonce under one key, and a replay window
- presence signals carry no ride key and no names; how fake presence signals are limited is an open question for the spec team
- an independent review of the design before the spec is frozen
- established, licensed crypto libraries only; no home-made primitives

Robustness:

- Validate every length, ID, hop count, message type and field.
- A malformed packet must never cause a reboot, a hang or memory corruption.
- Treat queue exhaustion and floods of packets as denial-of-service cases and test them.

## 13. Phone ↔ radio boundary

The phone app never needs to know how the mesh routes packets. The interface is small, and is defined message by message in the message spec:

- connect, disconnect and protocol-version check
- read radio identity, firmware version, battery and link status
- write ride settings (ride key, tier) with read-back
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
| Yes | Hardware and vendor documentation | Nordic nRF52840 product specification; nRF Connect SDK and Zephyr documentation, including Zephyr's RAK4631 board definition; Semtech SX1262 datasheet and application notes; Heltec T114 V2 schematic; Elecrow ThinkNode M1 documentation; RAKwireless RAK4631, RAK4630 and RAK19026 documentation; GNSS module manuals |
| Yes | Approved library documentation | RadioLib, nanopb, the UF2 bootloader |
| Yes | Standards and regulations | Bluetooth Core Specification, FCC Part 15 |
| Yes | Published literature | Papers on flooding and broadcast suppression, cited in DECISIONS.md |
| No | Other mesh messaging projects | Their code, documents, board files, protocol definitions, issue trackers, forums and blog posts, however obtained, including through web search or an AI chat |
| No | Anything outside the clean repository not on this list | Unless the spec team approves it and it is logged in SOURCE-REGISTER.md first |

Vendor documents are added to the repository by the product owner, so sessions don't need web access.

## 15. Libraries

| Purpose | Library | Licence |
| --- | --- | --- |
| RTOS and BLE | Zephyr / nRF Connect SDK | Apache-2.0 (verify) |
| LoRa driver | RadioLib, or Semtech's SX126x driver | MIT for RadioLib (verify); check Semtech's |
| Serialization | nanopb, from our own protobuf schema | zlib (verify) |
| Bootloader | Each board's UF2 bootloader | MIT (verify) |

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
- Diagnostics are compiled out of release builds.

## 17. Build order

Hardest first, so failures show up early. Unit tests come before hardware for every step. All steps run on the lead board first.

1. **Radio bring-up:** raw send and receive between two boards.
2. **Frame format and crypto:** all spec test vectors pass. Conformance tests run in CI from here on.
3. **BLE phone link** with a minimal test app.
4. **GPS and position** broadcast by tier.
5. **Mesh simulator** (1–20 virtual nodes), then relay and duplicate suppression.
6. **SOS:** priority and repeat.
7. **Nearby riders:** presence send and receive.
8. **Fill on Return.**
9. **Settings persistence** with read-back and power-loss safety.
10. **Firmware updates:** signed images and rollback.
11. **Further boards:** one board profile at a time, re-running the hardware tests on each.

## 18. Testing

| Layer | Tests | Passes when |
| --- | --- | --- |
| Unit | Encode/decode, IDs, hop count, duplicates, queues, backoff | Deterministic automated pass |
| Conformance | Every test vector, encrypted and decrypted | All vectors pass on every change |
| Protocol | Ride broadcast, unicast, presence, unknown sender, malformed and duplicate packets, replays | Expected state transitions; malformed input rejected |
| Mesh simulation | 1–20 virtual nodes; loss, duplicates, congestion, partitions, rejoining | No relaying beyond the hop limit; SOS delivered first under load; presence never delays SOS |
| Radio / channel access | Channel busy and clear, collisions, backoff, receive/transmit switching | No deadlocks; duty-cycle limits held |
| Hardware-in-the-loop | 3–5 boards: cold boot, sleep/wake, power loss during settings write, failed update, sustained traffic | Settings intact; rollback works; stable operation |
| Field | Range, Fill on Return timing, SOS, approaching-rider warnings, full-day battery, a 4-rider ride at each tier | Meets NFR-001 targets |
| Phone | Connect, version check, send, receive, tracking, nearby riders, reconnect, read-back | App sees only defined protocol events |

## 19. Repository layout and records

| Path | Contents |
| --- | --- |
| `CLAUDE.md` | Session instructions |
| `docs/` | Message spec drafts, as handed over |
| `docs/clean-room/` | POLICY.md, REQUIREMENTS.md |
| `docs/hardware/` | One hardware profile per board, with datasheet references |
| `spec/` | Frozen protocol spec (versioned) and test vectors |
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

- Builds reproducibly from a clean checkout; only approved board targets are enabled.
- All test vectors and conformance tests pass.
- SOS goes first and is never dropped, under simulated and bench load.
- Duplicates are suppressed and the hop limit stops flooding.
- Presence is sent and received, and never delays an SOS.
- Settings survive power loss, and every write is read back.
- Firmware updates roll back cleanly after a failure.
- The phone app connects and exchanges every message in the message spec.
- Every requirement has a passing test.
- Records in Section 19 are complete, and a clean reviewer has approved the code.
- Regulatory and release sign-off from the product owner before any release.
