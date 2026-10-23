# Trail Buddy radio: clean-room policy

Version 0.5 · 23 October 2026 · Issued together with message spec Draft v0.5 (`docs/SPEC-v0.5.md`) and `REQUIREMENTS.md` v0.5. Section numbers are kept from earlier versions; Section 9 (product requirements) is in `REQUIREMENTS.md`.

**Which document wins.** REQUIREMENTS says what the product must do, POLICY says how the clean team works, and the SPEC says exactly how the radio behaves. If they disagree, none of them wins: write the conflict in OPEN-QUESTIONS.md and build nothing that depends on it until the spec team answers. Until then, for anything affecting SOS, the reading that is safer for the rider applies.

This document is written for the implementation team. It deliberately does not name or describe any other project.

## 7. What we are building

Firmware for the Trail Buddy radio. A group of off-road riders each carry a phone running the Trail Buddy app and a small LoRa radio. The phone connects to its radio over Bluetooth Low Energy. The radios form a mesh: each one relays the others' messages, so riders can see each other on a map, message each other and send an SOS with no mobile signal. Any Trail Buddy radio can also hear a short presence signal from any other, so riders can see nearby Trail Buddy riders who are not in their ride.

## 8. Hardware

| Board | Notes |
| --- | --- |
| RAKwireless RAK19026 VC base board with RAK4631 core | **Version 1, lead board.** Reset button, one user button (SW1), two user LEDs, battery disconnection switch, integrated GNSS. Zephyr has a RAK4631 board definition. |
| Heltec Mesh Node T114 V2 | **Version 1.** V2 only; V1 is excluded. One user button; GPS module fitted on every Trail Buddy unit. Zephyr has a board definition. |
| Elecrow ThinkNode M1 | **Version 1.** Function, page and reset buttons; power switch; GPS switch; status LED; buzzer; e-paper screen; integrated GNSS; 2.5 dBi antenna. |
| RAKwireless WisMesh Pocket V3 | IP67 waterproof; RAK19026 baseboard; GPS; notification buzzer. Added when it ships. |

- All of them use a Nordic nRF52840 microcontroller (Bluetooth 5.0) and a Semtech SX1262 LoRa radio.
- Region: United States, 902–928 MHz. Every board transmits at 22 dBm with an antenna of at most 3 dBi (spec 2.1).
- One firmware serves every board, with a separate board profile for each: pins, GNSS, user button, status light, battery measurement and antenna gain.
- Version 1 runs on three boards: the **RAK19026 VC base board with the RAK4631 core** (lead board), the **Heltec T114 V2** and the **Elecrow ThinkNode M1**. Each build step is done on the lead board first; the radio bring-up and the hardware tests are run on all three (Sections 17, 18 and 20). Each has a written hardware profile in `docs/hardware/`. The Pocket V3 is added when it ships.
- Zephyr's RAK4631 and Heltec T114 V2 board definitions may be used as starting points. They are part of the open-source Zephyr project, not another mesh project's board files. The ThinkNode M1's definition is written by the clean team from Elecrow's documents.
- Version 1 uses each board's user button, GNSS and one status light, as mapped in spec Section 9. The status light shows power, mode and pairing only. Buzzers and screens are not used in version 1. SOS alerts are shown by the app.
- The radio never sleeps; the processor idles between events (spec 2.2).
- Anything a manufacturer document doesn't confirm is recorded as unknown in the board profile and raised in OPEN-QUESTIONS.md. Spec 9 lists the unknowns the product owner is chasing.
- Board profiles come only from the manufacturer: datasheets, schematics and the manufacturer's own board support. Never from any other mesh project's board files. Blacked-out text in the cleaned manufacturer copies isn't needed and is ignored.
- All board-specific GPIO, SPI, interrupt, power and radio-register logic stays in the board support package (BSP) and radio HAL. Mesh and application code never contains it.

## 10. Firmware architecture

| Layer | Responsibility |
| --- | --- |
| Phone link (BLE) | Commands, events, settings with read-back, version check, pairing control, connection limits, diagnostics. |
| Trail Buddy protocol | Message types, serialization, identity, addressing, acknowledgements, encryption envelope, SOS owner proof. |
| Mesh | Relay, duplicate suppression, hop limit, rebroadcast delay, queues and priority (SOS first, own SOS before relayed). |
| Channel access | Channel activity detection before every transmission, randomized backoff, the 1 s bound for SOS (spec 4.7), channel-busy measurement and the 30% airtime budget with its slow-down steps (spec 6.10), receive/transmit state machine. Version 1 serves the United States only, where no duty-cycle limit applies. |
| Settings and storage | Persistent settings, the ride record and counter ceiling, SOS state, pairings, position history for Fill on Return, safe defaults, the flash wear budget (spec 4.4). |
| Radio HAL | SX1262: frequency, spreading factor, bandwidth, coding rate, power, receive, transmit, channel activity detection. |
| BSP | nRF52840, SPI, GPIO, interrupts, timers, flash, power, GNSS, user button, status light, random number generator, per board profile. |

## 11. Packets and messages

The spec team designs the formats fresh; nothing is inherited from any existing project.

- **Over the air:** the spec defines every frame byte for byte (spec Sections 4 and 5): envelope, identities, encryption, counters, priorities and relay rules. The clean team implements those layouts exactly and does not change them.
- **Phone link:** the spec defines every message's fields, types, units and limits (spec Section 7). The clean team writes the protobuf schema from those tables, following spec 7.1.
- Anything the spec leaves to the implementation is recorded in DECISIONS.md with its rationale. Anything that would change a frame or message goes to OPEN-QUESTIONS.md instead.

## 12. Security

**Threat model (Decision):** spec 2.7 lists the attackers version 1 must resist and the risks accepted. Every security rule below serves it.

These are decided for building; the spec gives the detail (spec 2.7, 4.3 to 4.9, 6.11, 7.2 and 7.4 Updates). The security review approved the algorithms; only that review can change them, and any change comes as a new spec version.

- Ride messages use AES-128-CCM with an 8-byte authentication tag; keys and tags are derived from the ride key with HKDF-SHA256.
- No nonce is ever reused, by one radio or two: every frame carries the sender's join ID, and the frame counter rules in spec 4.4 hold through power loss, updates and rollback.
- A new ride key for every ride, made by the phone's cryptographic random number generator and shared in a QR code that expires after 7 days.
- Only the rider who raised an SOS can end it, proven by a secret only their radio holds (spec 5, 6.4).
- Presence frames carry no ride key, no names and only a rotating anonymous ID; fake presence is limited as in spec 6.6.
- The phone link is paired (Secure Connections Only) and encrypted before any message except HELLO and WELCOME. Pairing needs a button press at the radio. Nothing that identifies the radio is sent before encryption.
- Firmware images are signed with Ed25519 over the version, board, size and image hash (spec 7.4, Updates). **The bootloader checks the signature itself before every start-up**, whichever route delivered the image. Updates install over Bluetooth or USB; **shipped radios have no unsigned route**. Development radios may keep the UF2 bootloader; they are never shipped and never hold a production signing key.
- The signing keys (primary and backup) are held by the product owner, never in the repository.
- **Release radios lock the processor's debug port.** The accepted remaining risk is that a captured radio may give up its current ride key to laboratory attack (spec 2.7).
- Keys are never logged or reported, and are erased from every flash copy at End Ride, expiry and factory reset (spec 4.9).
- An independent review of the design before the spec is frozen. The v0.4 review is answered in spec v0.5; the reviewer confirms v0.5 before freeze.
- Established, licensed crypto libraries only; no home-made primitives. Build step 2 confirms on the device that the chosen library handles every algorithm and parameter set in the spec.

Robustness:

- Validate every length, ID, hop count, message type and field. Anything not listed as valid in the spec is invalid.
- A malformed packet or phone message must never cause a reboot, a hang or memory corruption.
- Treat queue exhaustion, SOS floods, presence floods and Bluetooth connection floods as denial-of-service cases and test them.

## 13. Phone ↔ radio boundary

The phone app never needs to know how the mesh routes packets. The interface is small, and is defined message by message in the message spec:

- connect, disconnect, protocol-version check and session start
- read radio identity (after encryption), firmware version, battery and link status
- write ride settings (ride key, tier, expiry) with read-back
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
| Yes | Hardware and vendor documentation | Nordic nRF52840 product specification; nRF Connect SDK and Zephyr documentation, including Zephyr's RAK4631 and Heltec T114 V2 board definitions; Semtech SX1262 datasheet and application notes; Heltec T114 datasheet and V2.0 schematic; Elecrow ThinkNode M1 documentation; RAKwireless RAK4631, RAK4630 and RAK19026 documentation; GNSS module manuals; antenna specifications |
| Yes | Approved library documentation | RadioLib, nanopb, the chosen signature-checking bootloader, the UF2 bootloader (development radios only) |
| Yes | Standards and regulations | The standards approved in spec Section 13, including the Bluetooth Core Specification and FCC Part 15. Read only the copies in `docs/standards/`. |
| Yes | Published literature | Papers on flooding and broadcast suppression, cited in DECISIONS.md |
| No | Other mesh messaging projects | Their code, documents, board files, protocol definitions, issue trackers, forums and blog posts, however obtained, including through web search or an AI chat |
| No | Anything outside the clean repository not on this list | Unless the spec team approves it and it is logged in SOURCE-REGISTER.md first |

Vendor documents and standards are added to the repository by the product owner, and each is logged in SOURCE-REGISTER.md, so sessions don't need web access.

## 15. Libraries

| Purpose | Library | Licence |
| --- | --- | --- |
| RTOS and BLE | Zephyr / nRF Connect SDK | Zephyr: Apache-2.0. Nordic SDK parts: as stated in each part (verify) |
| LoRa driver | RadioLib, or Semtech's SX126x driver | MIT for RadioLib (verify); check Semtech's |
| Phone-link serialization | nanopb, from our own protobuf schema | zlib (verify) |
| Over-the-air frames | Hand-written from the spec's byte layouts | Ours |
| Cryptography | The crypto libraries supplied with nRF Connect SDK (software and CryptoCell hardware drivers) | Expected to be Nordic's own licence (LicenseRef-Nordic-5-Clause), not Apache-2.0. Record the licence file actually shipped with each library. |
| Updates | A two-slot bootloader that checks signatures before every start-up, takes signed updates over USB, and rolls back, chosen by the clean team from the Zephyr ecosystem | Record in SOURCE-REGISTER.md (verify) |
| Development recovery | Each board's UF2 bootloader, over USB, on development radios only | MIT (verify) |

- Record every dependency's licence and purpose in SOURCE-REGISTER.md before using it, from the licence file shipped with that dependency.
- **The Nordic 5-Clause licence** (copy supplied by the product owner) allows use only with Nordic chips, which all version 1 boards are; allows binaries embedded in a Nordic-based product or its updates without a notice; forbids modifying or reverse-engineering Nordic's binary-only libraries; and forbids using Nordic's name to endorse the product. Firmware using these libraries can't move to a non-Nordic chip without replacing them. Any library under a different Nordic agreement goes to the product owner before use.
- Pin the SDK version, and record it in DECISIONS.md.
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
- Never log keys, derived keys or SOS secrets.
- Detailed logging is compiled out of release builds. The diagnostic counters required by DIAG-001 stay in release builds.

## 17. Build order

Hardest first, so failures show up early. Unit tests come before hardware for every step. All steps run on the lead board first.

1. **Radio bring-up:** raw send and receive between two lead boards, then between each pair of the three version 1 boards.
2. **Frame format and crypto:** all test vectors in spec Section 11 pass, including the rejection cases and the signature vector, on the device with the chosen crypto library. Conformance tests run in CI from here on.
3. **BLE phone link** with a minimal test app: advertising, connection limits, encryption deadline, pairing window, framing and re-synchronisation.
4. **GPS and position** broadcast by tier.
5. **Mesh simulator** (1–30 virtual nodes), then relay and duplicate suppression. The simulator confirms:
   - the airtime figures in spec 2.5
   - burst behaviour: Delivered and hello replies in a 30-rider ride, and Fill on Return at a busy trailhead (Q104)
   - the SOS delivery target in spec 4.7 (SOS-006)
   - SOS floods and SOS frames over the schedule limit (spec 4.5)
6. **Repeater mode.**
7. **SOS:** priority, repeat every 30 s, owner proof, reboot survival.
8. **Nearby riders:** presence send and receive, nearby table limits.
9. **Fill on Return.**
10. **Settings and storage:** persistence with read-back, power-loss safety, counter ceiling rules, flash wear budget.
11. **Firmware updates:** signed images over Bluetooth and USB, signature checked before every start-up, rollback, backup key.
12. **T114 V2 and ThinkNode M1:** complete each board profile (user button, status light, GNSS, battery, antenna gain) and re-run the hardware-in-the-loop and field tests on each.
13. **WisMesh Pocket V3,** when it ships.

## 18. Testing

| Layer | Tests | Passes when |
| --- | --- | --- |
| Unit | Encode/decode, IDs, hop count, duplicates, queues, backoff, counter ceiling rules, SOS owner proof | Deterministic automated pass |
| Conformance | Every test vector, encrypted and decrypted, including rejection cases | All vectors pass on every change |
| Protocol | Ride broadcast, unicast, presence, unknown sender, malformed and duplicate packets, replays, "sent as you", forged "I'm OK", time-suspect frames | Expected state transitions; malformed input rejected; no SOS dropped |
| Fuzz | (1) raw LoRa frames; (2) authenticated frames with malformed content, using a test ride key; (3) Bluetooth framing and Envelope decoding; (4) phone messages in the wrong order or state | No reboot, hang, watchdog reset or memory fault in any run |
| Mesh simulation | 1–30 virtual nodes; loss, duplicates, congestion, partitions, rejoining; channel busy per tier; bursts; SOS delivery; SOS floods | No relaying beyond the hop limit; SOS delivered first under load and within the SOS-006 target; presence never delays SOS |
| Radio / channel access | Channel busy and clear, collisions, listen before talking, backoff, the 1 s SOS bound, receive/transmit switching, slow-down steps | No deadlocks; normal traffic held at or below 30% busy; SOS never delayed by channel access for more than 1 s |
| Bluetooth security | Pairing window rules, one new phone per window, fifth phone refused, stranger holding a connection, encryption deadline, nothing identifying before encryption, phone time only after encryption, pairings kept through updates | All behave as spec 6.11 and 7.2 |
| Hardware-in-the-loop | 3–5 boards, including at least one of each version 1 board: cold boot, processor idle and wake, GPS duty cycling, power loss during settings and ceiling writes, failed update, rollback, sustained traffic, debug port locked in release builds | Settings and counters intact; rollback works; stable operation |
| Field | Range, Fill on Return timing, SOS, approaching-rider warnings, full-day battery, a 4-rider ride at each tier | Meets NFR-001 targets; with 4 riders at 10 s updates, the channel stays usable for an SOS |
| Phone | Connect, version check, session start, send, receive, tracking, nearby riders, reconnect, read-back, iPhone "forget this device" help | App sees only defined protocol events |

## 19. Repository layout and records

| Path | Contents |
| --- | --- |
| `CLAUDE.md` | Session instructions |
| `docs/` | Message spec drafts, as handed over |
| `docs/clean-room/` | POLICY.md, REQUIREMENTS.md |
| `docs/hardware/` | One hardware profile per board, with datasheet references, and the cleaned manufacturer documents |
| `docs/standards/` | Official copies of the approved standards (spec Section 13), added by the product owner |
| `docs/licences/` | Licence texts supplied by the product owner, such as the Nordic 5-Clause licence |
| `spec/` | Frozen protocol spec (versioned) |
| `spec/vectors/` | Test vectors: the spec's own (spec Section 11) and the recorded phone-link vectors |
| `firmware/bsp/`, `radio/`, `channel/`, `mesh/`, `protocol/`, `storage/`, `ble/`, `app/` | Firmware by layer (Section 10) |
| `tests/unit/`, `conformance/`, `fuzz/`, `simulation/`, `hardware/` | Tests by layer |
| `tools/` | Test and debug utilities |
| `logs/sessions/` | Every clean session transcript |

| Record | Purpose | Kept by |
| --- | --- | --- |
| SOURCE-REGISTER.md | Every external source and dependency, its licence and what it contributed | Implementers |
| DECISIONS.md | Implementation decisions and rationale, with literature citations; bootloader choice, flash layout, flash wear calculation, SDK version, worst-case message sizes | Implementers |
| OPEN-QUESTIONS.md | Questions for the spec team; never guessed | Implementers |
| TEST-MATRIX.md | Requirement-to-test traceability | Implementers |
| ATTESTATIONS/ | Signed statements from every human implementer and reviewer | Product owner |
| Clean-room log | Who saw what and when, spec hand-overs, questions and answers, security reviews. Kept outside the clean repo | Product owner |

## 20. Definition of done

- Builds reproducibly from a clean checkout for all three version 1 boards; only approved board targets are enabled.
- Hardware-in-the-loop tests pass on the RAK19026 VC, the T114 V2 and the ThinkNode M1, and a mixed ride using all three works.
- All test vectors and conformance tests pass, including rejection cases.
- Fuzz tests run without a reboot, hang, watchdog reset or memory fault.
- SOS goes first; the radio's own SOS and "I'm OK" are never dropped; SOS meets the SOS-006 delivery target in simulation; only the owner can end an SOS; an active SOS resumes after a reboot.
- Duplicates are suppressed and the hop limit stops flooding.
- Presence is sent and received, and never delays an SOS.
- DIAG-001 counters are present in the release build, with no key material.
- Settings, ride record and counters survive power loss, and every write is read back.
- Firmware updates over Bluetooth and USB accept only signed images; the bootloader checks before every start-up; rollback works.
- Release builds lock the debug port; shipped radios have no unsigned update route.
- The phone app connects and exchanges every message in the message spec.
- Every requirement has a passing test.
- Records in Section 19 are complete, and a clean reviewer has approved the code.
- The security reviewer has confirmed the frozen spec.
- Regulatory and release sign-off from the product owner before any release.

## Changes from v0.4

| Section | Change |
| --- | --- |
| 8 | Antenna limit 3 dBi; antenna gain in each board profile; no sleep mode; unknowns recorded, not guessed; blacked-out text ignored. |
| 10 | Pairing control, connection limits, SOS owner proof, own-SOS priority, 1 s SOS bound, storage rules, random number generator. |
| 12 | Threat model; nonce rule with join ID; new key every ride, 7-day codes; SOS owner proof; Secure Connections Only and button-press pairing; signatures checked by the bootloader before every start-up on both routes; no unsigned route on shipped radios; debug port locked; key erasure; review status. |
| 13 | Session start; identity only after encryption; ride expiry. |
| 14 | Heltec datasheet, antenna specifications and the signature-checking bootloader added. |
| 15 | Nordic 5-Clause licence on the crypto libraries; signature-checking bootloader takes USB updates; UF2 on development radios only; pin the SDK version. |
| 16 | Never log keys or SOS secrets. |
| 17 | Simulator scenarios for bursts, SOS delivery and SOS floods; Bluetooth security in step 3; storage rules in step 10; both update routes in step 11. |
| 18 | Fuzz tests and Bluetooth security tests added; "sleep/wake" replaced by processor idle and wake. |
| 19 | `docs/licences/` and `tests/fuzz/` added; more items recorded in DECISIONS.md. |
| 20 | Done-criteria for fuzzing, SOS delivery and ownership, signed updates on both routes, debug lock, security sign-off. |
