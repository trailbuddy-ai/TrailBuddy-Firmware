# Open questions for the spec team

Raised by: buildability review of `docs/SPEC-v0.2.md`, 22 October 2026.
Sources used: `CLAUDE.md`, `docs/clean-room/POLICY.md`, `docs/clean-room/REQUIREMENTS.md`, `docs/SPEC-v0.2.md`. Nothing else.

Each question gives the spec section, what is unclear, why it blocks implementation, and options. Options describe behaviour only. Where a question needs a product decision, it says so.

## Review basis

**Counts.** The phone-link table has 40 messages in 14 areas, which matches the spec's own claim. The over-the-air section has 9 message kinds plus the envelope.

**Freeze checklist.** The spec's own checklist has five items. Only the requirement ID is done. No message, on either link, has any of these:
- required or optional marking
- size limits or a byte budget
- units or precision
- sample values

Q55 covers this. The questions below are the specific gaps that stop a build.

**Requirement coverage (step 3).** These requirements have no message or behaviour in the spec, or only part of one:

| Requirement | Status | Question |
| --- | --- | --- |
| POS-003 (GPS duty cycle follows tier) | Not covered by any message or behaviour | Q34 |
| MESH-003 (randomized rebroadcast delay) | Not mentioned in the spec | Q14 |
| REG-001 (power, dwell time, duty cycle) | Not mentioned in the spec | Q1, Q16 |
| NFR-001 (range, battery, max packet size, return-to-filled-track time) | Targets not supplied | Q2 |
| FILL-001 (five rules) | REQUIREMENTS says the spec restates them; it does not | Q38 |
| RIDE-002 (four tiers, hop limit, interval) | Tier is a field, but no tier values exist | Q10 |
| NEAR-003 (presence intervals) | Left to the spec team and not done | Q42 |
| NEAR-005 (approaching warning) | App-only; no firmware part defined | Q45 |
| NEAR-008 (empty view not shown as clear) | App-only; the radio gives no state to support it | Q45 |
| BLE-003 ("link status") | No field in any message matches this | Q25 |

**Contradictions (step 4).** Raised as questions: Q4, Q5, Q12, Q15, Q16, Q34, Q37, Q43, Q48, Q53, Q54.

---

## A. Foundations that every message depends on

**Q1. Radio settings and channel plan are not defined.**
- Spec section: Over-the-air messages; REG-001. POLICY §11 and §12.
- Unclear: Frequency or channels within 902–928 MHz, spreading factor, bandwidth, coding rate, transmit power, preamble, and whether channels hop. No dwell-time or duty-cycle limit is given. REG-001 says "within the approved channel plan", and no plan is in the files.
- Blocks: POLICY §11 requires every message to be budgeted in bytes against the payload limit "at the chosen radio settings". Without settings there is no limit, no airtime budget, and no way to set presence or position intervals. It also blocks build step 1 (radio bring-up) and all channel-access limits.
- Options:
  - The product owner supplies one fixed channel plan.
  - Supply a short list of candidate plans and have the spec team pick one.
  - Supply the regulatory limits as numbers, so channel access can be tested against them.

Status (v0.3): ANSWERED — spec 2.1 (one fixed channel at 916.5 MHz, 500 kHz bandwidth, SF9, coding rate 4/5, preamble 12, sync word 0x12, explicit header with CRC, 22 dBm with an antenna of at most 2 dBi, no hopping, no duty-cycle limit if the lab confirms the digital-modulation route). Provisional until the compliance lab confirms (spec 14). The transmit procedure is still unstated: see Q64.

**Q2. NFR-001 targets are missing.**
- Spec section: NFR-001.
- Unclear: Maximum packet size, range, battery hours per riding day, and return-to-filled-track time are "set by the product owner" and not stated.
- Blocks: Maximum packet size is the input to every byte budget in the freeze checklist.
- Options: The product owner supplies the numbers. Alternatively, accept provisional numbers and mark them as such until v1.0.

Status (v0.3): PARTLY ANSWERED — spec 3 and NFR-001. Given: 12 hours per riding day, filled track within 60 s of return, maximum frame 176 bytes (consistent with spec 4.1 and 5). Missing: (a) range is "at least as good as the August field test", and that test's result is not in the repository, so there is no number to test against; (b) the 12-hour target has no conditions (tier, presence on or off, GPS duty, battery capacity, temperature, board); (c) "return" for the 60 s target is not defined, and the target conflicts with the Fill on Return rates (Q58).

**Q3. Do presence and ride traffic share a channel?**
- Spec section: Open questions (presence vs ride frequencies); Ride QR code; JOIN_RIDE.
- Unclear: The open question says "rides may use different frequencies". The QR code carries only format version, ride name, ride key and tier ("Nothing else"), so there is no way to tell a radio which frequency a ride uses.
- Blocks: The receive/transmit state machine, whether a radio listens on one channel or several, and presence airtime.
- Options:
  - One common channel for everything.
  - A common presence channel plus a common ride channel.
  - A per-ride channel derived from the ride key, with a defined presence timing slot.
  - If per-ride channels stay, the QR code and JOIN_RIDE both need a new field.

Status (v0.3): ANSWERED — spec 2.1 (Decision: one common channel for all Trail Buddy radios; rides are separated by keys only). The QR code needs no channel field (spec 8).

**Q4. Who designs the over-the-air layout, and is it protobuf?**
- Spec section: Encoding (How it works); Over-the-air messages intro. POLICY §11, §15.
- Unclear:
  - The spec says "the clean team designs the layout from scratch". POLICY §11 says "the spec team designs the packet format fresh" and that field sizes go in DECISIONS.md.
  - The spec says the encoding is protobuf. It does not say whether over-the-air messages also use protobuf, which has variable-length fields, or a fixed binary layout.
- Blocks: Who owns the layout, the byte budgets, and the test vectors. Byte budgets cannot be fixed if field sizes vary.
- Options:
  - The spec team defines exact bit and byte layouts for over-the-air messages and the clean team implements them.
  - The clean team designs layouts inside spec-given size budgets, with answers recorded in DECISIONS.md.
  - Protobuf is used on the phone link only, with a separate fixed layout on air.

Status (v0.3): ANSWERED — spec 1 (the spec team defines over-the-air frames byte for byte as fixed binary layouts, not protobuf; the clean team writes the phone-link protobuf schema from the spec tables). POLICY §11 now says the same. Schema details still open: Q92.

**Q5. Documents the spec refers to are not available, and one instruction conflicts with POLICY.**
- Spec section: Encoding; Clean-room note. POLICY §19.
- Unclear:
  - "How to Create Trail Buddy's Protocol Buffers" is cited and is not in the four files.
  - "Section 9 of the Clean-Room Firmware Development Specification" is cited. REQUIREMENTS.md appears to reproduce it, and I can't confirm that.
  - The spec says to log this version in `docs/PROVENANCE.md`. POLICY §19 says the clean-room log is kept outside the clean repo, and the repository layout has no PROVENANCE.md.
- Blocks: Schema conventions such as field naming, enum style and optional-field rules. It also leaves open where hand-over records go.
- Options: Supply the protobuf guide, or state the rules it carries. Decide whether PROVENANCE.md exists in the repo or the log stays outside it.

Status (v0.3): PARTLY ANSWERED — spec 1 (no docs/PROVENANCE.md; hand-overs go in the clean-room log kept outside the repository, as POLICY §19 says) and 7.1 (proto3, package trailbuddy.v1, names from the spec, field numbers never reused, enums start with an unspecified 0, bounded fields). The protobuf guide and the Clean-Room Firmware Development Specification are no longer cited. Missing: how required and optional (R/O) map onto proto3, who assigns field numbers and how the schema reaches the app team, and the message structure on the wire (Q92).

**Q6. The radio has no defined clock, and "time" is used everywhere.**
- Spec section: STATUS, TEXT_RECEIVED, GET_MISSED, GET_TRACK, GET_NEARBY, GET_DIAGNOSTICS, SEND_SOS, RIDERS_LIST ("last heard"), over-the-air SOS ("time sent").
- Unclear: Which clock the radio uses, and how it is set. No message lets the phone set the time. GPS time exists only with a fix. Also unspecified: time format (absolute or relative), resolution, rollover, and what happens when time is unknown.
- Blocks: "time received", "since (time)", "last heard", storage expiry and "time sent" can't be built. Stored history can't be ordered or aged.
- Options:
  - Add a phone-to-radio time-set message.
  - Take time from GPS only.
  - Use radio-local counters (relative ages) in place of absolute times.
  - Each field then needs a stated behaviour when time is unknown.

Status (v0.3): ANSWERED — spec 2.2 and 2.6 (seconds since 2026-01-01 UTC, unsigned 32-bit, 0 = unknown; clock from GPS, else phone time in HELLO; ages and timeouts use uptime). Rule wording is still ambiguous: Q75.

**Q7. Four kinds of ID are used, and none is defined.**
- Spec section: WELCOME (radio ID), most messages (rider ID), NEARBY_RIDER (anonymous rider ID), envelope (sender radio ID). REQUIREMENTS MSG-003 (short ID). POLICY §11.
- Unclear:
  - Whether radio ID, rider ID and sender radio ID are the same value.
  - How "short ID" differs, and where any message carries it. MSG-003 names it, and no message does.
  - Size, how the ID is created, whether it survives reflash or factory reset, and the reserved values.
  - POLICY §11 requires "how a radio's ID is created and kept" to be defined.
- Blocks: Envelope layout, duplicate suppression, addressing, riders list, and unicast.
- Options: One persistent radio ID that is used as the rider ID inside a ride. Or a separate ride-scoped rider ID assigned at JOIN_RIDE. Either way, define the short form as a derived value of fixed length.

Status (v0.3): PARTLY ANSWERED — spec 2.3 defines all four IDs (8-byte random radio ID kept until factory reset; 3-character short ID; ride-scoped 3-byte rider tag; 4-byte anonymous ID) and where each appears. Missing: (a) bit order and character order of the short ID (Q73); (b) reserved values — a rider tag of 000000 collides with "whole ride" (Q72); (c) "factory reset" is named but never defined (Q71).

**Q8. The cleartext envelope conflicts with anonymous presence.**
- Spec section: Envelope; Presence; NEAR-002. POLICY §12.
- Unclear:
  - The envelope must carry a "sender radio ID" on every message for duplicate suppression and relay.
  - If presence also carries the permanent radio ID, the daily anonymous ID gives no anonymity.
  - It is also unstated whether ride-message envelopes are readable by outsiders. That would let anyone track a radio's permanent ID.
  - The rotation rule is undefined: how it is generated, when "daily" changes without a reliable clock, whether it changes mid-encounter, and how an app avoids showing one rider as two.
- Blocks: Envelope layout, presence design, NEAR-002 testing, and the app's approaching-rider logic.
- Options:
  - Presence uses only the anonymous ID in the envelope.
  - Ride envelopes use a ride-scoped sender ID.
  - Rotate on a counter, not a clock.
  - Accept that the permanent ID is visible and document the privacy limit.

Status (v0.3): ANSWERED — spec 2.3 (ride envelopes carry only the ride-scoped rider tag; presence carries only the anonymous ID; the privacy limit is documented; the anonymous ID is random, new at each boot and every 6 hours of operation, applied only while stopped, and carried in the rider's hello). Gaps in the rotation rule: Q76.

**Q9. Ride size and ride identification on air.**
- Spec section: JOIN_RIDE; envelope destination "ride"; RIDERS_LIST.
- Unclear: Maximum riders per ride, and whether a radio holds exactly one ride at a time. There is no ride identifier on air, so a radio can't tell whether a packet is for its ride without trying to decrypt it.
- Blocks: List sizes, duplicate-cache size, airtime budget, memory use, and the receive path (what is tried, and how often it fails).
- Options: State a maximum ride size and a single active ride. Add a short ride identifier to the envelope, or derive one from the key.

Status (v0.3): ANSWERED — spec 2.4 (at most 30 riders, one active ride, 2-byte ride tag in every ride frame). How the limit of 30 is enforced: Q72.

**Q10. The four tiers are not defined.**
- Spec section: JOIN_RIDE; QR code. RIDE-002.
- Unclear: Tier names, how many bits, hop limit per tier, position interval per tier, GPS duty per tier, and presence interval per tier (the open question asks for this). Also whether tier can change during a ride and how.
- Blocks: POS-001, POS-003, hop limits (MESH-001), build step 4, the mesh simulation and the 4-rider field test at each tier.
- Options: A table of four tiers with named values for hop limit, position interval and GPS duty. Allow a mid-ride tier change only through a new JOIN_RIDE, or add an explicit message.

Status (v0.3): ANSWERED — spec 2.5 (four tiers with hop limit, position interval and Fill on Return gap threshold; fixed for the ride, changed only by ending the ride and joining again), 6.1 (GPS duty) and 6.6 (presence intervals). Conflict with the fresh-fix rule: Q57.

**Q11. The crypto design is not chosen.**
- Spec section: Envelope paragraph. POLICY §12.
- Unclear:
  - AES-CCM or ChaCha20-Poly1305 is not chosen.
  - Key size and tag size are not given.
  - The nonce scheme and replay window are not specified. Neither is where the per-sender counter lives, how it survives reboot and power loss without repeating a value, or what happens when a counter is lost.
  - No ride-key derivation or rotation is described, and no key check value is given for read-back (see Q27).
  - Which envelope fields are authenticated but not encrypted is not stated.
- Blocks: Build step 2 (frame format and crypto), tag and nonce overhead in every byte budget, and the independent review that POLICY requires before freeze.
- Options: The spec team picks the algorithm, the tag and nonce sizes, and a counter rule. The rule could use a persistent counter written ahead in blocks, or a random session value plus a counter.

Status (v0.3): ANSWERED — spec 4.3 (AES-128-CCM, 8-byte tag, 13-byte nonce, HKDF-SHA256 labels, key check value, envelope authenticated with hops zeroed) and 4.4 (counter ceiling 1024 ahead, written before sending at boot). Run-time ceiling advance and flash failure: Q71. The spec 11 vectors were not recalculated in this review.

**Q12. Can radios that don't hold a ride key relay?**
- Spec section: MESH-001, RPT-001; Envelope.
- Unclear:
  - MESH-001 says every radio relays ride traffic. Does it relay other rides' traffic? Does a repeater, which has no phone and so has no way to load a key through JOIN_RIDE, relay at all?
  - Authenticated encryption means only key holders can verify a packet. A keyless relay can't tell a forged flood from real traffic.
- Blocks: Relay rules, the DoS tests POLICY §12 requires, and repeater mode.
- Options:
  - Relay only packets that verify under the ride key held.
  - Relay any well-formed packet within hop and rate limits, with per-sender rate caps.
  - Repeaters hold a ride key, loaded some other way.

Status (v0.3): ANSWERED — spec 4.6 (a radio relays only frames that carry its own ride tag and pass authentication) and 6.8 (a repeater is joined to a ride by a phone and keeps the key through reboots).

**Q13. Duplicate suppression is not specified.**
- Spec section: Envelope; MESH-002.
- Unclear: Packet identity size and uniqueness scope. Duplicate-memory size and lifetime. The spec needs SOS to repeat every 5 minutes. A repeat must not be dropped as a duplicate, and a replay must still be rejected.
- Blocks: Unit tests for duplicates and replays, the SOS repeat logic, and memory sizing.
- Options: Every SOS repeat gets a new packet identity under the same alert ID. Define memory by number of entries and by age, and state which is used when they conflict.

Status (v0.3): ANSWERED — spec 4.4 (identity = sender rider tag + frame counter; last 256 identities for 10 minutes; replay window of 64 per sender; every SOS repeat is a new frame with a new counter and the same alert ID). Which limit wins when 256 entries and 10 minutes conflict is not stated: Q70.

**Q14. Rebroadcast delay (MESH-003) has no behaviour in the spec.**
- Spec section: not present. MESH-003 requires a randomized delay with a weighting chosen from published literature.
- Unclear: The delay range, any weighting, and whether delay depends on signal strength, hops remaining or priority. Does SOS skip the delay or use a shorter one?
- Blocks: The relay implementation and the mesh simulator.
- Options: The spec team chooses a scheme, cites literature in DECISIONS.md, and states different ranges per priority class.

Status (v0.3): ANSWERED — spec 4.6 (random delay; SOS and SOS replies 0–150 ms; other kinds a 1000 ms window scaled by SNR plus 0–20%; suppression after 2 other relays; Ni, Tseng, Chen and Sheu cited). The formula wording is ambiguous: Q68.

**Q15. The priority list is incomplete and queue rules are missing.**
- Spec section: Priority paragraph; envelope priority class. SOS-002, NEAR-007.
- Unclear:
  - Nine over-the-air kinds exist, and the priority order names seven. Rider hello and Fill ask are missing.
  - The number of priority classes in the envelope is not given.
  - Queue sizes are not given.
  - What happens when the queue is full of SOS packets is not defined.
  - Whether stale items are replaced is not defined. An example is an older queued position of the same rider.
  - A transmission already in progress can't be aborted, so "presence never delays an SOS" needs a maximum presence airtime.
- Blocks: The queue implementation, the SOS-first mesh test, and the NEAR-007 test.
- Options: A full ordered list with class numbers. State a queue length per class. State a replace-oldest rule for positions and presence.

Status (v0.3): PARTLY ANSWERED — spec 4.5 (classes 0–6 covering every kind, queue length and full-queue rule per class; longest frame about 230 ms, presence frame about 45 ms). Missing: what happens when the SOS class is full of different alerts and replies (Q59); whether relayed and own frames share queues (Q69).

**Q16. SOS-002 can conflict with duty-cycle limits.**
- Spec section: SOS-002 vs REG-001; POLICY §10 (channel access limits).
- Unclear: SOS is "never dropped" and "goes ahead of every other transmission". Channel access also holds airtime and duty-cycle limits. If the limit is reached, is an SOS delayed, or does it have its own allowance?
- Blocks: Channel-access design and the "duty-cycle limits held" and "SOS never dropped" tests, which can't both pass without a rule.
- Options: SOS may exceed the normal budget within a stated cap. Or SOS is delayed until the limit allows, never dropped. The product owner and regulatory sign-off decide.

Status (v0.3): ANSWERED — spec 2.1 (SOS and SOS replies are exempt from the 30% budget and never delayed by it; the Part 15 route has no duty-cycle limit if the lab confirms). POLICY §10 and §18 still speak of duty-cycle limits being held: Q62. SOS and channel detection: Q64.

**Q17. Envelope sizes, per-kind addressing and protocol-version handling are missing.**
- Spec section: Envelope; the over-the-air table. POLICY §11.
- Unclear:
  - There is no size for any envelope field and no rationale for the sizes.
  - There is no destination or direction for each over-the-air kind. For example, is Delivered sent to the sender only? Is Fill answer sent to one rider or to the ride?
  - There are no hop-limit rules per kind, including whether SOS or presence use a different hop count from the tier.
  - There is no rule for a packet with a protocol version the radio doesn't know: drop, or relay unread.
- Blocks: Build step 2, byte budgets, relay behaviour.
- Options: Add a table of kind, destination, unicast or broadcast, hop rule, and priority class. State a protocol-version rule.

Status (v0.3): ANSWERED — spec 4.1 (field sizes and offsets), 4.6 (destination, starting hops and class for every kind), 4.7 (frames with another version are dropped and not relayed). Rules for unknown kinds and out-of-range values: Q67. A rationale per size is not given: see Q55.

**Q18. One protocol version or two?**
- Spec section: "Every phone-link message carries … the protocol version"; HELLO/WELCOME; the envelope.
- Unclear: Whether phone-link and over-the-air versions are the same number, its format (single integer or major.minor), the compatibility rule, and why every phone-link message repeats the version after HELLO has already checked it.
- Blocks: The version check, BLE-001 tests, and message overhead on a small BLE packet.
- Options: One number with an exact-match rule. Or separate numbers, with the phone link allowed a range.

Status (v0.3): ANSWERED — spec 4.7 (one protocol version number, 1, for both links; exact match; the phone link checks it once, in HELLO and WELCOME).

## B. Phone link

**Q19. The BLE service itself is not defined.**
- Spec section: Phone link; BLE-001.
- Unclear: Service and characteristic layout (what the phone writes, what it is notified of), identifiers, maximum message size and how larger messages are split, ordering, flow control, and the number of simultaneous connections.
- Blocks: Build step 3, and every message that can exceed one BLE packet: TRACK_POINTS, NEARBY_LIST, RIDERS_LIST, DIAGNOSTICS, UPDATE_CHUNK.
- Options: The spec team defines the service contents and a framing rule. Choose between one write/notify pair for all messages and separate channels for bulk data such as updates.

Status (v0.3): ANSWERED — spec 7.2 (service and two characteristic UUIDs; 2-byte length framing across Bluetooth packets; largest packet size requested; 1024-byte maximum message; one write at a time with response; one phone at a time). Framing details: Q93.

**Q20. BLE pairing and link security are not stated.**
- Spec section: Phone link; JOIN_RIDE; START_UPDATE.
- Unclear: Whether the link is paired, bonded and encrypted, who may connect, and whether another phone can connect and read the ride key or messages. JOIN_RIDE sends the ride key, and START_UPDATE accepts firmware over the same link.
- Blocks: Connection setup, and the security review POLICY §12 requires.
- Options: Require an encrypted, bonded link before any message other than HELLO. Add a pairing step on first use, with a physical confirmation where the board has a button.

Status (v0.3): ANSWERED — spec 7.2 (LE Secure Connections pairing and encryption before any message except HELLO and WELCOME; new pairing only in the 2 minutes after power-on; 4 remembered phones; the ride key is never sent back to the app). The association model and how to add a phone later: Q94.

**Q21. Session rules are missing.**
- Spec section: Connect messages (HELLO, WELCOME, GOODBYE).
- Unclear:
  - What a radio does with any message sent before HELLO.
  - Session and idle timeouts.
  - Whether a second phone is refused or replaces the first.
  - What the radio keeps doing after GOODBYE or a drop: relay, SOS repeat, position broadcast, and storage of missed items.
  - Whether a radio in repeater mode accepts a phone at all.
- Blocks: The BLE state machine, reconnect tests, and "Phone" test layer.
- Options: State a session state machine in words: states, events, and what is refused in each.

Status (v0.3): ANSWERED — spec 7.2 Session (ERROR NOT_ALLOWED_NOW before HELLO; no idle timeout; a second phone is refused; the radio carries on after a disconnect or GOODBYE; repeaters accept phones).

**Q22. Request-ID and message-envelope rules are inconsistent.**
- Spec section: "Every phone-link message carries … a request ID"; Key fields column.
- Unclear:
  - Request ID is listed as a key field on some messages and not on others. That includes ones that answer a request, such as STATUS, SOS_STATUS and SETTINGS.
  - Its size, who allocates it, wraparound, and reuse are not given.
  - What the value is for unsolicited events such as TEXT_RECEIVED or SOS_RECEIVED is not stated.
  - How several replies to one request, such as GET_MISSED or GET_TRACK, are matched and ended is not stated.
  - Whether unknown fields are ignored or rejected, and how a missing required field is handled, is not stated.
- Blocks: Every encoder and decoder, and every phone-link test.
- Options: One header on every message: type, request ID, with a reserved "none" value. State a rule for multi-part replies, and state unknown-field behaviour once.

Status (v0.3): ANSWERED — spec 7.3 (every message has a type, a 32-bit request ID chosen by the app with 0 = unsolicited, and a last flag; unknown fields are ignored; an unknown type or missing required field gets ERROR). How a missing field is detected in the schema: Q92.

**Q23. The error model is split across several messages.**
- Spec section: ERROR; RIDE_RESULT; SEND_RESULT; SETTINGS_RESULT.
- Unclear:
  - Four places report failure, and each has its own "reason". No error-code list exists.
  - Which messages have no result and so rely on ERROR. Examples are SHARE_MY_LOCATION, SOS_REPLY, CLEAR_DIAGNOSTICS and every GET.
  - ERROR's "short description" has no length, and it is not stated whether it is shown to riders.
  - The behaviour for an unknown message type is not stated.
- Blocks: Error handling and the "malformed input rejected" tests.
- Options: One shared code list with each message listing which codes it may return. Drop the free-text description.

Status (v0.3): ANSWERED — spec 7.3 (one code list for every message; ERROR carries the request ID and a code only, no free text; messages that cannot otherwise fail silently report through ERROR). Overlapping codes: Q95.

**Q24. HELLO, WELCOME and GOODBYE details.**
- Spec section: Connect rows.
- Unclear:
  - Formats for app version and firmware version, and whether firmware versions must be ordered. START_UPDATE compares versions.
  - The mismatch refusal is a "clear reason", but it is not stated which message carries it: WELCOME with an error, ERROR, or GOODBYE.
  - The GOODBYE reason values are not listed.
  - The radio ID format is not given.
  - The board model is listed as names; the encoding is not given.
- Blocks: Connection and version-check tests.
- Options: Define the version formats and the mismatch path. List the reason values.

Status (v0.3): ANSWERED — spec 7.4 Connect (versions as three 32-bit numbers, 8-byte radio ID, short ID, board model enum, GOODBYE reasons listed) and the text after the Connect table (on a version mismatch the radio sends WELCOME, then GOODBYE with VERSION_MISMATCH, then disconnects).

**Q25. STATUS and "link status".**
- Spec section: GET_STATUS, STATUS. BLE-003, MESH-004. POLICY §13.
- Unclear:
  - BLE-003 and POLICY §13 require "link status". No field in any message matches. It could mean the phone link or the radio's contact with the ride.
  - "Channel busy" is not defined: measurement method, window, resolution, and whether it counts the radio's own transmissions.
  - "Messages waiting" is not defined (count of what, and cleared when).
  - When a change triggers an unsolicited STATUS, and at what rate, is not stated. A 1% battery change could produce a message every minute.
  - What is reported when a board has no battery measurement or no GPS module.
- Blocks: Status implementation, MESH-004 test, and the app's status screen.
- Options: Define "link status" in words and add fields. Add change thresholds with a minimum interval. Add an "unavailable" value.

Status (v0.3): PARTLY ANSWERED — spec 7.4 Status. Answered: link status = ride contact (members heard in the last 2 × position interval); channel busy = share of the last 60 s spent receiving, detecting or transmitting; STATUS triggers (change of charging, GPS or ride contact, battery by 5 points, at most every 30 s); UNAVAILABLE and 255 for missing hardware. Missing: when an event counts as "sent to the app" for events waiting (Q86); how activity is detected for channel busy (Q65); the ride contact window differs from the other "heard recently" windows (Q60).

**Q26. The app has no way to read ride or SOS state after reconnecting.**
- Spec section: Ride and SOS messages; SETTINGS; WELCOME. BLE-002.
- Unclear:
  - RIDE_RESULT is sent only in reply to a ride change. SETTINGS has no ride fields. SOS_STATUS is described as tracking a rider's own SOS, and nothing asks for it.
  - After a reconnect, the app can't learn whether a ride is active, which tier, or whether an SOS is running.
- Blocks: Reconnect (a listed phone test) and BLE-002's "reads every stored value back".
- Options: Add a "get ride" message and a "get SOS" message. Or have WELCOME carry ride and SOS state, or send RIDE_RESULT and SOS_STATUS automatically after connect.

Status (v0.3): ANSWERED — spec 7.2 On connect (after WELCOME: RIDE_STATE, SOS_STATUS if the rider's own SOS is active, SOS_RECEIVED for each active SOS from others, then STATUS), 7.4 Ride (GET_RIDE) and SOS (GET_SOS).

## C. Rides

**Q27. JOIN_RIDE fields and rules.**
- Spec section: JOIN_RIDE, RIDE_RESULT. RIDE-001, BLE-002.
- Unclear:
  - Ride name length and characters, key size and encoding, and tier encoding.
  - A request ID is not listed.
  - What happens when a ride is already active: replace, or refuse. This includes an active SOS.
  - Which keys are refused (all zero, wrong length).
  - Whether the ride survives a reboot.
  - RIDE_RESULT can't read the key back. A check value is needed to satisfy "reads every stored value back" without echoing the key.
- Blocks: Ride setup, persistence tests, and key-handling review.
- Options: A key check value in RIDE_RESULT. Refuse JOIN_RIDE during an active ride unless END_RIDE came first. Store the ride so that a reboot rejoins it.

Status (v0.3): ANSWERED — spec 7.4 Ride, JOIN_RIDE rules (name 1–24, 16-byte key, tier 1–4; refused with RIDE_ACTIVE, SOS_ACTIVE, or BAD_FIELD for an all-zero or wrong-length key or bad tier; one all-or-nothing write that survives reboot; RIDE_STATE reads back what was stored, with a key check value).

**Q28. END_RIDE: what is the "known default state"?**
- Spec section: END_RIDE. RIDE-003, SOS-003.
- Unclear:
  - Which things reset: ride key and name, tier, display name, presence on/off, repeater mode, GPS source.
  - What happens to queued transmissions, stored messages, stored track points, and any running SOS (cancelled?).
  - Whether other riders are told. No over-the-air message for this exists.
  - It also isn't stated whether RIDE-003's default state includes the presence default, which is a pending product decision.
- Blocks: END_RIDE implementation and its test.
- Options: A table: each stored item and its value after End Ride. State that End Ride affects only the local radio.

Status (v0.3): ANSWERED — spec 6.7 (table of what is erased, ended or kept; affects only the local radio). Items the table does not list (rider list, rider tag clash counter, pending fill state, paired phones, ID counters): Q85. End Ride during an SOS: Q83.

**Q29. Ride QR code format.**
- Spec section: Ride QR code. RIDE-001.
- Unclear: Binary or text, the encoding of the key, name length limit, the format-version rules, and an integrity check. A key size that the QR code can hold needs to fit.
- Blocks: Phone-to-radio provisioning tests and the app. Also blocks the key size choice in Q11.
- Options: State a byte layout and a maximum total size. Add a checksum.

Status (v0.3): ANSWERED — spec 8 (text form trailbuddy://ride/ plus Base64url; byte layout; 45-byte maximum; CRC-16/CCITT-FALSE; refusal rules). The byte order of the CRC is not stated: Q98.

## D. Messaging

**Q30. SEND_TEXT limits and addressing.**
- Spec section: SEND_TEXT. MSG-001.
- Unclear:
  - Maximum text length (listed as open) and character encoding.
  - Whether long text is split across packets or refused.
  - "Recipient (ride or rider ID)": no requirement covers a message to one rider (MSG-001 is group only), though the envelope and POLICY §11 require one-rider addressing.
  - Who assigns the message ID, its size and scope.
- Blocks: The text byte budget and unicast tests.
- Options: A fixed maximum (e.g. a single over-the-air packet). Decide whether unicast text is in v1. If so, add a requirement ID for it.

Status (v0.3): ANSWERED — spec 3 and 6.3 (whole ride only; 150 bytes of UTF-8; longer text refused, not split), 5 (16-bit message ID assigned by the sending radio), 7.4 (SEND_TEXT 1–150 bytes). Which error code a too-long text gets: Q95.

**Q31. What does "delivered" mean?**
- Spec section: SEND_RESULT; Delivered over the air. MSG-002.
- Unclear:
  - In a ride of N riders, does "delivered" mean one rider confirmed, all, or a set number?
  - Must every receiver send Delivered for a group message? That is N−1 acks per text.
  - Does a radio send Delivered when it holds the message but the phone is disconnected, or only after the phone has it?
  - Retries, timeouts and what makes a message "failed", and the failure reasons.
  - Whether SEND_RESULT reports each state change as its own message.
- Blocks: The ack protocol, ack airtime, and the SEND_RESULT tests.
- Options: Delivered means at least one other rider confirmed. Or report a count of confirmations. Add a retry count and timeout.

Status (v0.3): ANSWERED — spec 3 and 6.3 (sender sees a count; every receiving radio sends Delivered after 0–2 s whether or not its phone is connected; QUEUED, SENT, DELIVERED, FAILED; resend after 20 s up to 3 times; FAILED with NO_CONFIRMATION). Gaps: what the "of 4" counts (Q60); how a receiver handles a re-sent text (Q74).

**Q32. Missed messages and events.**
- Spec section: GET_MISSED, TEXT_RECEIVED, STATUS ("messages waiting"). Open question on storage.
- Unclear:
  - GET_MISSED has no reply message. If the reply is a series of TEXT_RECEIVED, nothing marks the end or ties it to the request.
  - "Since (time or last message ID)": one must be chosen.
  - Storage size, retention time and eviction are listed as open.
  - Only texts can be fetched. An SOS, a reply, or a rider update that arrives while the phone is away has no way to reach the app (SOS-004 relies on delivery).
- Blocks: Storage design, reconnect tests, and the safety case for SOS received while the phone is disconnected.
- Options: A defined reply with an end marker. Store SOS events separately and never evict them. Allow GET_MISSED to cover all event kinds.

Status (v0.3): ANSWERED — spec 6.3 Stored events (32-bit event numbers; last 200 events; SOS events kept until the SOS has ended plus 24 hours; GET_MISSED replays everything after an event number, each message carrying the request ID, the last one flagged) and 7.4 Messaging. Event details: Q86.

## E. Position and Fill on Return

**Q33. Location source, schedule, and heading and speed.**
- Spec section: SHARE_MY_LOCATION, LOCATION_UPDATE; Presence. POS-001, POS-002.
- Unclear:
  - Who decides when a position is broadcast: the radio on the tier interval, or the app pushing each update?
  - If the radio GPS is the source, why does the app send a "source" field?
  - SHARE_MY_LOCATION has no heading or speed, and presence requires both. With phone GPS, presence can't be built.
  - Whether the radio keeps broadcasting the last position after the phone disconnects, and for how long.
  - Behaviour on boards with no GPS.
  - The switch rule between phone and radio GPS is an open question.
  - There is no reply to SHARE_MY_LOCATION.
- Blocks: Build step 4 and presence.
- Options: The radio owns the schedule and uses the latest position from either source. Add heading and speed (optional) to SHARE_MY_LOCATION. State "phone preferred when connected and fresh, otherwise radio GPS".

Status (v0.3): ANSWERED — spec 6.1 (the radio owns the schedule; radio GPS first, phone fallback if fresh; SHARE_MY_LOCATION carries optional heading and speed and has no reply; the radio keeps sending after a disconnect; boards without GPS use the phone). Open follow-ups: Q57, Q61, Q78.

**Q34. GPS duty cycle (POS-003) is not specified, and it conflicts with NEAR-003.**
- Spec section: not present. POS-003, NEAR-003, NEAR-001.
- Unclear:
  - GPS duty follows the tier, with no values.
  - Presence must be sent often while moving, which needs frequent fixes whatever the tier. A radio with no ride has no tier at all.
- Blocks: Battery targets and presence frequency.
- Options: A GPS duty table per tier plus a separate rule when presence is on. Or fix the GPS rate to the presence rate while moving, and the tier rate otherwise.

Status (v0.3): ANSWERED — spec 6.1 GPS duty (continuous while moving or presence is on; once a minute after 5 minutes stopped with presence off; moving = 5 km/h for 10 s, stopped = below 3 km/h for 30 s). It follows movement and presence, not tier, which removes the conflict with NEAR-003. Conflict with the position freshness rule: Q57.

**Q35. Units and precision for numbers.**
- Spec section: all messages. Freeze checklist.
- Unclear: Latitude and longitude precision, accuracy (m), speed (km/h or mph, flagged by the spec itself), heading (range and "unknown" at standstill), signal strength (which measure, which unit), battery % (integer range, "unknown"), channel-busy % resolution, and time (see Q6).
- Blocks: Every field's type and size, and every byte budget.
- Options: A units table for the whole spec. Use one measurement system on the wire and convert in the app.

Status (v0.3): ANSWERED — spec 2.6 (units table with types and unknown values). Rounding when converting to wire units: Q67.

**Q36. Rider hello and MSG-003.**
- Spec section: Rider hello; GET_RIDERS. MSG-003.
- Unclear:
  - How radios that join later learn names. Hello has no "ask" form, so it must be repeated.
  - The repeat rule: when, how often, and the airtime cost.
  - Display name length and allowed characters.
  - What "short ID" is (see Q7).
  - Name clashes between riders.
- Blocks: Hello behaviour, the riders list, and airtime budget.
- Options: Send hello on join and on name change, then at a slow fixed interval. Or add a hello request that new radios send.

Status (v0.3): ANSWERED — spec 6.2 (hello on join, on name change, every 10 minutes and on request; hello request at most once a minute, replies after 0–5 s; name 1–24 bytes of UTF-8; default "Rider " + short ID; short ID shown beside duplicate names). Gaps: hello on anonymous ID change (Q76); hello name minimum (Q62).

**Q37. RIDERS_LIST and RIDER_UPDATE.**
- Spec section: Riders rows. MSG-003.
- Unclear:
  - "Battery % if shared": no over-the-air message carries battery, and no setting controls sharing.
  - "Signal strength" for a relayed packet describes the last hop, not the rider.
  - Maximum list size, and paging.
  - What triggers RIDER_UPDATE (any new position? name change only?).
  - When a rider is removed as gone.
- Blocks: The riders list and its test, and over-the-air battery sharing (not covered at all).
- Options: Drop the battery field from v1, or add it to a defined over-the-air message with an opt-in setting. Define "last heard" as time since any packet from that rider.

Status (v0.3): ANSWERED — spec 3 (no battery sharing in version 1) and 7.4 Riders (list of at most 30; signal strength only for direct frames; RIDER_UPDATE triggers with a 10 s rate limit; riders never removed; last heard defined). List size over the phone link: Q93; tag clashes: Q72.

**Q38. Fill on Return rules are missing.**
- Spec section: GET_TRACK, TRACK_POINTS; Fill ask, Fill answer. FILL-001.
- Unclear:
  - FILL-001 says the spec "restates the full five rules". The spec does not, so the rules are unknown. These include what triggers a request, who answers, retries, and holding back when the channel is busy.
  - GET_TRACK could read local history only, or could cause an over-the-air Fill ask. It doesn't say.
  - "Rejoin" is not defined.
  - Batch size, "more to come" handling, ordering, duplicates, and the history each radio keeps (open question).
  - How filled positions reach the app: as TRACK_POINTS only, or also as LOCATION_UPDATE.
- Blocks: Build step 8 completely. Also storage sizing and the "return-to-filled-track time" target.
- Options: The spec team supplies the five rules as behaviour. Define rejoin as a stated condition, for example "own radio hears a ride member after a silence of at least a stated time".

Status (v0.3): PARTLY ANSWERED — spec 6.5 gives the five rules, defines rejoin, says GET_TRACK reads local history only, and says filled points reach the app as TRACK_POINTS only. Missing: the order of points within TRACK_POINTS; handling of a point received twice (answers are sent twice, or the point is already stored); whether received fill points are added to this radio's stored history; how a receiver knows an answer is complete (Q84).

## F. SOS

**Q39. SOS sending and repeating.**
- Spec section: SEND_SOS, SOS_STATUS; SOS over the air. SOS-001, SOS-003.
- Unclear:
  - Who assigns the alert ID, and its size.
  - The schedule: "twice, 10 seconds apart, then every 5 minutes". The start of the interval, jitter allowed, and whether a missed slot is caught up are not stated.
  - SEND_SOS gives one position. Repeats could carry that position or fresh ones, and a moving rider gets stale data otherwise.
  - Whether a second SEND_SOS replaces or adds to an active one.
  - What happens to a running SOS when the phone disconnects.
  - When SOS_STATUS is sent: on change, on request, per repeat.
  - SOS_STATUS has only "sending, repeating, cancelled" with no state for "ended by I'm OK".
- Blocks: Build step 6.
- Options: Repeats use the latest position from the radio and keep the original send time. One active SOS per radio. The radio repeats independently of the phone.

Status (v0.3): PARTLY ANSWERED — spec 5 and 6.4 (the radio assigns a 16-bit alert ID; sent at once, 10 s later, then every 5 minutes with up to 5 s jitter; missed slots not caught up; repeats carry the latest position and the original send time; a second SEND_SOS only updates the position; repeats continue without the phone; SOS_STATUS has ENDED_OK and ENDED_LEFT_RIDE). Missing: when SOS_STATUS is sent unsolicited and what SEND_SOS is answered with (Q82); whether the jitter applies to the 10 s repeat; what an active SOS does across a reboot (Q81).

**Q40. SOS replies and who may cancel.**
- Spec section: SOS_REPLY, SOS_REPLY_RECEIVED, SOS_ENDED. SOS-004.
- Unclear:
  - "I'm OK" is for the rider who raised the SOS, and the spec doesn't say how the radio checks this. Every ride member holds the same key and could send it for another rider.
  - What a "Copy that" changes. It is only a notice, and it doesn't stop repeats.
  - What SOS_ENDED carries and who gets it.
  - Whether a radio that missed the cancel keeps relaying or showing an SOS.
- Blocks: SOS-004 tests and the protocol tests for replays and unknown senders.
- Options: The radio accepts "I'm OK" only when the sender ID matches the alert's rider ID. The shared-key limit is documented as accepted or raised to the security review.

Status (v0.3): ANSWERED — spec 6.4 (I'm OK is accepted only when the sender's rider tag matches the alert owner; Copy that is informational; radios that miss the end keep showing the SOS with its last-heard time), 7.4 SOS (SOS_ENDED and SOS_REPLY_RECEIVED fields) and 14 (forgery by a ride member goes to the security review).

**Q41. SOS received by the app.**
- Spec section: SOS_RECEIVED. SOS-001.
- Unclear: Whether each repeat reaches the app as a new message (the spec says "or repeated one"), or the radio reports each alert once and then updates. What the radio does on its own, such as the buzzer on boards that have one, is not stated.
- Blocks: App behaviour and the "phone" tests.
- Options: Report every repeat and let the app de-duplicate by alert ID. Or report once plus a "still active" notice.

Status (v0.3): ANSWERED — spec 6.4 To the app (SOS_RECEIVED for a new alert and for every repeat, the app de-duplicates by alert; SOS_ENDED when an end is heard; the buzzer sounds for a new alert on boards that have one). Buzzer behaviour: Q90.

## G. Nearby riders (presence)

**Q42. Presence intervals (NEAR-003).**
- Spec section: Open questions (intervals); Presence. NEAR-003.
- Unclear:
  - Interval while moving and while stopped, and the speed at which a radio counts as moving (with hysteresis).
  - Whether intervals depend on tier. A radio with no ride has none.
  - Any randomization.
  - The airtime budget they must fit, which depends on Q1 and Q2.
- Blocks: Build step 7 and the airtime budget.
- Options: The spec team sets two intervals and a movement threshold, with a worked airtime calculation.

Status (v0.3): ANSWERED — spec 6.6 (presence every 10 s while moving and every 60 s while stopped, ±1 s jitter, independent of tier and ride), 6.1 (moving and stopped thresholds) and 2.5 (airtime check). Edge cases of the moving state: Q77.

**Q43. Presence relay, duplicates and overlap with the ride.**
- Spec section: Presence; Envelope. MESH-001, MESH-002, NEAR-001, NEAR-004.
- Unclear:
  - Whether presence is relayed, and its hop limit. MESH-001 covers ride traffic only.
  - Whether presence is subject to duplicate suppression.
  - Whether a ride member's presence reaches the app as a NEARBY_RIDER, so the same person shows both as a ride rider and as an anonymous nearby rider.
  - Whether a radio's own presence is filtered.
- Blocks: The airtime budget, the mesh simulation, and nearby-rider tests.
- Options: Presence is one hop only and is not relayed. Or a small hop limit. The radio hides entries matching its own anonymous ID.

Status (v0.3): ANSWERED — spec 4.2, 4.6 and 6.6 (presence travels one hop and is never relayed; a radio ignores its own anonymous ID; ride members' presence is hidden using the anonymous IDs in their hellos). Keeping the hide list current: Q76.

**Q44. Fake presence limits.**
- Spec section: Open questions (fake presence). POLICY §12.
- Unclear: Presence has no ride key, so anyone can send it. Nothing limits per-ID rate, the number of IDs tracked, implausible movement, or the effect on approaching-rider warnings.
- Blocks: Robustness and flood tests, and the memory bound for the nearby table.
- Options:
  - Rate and table limits with eviction.
  - Plausibility checks.
  - A lightweight authentication scheme, if it can be done without breaking anonymity.
  - Accept the risk with a stated limit.

Status (v0.3): ANSWERED — spec 6.6 (at most 32 anonymous IDs with the oldest evicted; a report implying over 200 km/h is ignored; one report per ID per 2 s; the remaining risk is accepted and documented). The time basis for the speed test: Q78.

**Q45. Presence on/off, defaults, and radio-side states.**
- Spec section: SETTINGS; Open questions. NEAR-005, NEAR-006, NEAR-008.
- Unclear:
  - Default on or off, whether it switches on during the rider's own SOS, and its value after End Ride.
  - Whether presence off also stops receiving.
  - NEAR-005 and NEAR-008 are app requirements, and the spec doesn't say whether firmware has any role. The Pocket V3 buzzer is mentioned.
  - An empty NEARBY_LIST can't be told apart from "presence receive off" or "no GPS".
- Blocks: Settings defaults and the app's empty-view behaviour.
- Options: The product owner decides the default. Add a "presence receive active" status field. State that alerts are app-only in v1, or add a radio alert message.

Status (v0.3): ANSWERED — spec 3 and 6.6 (on by default, forced on during the rider's own SOS, on again after End Ride; turning it off stops sending only, receiving is always on; approaching-rider alerts are app-only in version 1) and 7.4 Status and Nearby (STATUS reports whether the radio has a position and whether presence is on, so the app can word an empty view).

**Q46. Presence precision and privacy.**
- Spec section: Presence; NEARBY_RIDER. NEAR-002.
- Unclear: Position precision sent to anyone, heading when speed is near zero, and whether position is rounded.
- Blocks: Presence byte budget and a privacy review.
- Options: Coarsen position to a stated precision. Mark heading "invalid" below a speed.

Status (v0.3): ANSWERED — spec 4.2 (position rounded to 0.0001°, about 11 m) and 2.6 (heading is 255 below 3 km/h).

**Q47. NEARBY_RIDER, GET_NEARBY and NEARBY_LIST.**
- Spec section: Nearby rows. NEAR-004.
- Unclear:
  - NEAR-004 says every presence signal goes to the app. In a crowded area this could overload the BLE link, and the spec gives no rate rule.
  - How many anonymous IDs the radio keeps, for how long, and what is evicted.
  - NEARBY_LIST size, paging, and how it is tied to its request.
  - "Since (time)" depends on Q6.
- Blocks: Nearby storage and BLE throughput design.
- Options: A per-ID report rate cap toward the app. A fixed table size with oldest-out eviction.

Status (v0.3): PARTLY ANSWERED — spec 6.6 (at most one NEARBY_RIDER per anonymous ID every 5 s; table of 32 IDs with the oldest evicted) and 7.4 Nearby (GET_NEARBY answered by NEARBY_LIST with the request ID). Missing: how long an entry stays in the table and in NEARBY_LIST before it expires; how many entries go in one NEARBY_LIST part, since 32 entries can exceed the 1024-byte message limit (Q93).

**Q48. Combining ride position and presence (open question).**
- Spec section: Open questions. NEAR-002.
- Unclear: One signal with an encrypted ride part would save airtime, but NEAR-002 says presence "never carries … ride details". It's unclear whether an encrypted ride blob counts. This choice changes the envelope.
- Blocks: Envelope and message list.
- Options: Keep two separate signals in v1. Or combine, with an explicit decision that the blob is allowed.

Status (v0.3): ANSWERED — spec 6.6 last bullet (presence and ride position stay separate signals in version 1).

## H. Settings, repeater, updates, diagnostics

**Q49. Settings rules.**
- Spec section: GET_SETTINGS, SETTINGS, CHANGE_SETTINGS, SETTINGS_RESULT. BLE-002.
- Unclear:
  - Whether setting names are text or numbered identifiers, and the type and range of each.
  - What an unknown setting does to the whole write.
  - SETTINGS_RESULT reports "every changed setting". BLE-002 says to read back "every stored value". It's unclear whether unchanged requested values, or all settings, are returned.
  - Default values for each setting (display name, presence, repeater, GPS source).
  - What happens on corrupted storage.
  - Whether ride data is inside the same atomic write.
- Blocks: Build step 9, power-loss tests, and boot safety.
- Options: A settings table with name, type, range, default. State that corrupted storage resets to defaults and reports it.

Status (v0.3): ANSWERED — spec 7.4 Settings (numbered IDs, types and defaults; all-or-nothing change; an unknown ID or invalid value changes nothing and returns BAD_FIELD; an equal value is not rewritten; SETTINGS_RESULT returns every setting as stored; damaged settings reset to defaults and are reported as SETTINGS_RESET; ride data is a separate record). Interplay and reporting details: Q87, Q88.

**Q50. Display name rules.**
- Spec section: SETTINGS; RIDERS_LIST; Rider hello. MSG-003.
- Unclear: Maximum length, allowed characters, empty name, and whether a name is required before joining.
- Blocks: Byte budget for hello and the settings write.
- Options: A byte limit that fits one hello, UTF-8 or a restricted set, and a default name derived from the short ID.

Status (v0.3): ANSWERED — spec 6.2 and 7.4 Settings (1–24 bytes of UTF-8, no control characters, default "Rider " + short ID). Whether the radio checks UTF-8 validity: Q67.

**Q51. Repeater mode.**
- Spec section: SETTINGS; REPEATER_STATUS. RPT-001.
- Unclear:
  - What a repeater relays (see Q12), whether it also sends its own position or presence, and whether it accepts BLE.
  - How repeater mode is turned on and how it is turned off with no phone.
  - REPEATER_STATUS has no trigger. It is "when repeater mode is on", yet it can only reach an app that is connected.
  - The units and counter behaviour of "messages relayed" and "uptime".
- Blocks: Repeater test and its place in the build order (no step names it).
- Options: Define repeater behaviour as a list of what it does and does not send. Make REPEATER_STATUS a reply to a request or a periodic update with a stated interval. Add a build step for it.

Status (v0.3): ANSWERED — spec 6.8 (the phone joins the radio to a ride and turns repeater mode on; ride and mode survive reboot; it relays its ride's frames, sends a hello every 10 minutes with the repeater flag, sends no position or presence, and accepts a phone; REPEATER_STATUS is sent with STATUS) and 7.4. Counting and answering details: Q89.

**Q52. Firmware update path.**
- Spec section: START_UPDATE, UPDATE_CHUNK, UPDATE_PROGRESS; open question on update path. DFU-001. POLICY §13, §15.
- Unclear:
  - The spec lists the messages, and its own open question asks whether updates go through them or through the board's update mechanism. POLICY §15 names each board's UF2 bootloader. Whether that bootloader checks signatures is not in these files.
  - Signature scheme, key handling, downgrade rules.
  - Image targeting per board.
  - Chunk size and flow control. There is no per-chunk acknowledgement.
  - There is no abort message, and no rule on resuming.
  - Battery minimum and behaviour during an SOS.
  - How the app learns the result after the reboot that follows "installed" or a rollback.
- Blocks: Build step 10 and the "failed update" hardware test.
- Options: Choose the path first. Then define the update as a state machine in words.

Status (v0.3): ANSWERED — spec 7.4 Update path (updates arrive through the phone link into a two-slot bootloader that checks an Ed25519 signature and rolls back after 60 s without confirmation; UF2 bootloader kept for USB recovery; UPDATE_REJECTED, LOW_BATTERY below 40% and SOS_ACTIVE refusals; ABORT_UPDATE; resume from bytes received; result in the next WELCOME). Details: Q97.

**Q53. Diagnostics.**
- Spec section: GET_DIAGNOSTICS, DIAGNOSTICS, CLEAR_DIAGNOSTICS. BLE-003. POLICY §16.
- Unclear:
  - POLICY says diagnostics are compiled out of release builds. The spec lists diagnostics messages, and the definition of done says the app exchanges every message in the spec. Are the diagnostics messages in release firmware?
  - BLE-003 covers battery, firmware version and link status only, so these three messages cite a requirement that doesn't cover them.
  - "Since (time)" on cumulative counters has no clear meaning.
  - Which message kinds are counted, which drop reasons exist, counter width and rollover.
  - The format of "channel busy history".
  - CLEAR_DIAGNOSTICS has no reply.
- Blocks: Diagnostics implementation, the field tests that need it, and the release definition of done.
- Options: Add a requirement ID, or remove the messages from release builds and say so in the spec. Define the counters in a table.

Status (v0.3): ANSWERED — spec 10 (DIAG-001; counters stay in release firmware, detailed logging is removed) and 7.4 (counter list, 32-bit wrap, CLEAR_DIAGNOSTICS replies with DIAGNOSTICS). Counter definitions: Q96.

## I. Hardware

**Q54. Lead board and board sources.**
- Spec section: Hardware under test; open questions on lead board and per-board features. POLICY §8, §14, §16. CLAUDE.md line 9.
- Unclear:
  - The lead board isn't named. POLICY says the product owner names it. The spec's open question mentions the T114 V2 as "the current plan's choice" and a plan that is not in these files.
  - CLAUDE.md says board details come only from the manufacturer's datasheets, schematics and board support. POLICY §14 also allows Zephyr's RAK4631 board definition. It isn't stated whether that definition counts as "manufacturer's board support".
  - The RAK19026 schematic is not in the repo.
  - Which board features v1 uses (screen, buttons, buzzer) is open.
- Blocks: Every build step, since all run on the lead board first, and the hardware profile in `docs/hardware/`.
- Options: The product owner names the lead board. The spec team states explicitly whether the Zephyr RAK4631 definition may be used. Vendor documents are added to the repo.

Status (v0.3): PARTLY ANSWERED — spec 9 and POLICY §8, §14 (lead board is the WisMesh Pocket V3; Zephyr's RAK4631 board definition may be the starting point; version 1 uses the buzzer and the power button, no screens). Missing: the RAK19026 / Pocket V3 documentation is not yet supplied (spec 14, fourth item), so pins, GNSS, buzzer, button and battery measurement cannot be written into the board profile. This blocks every step that runs on hardware.

Status (hardware docs added 22 October 2026): PARTLY ANSWERED — `docs/hardware/` now holds the RAK19026 VC WisMesh Base Board datasheet and the RAK19007 WisBlock Base Board 2nd Gen datasheet. The RAK19026 VC datasheet covers the base board: RAK4630 module (nRF52840 and SX1262), u-blox ZOE-M8Q GNSS, accelerometer, user and reset buttons, LEDs, battery connector and charger, and schematics (pp. 17–19); the pin-level readings from the schematic images are still to be confirmed at full size. Still missing: any document for the WisMesh Pocket V3 itself (neither file names it or says it uses the RAK19026 VC); a buzzer (in neither file); a firmware-readable power button (only a battery slide switch and a user button are described); and the RAK4630 module documentation for the radio wiring. The RAK19007 is a different base board, not the lead board. Not marked answered.

## J. Process

**Q55. Freeze checklist and traceability.**
- Spec section: "Before this spec can be frozen". REQUIREMENTS intro.
- Unclear:
  - None of the 49 messages meets required/optional, size, units, or sample values.
  - REQUIREMENTS says every protocol decision cites a requirement, a rationale and a test. The spec has no rationale or test column.
- Blocks: Freeze, and TEST-MATRIX.md.
- Options: One table per message with field, type, unit, required or optional, maximum size and sample value. Add a test reference per message.

Status (v0.3): PARTLY ANSWERED — spec 7.4 (R/O marking, units from 2.6, list limits, a requirement ID per message), 4 and 5 (byte sizes) and 11 (samples). Missing: samples for 33 of the 43 phone-link messages (promised for v0.4, spec 14); maximum encoded sizes for phone-link messages; the rationale and test reference that REQUIREMENTS says every protocol decision carries. Note: "None of the 49 messages" in Q55 should read 40 for v0.2 (as the Review basis says); v0.3 has 43, which matches the table and spec 15.

**Q56. Test vectors and review.**
- Spec section: not present. POLICY §12, §17 step 2, §19 (`spec/`).
- Unclear: Build step 2 needs "all spec test vectors". The spec doesn't say who produces them, from what inputs, or how the clean team can use them without writing them from its own implementation. It also doesn't say when the independent design review happens.
- Blocks: Build step 2 and CI conformance.
- Options: The spec team supplies vectors with the frozen spec. The review is completed before freeze.

Status (v0.3): PARTLY ANSWERED — spec 11 (the spec team supplies exact vectors for key derivation, a position frame, a presence frame and the QR code; phone-link vectors are recorded by the clean team in spec/vectors/ and checked by the spec team; 11.6 requires the independent security review before freeze). Missing: vectors for the other eight ride kinds (hello, text, SOS and fill in particular), for a relayed frame (hops changed, same tag), for rider tag clash counter 1, for the short ID, and for the replay and duplicate windows. I did not recalculate any vector in this review; their layouts and lengths match spec 4, 5 and 8.

---

## Status of the spec's own open questions

| Spec open question | Where it is raised here |
| --- | --- |
| Complete feature list (product owner) | Not raised; a product decision. Q26, Q30, Q32 and Q37 each point to a gap. |
| Presence default on/off, SOS override | Q45 |
| What "approaching" means | App-only; no firmware blocker. Needs speed and heading from Q33 and Q35. |
| Common channel for presence | Q3 |
| Combine position and presence | Q48 |
| Fake presence | Q44 |
| Intervals per tier | Q10, Q42 |
| Maximum text length | Q30 |
| Phone or radio GPS | Q33 |
| Missed message and track storage | Q32, Q38 |
| Update path | Q52 |
| Lead board; RAK19026; per-board features | Q54 |
| IP attorney | Out of scope for this review. |

---

## New questions from the v0.3 review

Raised by: review of `docs/SPEC-v0.3.md` against `docs/clean-room/POLICY.md` v0.3 and `docs/clean-room/REQUIREMENTS.md` v0.3, 22 October 2026.
Sources used: `CLAUDE.md`, the three v0.3 documents and this file. Nothing else.

**Cross-check of the three documents.**
- Matching: the four tiers (hops, intervals), 30 riders, one ride per radio, the 176-byte frame, the SOS schedule, DIAG-001, the lead board and the repeater build step agree across all three. Every change listed in spec Section 12 is present in REQUIREMENTS and POLICY v0.3. The phone link has 43 messages, as the tables and spec 15 say.
- Not matching: Q62 lists the wording that differs.
- Not checked: I did not recalculate the test vectors or the airtime table. For the vectors, the layouts and lengths in 11.2 to 11.4 agree with Sections 4, 5 and 8.

**Requirement coverage.** All 35 requirement IDs in REQUIREMENTS v0.3 have a home in the spec. These are covered only in part:

| Requirement | What is missing | Question |
| --- | --- | --- |
| POS-001, POS-003 | The fresh-fix rule cannot work with the once-a-minute GPS duty | Q57 |
| SOS-002 | "Never dropped" against a queue of 8 | Q59 |
| SOS-001 | No rule when there is no position, and no SOS without a ride | Q79, Q80 |
| MSG-002 | What "3 of 4" counts; re-sent texts | Q60, Q74 |
| FILL-001, NFR-001 | The 60 s target cannot be met with the fill rates | Q58 |
| NEAR-002, NEAR-004 | Spec behaviour differs from the requirement wording | Q62 |
| MESH-004 | Channel-busy measurement method | Q65 |
| NFR-001 (range, battery), REG-001 | Targets lack numbers or conditions (Q2); the lab has not confirmed 2.1 | Q2, spec 14 |
| NEAR-005, NEAR-008 | App only; the radio supplies what is needed (STATUS, NEARBY_LIST) | none |

### A. Contradictions inside v0.3 or between the three documents

**Q57. The "fresh fix" rule for positions cannot be met when the GPS samples once a minute.**
- Spec section: 6.1 (Source, GPS duty), 2.5, 6.5 rule 1. POS-001, POS-003.
- Unclear: A position is sent only if the radio's own fix is no older than the tier interval (10, 45, 120 or 90 s). After 5 minutes stopped with presence off, the GPS samples once a minute, so the fix is usually older than 10 s or 45 s. Read literally, no position is sent, and STATUS reports "no position". The spec does not say whether a stopped rider with presence off should keep sending a position at the tier interval, send one per GPS sample, or go silent.
- Blocks: POS-001 and POS-003 tests, the ride contact count, and Fill on Return rule 1: a rider who was silent while stopped and then moves more than 50 m looks like a returning rider and triggers fill asks for no reason.
- Options:
  - Send the latest fix, with its true fix time, at the tier interval (receivers already judge freshness by fix time).
  - Define "fresh" as no older than the GPS sampling period.
  - Send nothing while stopped, and exempt that silence from the Fill on Return gap rule.

**Q58. Fill on Return cannot meet "a filled track within 60 s of a rider's return" for any but short gaps.**
- Spec section: 6.5 rules 2 to 5, 5 (Fill ask, Fill answer), 3 (Targets). NFR-001, FILL-001.
- Unclear:
  - Rule 5 allows one fill frame every 10 s per radio. A fill answer holds at most 12 points. A 30-minute gap at one point per 10 s is about 180 points, which is 15 frames, or at least 150 s for one pass. In 60 s a radio can send about 6 frames, roughly 12 minutes of track, before the outline step, the second send of each answer and any asks.
  - Every answer is sent twice, 10 s apart, but the same one-frame-per-10-s limit applies. It is not said which comes first, the second copy of part 1 or part 2.
  - Asks name one rider each. A returning radio has a gap for every other rider, so it sends one ask per rider, each after a 2 to 8 s wait, and the asks use the same one-frame-per-10-s allowance.
  - "Filled track" and "return" are not defined: the whole 30 minutes, or the outline, and from which moment.
- Blocks: the FILL-001 and NFR-001 field test and the pass criteria for build step 9. The target cannot be met as written.
- Options:
  - Define "filled" as the outline within 60 s, with detail afterwards.
  - Raise the fill frame rate when channel busy is low.
  - State the maximum gap the 60 s target covers.
  - Let one ask cover several riders.
  - The product owner chooses.

**Q59. SOS "is never dropped" conflicts with a queue of 8 entries.**
- Spec section: 4.5 (class 0). SOS-002.
- Unclear:
  - Class 0 holds SOS and SOS replies, 8 entries, "never dropped; a newer frame for the same alert replaces an older queued one". In a ride of 30, several riders can have an SOS, every ride member can send Copy that, and relays of other riders' SOS frames also wait. More than 8 distinct entries can be queued, and the spec does not say what happens to the ninth.
  - "Same alert" does not separate replies from different riders. Replacing one rider's Copy that with another's would lose a reply.
  - It is not said whether the radio's own SOS repeats and relayed SOS frames share this queue.
- Blocks: the SOS-first and never-dropped tests under load (POLICY §18, §20).
- Options:
  - Size class 0 for the worst case (30 alerts plus replies) so it can never fill.
  - Define an eviction order inside class 0 (for example the oldest reply first, never an SOS frame).
  - Key replacement on alert, sender and reply value.

**Q60. "Heard recently" has three different windows, and it is unclear who is counted.**
- Spec section: 6.3 (Sender's side), 7.4 Status (ride contact), 7.4 Riders (in contact). MSG-002.
- Unclear:
  - Delivered counts are "out of the ride members heard in the last 10 minutes". Ride contact counts members heard in the last 2 × the position interval. RIDER_UPDATE reports contact lost after 3 × the interval. The same moment can show 3 in contact in STATUS and 4 in RIDERS_LIST.
  - For "Delivered to 3 of 4" (MSG-002, and the SEND_RESULT sample in 11.5: 3 confirmations, 4 ride members): does "4" include the sender? In a ride of 4, only 3 others can confirm.
  - Do repeaters count (see Q89)? Does a rider silent for 11 minutes drop out of the total?
- Blocks: SEND_RESULT, STATUS and RIDER_UPDATE tests, and the app's wording.
- Options: One definition of "in contact" used everywhere. State the total as "others in contact, not counting the sender", or "all known ride members".

**Q61. The GPS source preference setting contradicts the fixed source rule.**
- Spec section: 3 (GPS source), 6.1 (Source, decision), 7.4 Settings (setting 4), 6.7.
- Unclear: Sections 3 and 6.1 give one rule: radio GPS first, phone as fallback. Setting 4 offers RADIO_FIRST or PHONE_FIRST, and no section says what PHONE_FIRST does (when the phone's fix is fresh enough, what happens to presence).
- Blocks: settings tests (build step 10) and position source tests (build step 4).
- Options: Describe PHONE_FIRST as the mirror of the 6.1 rule. Or remove setting 4 from version 1.

**Q62. Requirement and policy wording that v0.3 no longer matches.**
- Spec section: REQUIREMENTS NEAR-002, NEAR-004. POLICY §10, §18. Spec 0, 5, 6.2, 12.
- Unclear:
  - NEAR-002 says presence carries "GPS fix time". Spec 4.2 carries a fix age.
  - NEAR-004 says the radio passes "every presence signal it hears" to the app. Spec 6.6 hides ride members' presence, ignores implausible or too-frequent reports and sends at most one NEARBY_RIDER per ID every 5 s. Spec Section 12 does not list NEAR-004.
  - POLICY §10 (Channel access) and §18 (Radio / channel access) talk about airtime and duty-cycle limits being held. Spec 2.1 says there is no duty-cycle limit and the 30% figure is Trail Buddy's own budget (Q64, Q66).
  - Spec Section 0 says updated copies of REQUIREMENTS and POLICY "will follow" and the spec wins until then. The v0.3 copies exist, so it is unclear which text wins on the points above.
  - Spec Section 5 allows a hello name of 0 to 24 bytes. Spec 6.2 and the settings table require 1 to 24.
- Blocks: TEST-MATRIX.md (every change maps to a requirement ID). A test for NEAR-002 or NEAR-004 would test wording the spec contradicts.
- Options: The spec team states, for each point, whether the requirement text or the spec text wins, and issues the amended wording.

**Facts gathered for the policy owner (22 October 2026)**

Facts only. No answer is proposed, and Q62 stays open until the policy owner decides. Quotes are from `REQUIREMENTS.md` v0.3, `POLICY.md` v0.3 and `SPEC-v0.3.md`. The hello name length (0–24 against 1–24 bytes) is left out because it belongs to the spec team.

| Topic | REQUIREMENTS or POLICY wording | SPEC-v0.3 wording | What's different |
| --- | --- | --- | --- |
| Fix time or fix age | NEAR-002: "The presence signal carries position, heading, speed and GPS fix time, plus an anonymous rider ID that changes at least daily." | 2.6: "Fix age (presence) \| unsigned 8-bit \| seconds, 254 = 254 s or older". 2.2: "Presence carries a fix **age** in seconds, not a time, so it works without a clock." 4.2 has a "Fix age" byte. | The requirement says the time of the fix. The spec sends how many seconds old the fix is. |
| Every presence signal or only some | NEAR-004: "The radio passes every presence signal it hears to the app." | 6.6: "a radio hides presence from its own ride's members"; "It ignores a report that implies a speed over 200 km/h"; "It accepts at most one report per ID every 2 s"; "at most one NEARBY_RIDER per anonymous ID every 5 s"; "a radio ignores presence frames carrying its own anonymous ID". NEAR-004 is not listed in spec Section 12. | The requirement says all of it reaches the app. The spec removes several kinds first. |
| Table of 32 | NEAR-004 (as above) | 6.6: "The radio tracks at most 32 anonymous IDs and evicts the oldest." 7.4: NEARBY_LIST is "list (at most 32)". | A further limit on what can reach the app under NEAR-004. |
| Precision of position | NEAR-002: presence carries "position" | 4.2: "Latitude, rounded to the nearest 0.0001° (about 11 m)"; longitude "rounded the same way" | Detail, not a clash: the requirement does not state a precision. |
| Heading at low speed | NEAR-002: presence carries "heading" | 2.6: heading unknown value "255; always 255 below 3 km/h" | Detail, not a clash: below 3 km/h the heading is sent as "unknown". |
| Duty-cycle limits (architecture) | POLICY §10, Channel access row: "Channel activity detection before transmit, randomized backoff, airtime and duty-cycle limits, receive/transmit state machine." | 2.1: "FCC Part 15, which has no duty-cycle limit in the United States. The lab must confirm this." 2.1: "Trail Buddy's own limit is that normal traffic keeps the channel at or below 30% busy. SOS and SOS replies are exempt from this budget" | POLICY lists duty-cycle limits. The spec says none applies and uses its own 30% airtime budget, with SOS exempt. |
| Duty-cycle limits (tests) | POLICY §18, Radio / channel access row: passes when "No deadlocks; duty-cycle limits held" | Same 2.1 wording as the row above | A test would check a limit the spec says does not exist. The spec's 30% is not named as the pass criterion. |
| Listening before sending; backoff (gap, not a clash) | POLICY §10, Channel access row: "Channel activity detection before transmit, randomized backoff…". POLICY §18 Unit row: "backoff". POLICY §18 Radio / channel access row: "Channel busy and clear, collisions, backoff, receive/transmit switching". | No rule found for checking the channel before a radio sends its own frames, and no backoff rule. The random delays that exist are: relaying (4.6: "the radio waits a random time"); hello replies (6.2: "replies after a random 0–5 s delay"); Delivered (6.3: "after a random 0–2 s delay"); the wait before a fill ask (6.5: "Wait a random 2–8 s"); and random jitter on send times (6.1 position, 6.4 SOS repeats, 6.6 presence). The only mention of detecting activity is the busy measurement in 7.4 ("receiving or detecting LoRa activity"), which is a reading, not a rule for transmitting. | POLICY expects a check-before-send and a backoff scheme. The spec describes neither. The jitter values are timing spread on schedules, not backoff. The transmit procedure is also raised as Q64. |
| Screens | POLICY §10, BSP row: "nRF52840, SPI, GPIO, interrupts, timers, flash, power, GNSS, display, buttons, buzzer, per board profile." (POLICY §8 also says: "Screens are not used in version 1.") | 9: "Screens are not used in version 1." | Not a clash between the documents. POLICY §10 lists a display as something the board support layer covers per board, and POLICY §8 and spec 9 both say version 1 does not use screens. |
| Which text wins | REQUIREMENTS and POLICY both say "Version 0.3 · 22 October 2026 · Matches message spec Draft v0.3" | 0: "Updated copies of those files will follow; until then, this spec wins where they differ." 12: "These take effect with this version. Updated files will follow." Section 12 does not list NEAR-002, NEAR-004, POLICY §10 or POLICY §18. | The v0.3 copies now exist, but the spec's precedence sentence still says "until then". |
| Whether the encryption choices are final | POLICY §12 (Security), line 51: "These are decided; the spec gives the detail (spec 4.3 to 4.6 and 7.2):". Line 53: "ride messages use AES-128-CCM with an 8-byte authentication tag; keys and tags are derived from the ride key with HKDF-SHA256". | 4.3, line 186: "The choice of algorithm, tag size and counter rule goes to the independent security review before freeze." | POLICY says the algorithms are decided; the spec says they can still change after the security review. |

See Q64 for the missing check-before-sending rule.

**Q63. Standards the spec relies on are not on the POLICY §14 list.**
- Spec section: 4.3 (HKDF, RFC 5869; AES-CCM), 7.4 Updates (Ed25519), 8 (Base64url; CRC-16/CCITT-FALSE), 7.1 (proto3). POLICY §14.
- Unclear: §14 lists only the Bluetooth Core Specification and FCC Part 15 as standards. The clean team needs the definitions of these (CCM parameter conventions for a 13-byte nonce and 8-byte tag, Ed25519, the Base64url alphabet, the CRC parameters, the proto3 language rules). Under CLAUDE.md it may not consult anything not on the list or logged in SOURCE-REGISTER.md.
- Blocks: build steps 2 and 3.
- Options: The spec team approves these standards by name so the product owner can add them to §14 and the repository. Or the spec restates the parameters it needs.

**Facts gathered for the policy owner (22 October 2026)**

Facts only. No answer is proposed, and Q63 stays open until the policy owner decides. Sources: `POLICY.md` v0.3 and `SPEC-v0.3.md` only. The "usual defining document" column was supplied by the policy owner. **None of those documents is an approved source**, and none was opened, searched for or used.

POLICY §14 lists only two standards (the Bluetooth Core Specification and FCC Part 15). Its last row bars anything else "Unless the spec team approves it and it is logged in SOURCE-REGISTER.md first". No `SOURCE-REGISTER.md` is in the repository, so no such approval is visible. "No" below means "not on the §14 list".

| Standard | What it is used for | Spec section | Defining document named in the spec | Usual defining document (not an approved source) | Allowed by POLICY §14 |
| --- | --- | --- | --- | --- | --- |
| HKDF | Derives the encryption key, ride tag, rider tag and key check value from the ride key | 4.3, 11.1 | RFC 5869 | RFC 5869 (named in the spec) | No |
| SHA-256 | Inside HKDF; also the hash of a firmware image in START_UPDATE | 4.3, 7.4 | None | FIPS 180-4 | No |
| AES-128-CCM (8-byte tag, 13-byte nonce) | Encrypts and authenticates every ride frame | 4.3 | None | AES: FIPS 197. CCM mode: NIST SP 800-38C and RFC 3610 | No |
| Ed25519 | The bootloader checks the signature on a firmware image | 7.4 (Update path) | None | RFC 8032 | No |
| CRC-16/CCITT-FALSE | Error check on the ride QR code | 8, 11.4 | None (the spec gives the start value 0xFFFF) | Polynomial from ITU-T V.41; the name is a parameter set from a published CRC catalogue, not a standard | No |
| Base64url, no padding | Text form of the QR code | 8 | None | RFC 4648 §5 | No |
| Base32 alphabet (A–Z, 2–7) | The 3-character short ID. The spec gives the character set and does not use the word "Base32" | 2.3 | None | RFC 4648 §6 | No |
| UTF-8 | Text, names, ride name | 2.6, 6.2, 8 | None | RFC 3629 | No |
| Protocol Buffers, proto3 | Phone-link message format | 7.1, 7.3 | None | Google's proto3 language specification | Unclear (nanopb documentation is allowed; the language rules are not listed) |
| UTC, GPS time | Epoch for Trail Buddy time; GPS fix time sets the clock | 2.2 | None | UTC: ITU-R TF.460. GPS time: IS-GPS-200 | Unclear (GNSS module manuals are allowed; these are not listed) |
| QR code symbol | The ride code the app shows and scans | 8 | None | ISO/IEC 18004 | No |
| Bluetooth LE, including 128-bit UUIDs | Phone link: pairing, encryption, service and characteristic UUIDs | 7.2 | The spec says "Bluetooth LE Secure Connections" and gives three UUIDs; it names no document | Bluetooth Core Specification | Yes |
| FCC digital-modulation route | Why the spec says there is no duty-cycle limit | 2.1 | "FCC Part 15" (name only) | 47 CFR §15.247 | Yes (FCC Part 15) |
| UF2 | USB recovery route on each board | 7.4 (Update path) | "UF2 bootloader" (name only) | Microsoft's published UF2 format | Yes for the UF2 bootloader (listed under approved library documentation); the format document is not named |
| LoRa settings (bandwidth, spreading factor, coding rate, sync word, header CRC) | Over-the-air radio | 2.1 | None | — | Yes (Semtech SX1262 datasheet) |
| nRF52840 hardware random number generator and AES hardware | Creates the radio ID; speeds up encryption | 2.3, 4.3 | None | — | Yes (nRF52840 product specification) |
| "The Broadcast Storm Problem in a Mobile Ad Hoc Network" (Ni, Tseng, Chen, Sheu, MobiCom 1999) | Basis of the relay delay and suppression | 4.6 | Cited in full | — | Yes (published literature, cited in DECISIONS.md) |

Source of these document names: supplied by the spec team through the product owner, 22–23 October 2026, as names only. Not approved sources; not opened or used.

Further facts:
- **POLICY §12 and §14 do not line up.** POLICY §12 (Security), line 53, says "ride messages use AES-128-CCM with an 8-byte authentication tag; keys and tags are derived from the ride key with HKDF-SHA256". Line 58 says "firmware images are signed; the signing key never enters the repository". It does not name Ed25519. Its lead-in says "These are decided". Spec 4.3 ends: "The choice of algorithm, tag size and counter rule goes to the independent security review before freeze." POLICY §14 does not list the defining documents of these algorithms.
- **UUIDs.** The spec uses three 128-bit UUIDs (7.2, lines 398–400). That the Bluetooth Core Specification defines them could not be checked from the repository, because that specification is not in it.
- **Libraries.** The spec names no code library. Two passages come close. Spec 7.4 (Update path) says: "The bootloader and signing tool are chosen by the clean team from the approved libraries and recorded in SOURCE-REGISTER.md." Spec 11.5 says the clean team encodes the phone-link samples "with the generated code", with no tool named. Libraries are named only in POLICY §15: nanopb for phone-link messages, "the crypto library supplied with nRF Connect SDK / Zephyr" for cryptography, and a two-slot bootloader "chosen by the clean team from the Zephyr ecosystem". Nothing in the repository says whether that crypto library supports every algorithm above, in particular Ed25519 and HKDF.

### B. Over the air

**Q64. The transmit procedure is not defined: channel detection, backoff, retries and SOS.**
- Spec section: 2.1, 4.5, 4.6. POLICY §10 (Channel access), §18.
- Unclear: The spec gives queue order, relay delays and a 30% budget. It does not say how a radio decides to transmit once a frame is queued: whether it listens for activity first; the backoff window and how it grows when busy; how long or how many tries before it gives up; whether the relay delay in 4.6 is the only wait; whether SOS frames skip detection and backoff; what happens if a frame arrives while a transmission is pending. The claim that an SOS waits at most 230 ms behind one frame (4.5) holds only if SOS ignores a busy channel.
- Blocks: build steps 1 and 5, and the channel-access tests in POLICY §18 (collisions, backoff, no deadlocks).
- Options:
  - Detect before every non-SOS transmission with a random backoff of a stated size and a retry cap. SOS uses a short fixed backoff and never waits for a clear channel.
  - No detection for any frame, relying on random delay alone.

**Q65. How "channel busy" is measured.**
- Spec section: 7.4 Status (Channel busy), 6.5 rule 5, 2.1. MESH-004, NFR-001, DIAG-001.
- Unclear: "The share of the last 60 s that the radio spent receiving or detecting LoRa activity, or transmitting." What counts as activity: a preamble heard, a valid packet, signal strength over a threshold, or other signals on the channel? How often the reading is refreshed (fill needs a reading under 60 s old). The value before the radio has run 60 s. Whether time spent on frames from other rides and on presence frames counts (it should, for the 30% check).
- Blocks: the MESH-004 test, Fill on Return rule 5, the 60-minute busy history and the 30% field check.
- Options: Count only time spent receiving valid Trail Buddy frames and transmitting, updated each second. Or sample signal strength against a stated threshold. State the update period and the value for the first minute.

**Q66. Scope of the 30% airtime budget, and what the radio does above it.**
- Spec section: 2.1 (Airtime budget), 2.5 (Airtime check). NFR-001. POLICY §18.
- Unclear:
  - The 2.5 table counts only the radios of one ride. All Trail Buddy radios share one channel, so other rides' traffic and every nearby rider's presence add to the load (two groups at a trailhead, for example).
  - The table leaves out hellos and their replies, texts, Delivered (up to one per rider per text), Fill on Return traffic and SOS.
  - It is not said what a radio does when measured busy goes above 30%: report only, slow its own position and presence, or nothing.
- Blocks: the simulator scenarios in build step 5 and the pass criteria "channel at most 30% busy".
- Options: State the assumed density (number of groups in range) and add the missing kinds to the table. State whether 30% is a design target checked in the simulator or a runtime limit, and if runtime, which traffic slows first.

**Q67. Receive-side validation rules are not given for each frame and kind.**
- Spec section: 4.1, 4.2, 5, 6.2, 2.6. POLICY §12.
- Unclear: POLICY requires validating "every length, ID, hop count, message type and field". The spec gives sizes but not the rules for rejecting or accepting:
  - minimum frame length (a ride frame is at least 14 + 1 + 8 bytes) and a length that does not match the kind (text length against frame length, name length, point count against size, a point count of 0 or above 12)
  - an unknown kind byte inside an authenticated frame (dropped, or relayed unread, and in which class), and an unknown frame type with version 1
  - hops remaining above the kind's starting hops (the hops bits are not authenticated)
  - latitude beyond ±90°, longitude beyond ±180°, a fix time of 0 or in the future
  - reserved bits in the hello flags and the fill part byte
  - names and text that are not valid UTF-8 or contain control characters
  - rounding when converting GPS values to wire units (speed to km/h, heading to 1.5° steps, presence coordinates at the halfway point for negative numbers)
- Blocks: the build step 2 unit and protocol tests ("malformed input rejected") and the flood and fuzz tests POLICY §12 requires.
- Options: A table per kind with field, valid range and action (drop; drop and do not relay; accept and clamp). A general rule that anything not listed as valid is dropped and not relayed.

**Q68. Relay delay and suppression: the wording is ambiguous.**
- Spec section: 4.6. MESH-003.
- Unclear:
  - "Waits a random time", "the wait window is 1000 ms, scaled" and "a random 0–20% is added on top": is the delay uniform between 0 and the scaled window, with 0 to 20% of the window added? Is there a minimum delay?
  - Which SNR is used: this reception, or the best or first of several copies heard? SNR is in dB for the scale and in quarter-dB steps on the wire.
  - "Hears the same frame relayed by 2 other radios": does the copy the radio first received count? Does a copy with fewer hops remaining count?
- Blocks: build step 5 relaying and the simulator figures; the "4 transmissions per position" assumption in 2.5.
- Options: Give the delay as a written range with the distribution stated, and say which copies are counted for suppression.

**Q69. Queues: the radio's own frames and relayed frames.**
- Spec section: 4.5, 4.6, 6.3.
- Unclear:
  - Whether relayed frames use the same queues as the radio's own. The position queue is "1 per rider": does that include other riders' relayed positions?
  - For text, "new text refused with BUSY" is a phone-link error. For a relayed text with the queue full, "refused" must mean dropped. Relays can fill the 8 entries so the rider's own text gets BUSY.
  - Delivered and Fill drop the "oldest" entry, which could be the radio's own.
  - Whether a relay cancelled by suppression leaves the queue, and whether queue-full drops are counted (DIAGNOSTICS has a "queue full" counter).
- Blocks: the queue implementation and the load tests.
- Options: Separate own and relay queues, or tag each entry and state the rule for each case.

**Q70. Duplicate memory: 256 entries or 10 minutes?**
- Spec section: 4.4. MESH-002.
- Unclear:
  - "The last 256 identities for 10 minutes": when more than 256 frames arrive within 10 minutes (30 riders sending positions plus texts, Delivered and fill frames can), which limit wins?
  - The replay window (64 per sender) already rejects a counter seen twice. What does the identity memory add?
  - Suppression needs duplicates counted while a relay waits. Is a duplicate counted before it is dropped?
  - Are the radio's own frames entered, so an echo is not relayed again?
- Blocks: duplicate unit tests and memory sizing.
- Options: 256 entries with the oldest evicted, plus expiry at 10 minutes. State that own and relayed frames are entered.

**Q71. The frame counter ceiling at run time, flash failure, and "factory reset".**
- Spec section: 4.4, 2.3, 6.7.
- Unclear:
  - 4.4 gives the boot rule. When the counter reaches the stored ceiling while running, must the next ceiling be written before the frame is sent?
  - If the flash write fails, does the radio stop sending ride frames, and what is reported (STORAGE_FAILED)? If the stored ceiling is damaged at boot?
  - 2.3 says the radio ID changes "only on factory reset". Nothing defines a factory reset: how it is triggered and what it erases (settings, ride, paired phones, counter ceiling, radio ID).
- Blocks: build step 2 (a nonce must never repeat), step 10 persistence and the power-loss hardware tests.
- Options: Write the next ceiling before any frame at the current ceiling is sent, and stop sending ride frames if the write fails. Define factory reset as a table of what it erases.

**Q72. Rider tag: reserved value, clash handling and the 30-rider limit.**
- Spec section: 2.3 (Rider tag clash), 2.4, 4.1 (destination 000000), 4.3, 7.4 Riders. RIDE-004.
- Unclear:
  - A rider tag of 000000 would read as "whole ride". Is it reserved, and is a tag derived as 000000 re-derived with the next clash counter?
  - Where the clash counter is stored, and whether End Ride resets it.
  - After a clash the radio has a new tag. Other radios see a new rider tag in a hello with a radio ID they already know. Riders are never removed, so the old tag stays in RIDERS_LIST and the list can show one rider twice and pass 30.
  - Nothing stops a 31st radio joining with the same QR code. What do the others do with frames from a 31st tag?
- Blocks: riders list and RIDER_UPDATE tests, rider tag unit tests, RIDE-004.
- Options: Reserve 000000. Merge entries that share a radio ID. State the 31st-rider rule (ignored and not listed, or listed and flagged).

**Q73. Short ID derivation.**
- Spec section: 2.3. MSG-003.
- Unclear: "The first 15 bits of the radio ID, written as 3 characters from A–Z and 2–7." Which bit comes first (most significant bit of the first byte?), and which character stands for each 5-bit value (A to Z as 0 to 25 and 2 to 7 as 26 to 31?). There is no sample.
- Blocks: WELCOME, RIDERS_LIST, the default display name, and agreement between the radio and the app.
- Options: State the bit order and the character order, and add the short ID of the radio ID in 11.1 to Section 11.

**Q74. Message IDs, alert IDs and re-sent texts at the receiver.**
- Spec section: 5 (Message IDs and alert IDs), 6.3, 4.4, 6.4.
- Unclear:
  - Message and alert IDs are 16-bit counters starting at 1. Do they survive a reboot? If they restart at 1, a receiver holding (sender, ID 1) may treat a new text as a re-send, and a new alert may be taken for an old ended one.
  - A re-sent text has the same message ID and a new frame counter. A receiver that already holds it: does it store it again, tell the app again, send Delivered again (needed when the first Delivered was lost)? How long does it remember a (sender, message ID) pair?
  - Is Delivered counted once per confirming rider if it arrives twice?
  - Behaviour at wrap after 65535.
- Blocks: MSG-002 tests, stored events, SOS ID handling.
- Options: Keep the counters in storage, or start them at a random value. A receiver keeps recent (sender, ID) pairs with an age. A re-sent text creates no new event but is confirmed again.

**Q75. Clock rules.**
- Spec section: 2.2.
- Unclear:
  - "Adopts [the phone's time] if its own clock is unknown or more than 2 seconds different and it has no GPS fix": is that (unknown OR differs by more than 2 s) AND no fix?
  - The phone's time arrives only in HELLO. A radio without a fix that connects once keeps a free-running clock for hours: is there a drift limit, and do the fix times in SHARE_MY_LOCATION correct it?
  - Trail Buddy time is UTC-based. Does the GPS module's time need conversion (time scale, leap seconds)?
  - When the clock first becomes known, do time fields of items stored earlier stay 0?
  - What happens when the clock steps backwards.
- Blocks: Fill ranges (from and to time), "time received", event times, intervals tests.
- Options: Write the adoption rule as a numbered list. State the GPS conversion and a maximum drift.

**Q76. Anonymous ID change: a hello on change, and a limit on how long it can be deferred.**
- Spec section: 2.3, 6.2, 6.6. NEAR-002.
- Unclear:
  - A ride member's hello carries their anonymous ID so ride radios can hide them from Nearby. 6.2 does not list a hello as a trigger when the ID changes (every boot and every 6 hours). Until the next hello, up to 10 minutes, a rider's own ride mates see them as a nearby rider, and the app could warn of an "approaching rider" who is in the ride.
  - The change is applied only while stopped, with no upper limit. A radio that never stops, or never has a fix, keeps one ID for more than a day, against "changes at least daily" in NEAR-002.
  - Are the "6 hours of operation" counted in uptime, including sleep, and does the timer restart at each change?
- Blocks: the NEAR-002 test, privacy, and the app's approaching-rider logic.
- Options: Send a hello at once after every change. Apply the change after a stated maximum time even if moving, accepting that one encounter may show twice.

**Q77. Moving and stopped state at boot and without a fix.**
- Spec section: 6.1 (moving and stopped thresholds), 6.6.
- Unclear: The starting state at boot. The state between 3 and 5 km/h (keeps the last?). The state with no fix, or with a phone position that has no speed. Whether the GPS duty and the presence interval use the same state.
- Blocks: POS-003 and NEAR-003 tests.
- Options: Start as moving, keep the previous state between the thresholds, and treat unknown speed as moving.

**Q78. Presence: position source, fix age, and the basis for the plausibility check.**
- Spec section: 4.2, 6.1, 6.6, 7.4 (SHARE_MY_LOCATION). NEAR-002.
- Unclear:
  - Which fix presence uses: the radio's GPS only, or the phone's when the radio has none. Heading and speed are optional in SHARE_MY_LOCATION; when absent, presence sends 255. A fix age for a phone fix needs the radio clock, which can be unknown.
  - The 200 km/h check: distance over which time, the time between receipts or corrected by each report's fix age? Does a rejected report become the "last report" for the next test? The first report from an ID has nothing to compare with.
- Blocks: build step 8 tests and NEAR-008 behaviour.
- Options: State the source order, the unknown-value rules and the time basis. State that a rejected report does not replace the last accepted one.

### C. Behaviour

**Q79. SOS when the radio has no ride.**
- Spec section: 5 (SOS is a ride frame), 7.3 (NO_RIDE), 7.4 SOS. SOS-001 to SOS-004, RIDE-001.
- Unclear: SOS frames are encrypted with the ride key, so a radio with no active ride can neither send nor hear an SOS. The spec does not say SEND_SOS needs a ride, and does not list NO_RIDE for it. Is it intended that a rider outside any ride has no SOS?
- Blocks: SEND_SOS handling, the field test and the safety case. This is a product decision.
- Options:
  - SOS requires a ride; SEND_SOS without one returns NO_RIDE and the app says so.
  - Add an SOS frame that needs no ride key and is heard by every Trail Buddy radio. That changes Section 4 and the security review.

**Q80. SOS when there is no position fix.**
- Spec section: 2.6 ("a position is never sent without a fix"), 5 (SOS fields), 7.4 SEND_SOS. SOS-001.
- Unclear: SEND_SOS may carry no position, and the radio may have no fix and none from the phone. The SOS frame has no "unknown" value for latitude or longitude. Is the SOS sent at once with the last known position and its true fix time, held until a fix arrives, or sent with an unknown marker? How old may a last known position be? How does the receiving app show a missing position?
- Blocks: SOS-001 tests and the safety case.
- Options: Send at once with the last known position and its real fix time, and replace it on later repeats. Or add an explicit "no position" marker. Or hold until a fix, with a stated limit.

**Q81. SOS schedule details, and an SOS across a reboot.**
- Spec section: 6.4, 4.4, 5. SOS-003.
- Unclear:
  - "Every 5 minutes after the first send": at 300 s, 600 s and so on, measured from the first send? Does the up-to-5 s jitter apply to the repeat at 10 s? Is jitter only added, or added and subtracted?
  - The repeat number is one byte: after 255 repeats (about 21 hours) does it wrap or stay at 255?
  - If the radio reboots or loses power during an active SOS (battery swap, brownout, crash), does it resume repeating? If so, the alert ID, repeat number and first-sent time must be stored, and how often.
- Blocks: build step 7, the power-loss hardware test, SOS-003 test.
- Options: State the schedule as send times from the first send. State that an active SOS resumes after a reboot and how often its state is stored.

**Q82. SOS_STATUS timing, the reply to SEND_SOS, and the Copy that list.**
- Spec section: 7.4 SOS, 7.2 (On connect), 6.4.
- Unclear: When the radio sends SOS_STATUS without a request (each repeat, each state change, each Copy that, never). What SEND_SOS is answered with. How "copy-that riders" is built (distinct riders who replied for this alert) and when it is cleared. "Last sent" when the clock is unknown. "Times sent" is a u8 (wrap).
- Blocks: the phone tests for SOS and the app.
- Options: SOS_STATUS after every send, repeat, Copy that and end. SEND_SOS answered by SOS_STATUS, or by ERROR.

**Q83. End Ride during the rider's own SOS: how reliable is the "left the ride" reply?**
- Spec section: 6.7, 6.4 (Ending), 4.5.
- Unclear: END_RIDE sends one SOS reply "ended because the owner left the ride", then erases the key and drops queued ride frames. Is the reply sent once, or twice like an SOS? The key must stay until the reply has actually gone out (the channel may be busy). If the reply is lost, other radios show the SOS forever, because an SOS is never ended for lack of signal. When does RIDE_STATE (not active) reach the app: at once, or after the reply is sent?
- Blocks: End Ride and SOS-004 tests.
- Options: Send the reply twice, 10 s apart, and erase the key after the last send or after a stated timeout. Report RIDE_STATE after the erase.

**Q84. Fill on Return: gaps in the ask and answer frames and in the rules.**
- Spec section: 5 (Fill ask, Fill answer), 6.5.
- Unclear:
  - The answer frame carries a rider tag, a part byte and points, but not the range it covers, an ask reference, the number of parts or a last-part marker. A radio cannot tell when an answer is complete, or whether it answers its own ask ("no answer within 30 s means ask again").
  - Outline and detail: are outline points repeated in the detail parts? Do outline and detail parts number separately? What is sent when the range has fewer than 12 points?
  - The ask range: "at most 30 minutes" from the last position before the silence, or the latest 30 minutes? For a longer gap, one ask or several? What are from and to time when the clock is unknown?
  - Rule 1: is "moved more than 50 m" measured between the last position before the silence and the first one heard after?
  - What counts as an answer for rule 4: any answer for that rider, or one covering the asked range?
  - "Heard again": by any authenticated frame, including a relayed one?
- Blocks: build step 9 and the FILL-001 tests.
- Options: Add an ask reference, the range and the total parts to the answer. State "answered" as a covering answer, and write each range rule as a worked example.

**Q85. What is stored, where, and what survives a reboot.**
- Spec section: 6.5 (2-hour history), 6.3, 6.4, 6.2, 6.7. POLICY §10 (Settings and storage), §18.
- Unclear: The spec says the ride, settings, repeater mode and the counter ceiling persist, and that replay and duplicate records do not. It does not say for: own and heard position history (RAM or flash; at one point per 10 s, flash wear matters), stored events and event numbers, the known riders (names, radio IDs), a running SOS (Q81), the ID counters (Q74), paired phones, pending fill asks, and the clash counter. Capacity is also open: heard positions for 30 riders over 2 hours is thousands of points, with no per-rider limit or thinning rule. Own history is "at least one point every 10 s" while the GPS may sample once a minute (Q57).
- Blocks: build steps 9 and 10, memory sizing and the power-loss hardware tests.
- Options: A table of each record with its medium, whether it survives a reboot, its capacity and its eviction rule. State what End Ride erases from the same table.

**Q86. Stored events: numbering, SOS repeats, and "events waiting".**
- Spec section: 6.3 (Stored events), 7.2 (On connect), 7.4 (GET_MISSED; STATUS events waiting).
- Unclear:
  - "Events waiting" counts stored events "not yet sent to the app". There is no acknowledgement, and the connect sequence in 7.2 does not push stored events. When does an event count as sent?
  - SOS_RECEIVED is sent "for every repeat" with an event number. Does each repeat create a new stored event? SOS events are never evicted, so the count could pass 200.
  - Do event numbers continue after a reboot and after End Ride?
  - GET_MISSED with a number older than the oldest retained event: how does the app learn events were lost?
- Blocks: GET_MISSED and reconnect tests, and the STATUS events-waiting field.
- Options: An event is "sent" when written to a connected app. Repeats update the existing SOS event. Event numbers never go back. Add a "first available event number" to the replay.

**Q87. Settings: how they interact with SOS, repeater mode and End Ride.**
- Spec section: 7.4 Settings, 6.6, 6.7, 6.8.
- Unclear:
  - During the rider's own SOS, presence is forced on. If setting 2 is changed to off then, is it stored and not effective, or refused? SETTINGS returns the stored value while STATUS reports presence sending as on. Does it revert to the stored value when the SOS ends?
  - Is repeater mode allowed with no ride (NO_RIDE?) or during the rider's own SOS?
  - End Ride sets presence on and repeater mode off: these are writes that can fail. What is reported?
- Blocks: settings tests in build step 10.
- Options: State each case as a rule with its error code.

**Q88. Damaged records at boot, and when SETTINGS_RESET is reported.**
- Spec section: 7.4 Settings (Rules), 7.2 (On connect), 4.4, 6.8.
- Unclear: SETTINGS_RESET is reported "on the next connection": as an ERROR message, and at which point in the connect sequence in 7.2? If the ride record is damaged, is the ride erased, and how does the app learn (RIDE_STATE with no ride, or an error)? What happens if repeater mode was on, or an SOS was active?
- Blocks: the power-loss hardware tests and BLE-002 ("no settings write can leave the radio unable to boot").
- Options: List each record, what a damaged copy does, and what is reported and when.

**Q89. Repeater mode: counting and answering.**
- Spec section: 6.8, 6.3, 6.2, 6.4, 7.4 Riders. RPT-001.
- Unclear: Does a repeater send Delivered for texts it stores? That would raise the confirmation count with a radio that has no rider. Does it count among the 30 riders and appear in RIDERS_LIST (the repeater flag suggests yes)? Does it answer hello requests? Does it report presence it hears to a connected phone? Does it sound the buzzer for a new SOS? Does it keep and answer Fill on Return data? Does it take part in the 2-minute pairing window?
- Blocks: RPT-001 tests and the MSG-002 count.
- Options: A list of what a repeater does and does not do, kind by kind.

**Q90. Buzzer behaviour.**
- Spec section: 6.4 (To the app), 9. POLICY §8.
- Unclear: Pattern, duration and loudness. Whether it sounds only for the first frame of a new alert or for every repeat. What silences it (button, phone, timeout). Whether it sounds when the phone is connected. What it does when the SOS ends, when two alerts arrive, and at low battery.
- Blocks: the board support interface for the buzzer and the SOS hardware test.
- Options: One short sound pattern for a new alert, sounded once, silenced by the power button.

**Q91. LOCATION_UPDATE: what is reported.**
- Spec section: 7.4 Location, 6.1, 6.5.
- Unclear: Is the radio's own position reported to the app (with the rider's own tag), and how often? Is every received position reported, or only the newest by fix time? Is a position older than one already held (out of order) reported? How is "hops travelled" set for a frame relayed through a repeater?
- Blocks: build step 4 and the phone tests.
- Options: Report each new authenticated position once, in fix-time order, and state the own-position rule.

### D. Phone link

**Q92. Top-level message structure, and how "required" is checked.**
- Spec section: 7.1, 7.3, 7.4. BLE-001.
- Unclear:
  - Each message has a type, a request ID and a last flag, and is sent as a length plus protobuf bytes. The spec does not say how a receiver learns the message type from the bytes: one wrapper message with a type and one payload field per message, or a separate header.
  - proto3 cannot tell an unset number from 0 unless the field is marked optional. The spec says a missing required field gets ERROR, and 0 is valid for latitude, accuracy and fix time. Which fields need presence tracking?
  - Who assigns field and enum numbers, and how the app team receives the schema. The app must use identical numbers.
  - Naming: message names in the spec are in capitals (HELLO).
- Blocks: build step 3 and the recorded vectors in 11.5.
- Options: The spec team prescribes the wrapper and a numbering rule (for example table order from 1). The clean team writes the schema and hands it to the product owner for the app team.

**Q93. Framing checks and the size of list messages.**
- Spec section: 7.2 (Framing), 7.3, 7.4.
- Unclear:
  - The byte order of the 2-byte length (the over-the-air rule in Section 4 is little-endian; 7.2 is silent).
  - A length above 1024, a length of 0, or a message that never completes: which error, whether the link is dropped, a timeout, how the receiver re-synchronises.
  - RIDERS_LIST (30 entries, names up to 24 bytes) and NEARBY_LIST (32 entries) can pass 1024 bytes. Only TRACK_POINTS has a part cap (64 points). The maximum entries per part for the other lists, and whether they use the last flag.
- Blocks: build step 3 and the list tests.
- Options: Little-endian length. A bad length gets ERROR and the partial message is discarded after a stated timeout. A fixed entry cap per part for each list.

**Q94. Pairing: association model and adding a phone later.**
- Spec section: 7.2 (Pairing and security), 9. POLICY §8.
- Unclear:
  - Screens are not used in version 1, so Bluetooth LE Secure Connections cannot use numeric comparison. With "just works" pairing, any phone in range can pair during the 2-minute window, and then read texts and positions. What confirms the rider's own phone: a button press, a passkey, or nothing?
  - New pairing is accepted only in the 2 minutes after power-on. To add or replace a phone during a ride, the rider must power-cycle the radio, which stops relaying and any running SOS for the restart. Does a button press open the window at any time? POLICY §8 says the power button is for "pairing window".
  - When 4 phones are remembered, is a fifth refused or does it replace the oldest? How is a remembered phone removed?
  - Who starts encryption after HELLO and WELCOME, and what a bonded phone does outside the window.
- Blocks: build step 3 and the independent security review (the ride key and texts cross this link).
- Options: Require a button press to open the window at any time. Define replacement and removal of remembered phones. State the order: connect, HELLO, WELCOME, security start.

**Q95. Overlapping error codes.**
- Spec section: 7.3.
- Unclear: BAD_FIELD ("too long") and TOO_LARGE ("text or message over its limit") overlap. Which code for a 151-byte text, an empty text, a 25-byte name, a message above 1024 bytes? Which code wins when several apply (for example NO_RIDE, SOS_ACTIVE and NOT_ALLOWED_NOW)? Which code for an update chunk at the wrong offset?
- Blocks: the "malformed input rejected" tests for the phone link.
- Options: A table of message, condition and code, with an order of precedence.

**Q96. Diagnostics: counter definitions.**
- Spec section: 7.4 (Repeater, errors and diagnostics), 10. DIAG-001.
- Unclear: Which kinds are counted (nine ride kinds, presence?). Whether "received" counts before or after authentication and duplicate checks. Whether "relayed" counts frames sent or relays cancelled by suppression. Whether the six drop reasons are exclusive and in what order they are checked. The order of the 60 busy values (oldest first?) and the value of the current partial minute. Whether End Ride clears counters.
- Blocks: the DIAG-001 test and the field tests.
- Options: A table of each counter with the event that increments it and the order of checks.

**Q97. Firmware update: remaining details.**
- Spec section: 7.4 Updates. DFU-001.
- Unclear:
  - Where the signature travels (inside the image or in START_UPDATE) and what the SHA-256 covers. The type of "new version" (three numbers as in WELCOME?). Whether an image of the same version is refused.
  - An UPDATE_CHUNK at an offset other than the bytes received (gap or repeat): which code. An image larger than the slot. A timeout if the app stops sending. START_UPDATE while one is in progress.
  - Whether relaying and SOS repeats continue while receiving, and that the radio is offline during the reboot and the 60 s confirmation period. What an SOS raised mid-update does.
  - What the new firmware must do to "confirm itself".
  - The UF2 recovery bootloader accepts images over USB. Does it check signatures? How does it sit in flash beside the two-slot bootloader? DFU-001 says updates use signed images.
- Blocks: build step 11, the failed-update hardware test and the security review.
- Options: State each as a rule with its code. State whether the USB route is accepted as outside DFU-001.

### E. Ride QR code

**Q98. QR code: CRC byte order and trailing data.**
- Spec section: 8, 11.4. RIDE-001.
- Unclear: The two CRC bytes: low byte first (as over the air) or high byte first? The sample ends in `6b0b`, and the spec does not say which. Whether bytes after the CRC are ignored or cause refusal, and whether the Base64url text may contain padding or capital letters.
- Blocks: the QR vector in 11.4 and app and radio agreement on the code.
- Options: State the CRC byte order and that any extra or malformed text is refused.
