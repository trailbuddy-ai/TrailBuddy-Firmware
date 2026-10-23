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

Status (v0.4): ANSWERED — spec 2.1, unchanged. Still provisional until the compliance lab confirms it (spec 14). The transmit procedure is now answered (spec 4.7, Q64). New: the ThinkNode M1's antenna gain is above the 2 dBi in 2.1 (Q99).

Status (v0.5): ANSWERED — spec 2.1. The antenna limit is now 3 dBi, not 2 dBi (Q99), the lab tests each board with its own supplied antenna, and the lab must also check the ThinkNode M1's existing FCC grant. Still provisional until the compliance lab confirms the whole plan (spec 14). The transmit procedure is answered in spec 4.7 (Q64).

**Q2. NFR-001 targets are missing.**
- Spec section: NFR-001.
- Unclear: Maximum packet size, range, battery hours per riding day, and return-to-filled-track time are "set by the product owner" and not stated.
- Blocks: Maximum packet size is the input to every byte budget in the freeze checklist.
- Options: The product owner supplies the numbers. Alternatively, accept provisional numbers and mark them as such until v1.0.

Status (v0.3): PARTLY ANSWERED — spec 3 and NFR-001. Given: 12 hours per riding day, filled track within 60 s of return, maximum frame 176 bytes (consistent with spec 4.1 and 5). Missing: (a) range is "at least as good as the August field test", and that test's result is not in the repository, so there is no number to test against; (b) the 12-hour target has no conditions (tier, presence on or off, GPS duty, battery capacity, temperature, board); (c) "return" for the 60 s target is not defined, and the target conflicts with the Fill on Return rates (Q58).

Status (v0.4): PARTLY ANSWERED — spec 3 and NFR-001 v0.4. Answered: battery conditions (tier 1, presence on, moving half the time, the board's standard battery, about 20 °C); "filled" and "return" defined for the 60 s target (Q58); range target defined as total blackout time and longest blackout (43 minutes) no worse than the August test. Missing: (a) the August total blackout time and route data are not in the repository, so only the 43-minute figure can be tested against; (b) "the board's standard battery" is not named for any board (the ThinkNode M1 datasheet gives 1200 mAh; the others are not documented); (c) the ThinkNode M1 operating range is −20 to +70 °C and the RAK19026 VC is −35 to +75 °C, and the battery test is only at 20 °C.

Status (v0.5): PARTLY ANSWERED — spec 3 and NFR-001 v0.5. Answered: battery conditions (tier 1, presence on, moving half the time, the board's standard battery, about 20 °C); "filled" and "return" for the 60 s target (Q58); range target as total blackout time and longest blackout (43 minutes) no worse than the August test; frames up to 184 bytes. Missing: (a) the August total blackout time and route data are not in the repository; (b) "the board's standard battery" is named for no board (the ThinkNode M1 datasheet gives 1200 mAh; the others are not documented); (c) operating ranges differ (RAK19026 VC −35 to +75 °C; ThinkNode M1 −20 to +70 °C in its datasheet, but spec 9 records −10 to +50 °C from a user manual that is not in the repository, Q130) and the battery test is only at 20 °C.

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

Status (v0.4): ANSWERED — spec 2.1 and 8, unchanged.

Status (v0.5): ANSWERED — spec 2.1 and 8, unchanged.

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

Status (v0.4): ANSWERED — spec 1, 4, 5 and 7.1. The schema questions left open in v0.3 are answered (Q92).

Status (v0.5): ANSWERED — spec 1, 4, 5 and 7.1. Frames now carry a 4-byte join ID (4.1).

**Q5. Documents the spec refers to are not available, and one instruction conflicts with POLICY.**
- Spec section: Encoding; Clean-room note. POLICY §19.
- Unclear:
  - "How to Create Trail Buddy's Protocol Buffers" is cited and is not in the four files.
  - "Section 9 of the Clean-Room Firmware Development Specification" is cited. REQUIREMENTS.md appears to reproduce it, and I can't confirm that.
  - The spec says to log this version in `docs/PROVENANCE.md`. POLICY §19 says the clean-room log is kept outside the clean repo, and the repository layout has no PROVENANCE.md.
- Blocks: Schema conventions such as field naming, enum style and optional-field rules. It also leaves open where hand-over records go.
- Options: Supply the protobuf guide, or state the rules it carries. Decide whether PROVENANCE.md exists in the repo or the log stays outside it.

Status (v0.3): PARTLY ANSWERED — spec 1 (no docs/PROVENANCE.md; hand-overs go in the clean-room log kept outside the repository, as POLICY §19 says) and 7.1 (proto3, package trailbuddy.v1, names from the spec, field numbers never reused, enums start with an unspecified 0, bounded fields). The protobuf guide and the Clean-Room Firmware Development Specification are no longer cited. Missing: how required and optional (R/O) map onto proto3, who assigns field numbers and how the schema reaches the app team, and the message structure on the wire (Q92).

Status (v0.4): ANSWERED — spec 7.1: R and O map to `optional` scalar fields checked by the receiver; field numbers follow table order from 1 and are never reused; one `Envelope` wrapper carries every message; the clean team writes the .proto and hands it to the product owner for the app team (spec 1). The two documents the spec used to cite are no longer cited.

Status (v0.5): ANSWERED — spec 7.1 (optional scalar fields, Envelope wrapper, fixed-width types, sizes recorded in DECISIONS.md and tested at 2048 bytes or less).

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

Status (v0.4): ANSWERED — spec 2.2, rewritten as a numbered rule list (Q75). A phone fix's age for presence is still measured from arrival, not from the fix: Q116.

Status (v0.5): ANSWERED — spec 2.2, rewritten: phone time only from a bonded phone over an encrypted link, a 5-minute step limit with a "clock conflict" flag, and no frame ever dropped because of its time. The age of a phone fix is answered in 6.1 (Q116). New: ride expiry uses this clock (Q124).

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

Status (v0.4): ANSWERED — spec 2.3: short ID bit and character order given (Q73), rider tag 000000 reserved and clashes handled (Q72), factory reset defined in 6.11 (Q71). I recomputed the short ID for radio ID 1122334455667788 and it is CER, as in 11.1.

Status (v0.5): ANSWERED — spec 2.3: join ID added; the radio ID is never sent unencrypted; the clash rule now says which radio changes and limits changes to one per 10 minutes. Short ID CER and the rider tags eea021 and c4b67b recompute correctly.

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

Status (v0.4): ANSWERED — spec 2.3, 6.2 and 6.6. A hello is now sent when the anonymous ID changes, and the change is applied within 24 hours even while moving (Q76). New: the permanent radio ID is sent in clear over Bluetooth in WELCOME before encryption (Q101).

Status (v0.5): ANSWERED — spec 2.3, 6.2, 6.6 and 7.2. WELCOME no longer carries the radio ID (Q101).

**Q9. Ride size and ride identification on air.**
- Spec section: JOIN_RIDE; envelope destination "ride"; RIDERS_LIST.
- Unclear: Maximum riders per ride, and whether a radio holds exactly one ride at a time. There is no ride identifier on air, so a radio can't tell whether a packet is for its ride without trying to decrypt it.
- Blocks: List sizes, duplicate-cache size, airtime budget, memory use, and the receive path (what is tried, and how often it fails).
- Options: State a maximum ride size and a single active ride. Add a short ride identifier to the envelope, or derive one from the key.

Status (v0.3): ANSWERED — spec 2.4 (at most 30 riders, one active ride, 2-byte ride tag in every ride frame). How the limit of 30 is enforced: Q72.

Status (v0.4): ANSWERED — spec 2.4 (30 riders, repeaters included; one active ride; 2-byte ride tag; a 31st radio still works and is flagged). The enforcement questions (Q72) are answered.

Status (v0.5): ANSWERED — spec 2.4, unchanged in substance.

**Q10. The four tiers are not defined.**
- Spec section: JOIN_RIDE; QR code. RIDE-002.
- Unclear: Tier names, how many bits, hop limit per tier, position interval per tier, GPS duty per tier, and presence interval per tier (the open question asks for this). Also whether tier can change during a ride and how.
- Blocks: POS-001, POS-003, hop limits (MESH-001), build step 4, the mesh simulation and the 4-rider field test at each tier.
- Options: A table of four tiers with named values for hop limit, position interval and GPS duty. Allow a mid-ride tier change only through a new JOIN_RIDE, or add an explicit message.

Status (v0.3): ANSWERED — spec 2.5 (four tiers with hop limit, position interval and Fill on Return gap threshold; fixed for the ride, changed only by ending the ride and joining again), 6.1 (GPS duty) and 6.6 (presence intervals). Conflict with the fresh-fix rule: Q57.

Status (v0.4): ANSWERED — spec 2.5 (tier table now also holds presence interval, Delivered hop limit and Fill gap threshold), 6.1 and 6.6. The fresh-fix conflict (Q57) is answered.

Status (v0.5): ANSWERED — spec 2.5 (tier table, unchanged).

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

Status (v0.4): ANSWERED — spec 4.3 and 4.4 (run-time ceiling advance and flash failure now covered, Q71). I recomputed the key derivation and all eight encrypted ride-frame vectors in 11.2 (position, relayed position, hello, text, SOS, SOS with no position, fill ask, fill answer) with an independent AES-CCM, and every one matches byte for byte. The independent security review is still required before freeze (spec 11.6, 14). New: nonce uniqueness depends on rider tags being unique (Q100).

Status (v0.5): ANSWERED — spec 4.3 and 4.4. The nonce is now rider tag + join ID + frame counter + ride tag (Q100). I recomputed the key derivation, the SOS commitment, all 13 ride frames in 11.2, the three rejection frames with bytes in 11.3, the presence frame, the QR code and the Ed25519 signature in 11.6 with independent code, and every value matches. The independent security review of v0.5 is still required before freeze (11.8, 14). New: a radio that rejoins the same ride (Q123).

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

Status (v0.4): ANSWERED — spec 4.6 (only frames with the radio's own ride tag that pass authentication are relayed) and 6.8 (a repeater is joined to a ride by a phone).

Status (v0.5): ANSWERED — spec 4.6 and 6.8, unchanged.

**Q13. Duplicate suppression is not specified.**
- Spec section: Envelope; MESH-002.
- Unclear: Packet identity size and uniqueness scope. Duplicate-memory size and lifetime. The spec needs SOS to repeat every 5 minutes. A repeat must not be dropped as a duplicate, and a replay must still be rejected.
- Blocks: Unit tests for duplicates and replays, the SOS repeat logic, and memory sizing.
- Options: Every SOS repeat gets a new packet identity under the same alert ID. Define memory by number of entries and by age, and state which is used when they conflict.

Status (v0.3): ANSWERED — spec 4.4 (identity = sender rider tag + frame counter; last 256 identities for 10 minutes; replay window of 64 per sender; every SOS repeat is a new frame with a new counter and the same alert ID). Which limit wins when 256 entries and 10 minutes conflict is not stated: Q70.

Status (v0.4): ANSWERED — spec 4.4. When 256 entries and 10 minutes conflict, the oldest entry is evicted when full and each expires after 10 minutes (Q70).

Status (v0.5): ANSWERED — spec 4.4. A frame's identity is now sender tag, join ID and counter. Replay after a reboot is limited, not removed (Q122).

**Q14. Rebroadcast delay (MESH-003) has no behaviour in the spec.**
- Spec section: not present. MESH-003 requires a randomized delay with a weighting chosen from published literature.
- Unclear: The delay range, any weighting, and whether delay depends on signal strength, hops remaining or priority. Does SOS skip the delay or use a shorter one?
- Blocks: The relay implementation and the mesh simulator.
- Options: The spec team chooses a scheme, cites literature in DECISIONS.md, and states different ranges per priority class.

Status (v0.3): ANSWERED — spec 4.6 (random delay; SOS and SOS replies 0–150 ms; other kinds a 1000 ms window scaled by SNR plus 0–20%; suppression after 2 other relays; Ni, Tseng, Chen and Sheu cited). The formula wording is ambiguous: Q68.

Status (v0.4): ANSWERED — spec 4.6, rewritten with an exact formula, the SNR of the first copy, and which receptions count towards suppression (Q68). SOS and SOS replies wait a uniform 20–150 ms. The published source is cited.

Status (v0.5): ANSWERED — spec 4.6. Normal frames: unchanged formula. SOS and SOS replies now wait a uniform 20–500 ms (was 20–150 ms) to reduce collisions.

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

Status (v0.4): ANSWERED — spec 4.5. Class 0 holds 64 shared entries and never drops an SOS or an "I'm OK"; each class now states own and relay space (Q59, Q69). I recomputed the airtime of the longest frame (176 bytes: about 230 ms) and of presence (16 bytes: about 45 ms), and both match.

Status (v0.5): ANSWERED — spec 4.5: class 0 holds 16 own and 48 relay entries; the radio's own SOS always has space; relays over the SOS schedule are treated as class 6. New: the schedule limit is counted per sender identity, which a ride-key holder can invent (Q118). Frame times for the longest frame (184 bytes, about 240 ms) and presence (about 45 ms) recompute correctly.

**Q16. SOS-002 can conflict with duty-cycle limits.**
- Spec section: SOS-002 vs REG-001; POLICY §10 (channel access limits).
- Unclear: SOS is "never dropped" and "goes ahead of every other transmission". Channel access also holds airtime and duty-cycle limits. If the limit is reached, is an SOS delayed, or does it have its own allowance?
- Blocks: Channel-access design and the "duty-cycle limits held" and "SOS never dropped" tests, which can't both pass without a rule.
- Options: SOS may exceed the normal budget within a stated cap. Or SOS is delayed until the limit allows, never dropped. The product owner and regulatory sign-off decide.

Status (v0.3): ANSWERED — spec 2.1 (SOS and SOS replies are exempt from the 30% budget and never delayed by it; the Part 15 route has no duty-cycle limit if the lab confirms). POLICY §10 and §18 still speak of duty-cycle limits being held: Q62. SOS and channel detection: Q64.

Status (v0.4): ANSWERED — spec 2.1 and 4.7 step 5: an SOS never waits for a clear channel and is exempt from the 30% budget. POLICY §10 and §18 now say the same (no duty-cycle limit in the United States; 30% budget).

Status (v0.5): ANSWERED — spec 2.1 and 4.7. Changed from v0.4: an SOS now listens before talking, and channel access never delays it by more than 1 s. It is still exempt from the 30% budget.

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

Status (v0.4): ANSWERED — spec 4.1, 4.6 and 4.8 (a frame with another protocol version is dropped and counted; unknown kinds are dropped, not relayed). A rationale per field size is still not given: see Q55.

Status (v0.5): ANSWERED — spec 4.1, 4.6 and 4.8. The reason for the join ID's size is given (4.4). Reasons for the other field sizes are still not stated one by one (Q55).

**Q18. One protocol version or two?**
- Spec section: "Every phone-link message carries … the protocol version"; HELLO/WELCOME; the envelope.
- Unclear: Whether phone-link and over-the-air versions are the same number, its format (single integer or major.minor), the compatibility rule, and why every phone-link message repeats the version after HELLO has already checked it.
- Blocks: The version check, BLE-001 tests, and message overhead on a small BLE packet.
- Options: One number with an exact-match rule. Or separate numbers, with the phone link allowed a range.

Status (v0.3): ANSWERED — spec 4.7 (one protocol version number, 1, for both links; exact match; the phone link checks it once, in HELLO and WELCOME).

Status (v0.4): ANSWERED — implied by spec 4.1 (version 1 in the frame), 4.8 check 1 (exact match) and 7.3 (VERSION_MISMATCH, HELLO and WELCOME). The v0.3 sentence saying one number serves both links is no longer in the spec; the rule still reads the same.

Status (v0.5): ANSWERED — spec 4.1, 4.8 check 1 and 7.3, unchanged.

## B. Phone link

**Q19. The BLE service itself is not defined.**
- Spec section: Phone link; BLE-001.
- Unclear: Service and characteristic layout (what the phone writes, what it is notified of), identifiers, maximum message size and how larger messages are split, ordering, flow control, and the number of simultaneous connections.
- Blocks: Build step 3, and every message that can exceed one BLE packet: TRACK_POINTS, NEARBY_LIST, RIDERS_LIST, DIAGNOSTICS, UPDATE_CHUNK.
- Options: The spec team defines the service contents and a framing rule. Choose between one write/notify pair for all messages and separate channels for bulk data such as updates.

Status (v0.3): ANSWERED — spec 7.2 (service and two characteristic UUIDs; 2-byte length framing across Bluetooth packets; largest packet size requested; 1024-byte maximum message; one write at a time with response; one phone at a time). Framing details: Q93.

Status (v0.4): ANSWERED — spec 7.2: UUIDs, little-endian 2-byte length framing, 1024-byte limit, 2 s timeout, list sizes per part (Q93). New: how the receiver recovers after a bad length (Q106); the size of a full TRACK_POINTS part (Q105).

Status (v0.5): ANSWERED — spec 7.2: UUIDs, framing, 2048-byte limit, 2 s timeout, re-synchronisation after a bad length (Q106), list sizes per part with TRACK_POINTS at 40 points (Q105).

**Q20. BLE pairing and link security are not stated.**
- Spec section: Phone link; JOIN_RIDE; START_UPDATE.
- Unclear: Whether the link is paired, bonded and encrypted, who may connect, and whether another phone can connect and read the ride key or messages. JOIN_RIDE sends the ride key, and START_UPDATE accepts firmware over the same link.
- Blocks: Connection setup, and the security review POLICY §12 requires.
- Options: Require an encrypted, bonded link before any message other than HELLO. Add a pairing step on first use, with a physical confirmation where the board has a button.

Status (v0.3): ANSWERED — spec 7.2 (LE Secure Connections pairing and encryption before any message except HELLO and WELCOME; new pairing only in the 2 minutes after power-on; 4 remembered phones; the ride key is never sent back to the app). The association model and how to add a phone later: Q94.

Status (v0.4): ANSWERED — spec 7.2 and 6.11 (Just Works with bonding; new pairing only while the pairing window is open; a 2 s button press opens it at any time; 4 phones remembered, a fifth replaces the least recently used; 10 s hold forgets all phones) (Q94). New: what an unpaired phone can do before encryption (Q101).

Status (v0.5): ANSWERED — spec 7.2 and 6.11, changed from v0.4: new pairing only while the pairing window is open, which opens only on a button press or at first set-up; one new phone per window; Secure Connections only; a fifth phone is refused (v0.4 let it replace the oldest) (Q94). New: two small gaps in the button rules (Q126).

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

Status (v0.4): ANSWERED — spec 7.2 Session. New: with no idle timeout and one phone at a time, an unpaired phone can hold the only connection (Q101).

Status (v0.5): ANSWERED — spec 7.2 Session and Connections and time limits: up to two connections (one encrypted), a 5 s encryption deadline, a 30 s refusal after a failed encryption, no idle timeout once encrypted. New: the two places can still be taken by unencrypted connections (Q117).

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

Status (v0.4): ANSWERED — spec 7.1 (Envelope with request_id and last) and 7.3. Missing-field detection is now defined by `optional` fields (Q92).

Status (v0.5): ANSWERED — spec 7.1 and 7.3, unchanged.

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

Status (v0.4): ANSWERED — spec 7.3: one code list, a precedence order and worked examples (Q95). UPDATE_REJECTED and NO_CONFIRMATION are not in the precedence order; for a START_UPDATE with several faults, the order between UPDATE_REJECTED, LOW_BATTERY and SOS_ACTIVE is not stated.

Status (v0.5): PARTLY ANSWERED — spec 7.3: the precedence order is longer, but UPDATE_REJECTED and NO_CONFIRMATION are still not in it, and the order between UPDATE_REJECTED, LOW_BATTERY and SOS_ACTIVE for a START_UPDATE with several faults is not stated. (Marked answered in the v0.4 review; changed on re-reading because the gap is still there in v0.5.)

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

Status (v0.4): ANSWERED — spec 7.4 Connect, unchanged. The board model list includes RAK19007 and POCKET_V3, which are not version 1 boards.

Status (v0.5): ANSWERED — spec 7.4 Connect. WELCOME no longer identifies the radio; SESSION_START and RADIO_INFO are new. The board list still contains RAK19007 and POCKET_V3, which are not version 1 boards (Q131).

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

Status (v0.4): ANSWERED — spec 6.3 (one definition of "in contact" used by STATUS, RIDERS_LIST and Delivered; an event counts as sent once written to a connected app), 6.10 (channel busy measured every second over 60 s from all LoRa frames and transmissions) and 7.4 Status (Q60, Q65, Q86).

Status (v0.5): ANSWERED — spec 6.3, 6.10 and 7.4 Status (new fields: clock conflict, slow-down step, storage failed, nearby view incomplete, frames sent as you).

**Q26. The app has no way to read ride or SOS state after reconnecting.**
- Spec section: Ride and SOS messages; SETTINGS; WELCOME. BLE-002.
- Unclear:
  - RIDE_RESULT is sent only in reply to a ride change. SETTINGS has no ride fields. SOS_STATUS is described as tracking a rider's own SOS, and nothing asks for it.
  - After a reconnect, the app can't learn whether a ride is active, which tier, or whether an SOS is running.
- Blocks: Reconnect (a listed phone test) and BLE-002's "reads every stored value back".
- Options: Add a "get ride" message and a "get SOS" message. Or have WELCOME carry ride and SOS state, or send RIDE_RESULT and SOS_STATUS automatically after connect.

Status (v0.3): ANSWERED — spec 7.2 On connect (after WELCOME: RIDE_STATE, SOS_STATUS if the rider's own SOS is active, SOS_RECEIVED for each active SOS from others, then STATUS), 7.4 Ride (GET_RIDE) and SOS (GET_SOS).

Status (v0.4): ANSWERED — spec 7.2 On connect and 7.4. The order of boot-time errors differs between 6.9 and 7.2: Q102.

Status (v0.5): ANSWERED — spec 7.2 Connection order (the boot-error order is settled, Q102).

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

Status (v0.4): ANSWERED — spec 7.4 Ride, unchanged.

Status (v0.5): ANSWERED — spec 7.4 JOIN_RIDE rules, with expiry and RIDE_EXPIRED added.

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

Status (v0.4): ANSWERED — spec 6.7 and 6.12 (clash counter, pending fill asks and known riders erased; paired phones, counters and display name kept) and 6.4 (End Ride during an SOS) (Q83, Q85).

Status (v0.5): ANSWERED — spec 6.7 and 6.12. Key material is now erased from every flash copy (4.9); expiry has the same effect.

**Q29. Ride QR code format.**
- Spec section: Ride QR code. RIDE-001.
- Unclear: Binary or text, the encoding of the key, name length limit, the format-version rules, and an integrity check. A key size that the QR code can hold needs to fit.
- Blocks: Phone-to-radio provisioning tests and the app. Also blocks the key size choice in Q11.
- Options: State a byte layout and a maximum total size. Add a checksum.

Status (v0.3): ANSWERED — spec 8 (text form trailbuddy://ride/ plus Base64url; byte layout; 45-byte maximum; CRC-16/CCITT-FALSE; refusal rules). The byte order of the CRC is not stated: Q98.

Status (v0.4): ANSWERED — spec 8: CRC low byte first, full CRC parameters, strict refusal rules (Q98). I recomputed the CRC (0x0B6B), the 33 bytes and the text form in 11.4, and all match.

Status (v0.5): ANSWERED — spec 8, format version 2 with a 7-day expiry. The CRC (0xB19F), the 37 bytes and the text form in 11.5 recompute correctly.

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

Status (v0.4): ANSWERED — spec 3, 6.3 and 7.3 (a 151-byte text gets TOO_LARGE; an empty text gets BAD_FIELD) (Q95). UTF-8 checks on text from the phone are not stated: Q110.

Status (v0.5): ANSWERED — spec 3, 6.3 and 7.3. Checks on text from the phone are stated (Q110).

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

Status (v0.4): ANSWERED — spec 6.3: "in contact" counted once everywhere (Q60), re-sent texts handled at the receiver (Q74). New: Delivered travels fewer hops than texts in tiers 3 and 4 (Q103) and Delivered frames arrive in a burst (Q104).

Status (v0.5): ANSWERED — spec 6.3: the Delivered count covers only riders within Delivered range (Q103), and a text is re-sent only if nobody has confirmed. New: a text sent when nobody is within Delivered range (Q127).

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

Status (v0.4): ANSWERED — spec 6.3 Stored events (event numbers survive reboot and End Ride; SOS repeats update one event; EVENTS_INFO tells the app when events were lost) (Q86).

Status (v0.5): ANSWERED — spec 6.3 Stored events, with the app's cursor rule after a factory reset (SR-20).

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

Status (v0.4): ANSWERED — spec 6.1 (radio first, phone fallback; the stopped-rider case is solved by rule 3) (Q57), 3 and 7.4 Settings (the GPS source setting is removed, Q61). A phone fix's age for presence: Q116.

Status (v0.5): ANSWERED — spec 6.1 and 7.4 Location. The age of a phone fix is answered (Q116).

**Q34. GPS duty cycle (POS-003) is not specified, and it conflicts with NEAR-003.**
- Spec section: not present. POS-003, NEAR-003, NEAR-001.
- Unclear:
  - GPS duty follows the tier, with no values.
  - Presence must be sent often while moving, which needs frequent fixes whatever the tier. A radio with no ride has no tier at all.
- Blocks: Battery targets and presence frequency.
- Options: A GPS duty table per tier plus a separate rule when presence is on. Or fix the GPS rate to the presence rate while moving, and the tier rate otherwise.

Status (v0.3): ANSWERED — spec 6.1 GPS duty (continuous while moving or presence is on; once a minute after 5 minutes stopped with presence off; moving = 5 km/h for 10 s, stopped = below 3 km/h for 30 s). It follows movement and presence, not tier, which removes the conflict with NEAR-003. Conflict with the position freshness rule: Q57.

Status (v0.4): ANSWERED — spec 6.1 GPS duty. The conflict with the freshness rule is gone: a stopped rider keeps sending at the tier interval from the latest sample (Q57).

Status (v0.5): ANSWERED — spec 6.1 GPS duty. The L76K protocol document in the repository lists only PCAS01 to PCAS04 and PCAS10 and no standby command, which agrees with spec 9; GPS duty uses each board's power or standby pin.

**Q35. Units and precision for numbers.**
- Spec section: all messages. Freeze checklist.
- Unclear: Latitude and longitude precision, accuracy (m), speed (km/h or mph, flagged by the spec itself), heading (range and "unknown" at standstill), signal strength (which measure, which unit), battery % (integer range, "unknown"), channel-busy % resolution, and time (see Q6).
- Blocks: Every field's type and size, and every byte budget.
- Options: A units table for the whole spec. Use one measurement system on the wire and convert in the app.

Status (v0.3): ANSWERED — spec 2.6 (units table with types and unknown values). Rounding when converting to wire units: Q67.

Status (v0.4): ANSWERED — spec 2.6, with rounding stated (half away from zero for coordinates, nearest for speed, nearest 1.5° step for heading). I recomputed the presence frame in 11.3 and it matches.

Status (v0.5): ANSWERED — spec 2.6 (unit table with unknown values) and 4.2 (the presence unit is now stated). The presence frame in 11.4 recomputes correctly.

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

Status (v0.4): ANSWERED — spec 6.2: a hello is also sent when the anonymous ID changes (Q76); names are 1–24 bytes everywhere (spec 2.6 and 5).

Status (v0.5): ANSWERED — spec 6.2 (at most one hello per minute in reply to requests; reply delay widens with the number of riders in contact).

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

Status (v0.4): ANSWERED — spec 7.2 (RIDERS_LIST 10 per part), 7.4 and 2.3 (tag clashes merge by radio ID) (Q72, Q93).

Status (v0.5): ANSWERED — spec 7.4 Riders (RIDER_UPDATE at most once per 10 s per rider) and 2.3 (entries merge by radio ID).

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

Status (v0.4): ANSWERED — spec 6.5: points are sorted by fix time with the outline first, duplicates are dropped by rider and fix time, received points are added to the stored history, and every part names its range, part number and total (Q84). New: what the returning radio sends when it has no history (Q108).

Status (v0.5): ANSWERED — spec 6.5, including the empty "no history" answer (Q108).

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

Status (v0.4): ANSWERED — spec 6.4: exact schedule (first send, +10 s with no jitter, then every 300 s with 0–5 s added), repeat number stops at 255, SOS_STATUS timing, resume after a reboot (Q81, Q82).

Status (v0.5): ANSWERED — spec 6.4, changed from v0.4: first send, +10 s, then every 25 s plus 0–5 s, so never more than 30 s apart (was 300 s); the repeat number is 2 bytes and stops at 65535. SOS-003 says the same. New: spec 4.7 describes the repeats differently (Q125).

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

Status (v0.4): ANSWERED — spec 6.4, unchanged. Forgery of "I'm OK" by a ride member stays on the security review list (spec 14).

Status (v0.5): ANSWERED — spec 5 and 6.4: only the owner can end an SOS, proven by a secret whose commitment is in every SOS frame; a radio never ends its own alert because of a received frame. Two gaps remain in how an alert is identified and how forged frames are detected (Q119, Q120).

**Q41. SOS received by the app.**
- Spec section: SOS_RECEIVED. SOS-001.
- Unclear: Whether each repeat reaches the app as a new message (the spec says "or repeated one"), or the radio reports each alert once and then updates. What the radio does on its own, such as the buzzer on boards that have one, is not stated.
- Blocks: App behaviour and the "phone" tests.
- Options: Report every repeat and let the app de-duplicate by alert ID. Or report once plus a "still active" notice.

Status (v0.3): ANSWERED — spec 6.4 To the app (SOS_RECEIVED for a new alert and for every repeat, the app de-duplicates by alert; SOS_ENDED when an end is heard; the buzzer sounds for a new alert on boards that have one). Buzzer behaviour: Q90.

Status (v0.4): ANSWERED — spec 6.4 To the app (SOS_RECEIVED for every repeat; the app de-duplicates by alert). Changed from v0.3: the radio no longer sounds a buzzer; the app alerts the rider (SOS-005). See Q112.

Status (v0.5): ANSWERED — spec 6.4 To the app. SOS_RECEIVED carries the first-sent time; the alert is shown by the app only (Q112).

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

Status (v0.4): ANSWERED — spec 2.5 and 6.6: presence every 10 s while moving in tiers 1 and 2 and every 20 s in tiers 3 and 4, every 60 s while stopped, 10 s with no ride, ±1 s jitter. The moving state is defined in 6.1 (Q77).

Status (v0.5): ANSWERED — spec 2.5 and 6.6, unchanged.

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

Status (v0.4): ANSWERED — spec 4.2, 4.6 and 6.6 (presence is one hop; a ride mate's presence is hidden using the anonymous ID in their hello, which is now re-sent when the ID changes) (Q76).

Status (v0.5): ANSWERED — spec 4.2, 4.6 and 6.6, unchanged.

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

Status (v0.4): ANSWERED — spec 6.6: 32 IDs with the least recently heard evicted, entries expire 5 minutes after the last accepted report, repeats within 5 s ignored, reports implying over 200 km/h rejected, with the time basis and the "rejected report doesn't replace the last" rule stated (Q78).

Status (v0.5): ANSWERED — spec 6.6: a confirmed entry heard in the last 60 s is never pushed out of the 32-entry table; unplaced IDs are counted and flagged "nearby view incomplete"; at most 4 NEARBY_RIDER messages per second.

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

Status (v0.4): ANSWERED — spec 3, 6.6 and 6.9: on by default, forced on during the rider's own SOS, on again after End Ride; turning it off stops sending only. REQUIREMENTS NEAR-005 now says the app alone makes the warning and "approaching" is deferred.

Status (v0.5): ANSWERED — spec 3, 6.6 and 6.9; NEAR-006 and NEAR-009 now say what the rider is told.

**Q46. Presence precision and privacy.**
- Spec section: Presence; NEARBY_RIDER. NEAR-002.
- Unclear: Position precision sent to anyone, heading when speed is near zero, and whether position is rounded.
- Blocks: Presence byte budget and a privacy review.
- Options: Coarsen position to a stated precision. Mark heading "invalid" below a speed.

Status (v0.3): ANSWERED — spec 4.2 (position rounded to 0.0001°, about 11 m) and 2.6 (heading is 255 below 3 km/h).

Status (v0.4): ANSWERED — spec 4.2 and 2.6, unchanged.

Status (v0.5): ANSWERED — spec 4.2, 2.3 and NEAR-002. The app tells the rider at set-up what presence reveals.

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

Status (v0.4): ANSWERED — spec 6.6 (entries expire 5 minutes after the last accepted report) and 7.2 (NEARBY_LIST 16 entries per part). Worst-case size of that part is about 0.6 KB, under the 1024-byte limit.

Status (v0.5): ANSWERED — spec 6.6 and 7.2 (NEARBY_LIST 16 per part).

**Q48. Combining ride position and presence (open question).**
- Spec section: Open questions. NEAR-002.
- Unclear: One signal with an encrypted ride part would save airtime, but NEAR-002 says presence "never carries … ride details". It's unclear whether an encrypted ride blob counts. This choice changes the envelope.
- Blocks: Envelope and message list.
- Options: Keep two separate signals in v1. Or combine, with an explicit decision that the blob is allowed.

Status (v0.3): ANSWERED — spec 6.6 last bullet (presence and ride position stay separate signals in version 1).

Status (v0.4): ANSWERED — spec 6.6 last bullet, unchanged.

Status (v0.5): ANSWERED — spec 6.6 last bullet, unchanged.

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

Status (v0.4): ANSWERED — spec 6.9 and 7.4 Settings, including the damaged-record table (Q87, Q88).

Status (v0.5): ANSWERED — spec 6.9 and 7.4 Settings.

**Q50. Display name rules.**
- Spec section: SETTINGS; RIDERS_LIST; Rider hello. MSG-003.
- Unclear: Maximum length, allowed characters, empty name, and whether a name is required before joining.
- Blocks: Byte budget for hello and the settings write.
- Options: A byte limit that fits one hello, UTF-8 or a restricted set, and a default name derived from the short ID.

Status (v0.3): ANSWERED — spec 6.2 and 7.4 Settings (1–24 bytes of UTF-8, no control characters, default "Rider " + short ID). Whether the radio checks UTF-8 validity: Q67.

Status (v0.4): ANSWERED — spec 2.6 (names 1–24 bytes, no control characters) and 4.8 check 13 for names received over the air. Checks on names entered from the phone: Q110.

Status (v0.5): ANSWERED — spec 2.6, 6.9 and 7.3. Checks on names from the phone are stated (Q110).

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

Status (v0.4): ANSWERED — spec 6.8 (a table of what a repeater does and does not do) (Q89).

Status (v0.5): ANSWERED — spec 6.8. A repeater opens the pairing window only on a button press, like any radio.

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

Status (v0.4): ANSWERED — spec 7.4 Update rules (Q97): signature in START_UPDATE, refusals with codes, 60 s chunk timeout, SOS rules, confirm within 60 s or roll back, result in the next WELCOME. The unsigned USB recovery route is not reflected in DFU-001 or POLICY §12: Q109.

Status (v0.5): ANSWERED — spec 7.4 Updates. The unsigned USB route is settled: shipped radios have none (Q109).

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

Status (v0.4): ANSWERED — spec 10 and 7.4 Diagnostics (counter definitions, Q96). New: how "received" is counted per kind (Q107).

Status (v0.5): ANSWERED — spec 10 and 7.4 Diagnostics. Counters per kind are counted after authentication (Q107).

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

Status (v0.4): PARTLY ANSWERED — Answered by spec 9, POLICY §8 and §14, and HW-001: the lead board is the RAK19026 VC base board with the RAK4631 core; version 1 has three boards (RAK19026 VC, Heltec T114 V2, Elecrow ThinkNode M1); Zephyr's RAK4631 and Heltec T114 V2 board definitions may be starting points (POLICY §14 now names them); version 1 uses each board's user button, GNSS and one status light, with no buzzer or screen; the Pocket V3 waits until it ships. Documents now in `docs/hardware/`: RAK19026 VC datasheet, RAK19007 datasheet (not a version 1 board), Heltec T114 V2.0 schematic, ThinkNode M1 datasheet V1.0 (16 April 2025), and the RAK19026 profile. Still missing: the three board profiles are not written; none of the eight open gaps in the RAK19026 profile is closed by the new documents (see Q114 and Q115 for what the new documents leave open); spec 9 lists a Heltec "datasheet" and a ThinkNode "schematic" that are not in the repository; the RAK4631 module documentation and every GNSS module manual are not in the repository. Spec 9 picks LED 1 as the RAK19026 status light, and the profile says the datasheet disagrees with itself about what LED 1 is. Not marked answered.

Status (v0.5): PARTLY ANSWERED — spec 9 now has a profile-source table and a role-by-board table; POLICY §8 and §14 and HW-001 agree. New documents in `docs/hardware/`: the Zephyr T114 V2 board page (confirms USER button SW1 = P1.10 and green LED4 = P1.03, which match my reading of the schematic), the VAE antenna specification (peak 2.39 dBi, 1.77 dBi at 915 MHz: both recomputed from its gain table and correct), a retailer page for the Heltec L76K GNSS module (9600 baud, 3.3 V, built-in antenna), the Quectel L76K protocol specification, the Waveshare L76K wiki and the u-blox ZOE-M8 datasheet. Not closed: the three board profiles are still unwritten; none of the eight open gaps in the RAK19026 profile is closed by a document (buzzer and power button are settled by decision, not by a document); spec 9 names documents that are not in the repository (Q130). Not marked answered.

## J. Process

**Q55. Freeze checklist and traceability.**
- Spec section: "Before this spec can be frozen". REQUIREMENTS intro.
- Unclear:
  - None of the 49 messages meets required/optional, size, units, or sample values.
  - REQUIREMENTS says every protocol decision cites a requirement, a rationale and a test. The spec has no rationale or test column.
- Blocks: Freeze, and TEST-MATRIX.md.
- Options: One table per message with field, type, unit, required or optional, maximum size and sample value. Add a test reference per message.

Status (v0.3): PARTLY ANSWERED — spec 7.4 (R/O marking, units from 2.6, list limits, a requirement ID per message), 4 and 5 (byte sizes) and 11 (samples). Missing: samples for 33 of the 43 phone-link messages (promised for v0.4, spec 14); maximum encoded sizes for phone-link messages; the rationale and test reference that REQUIREMENTS says every protocol decision carries. Note: "None of the 49 messages" in Q55 should read 40 for v0.2 (as the Review basis says); v0.3 has 43, which matches the table and spec 15.

Status (v0.4): PARTLY ANSWERED — spec 11.5 now holds a sample for all 44 phone-link messages, and 11.1 to 11.4 hold vectors. Missing: (a) the largest encoded size of each phone-link message (a full TRACK_POINTS part may be over 1024 bytes: Q105); (b) the rationale and test reference that REQUIREMENTS says every protocol decision carries; (c) the new requirements MESH-005, SOS-005 and HW-001 are not cited by any spec section (Q111).

Status (v0.5): ANSWERED — spec 11.7 holds a sample for all 46 phone-link messages (checked against the 7.4 tables); the clean team encodes each once and saves the bytes in `spec/vectors/` for the spec team to check; worst-case sizes are recorded in DECISIONS.md and tested (7.1); MESH-005, SOS-005 and HW-001 are now cited (Q111). The wording that every protocol decision carries a rationale is met only where the text gives a reason.

**Q56. Test vectors and review.**
- Spec section: not present. POLICY §12, §17 step 2, §19 (`spec/`).
- Unclear: Build step 2 needs "all spec test vectors". The spec doesn't say who produces them, from what inputs, or how the clean team can use them without writing them from its own implementation. It also doesn't say when the independent design review happens.
- Blocks: Build step 2 and CI conformance.
- Options: The spec team supplies vectors with the frozen spec. The review is completed before freeze.

Status (v0.3): PARTLY ANSWERED — spec 11 (the spec team supplies exact vectors for key derivation, a position frame, a presence frame and the QR code; phone-link vectors are recorded by the clean team in spec/vectors/ and checked by the spec team; 11.6 requires the independent security review before freeze). Missing: vectors for the other eight ride kinds (hello, text, SOS and fill in particular), for a relayed frame (hops changed, same tag), for rider tag clash counter 1, for the short ID, and for the replay and duplicate windows. I did not recalculate any vector in this review; their layouts and lengths match spec 4, 5 and 8.

Status (v0.4): PARTLY ANSWERED — spec 11.1 to 11.4 now cover hello, text, SOS, SOS with no position, fill ask, fill answer, a relayed frame, rider tag clash counter 1 and the short ID. I recomputed key derivation, tags, the check value, the short ID, all eight ride frames, the presence frame and the QR code, and all match. Missing: vectors for the three kinds with none (hello request 0x03, Delivered 0x05, SOS reply 0x07); vectors for replay, duplicate and receive-check rejections; the independent review (11.6) is still to be arranged.

Status (v0.5): ANSWERED — spec 11.1 to 11.6 now cover every kind (including hello request, Delivered and both SOS replies), the join ID, four rejection cases with bytes, and a real Ed25519 signature. I recomputed all of them and they match. The "sent as you" and "wrong secret" rejection cases are described but have no bytes. The reviewer must also recompute them (11.8).

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

Status (v0.4): ANSWERED — spec 6.1: rule 3 uses the newer of the radio and phone fixes up to 10 minutes old, with its true fix time; a stopped rider keeps sending at the tier interval, so Fill on Return is not triggered by silence.

Status (v0.5): ANSWERED — spec 6.1, unchanged.

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

Status (v0.4): ANSWERED — spec 3, 6.5: "filled" means the outline of the whole gap (up to 30 minutes) within 60 s and the full detail within 3 minutes; one ask covers up to 8 riders; fill rate is 1 frame per 3 s below 15% busy and 1 per 10 s from 15% to 25%. Above 25% busy, or while any SOS is queued, no fill is sent, so the target can't be met in that case (see Q104).

Status (v0.5): ANSWERED — spec 3 and 6.5: the target applies below 15% busy; between 15% and 25% fill is slower; above 25% only outline parts go, at one frame per 30 s; at slow-down step 3 fill stops. Burst behaviour is in Q104.

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

Status (v0.4): ANSWERED — spec 4.5: class 0 holds 64 entries; replies are keyed on alert owner, alert ID, sender and reply value; when full, the oldest "Copy that" is dropped, never an SOS, "I'm OK" or "left the ride" reply. SOS-002 now says so.

Status (v0.5): ANSWERED — spec 4.5: the radio's own SOS and replies always have queue space and are sent first; replies are keyed on owner, alert, sender and reply value; when the relay part is full, relayed entries are dropped, oldest "Copy that" first. New: Q118.

**Q60. "Heard recently" has three different windows, and it is unclear who is counted.**
- Spec section: 6.3 (Sender's side), 7.4 Status (ride contact), 7.4 Riders (in contact). MSG-002.
- Unclear:
  - Delivered counts are "out of the ride members heard in the last 10 minutes". Ride contact counts members heard in the last 2 × the position interval. RIDER_UPDATE reports contact lost after 3 × the interval. The same moment can show 3 in contact in STATUS and 4 in RIDERS_LIST.
  - For "Delivered to 3 of 4" (MSG-002, and the SEND_RESULT sample in 11.5: 3 confirmations, 4 ride members): does "4" include the sender? In a ride of 4, only 3 others can confirm.
  - Do repeaters count (see Q89)? Does a rider silent for 11 minutes drop out of the total?
- Blocks: SEND_RESULT, STATUS and RIDER_UPDATE tests, and the app's wording.
- Options: One definition of "in contact" used everywhere. State the total as "others in contact, not counting the sender", or "all known ride members".

Status (v0.4): ANSWERED — spec 6.3: one definition of "in contact" (heard within 3 × the position interval, at least 60 s), used by STATUS, RIDERS_LIST and Delivered; the Delivered total excludes the sender and repeaters.

Status (v0.5): ANSWERED — spec 6.3, unchanged.

**Q61. The GPS source preference setting contradicts the fixed source rule.**
- Spec section: 3 (GPS source), 6.1 (Source, decision), 7.4 Settings (setting 4), 6.7.
- Unclear: Sections 3 and 6.1 give one rule: radio GPS first, phone as fallback. Setting 4 offers RADIO_FIRST or PHONE_FIRST, and no section says what PHONE_FIRST does (when the phone's fix is fresh enough, what happens to presence).
- Blocks: settings tests (build step 10) and position source tests (build step 4).
- Options: Describe PHONE_FIRST as the mirror of the 6.1 rule. Or remove setting 4 from version 1.

Status (v0.4): ANSWERED — spec 3 and 7.4 Settings: radio GPS first, phone as fallback, no setting; ID 4 is removed and never reused.

Status (v0.5): ANSWERED — spec 3 and 7.4 Settings, unchanged.

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

Status (v0.4): ANSWERED — by the three v0.4 documents together. (1) NEAR-002 now says fix age. (2) NEAR-004 now lists what is left out, matching spec 6.6. (3) POLICY §10 and §18 now use the channel check and the 30% budget; REG-001 says version 1 is United States only. (4) POLICY §12 now says the encryption choices are decided for building and only the security review can change them. (5) A "which document wins" rule is in all three documents: none wins, write a question, build nothing that depends on it, and for SOS the safer reading applies. The hello name length is 1–24 everywhere. This settles all five items in TODO.md Q62; the owner records the decisions and the provenance-log entry. One small mismatch remains: spec 12 does not list SOS-005, MESH-005, HW-001, or the MESH-003, MESH-004, RPT-001 and NEAR-005 edits (Q111).

Status (v0.5): ANSWERED — and still answered by the v0.5 documents. The rule (none wins; write a question; SOS safer reading) is in all three. I checked that every requirement ID the spec cites exists in REQUIREMENTS, and that SOS-005, MESH-005 and HW-001 are now cited and listed in spec 12 (the one mismatch noted in v0.4 is gone). TODO.md Q62: the owner still records the decisions and the provenance-log entry.

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

Status (v0.4): PARTLY ANSWERED — spec 13 approves 15 standards by name, with a defining document for each (CRC-16/CCITT-FALSE is fully defined in spec 8), and POLICY §14 now allows them, to be read only from copies in `docs/standards/`. Still open: `docs/standards/` does not exist and none of the documents is in the repository; SOURCE-REGISTER.md does not exist, so no approval is logged; whether the clean team may use only the published standard, or also reference code or test vectors from a library, is not stated; the product owner has not yet confirmed that the chosen crypto library supports AES-128-CCM, HKDF-SHA256 and Ed25519 (spec 14). TODO.md Q63 stays open.

Status (v0.5): PARTLY ANSWERED — spec 13 approves 15 standards; POLICY §14 allows reading them only from `docs/standards/`; spec 14 and POLICY §15 now say what to check about the crypto library, and the security review (SR-21) found Nordic's documentation shows support for all four algorithms but could not confirm the exact parameter set. The Nordic 5-Clause licence copy is now in the repository, in `license/` (POLICY §19 says `docs/licences/`: Q131). Still open: `docs/standards/` does not exist and holds none of the documents; SOURCE-REGISTER.md does not exist; whether the clean team may use only the published standard or also a library's reference code or vectors is not stated; the library and signature-checking bootloader are unconfirmed (build step 2 and step 11). TODO.md Q63 stays open.

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

Status (v0.4): ANSWERED — spec 4.7: one channel activity check before every transmission; busy gives a random wait of 0 to 50 ms × 2^k (k 1 to 5) and a drop after 8 busy checks (counted "gave up"); an SOS waits a random 0–100 ms once and then transmits whether or not the channel is clear. The CAD detection settings are left to the radio HAL.

Status (v0.5): ANSWERED — spec 4.7: one channel check; normal frames back off 0–50 ms × 2^k and are dropped after 8 busy checks; an SOS re-checks every 20–100 ms and transmits after at most 1 s whether or not the channel is clear.

**Q65. How "channel busy" is measured.**
- Spec section: 7.4 Status (Channel busy), 6.5 rule 5, 2.1. MESH-004, NFR-001, DIAG-001.
- Unclear: "The share of the last 60 s that the radio spent receiving or detecting LoRa activity, or transmitting." What counts as activity: a preamble heard, a valid packet, signal strength over a threshold, or other signals on the channel? How often the reading is refreshed (fill needs a reading under 60 s old). The value before the radio has run 60 s. Whether time spent on frames from other rides and on presence frames counts (it should, for the 30% check).
- Blocks: the MESH-004 test, Fill on Return rule 5, the 60-minute busy history and the 30% field check.
- Options: Count only time spent receiving valid Trail Buddy frames and transmitting, updated each second. Or sample signal strength against a stated threshold. State the update period and the value for the first minute.

Status (v0.4): ANSWERED — spec 6.10: the radio adds up transmit time, receive time of every LoRa frame with a valid header (any ride, presence, or other) and 10 ms per unexplained channel-check hit; the value is the percentage over the last 60 s, updated every second, and 255 for the first 10 s.

Status (v0.5): ANSWERED — spec 6.10, unchanged.

**Q66. Scope of the 30% airtime budget, and what the radio does above it.**
- Spec section: 2.1 (Airtime budget), 2.5 (Airtime check). NFR-001. POLICY §18.
- Unclear:
  - The 2.5 table counts only the radios of one ride. All Trail Buddy radios share one channel, so other rides' traffic and every nearby rider's presence add to the load (two groups at a trailhead, for example).
  - The table leaves out hellos and their replies, texts, Delivered (up to one per rider per text), Fill on Return traffic and SOS.
  - It is not said what a radio does when measured busy goes above 30%: report only, slow its own position and presence, or nothing.
- Blocks: the simulator scenarios in build step 5 and the pass criteria "channel at most 30% busy".
- Options: State the assumed density (number of groups in range) and add the missing kinds to the table. State whether 30% is a design target checked in the simulator or a runtime limit, and if runtime, which traffic slows first.

Status (v0.4): ANSWERED — spec 2.5 (airtime table now includes hellos, texts and Delivered, and fill; one ride on its own is 15–25% in every tier; two rides plus 10 riders go over 30%) and 6.10 (runtime rule: slow presence, then positions, then stop fill; undo below 20%; SOS never slowed). I recomputed the table from the airtime formula and it matches within rounding. The average does not show bursts: Q104.

Status (v0.5): ANSWERED — spec 2.5 and 6.10. I recomputed the table from the airtime formula: positions, presence, hellos, texts with Delivered, and the SOS add-on (about 9%) all match; the fill column (0.7%) I checked only for plausibility. Two rides plus 10 riders still go over 30%, which 6.10 handles. Bursts are not shown by the average (Q104).

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

Status (v0.4): ANSWERED — spec 4.8: 14 numbered checks in a fixed order, each with its drop reason; presence uses checks 1–3 and 10 plus 6.6.

Status (v0.5): ANSWERED — spec 4.8: 18 numbered checks in a fixed order (was 14), including a maximum length before decryption, reserved sender tag, class matches kind, value ranges and the "anything not listed is invalid" rule.

**Q68. Relay delay and suppression: the wording is ambiguous.**
- Spec section: 4.6. MESH-003.
- Unclear:
  - "Waits a random time", "the wait window is 1000 ms, scaled" and "a random 0–20% is added on top": is the delay uniform between 0 and the scaled window, with 0 to 20% of the window added? Is there a minimum delay?
  - Which SNR is used: this reception, or the best or first of several copies heard? SNR is in dB for the scale and in quarter-dB steps on the wire.
  - "Hears the same frame relayed by 2 other radios": does the copy the radio first received count? Does a copy with fewer hops remaining count?
- Blocks: build step 5 relaying and the simulator figures; the "4 transmissions per position" assumption in 2.5.
- Options: Give the delay as a written range with the distribution stated, and say which copies are counted for suppression.

Status (v0.4): ANSWERED — spec 4.6: delay = 20 ms + s × 1000 ms × u + 200 ms × v, s from the SNR of the first copy, suppression after 2 further receptions of the same identity. The first copy does not count.

Status (v0.5): ANSWERED — spec 4.6, unchanged for normal frames.

**Q69. Queues: the radio's own frames and relayed frames.**
- Spec section: 4.5, 4.6, 6.3.
- Unclear:
  - Whether relayed frames use the same queues as the radio's own. The position queue is "1 per rider": does that include other riders' relayed positions?
  - For text, "new text refused with BUSY" is a phone-link error. For a relayed text with the queue full, "refused" must mean dropped. Relays can fill the 8 entries so the rider's own text gets BUSY.
  - Delivered and Fill drop the "oldest" entry, which could be the radio's own.
  - Whether a relay cancelled by suppression leaves the queue, and whether queue-full drops are counted (DIAGNOSTICS has a "queue full" counter).
- Blocks: the queue implementation and the load tests.
- Options: Separate own and relay queues, or tag each entry and state the rule for each case.

Status (v0.4): ANSWERED — spec 4.5: each class states own and relay queue space and what happens when full; a suppressed relay leaves its queue; every full-queue drop is counted.

Status (v0.5): ANSWERED — spec 4.5, with the class-0 changes above.

**Q70. Duplicate memory: 256 entries or 10 minutes?**
- Spec section: 4.4. MESH-002.
- Unclear:
  - "The last 256 identities for 10 minutes": when more than 256 frames arrive within 10 minutes (30 riders sending positions plus texts, Delivered and fill frames can), which limit wins?
  - The replay window (64 per sender) already rejects a counter seen twice. What does the identity memory add?
  - Suppression needs duplicates counted while a relay waits. Is a duplicate counted before it is dropped?
  - Are the radio's own frames entered, so an echo is not relayed again?
- Blocks: duplicate unit tests and memory sizing.
- Options: 256 entries with the oldest evicted, plus expiry at 10 minutes. State that own and relayed frames are entered.

Status (v0.4): ANSWERED — spec 4.4: up to 256 identities, each expires after 10 minutes, the oldest is evicted when full; the radio's own frames are entered; a duplicate is counted for suppression before it is dropped.

Status (v0.5): ANSWERED — spec 4.4, unchanged in substance.

**Q71. The frame counter ceiling at run time, flash failure, and "factory reset".**
- Spec section: 4.4, 2.3, 6.7.
- Unclear:
  - 4.4 gives the boot rule. When the counter reaches the stored ceiling while running, must the next ceiling be written before the frame is sent?
  - If the flash write fails, does the radio stop sending ride frames, and what is reported (STORAGE_FAILED)? If the stored ceiling is damaged at boot?
  - 2.3 says the radio ID changes "only on factory reset". Nothing defines a factory reset: how it is triggered and what it erases (settings, ride, paired phones, counter ceiling, radio ID).
- Blocks: build step 2 (a nonce must never repeat), step 10 persistence and the power-loss hardware tests.
- Options: Write the next ceiling before any frame at the current ceiling is sent, and stop sending ride frames if the write fails. Define factory reset as a table of what it erases.

Status (v0.4): ANSWERED — spec 4.4 (next ceiling written before the counter gets within 64 of it; on a failed write the radio stops sending ride frames and reports STORAGE_FAILED) and 6.11 (factory reset erases the radio ID, settings, ride, paired phones, events, history, counter ceiling, ID counters and event number).

Status (v0.5): ANSWERED — spec 4.4 and 6.11: two copies, the higher valid one wins, written and read back before use, a missing ceiling counts as damaged, fixed format for every image, never wraps, and a flash wear budget the clean team records. SOS still goes out if storage fails.

**Q72. Rider tag: reserved value, clash handling and the 30-rider limit.**
- Spec section: 2.3 (Rider tag clash), 2.4, 4.1 (destination 000000), 4.3, 7.4 Riders. RIDE-004.
- Unclear:
  - A rider tag of 000000 would read as "whole ride". Is it reserved, and is a tag derived as 000000 re-derived with the next clash counter?
  - Where the clash counter is stored, and whether End Ride resets it.
  - After a clash the radio has a new tag. Other radios see a new rider tag in a hello with a radio ID they already know. Riders are never removed, so the old tag stays in RIDERS_LIST and the list can show one rider twice and pass 30.
  - Nothing stops a 31st radio joining with the same QR code. What do the others do with frames from a 31st tag?
- Blocks: riders list and RIDER_UPDATE tests, rider tag unit tests, RIDE-004.
- Options: Reserve 000000. Merge entries that share a radio ID. State the 31st-rider rule (ignored and not listed, or listed and flagged).

Status (v0.4): ANSWERED — spec 2.3 and 2.4: 000000 is reserved; a clash makes the radio re-derive its tag with the next clash counter and send a hello at once; the counter is stored with the ride and reset at End Ride; entries merge by radio ID; a 31st radio works and is flagged. New: frames sent before a clash is noticed can reuse a nonce (Q100).

Status (v0.5): ANSWERED — spec 2.3 and 4.4: the greater radio ID changes tag; a radio with an active SOS never changes; one change per 10 minutes. Nonce safety no longer depends on tags being unique (Q100).

**Q73. Short ID derivation.**
- Spec section: 2.3. MSG-003.
- Unclear: "The first 15 bits of the radio ID, written as 3 characters from A–Z and 2–7." Which bit comes first (most significant bit of the first byte?), and which character stands for each 5-bit value (A to Z as 0 to 25 and 2 to 7 as 26 to 31?). There is no sample.
- Blocks: WELCOME, RIDERS_LIST, the default display name, and agreement between the radio and the app.
- Options: State the bit order and the character order, and add the short ID of the radio ID in 11.1 to Section 11.

Status (v0.4): ANSWERED — spec 2.3: first 15 bits, most significant bit first, three 5-bit groups, characters from ABCDEFGHIJKLMNOPQRSTUVWXYZ234567; sample CER in 11.1 (recomputed, matches).

Status (v0.5): ANSWERED — spec 2.3, unchanged.

**Q74. Message IDs, alert IDs and re-sent texts at the receiver.**
- Spec section: 5 (Message IDs and alert IDs), 6.3, 4.4, 6.4.
- Unclear:
  - Message and alert IDs are 16-bit counters starting at 1. Do they survive a reboot? If they restart at 1, a receiver holding (sender, ID 1) may treat a new text as a re-send, and a new alert may be taken for an old ended one.
  - A re-sent text has the same message ID and a new frame counter. A receiver that already holds it: does it store it again, tell the app again, send Delivered again (needed when the first Delivered was lost)? How long does it remember a (sender, message ID) pair?
  - Is Delivered counted once per confirming rider if it arrives twice?
  - Behaviour at wrap after 65535.
- Blocks: MSG-002 tests, stored events, SOS ID handling.
- Options: Keep the counters in storage, or start them at a random value. A receiver keeps recent (sender, ID) pairs with an age. A re-sent text creates no new event but is confirmed again.

Status (v0.4): ANSWERED — spec 5 and 6.3: 16-bit counters kept in flash, written 16 ahead, start at 1, skip 0, wrap 65535 to 1; a receiver remembers (sender, message ID) for 30 minutes, creates no new event for a re-send, and sends Delivered again.

Status (v0.5): ANSWERED — spec 5 and 6.3, unchanged.

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

Status (v0.4): ANSWERED — spec 2.2: numbered rules for setting the clock (GPS when it differs by over 1 s; else the phone's time after 10 minutes with no fix), GPS to UTC conversion, drift of 2 s per hour, unknown time sent as 0, backward steps never affect intervals.

Status (v0.5): ANSWERED — spec 2.2 (see Q6). New: Q124.

**Q76. Anonymous ID change: a hello on change, and a limit on how long it can be deferred.**
- Spec section: 2.3, 6.2, 6.6. NEAR-002.
- Unclear:
  - A ride member's hello carries their anonymous ID so ride radios can hide them from Nearby. 6.2 does not list a hello as a trigger when the ID changes (every boot and every 6 hours). Until the next hello, up to 10 minutes, a rider's own ride mates see them as a nearby rider, and the app could warn of an "approaching rider" who is in the ride.
  - The change is applied only while stopped, with no upper limit. A radio that never stops, or never has a fix, keeps one ID for more than a day, against "changes at least daily" in NEAR-002.
  - Are the "6 hours of operation" counted in uptime, including sleep, and does the timer restart at each change?
- Blocks: the NEAR-002 test, privacy, and the app's approaching-rider logic.
- Options: Send a hello at once after every change. Apply the change after a stated maximum time even if moving, accepting that one encounter may show twice.

Status (v0.4): ANSWERED — spec 6.2 and 6.6: a hello is sent as soon as the anonymous ID changes; a change is always applied within 24 hours of the last one even while moving. Still unstated: whether the 6-hour uptime count runs through any sleep (Q113).

Status (v0.5): ANSWERED — spec 6.2 and 6.6. The radio never sleeps, so uptime counts continuously (Q113).

**Q77. Moving and stopped state at boot and without a fix.**
- Spec section: 6.1 (moving and stopped thresholds), 6.6.
- Unclear: The starting state at boot. The state between 3 and 5 km/h (keeps the last?). The state with no fix, or with a phone position that has no speed. Whether the GPS duty and the presence interval use the same state.
- Blocks: POS-003 and NEAR-003 tests.
- Options: Start as moving, keep the previous state between the thresholds, and treat unknown speed as moving.

Status (v0.4): ANSWERED — spec 6.1: moving is 5 km/h for 10 s, stopped is below 3 km/h for 30 s, the previous state holds in between, and the state is "moving" at boot and when speed is unknown. GPS duty and presence use the same state.

Status (v0.5): ANSWERED — spec 6.1, unchanged.

**Q78. Presence: position source, fix age, and the basis for the plausibility check.**
- Spec section: 4.2, 6.1, 6.6, 7.4 (SHARE_MY_LOCATION). NEAR-002.
- Unclear:
  - Which fix presence uses: the radio's GPS only, or the phone's when the radio has none. Heading and speed are optional in SHARE_MY_LOCATION; when absent, presence sends 255. A fix age for a phone fix needs the radio clock, which can be unknown.
  - The 200 km/h check: distance over which time, the time between receipts or corrected by each report's fix age? Does a rejected report become the "last report" for the next test? The first report from an ID has nothing to compare with.
- Blocks: build step 8 tests and NEAR-008 behaviour.
- Options: State the source order, the unknown-value rules and the time basis. State that a rejected report does not replace the last accepted one.

Status (v0.4): ANSWERED — spec 6.6: presence uses the radio's fix if no older than 10 s, else the phone's; fix age from the uptime counter; plausibility compares each report with the last accepted one, and a rejected report doesn't replace it. A phone fix's age is measured from when SHARE_MY_LOCATION arrived (Q116).

Status (v0.5): ANSWERED — spec 6.6. The age of a phone fix is answered in 6.1 (Q116).

### C. Behaviour

**Q79. SOS when the radio has no ride.**
- Spec section: 5 (SOS is a ride frame), 7.3 (NO_RIDE), 7.4 SOS. SOS-001 to SOS-004, RIDE-001.
- Unclear: SOS frames are encrypted with the ride key, so a radio with no active ride can neither send nor hear an SOS. The spec does not say SEND_SOS needs a ride, and does not list NO_RIDE for it. Is it intended that a rider outside any ride has no SOS?
- Blocks: SEND_SOS handling, the field test and the safety case. This is a product decision.
- Options:
  - SOS requires a ride; SEND_SOS without one returns NO_RIDE and the app says so.
  - Add an SOS frame that needs no ride key and is heard by every Trail Buddy radio. That changes Section 4 and the security review.

Status (v0.4): ANSWERED — spec 3 and 6.4 (Decision): an SOS needs a ride; SEND_SOS with no ride gets NO_RIDE and the app says "Create or join a ride to use SOS".

Status (v0.5): ANSWERED — spec 3 and 6.4 (NO_RIDE), SOS-001.

**Q80. SOS when there is no position fix.**
- Spec section: 2.6 ("a position is never sent without a fix"), 5 (SOS fields), 7.4 SEND_SOS. SOS-001.
- Unclear: SEND_SOS may carry no position, and the radio may have no fix and none from the phone. The SOS frame has no "unknown" value for latitude or longitude. Is the SOS sent at once with the last known position and its true fix time, held until a fix arrives, or sent with an unknown marker? How old may a last known position be? How does the receiving app show a missing position?
- Blocks: SOS-001 tests and the safety case.
- Options: Send at once with the last known position and its real fix time, and replace it on later repeats. Or add an explicit "no position" marker. Or hold until a fix, with a stated limit.

Status (v0.4): ANSWERED — spec 6.4 (Decision): an SOS is never held back. It uses the best position, else the last known position of any age with its true fix time, else the no-position marker (0x7FFFFFFF, accuracy 255, fix time 0). The no-position SOS vector in 11.2 recomputes correctly.

Status (v0.5): ANSWERED — spec 6.4. The no-position SOS frame in 11.2 recomputes correctly.

**Q81. SOS schedule details, and an SOS across a reboot.**
- Spec section: 6.4, 4.4, 5. SOS-003.
- Unclear:
  - "Every 5 minutes after the first send": at 300 s, 600 s and so on, measured from the first send? Does the up-to-5 s jitter apply to the repeat at 10 s? Is jitter only added, or added and subtracted?
  - The repeat number is one byte: after 255 repeats (about 21 hours) does it wrap or stay at 255?
  - If the radio reboots or loses power during an active SOS (battery swap, brownout, crash), does it resume repeating? If so, the alert ID, repeat number and first-sent time must be stored, and how often.
- Blocks: build step 7, the power-loss hardware test, SOS-003 test.
- Options: State the schedule as send times from the first send. State that an active SOS resumes after a reboot and how often its state is stored.

Status (v0.4): ANSWERED — spec 6.4: schedule fixed (t0, t0 + 10 s, then t0 + 300 s × n with 0–5 s added), repeat number stops at 255, and an active SOS is stored in flash at the start and at every repeat and resumes after a reboot.

Status (v0.5): ANSWERED — spec 6.4: stored at the start and at least once a minute; sends a repeat as soon as it is running again after a reboot.

**Q82. SOS_STATUS timing, the reply to SEND_SOS, and the Copy that list.**
- Spec section: 7.4 SOS, 7.2 (On connect), 6.4.
- Unclear: When the radio sends SOS_STATUS without a request (each repeat, each state change, each Copy that, never). What SEND_SOS is answered with. How "copy-that riders" is built (distinct riders who replied for this alert) and when it is cleared. "Last sent" when the clock is unknown. "Times sent" is a u8 (wrap).
- Blocks: the phone tests for SOS and the app.
- Options: SOS_STATUS after every send, repeat, Copy that and end. SEND_SOS answered by SOS_STATUS, or by ERROR.

Status (v0.4): ANSWERED — spec 6.4: SEND_SOS is answered by SOS_STATUS; SOS_STATUS follows every send, repeat, new Copy that and the end; the Copy-that list holds distinct riders and clears at the end; "last sent" is 0 when the clock is unknown; times sent is 16 bits.

Status (v0.5): ANSWERED — spec 6.4 SOS_STATUS, unchanged in substance.

**Q83. End Ride during the rider's own SOS: how reliable is the "left the ride" reply?**
- Spec section: 6.7, 6.4 (Ending), 4.5.
- Unclear: END_RIDE sends one SOS reply "ended because the owner left the ride", then erases the key and drops queued ride frames. Is the reply sent once, or twice like an SOS? The key must stay until the reply has actually gone out (the channel may be busy). If the reply is lost, other radios show the SOS forever, because an SOS is never ended for lack of signal. When does RIDE_STATE (not active) reach the app: at once, or after the reply is sent?
- Blocks: End Ride and SOS-004 tests.
- Options: Send the reply twice, 10 s apart, and erase the key after the last send or after a stated timeout. Report RIDE_STATE after the erase.

Status (v0.4): ANSWERED — spec 6.4: the "owner left the ride" reply is sent twice, 10 s apart; the key is kept until both are sent or 60 s pass; RIDE_STATE (not active) follows the erase.

Status (v0.5): ANSWERED — spec 6.4: "left the ride" and "I'm OK" are each sent twice, 10 s apart, with the owner secret; the key is kept until both are sent or 60 s pass; a factory reset during an SOS does the same first. Damaged-record cases are in Q121.

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

Status (v0.4): ANSWERED — spec 5 and 6.5: the answer carries its range, part number (outline flag in bit 7) and total parts; outline then detail, with the outline's points repeated in the detail; the gap is the latest 30 minutes when longer; "answered" means every part of the detail pass was received; return is any authenticated frame after the silence; 50 m is measured between the last position before and the first after.

Status (v0.5): ANSWERED — spec 5 and 6.5, unchanged in substance.

**Q85. What is stored, where, and what survives a reboot.**
- Spec section: 6.5 (2-hour history), 6.3, 6.4, 6.2, 6.7. POLICY §10 (Settings and storage), §18.
- Unclear: The spec says the ride, settings, repeater mode and the counter ceiling persist, and that replay and duplicate records do not. It does not say for: own and heard position history (RAM or flash; at one point per 10 s, flash wear matters), stored events and event numbers, the known riders (names, radio IDs), a running SOS (Q81), the ID counters (Q74), paired phones, pending fill asks, and the clash counter. Capacity is also open: heard positions for 30 riders over 2 hours is thousands of points, with no per-rider limit or thinning rule. Own history is "at least one point every 10 s" while the GPS may sample once a minute (Q57).
- Blocks: build steps 9 and 10, memory sizing and the power-loss hardware tests.
- Options: A table of each record with its medium, whether it survives a reboot, its capacity and its eviction rule. State what End Ride erases from the same table.

Status (v0.4): ANSWERED — spec 6.12: a table of every record with its medium, survival of reboot, capacity, eviction and whether End Ride erases it.

Status (v0.5): ANSWERED — spec 6.12 (updated for the join ID, owner secret and ended alerts).

**Q86. Stored events: numbering, SOS repeats, and "events waiting".**
- Spec section: 6.3 (Stored events), 7.2 (On connect), 7.4 (GET_MISSED; STATUS events waiting).
- Unclear:
  - "Events waiting" counts stored events "not yet sent to the app". There is no acknowledgement, and the connect sequence in 7.2 does not push stored events. When does an event count as sent?
  - SOS_RECEIVED is sent "for every repeat" with an event number. Does each repeat create a new stored event? SOS events are never evicted, so the count could pass 200.
  - Do event numbers continue after a reboot and after End Ride?
  - GET_MISSED with a number older than the oldest retained event: how does the app learn events were lost?
- Blocks: GET_MISSED and reconnect tests, and the STATUS events-waiting field.
- Options: An event is "sent" when written to a connected app. Repeats update the existing SOS event. Event numbers never go back. Add a "first available event number" to the replay.

Status (v0.4): ANSWERED — spec 6.3: an event counts as sent when written to a connected app; SOS repeats update one event; event numbers never go back and survive End Ride; EVENTS_INFO gives the oldest and newest held.

Status (v0.5): ANSWERED — spec 6.3, unchanged in substance.

**Q87. Settings: how they interact with SOS, repeater mode and End Ride.**
- Spec section: 7.4 Settings, 6.6, 6.7, 6.8.
- Unclear:
  - During the rider's own SOS, presence is forced on. If setting 2 is changed to off then, is it stored and not effective, or refused? SETTINGS returns the stored value while STATUS reports presence sending as on. Does it revert to the stored value when the SOS ends?
  - Is repeater mode allowed with no ride (NO_RIDE?) or during the rider's own SOS?
  - End Ride sets presence on and repeater mode off: these are writes that can fail. What is reported?
- Blocks: settings tests in build step 10.
- Options: State each case as a rule with its error code.

Status (v0.4): ANSWERED — spec 6.8 and 6.9: repeater mode gets NO_RIDE with no ride and SOS_ACTIVE during the rider's own SOS; presence turned off during an SOS is stored and takes effect when it ends; a failed write at End Ride is reported as STORAGE_FAILED in RIDE_STATE.

Status (v0.5): ANSWERED — spec 6.8 and 6.9.

**Q88. Damaged records at boot, and when SETTINGS_RESET is reported.**
- Spec section: 7.4 Settings (Rules), 7.2 (On connect), 4.4, 6.8.
- Unclear: SETTINGS_RESET is reported "on the next connection": as an ERROR message, and at which point in the connect sequence in 7.2? If the ride record is damaged, is the ride erased, and how does the app learn (RIDE_STATE with no ride, or an error)? What happens if repeater mode was on, or an SOS was active?
- Blocks: the power-loss hardware tests and BLE-002 ("no settings write can leave the radio unable to boot").
- Options: List each record, what a damaged copy does, and what is reported and when.

Status (v0.4): ANSWERED — spec 6.9 table: each damaged record, what the radio does and which error is reported. When the error is sent is described differently in 6.9 and 7.2: Q102.

Status (v0.5): ANSWERED — spec 6.9: the table now says errors are reported after encryption (Q102).

**Q89. Repeater mode: counting and answering.**
- Spec section: 6.8, 6.3, 6.2, 6.4, 7.4 Riders. RPT-001.
- Unclear: Does a repeater send Delivered for texts it stores? That would raise the confirmation count with a radio that has no rider. Does it count among the 30 riders and appear in RIDERS_LIST (the repeater flag suggests yes)? Does it answer hello requests? Does it report presence it hears to a connected phone? Does it sound the buzzer for a new SOS? Does it keep and answer Fill on Return data? Does it take part in the 2-minute pairing window?
- Blocks: RPT-001 tests and the MSG-002 count.
- Options: A list of what a repeater does and does not do, kind by kind.

Status (v0.4): ANSWERED — spec 6.8: a repeater sends hellos and answers hello requests, sends no positions, presence or Delivered, counts towards 30 and is flagged in RIDERS_LIST, does not answer Fill on Return asks, passes presence to a phone, and opens the pairing window.

Status (v0.5): ANSWERED — spec 6.8.

**Q90. Buzzer behaviour.**
- Spec section: 6.4 (To the app), 9. POLICY §8.
- Unclear: Pattern, duration and loudness. Whether it sounds only for the first frame of a new alert or for every repeat. What silences it (button, phone, timeout). Whether it sounds when the phone is connected. What it does when the SOS ends, when two alerts arrive, and at low battery.
- Blocks: the board support interface for the buzzer and the SOS hardware test.
- Options: One short sound pattern for a new alert, sounded once, silenced by the power button.

Status (v0.4): ANSWERED — spec 6.4 and 9 (Decision) and SOS-005: the radio makes no SOS light or sound and uses no buzzer; the app alerts the rider. See Q112.

Status (v0.5): ANSWERED — spec 6.4 and 9 (Decision): no SOS light or sound on the radio (Q112).

**Q91. LOCATION_UPDATE: what is reported.**
- Spec section: 7.4 Location, 6.1, 6.5.
- Unclear: Is the radio's own position reported to the app (with the rider's own tag), and how often? Is every received position reported, or only the newest by fix time? Is a position older than one already held (out of order) reported? How is "hops travelled" set for a frame relayed through a repeater?
- Blocks: build step 4 and the phone tests.
- Options: Report each new authenticated position once, in fix-time order, and state the own-position rule.

Status (v0.4): ANSWERED — spec 7.4 Location rules: each new authenticated position is reported once; older ones are stored but not reported; hops travelled is starting hops minus hops remaining; the radio reports its own position when the source is its GPS.

Status (v0.5): ANSWERED — spec 7.4 Location rules, unchanged.

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

Status (v0.4): ANSWERED — spec 7.1: one `Envelope` message with request_id (1), last (2) and a oneof body numbered from 10 in table order; every scalar is `optional`; CamelCase message names; the .proto goes to the product owner for the app team.

Status (v0.5): ANSWERED — spec 7.1.

**Q93. Framing checks and the size of list messages.**
- Spec section: 7.2 (Framing), 7.3, 7.4.
- Unclear:
  - The byte order of the 2-byte length (the over-the-air rule in Section 4 is little-endian; 7.2 is silent).
  - A length above 1024, a length of 0, or a message that never completes: which error, whether the link is dropped, a timeout, how the receiver re-synchronises.
  - RIDERS_LIST (30 entries, names up to 24 bytes) and NEARBY_LIST (32 entries) can pass 1024 bytes. Only TRACK_POINTS has a part cap (64 points). The maximum entries per part for the other lists, and whether they use the last flag.
- Blocks: build step 3 and the list tests.
- Options: Little-endian length. A bad length gets ERROR and the partial message is discarded after a stated timeout. A fixed entry cap per part for each list.

Status (v0.4): ANSWERED — spec 7.2: little-endian length; a length of 0 or over 1024 gets ERROR TOO_LARGE and the buffer is discarded; 2 s completion timeout; list sizes per part (RIDERS_LIST 10, NEARBY_LIST 16, TRACK_POINTS 64). Open: re-synchronising after a bad length (Q106) and whether a full TRACK_POINTS part fits in 1024 bytes (Q105).

Status (v0.5): ANSWERED — spec 7.2: the limit is now 2048 bytes; list sizes are RIDERS_LIST 10, NEARBY_LIST 16, TRACK_POINTS 40. Q105 and Q106 are answered.

**Q94. Pairing: association model and adding a phone later.**
- Spec section: 7.2 (Pairing and security), 9. POLICY §8.
- Unclear:
  - Screens are not used in version 1, so Bluetooth LE Secure Connections cannot use numeric comparison. With "just works" pairing, any phone in range can pair during the 2-minute window, and then read texts and positions. What confirms the rider's own phone: a button press, a passkey, or nothing?
  - New pairing is accepted only in the 2 minutes after power-on. To add or replace a phone during a ride, the rider must power-cycle the radio, which stops relaying and any running SOS for the restart. Does a button press open the window at any time? POLICY §8 says the power button is for "pairing window".
  - When 4 phones are remembered, is a fifth refused or does it replace the oldest? How is a remembered phone removed?
  - Who starts encryption after HELLO and WELCOME, and what a bonded phone does outside the window.
- Blocks: build step 3 and the independent security review (the ride key and texts cross this link).
- Options: Require a button press to open the window at any time. Define replacement and removal of remembered phones. State the order: connect, HELLO, WELCOME, security start.

Status (v0.4): ANSWERED — spec 7.2 and 6.11: Just Works with bonding; a 2 s press opens a 2-minute pairing window at any time; the first 2 minutes after start also open it; a fifth phone replaces the least recently used; a 10 s hold forgets all phones; connection order is connect, HELLO, WELCOME, then encryption. The Just Works risk is on the security review list (spec 14).

Status (v0.5): ANSWERED — spec 6.11 and 7.2 (see Q20).

**Q95. Overlapping error codes.**
- Spec section: 7.3.
- Unclear: BAD_FIELD ("too long") and TOO_LARGE ("text or message over its limit") overlap. Which code for a 151-byte text, an empty text, a 25-byte name, a message above 1024 bytes? Which code wins when several apply (for example NO_RIDE, SOS_ACTIVE and NOT_ALLOWED_NOW)? Which code for an update chunk at the wrong offset?
- Blocks: the "malformed input rejected" tests for the phone link.
- Options: A table of message, condition and code, with an order of precedence.

Status (v0.4): ANSWERED — spec 7.3: 151-byte text TOO_LARGE, empty text BAD_FIELD, 25-byte name BAD_FIELD, message over 1024 bytes TOO_LARGE, wrong update chunk offset BAD_FIELD, plus a precedence order. UPDATE_REJECTED is not in the order (see Q23).

Status (v0.5): ANSWERED — spec 7.3 examples and precedence order. The one gap in the order is tracked in Q23.

**Q96. Diagnostics: counter definitions.**
- Spec section: 7.4 (Repeater, errors and diagnostics), 10. DIAG-001.
- Unclear: Which kinds are counted (nine ride kinds, presence?). Whether "received" counts before or after authentication and duplicate checks. Whether "relayed" counts frames sent or relays cancelled by suppression. Whether the six drop reasons are exclusive and in what order they are checked. The order of the 60 busy values (oldest first?) and the value of the current partial minute. Whether End Ride clears counters.
- Blocks: the DIAG-001 test and the field tests.
- Options: A table of each counter with the event that increments it and the order of checks.

Status (v0.4): ANSWERED — spec 7.4 Diagnostics: counters for the 9 ride kinds and presence (sent, received, relayed, suppressed, gave up), drops by the first failing check in 4.8, queue full, and the last 60 complete minutes of channel busy, oldest first; End Ride doesn't clear them. "Received" per kind is hard to apply before a frame's kind is known: Q107.

Status (v0.5): ANSWERED — spec 7.4 Diagnostics (counters per kind after authentication; new security and time counters).

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

Status (v0.4): ANSWERED — spec 7.4 Update rules: signature over the SHA-256 in START_UPDATE; refusals (UPDATE_REJECTED, LOW_BATTERY under 40%, SOS_ACTIVE, NOT_ALLOWED_NOW); wrong offset gets BAD_FIELD plus UPDATE_PROGRESS; 60 s without a chunk aborts; relaying and SOS continue while receiving; confirm within 60 s or roll back; USB recovery by the UF2 bootloader is unsigned and is accepted as outside DFU-001 (Q109).

Status (v0.5): ANSWERED — spec 7.4 Updates: Ed25519 over 65 signed bytes (label, version, board, size, SHA-256), checked by the bootloader before every start-up, primary and backup keys, rollback only to the previous confirmed image. The clean team must still confirm the chosen bootloader can do all of this (spec 14).

### E. Ride QR code

**Q98. QR code: CRC byte order and trailing data.**
- Spec section: 8, 11.4. RIDE-001.
- Unclear: The two CRC bytes: low byte first (as over the air) or high byte first? The sample ends in `6b0b`, and the spec does not say which. Whether bytes after the CRC are ignored or cause refusal, and whether the Base64url text may contain padding or capital letters.
- Blocks: the QR vector in 11.4 and app and radio agreement on the code.
- Options: State the CRC byte order and that any extra or malformed text is refused.

Status (v0.4): ANSWERED — spec 8: CRC low byte first, CRC parameters stated in full, and the app refuses a wrong format version, bad CRC, wrong name length, trailing bytes or any character outside the set. The 11.4 vector recomputes correctly.

Status (v0.5): ANSWERED — spec 8, unchanged in substance (version 2 format).

---

## New questions from the v0.4 review

Raised by: review of `docs/SPEC-v0.4.md`, `docs/clean-room/POLICY.md` v0.4 and `docs/clean-room/REQUIREMENTS.md` v0.4, 22 October 2026, with the two new hardware documents.
Sources used: `CLAUDE.md`, the three v0.4 documents, this file, `TODO.md`, `docs/hardware/` (the RAK19026 VC datasheet and profile, the Heltec T114 V2.0 schematic, the ThinkNode M1 datasheet V1.0). Nothing else.

**Cross-check of the three documents.**
- Matching: the four tiers, 30 riders, one ride per radio, the 176-byte frame, the SOS schedule, the three version 1 boards, the status light states, the 30% budget, the "which document wins" rule (the same sentence in all three), and the REQUIREMENTS and POLICY change tables all agree with the spec. The phone link has 44 messages, as the tables and spec 15 say. The hello name is 1–24 bytes everywhere.
- Recomputed: key derivation, ride tag, both rider tags, key check value, short ID, all eight encrypted ride frames in 11.2, the presence frame, the QR code bytes, CRC and text, and every airtime figure in the 2.5 table and the frame times in 4.5. All match. The phone-link samples in 11.5 are descriptions, not bytes, and were not checked.
- Not matching: Q62's items are settled. New differences are in Q109 and Q111.
- `SPEC-v0.4.pdf` and `SPEC-v0.4.md` carry the same text (word-level comparison; only line-break and table-layout differences).

### A. Spec text that conflicts or has gaps

**Q99. The ThinkNode M1's antenna is 2.5 dBi, and spec 2.1 allows at most 2 dBi at 22 dBm.**
- Spec section: 2.1 (Transmit power row), 9. REG-001. ThinkNode M1 datasheet, page 16 (LoRa antenna: gain 2.5 dBi; transmit power +22 dBm).
- Unclear: Whether 2 dBi is a hard condition of the compliance route or a design figure. If hard, whether the ThinkNode M1 must run below 22 dBm, or the limit changes. The antenna gain of the RAK19026 VC and T114 V2 as supplied with Trail Buddy is not in any repository document. Whether the lab tests each board with its own antenna.
- Blocks: REG-001 sign-off, the transmit-power setting in each board profile, and the radio bring-up on the ThinkNode M1.
- Options: State the limit as a combined figure the lab confirms. Set a per-board power in each profile. Fix the antenna for each board.

Status (v0.5): PARTLY ANSWERED — spec 2.1: 2 dBi is withdrawn as a design figure; the limit is 3 dBi at 22 dBm on every board, and the lab tests each board with its own antenna. The ThinkNode M1 (2.5 dBi) and the T114 V2 (2.39 dBi peak, 1.77 dBi at 915 MHz; I checked this against the antenna specification) are within 3 dBi. Still open: the RAK19026 VC's antenna model and gain are unknown (spec 14); the compliance lab has not confirmed the 3 dBi figure; the ThinkNode M1's manual (not in the repository, Q130) says its Canadian approval allows only 1 dBi.

**Q100. Nonce uniqueness depends on rider tags being unique, and a clash is only noticed afterwards.**
- Spec section: 4.3 (nonce), 4.4, 2.3 (Clash). Security review (spec 14).
- Unclear: The nonce is rider tag + frame counter + ride tag. Two radios in one ride can derive the same 3-byte tag (about 1 chance in 38,000 for a 30-rider ride). Both start their counters from small ceilings, so both can send frames with the same nonce under the same key before a hello reveals the clash. Reusing a nonce under CCM exposes the content of both frames and weakens authentication. 6.11 says a factory reset is safe because "no nonce can repeat", which holds for one radio but not for two radios with one tag.
- Blocks: build step 2, and the security review of 4.3 to 4.8.
- Options: Use the 8-byte radio ID, or a hash of it, in the nonce. Start each radio's counter from a random point per ride. Have the security review decide, and list it in spec 14.

Status (v0.5): ANSWERED — spec 4.1, 4.3 and 4.4: a 4-byte random join ID is chosen each time a radio joins a ride and is part of the nonce. I recomputed the chance of two radios in a 30-rider ride sharing tag and join ID: 435 pairs in 2^56, about 1 in 1.7 × 10^14 (2^−47), as the spec says. New: the same radio rejoining the same ride (Q123). The reviewer confirms (11.8).

**Q101. What an unpaired phone can do before encryption.**
- Spec section: 7.2 (Connection order, Pairing and security, Session), 2.3, 6.11.
- Unclear:
  - WELCOME is sent before encryption and carries the permanent radio ID and short ID, so any phone in range can read them without pairing. 2.3 says the radio ID is "never sent in clear", meaning over the air.
  - One phone may be connected at a time and there is no idle timeout. An unpaired phone that connects, sends HELLO and then stays silent keeps the only connection and locks out the rider's own phone, including SEND_SOS.
  - Nothing says what the radio advertises (name, address privacy, service UUID) or how long it waits for encryption to start after WELCOME.
- Blocks: build step 3 and the security review of 7.2.
- Options: A timeout for encryption to start after WELCOME, after which the connection is dropped. A bonded phone may replace an unpaired connection. Send the radio ID only after encryption. State the advertising contents.

Status (v0.5): ANSWERED — spec 7.2 and 6.11: WELCOME no longer identifies the radio; a 5 s encryption deadline; two connection places with at most one encrypted; private addresses and no name outside the pairing window; a stranger's phone can't pair without a button press. New: Q117 (the connection places).

**Q102. When a boot-time error is sent: right after WELCOME, or after encryption.**
- Spec section: 6.9 (table heading: "sent right after WELCOME, before RIDE_STATE"), 7.2 (Connection order; On connect).
- Unclear: 7.2 says every message other than HELLO and WELCOME waits for encryption, and lists the boot errors first among the messages sent "after encryption". 6.9 puts them right after WELCOME, before encryption.
- Blocks: the BLE state machine and the connect tests.
- Options: State one order: connect, HELLO, WELCOME, encryption, boot errors, RIDE_STATE.

Status (v0.5): ANSWERED — spec 7.2 Connection order and 6.9: boot errors come after encryption.

**Q103. Delivered travels fewer hops than texts in tiers 3 and 4.**
- Spec section: 2.5 (Delivered hop limit 2 in tiers 3 and 4), 4.6, 6.3. MSG-002.
- Unclear: Texts go 5 hops in tier 3 and 7 in tier 4. Delivered goes 2 hops. A rider 3 or more hops from the sender receives the text, but the sender can't receive the confirmation. "In contact" counts that rider, so the sender sees fewer confirmations than members in contact, may see "failed" with NO_CONFIRMATION and resend up to 3 times for riders that did receive the text.
- Blocks: MSG-002 tests in tiers 3 and 4 and the airtime figures in 2.5.
- Options: Make the "of N" count only riders within the Delivered range. Raise the Delivered hop limit. Count the failure only when no rider at all confirms.

Status (v0.5): ANSWERED — spec 6.3: the second number counts only riders within Delivered range; a text with at least one confirmation is never re-sent; NO_CONFIRMATION only when nobody confirms. New edge case: Q127.

**Q104. Bursts are not covered by the 30% average.**
- Spec section: 2.5, 4.7, 6.2 (replies after 0–5 s), 6.3 (Delivered after 0–2 s), 6.5, 6.10.
- Unclear: In tier 4, up to 29 radios send Delivered within the same 2 s window, each about 60 ms on air, plus relays. A frame that finds the channel busy 8 times is dropped ("gave up") and nobody is told, so the sender's count is low. The same applies to hello replies within 5 s. Fill is suspended above 25% busy and while any SOS is queued, so a rider returning to a busy trailhead gets no fill at all (spec 6.5).
- Blocks: the simulator scenarios in build step 5 and the Fill on Return field test.
- Options: Widen the random windows with the number of riders. Add burst scenarios to the simulator. State what Delivered and fill do when a frame gives up.

Status (v0.5): PARTLY ANSWERED — spec 6.2, 6.3, 6.5 and 2.5: reply windows widen with the number of riders in contact; a Delivered that gives up is not retried; fill above 25% busy sends outline parts only. Still open: the mesh simulator must show these hold in a 30-rider burst and at a busy trailhead (spec 14, build step 5).

**Q105. A full TRACK_POINTS part may be larger than 1024 bytes.**
- Spec section: 7.2 (Framing, List sizes per part), 7.4 Fill on Return, 7.1.
- Unclear: Each point holds latitude, longitude and fix time. With protobuf's usual number encoding, 64 points come to roughly 1.1 to 1.3 KB (about 1.1 KB even with fixed 32-bit fields), above the 1024-byte limit that is answered with TOO_LARGE. DIAGNOSTICS (about 116 counters) comes to about 0.8 KB, close to the limit. The spec doesn't say which protobuf types the schema uses.
- Blocks: the schema (build step 3) and the recorded vectors.
- Options: Lower the limit per part, say to 40 points. Say which types the schema uses. Add a worst-case size column to 7.4.

Status (v0.5): ANSWERED — spec 7.1 and 7.2: fixed-width types, the limit is 2048 bytes, TRACK_POINTS is 40 points per part (about 0.7 KB), and the clean team records worst-case sizes and tests them.

**Q106. How the receiver re-synchronises after a bad length.**
- Spec section: 7.2 (Framing).
- Unclear: A length of 0 or over 1024 gets ERROR TOO_LARGE and the buffer is discarded. The rest of that message keeps arriving, and its bytes would be read as new length prefixes. Nothing says how the receiver finds the next message start.
- Blocks: the framing tests and the malformed-input tests in build step 3.
- Options: After a bad length, discard bytes until 2 s pass with none. Or drop the connection.

Status (v0.5): ANSWERED — spec 7.2 Framing: after a bad length the receiver discards bytes until 2 s pass with none; after 3 bad lengths in a minute the radio drops the connection.

**Q107. Per-kind "received" counters before the kind is known.**
- Spec section: 7.4 Diagnostics, 4.8.
- Unclear: "Received" is counted per kind "before any other check", but the kind is inside the encrypted content and is read only after authentication (check 7). Frames that fail earlier have no known kind. Presence also has "relayed" and "suppressed" counters that can never count.
- Blocks: the DIAG-001 test.
- Options: Count received per kind only after authentication, and count earlier drops only under their reason. Say which counters apply to presence.

Status (v0.5): ANSWERED — spec 7.4 Diagnostics: per-kind counters count only after authentication; presence has sent, received and gave-up only.

**Q108. What a returning radio sends when it holds no history.**
- Spec section: 6.5 rules 3 and 4, 6.12 (own history is in RAM).
- Unclear: Own position history is lost at reboot. A radio that rebooted during the gap has nothing to answer with. No "no points" answer exists, so the others retry the ask 4 times, each time using the fill airtime allowance.
- Blocks: the Fill on Return tests and the airtime check.
- Options: Add an empty answer. Or let a radio that holds no history stay silent and the others stop after one retry.

Status (v0.5): ANSWERED — spec 6.5 rule 3: a radio with no history sends one empty answer, and receivers stop asking until the next return. The bytes are in 11.2 (recomputed, match).

**Q109. The unsigned USB recovery route is not reflected in DFU-001 or POLICY §12.**
- Spec section: 7.4 (Update rules, USB recovery), 0. DFU-001, POLICY §12.
- Unclear: Spec 7.4 accepts the UF2 bootloader's unsigned USB recovery as "outside DFU-001". REQUIREMENTS DFU-001 ("updates use signed images") and POLICY §12 ("firmware images are signed with Ed25519") say nothing about the exception. Under spec 0, a difference between documents stops work on the point. It needs a decision by the policy owner. It also means anyone holding the radio can flash unsigned firmware over USB.
- Blocks: build step 11 and the security review.
- Options: Add the exception to DFU-001 and POLICY §12. Or require the recovery route to check a signature.

Status (v0.5): ANSWERED — by a policy decision written into all three documents: shipped radios have no unsigned route; both Bluetooth and USB updates need signed images checked by the bootloader before every start-up; development radios may keep the UF2 bootloader (spec 7.4, POLICY §12 and §15, DFU-001). The clean team must still confirm a suitable bootloader exists (spec 14).

**Q110. Checks on text entered through the phone link.**
- Spec section: 4.8 check 13, 2.6, 7.3, 7.4 (SEND_TEXT, JOIN_RIDE, CHANGE_SETTINGS).
- Unclear: Check 13 covers names and text received over the air. For the display name, ride name and text coming from the app, 7.3 names only empty, too long and missing as BAD_FIELD. It isn't said whether invalid UTF-8 or control characters from the phone are refused, and with which code.
- Blocks: the phone-link malformed-input tests.
- Options: Apply the same rules as check 13 to every phone-link text and return BAD_FIELD.

Status (v0.5): ANSWERED — spec 6.9 and 7.3: text from the phone follows the same rules as over the air; anything else is BAD_FIELD.

**Q111. New requirements are not cited in the spec, and spec 12 is incomplete.**
- Spec section: 12, 4.7, 6.4, 7.4, 9. REQUIREMENTS MESH-005, SOS-005, HW-001. CLAUDE.md (every change maps to a requirement ID).
- Unclear: MESH-005 (listen before talking), SOS-005 (the app alerts the rider) and HW-001 (three boards) are not cited by any spec section or message. SOS_RECEIVED cites SOS-001 only. Spec 12 lists seven requirement changes but not these three, nor the MESH-003, MESH-004, RPT-001 and NEAR-005 edits.
- Blocks: TEST-MATRIX.md and the "every change maps to a requirement" rule.
- Options: Add the requirement IDs to 4.7, 6.4, 7.4 and 9 and complete spec 12.

Status (v0.5): ANSWERED — MESH-005 (4.7), SOS-005 (6.4, SOS_RECEIVED) and HW-001 (9) are cited, and spec 12 lists the changes. I checked that every requirement ID the spec cites exists in REQUIREMENTS and that every ID in REQUIREMENTS is cited, directly or in a range.

**Q112. SOS alerts exist only in the app (decision to confirm).**
- Spec section: 6.4 (Alert on the radio, Decision), 9. SOS-005, POLICY §8.
- Unclear: The radio makes no SOS light or sound on any board. The rider learns of a ride mate's SOS only from the app. If the phone is disconnected, the app is closed, or the phone is silent, nothing on the radio shows it; the status light shows only power and mode. The radio stores the event and the app learns of it on reconnect (7.2). Spec 0 says that for anything affecting SOS, the reading safer for the rider applies.
- Blocks: nothing in firmware. The SOS field test and the safety case need the owner's confirmation.
- Options: Confirm app-only alerts for version 1 and record the reason. Or add a radio alert in a later version.

Status (v0.5): ANSWERED — a Decision in spec 3, 6.4 and 9, and SOS-005: the radio makes no SOS light or sound. "SOS without a phone" is also a Decision: not in version 1. Both are the owner's call and are marked as such.

**Q113. Sleep is not defined, and the 6-hour anonymous ID timer counts "uptime".**
- Spec section: 6.6, 6.1. POLICY §18 (hardware-in-the-loop: "sleep/wake").
- Unclear: The spec has no sleep mode. The radio must keep receiving to relay. Whether uptime keeps counting during any low-power state, and what "sleep/wake" in POLICY §18 tests, are not stated.
- Blocks: the hardware test list and the anonymous ID timer test.
- Options: State that the radio never sleeps (the processor idles between events), or define the sleep states and how uptime counts through them.

Status (v0.5): ANSWERED — spec 2.2 and 9: the radio never sleeps; POLICY §18 now tests processor idle and wake instead of sleep and wake.

### B. Board documents

**Q114. ThinkNode M1: what the datasheet leaves open for the board profile.**
- Spec section: 9, 6.11. ThinkNode M1 datasheet V1.0, 16 April 2025 (pages 9–15 read; two table cells are blacked out in the cleaned copy).
- Unclear:
  - User button: the Function button is silk-screened FK, signal BUTTON_TOUCH, pin P1.07. Its function text is blacked out. Whether it is a push or touch sensor, its active level and debounce are not stated. The casing version has buttons and a knob on the casing; whether version 1 uses the casing is not stated.
  - Starting: power is a rotary switch with no processor pin, so "hold the button while the radio starts" (6.11) means holding FK while turning the knob. The reset button resets on one click and enters the burning (bootloader) mode on a double click.
  - Status light: DATA, LED3, P0.13, blue. Its description is blacked out. The PWR and CHGLED lights are red and tied to charging.
  - GNSS (Quectel L76K): GPS_RX P1.09, GPS_TX P1.08 (direction not stated), standby P1.02, reinit P1.05, 1PPS with no processor pin. The GPS switch has no processor pin, so the radio can't read it and can only see no response from the module.
  - Battery measurement: not in the datasheet at all.
  - Radio pins are listed (CS P0.24, SCK P0.19, MOSI P0.22, MISO P0.23, RESET P0.25, BUSY P0.17, DIO1 P0.20). DIO2 drives the RF switch and DIO3 the TCXO supply, but the table also lists P0.21 against DIO3. The TCXO voltage and RF switch polarity are not given.
  - No schematic is in the repository, though spec 9 lists one. The certifications section is empty. Battery 1200 mAh; operating range −20 to +70 °C; antenna 2.5 dBi (Q99). The L76K manual is not in the repository.
- Blocks: the ThinkNode M1 profile and its hardware tests (build step 12).
- Options: Supply the full Elecrow schematic and the L76K manual, and the blacked-out descriptions or Elecrow's answers. Record what remains as unknown in the profile.

Status (v0.5): PARTLY ANSWERED — spec 9 settles what it can: the blacked-out text is declared not needed (Decision); the Function button FK (P1.07) is the user button, and the status light is LED3 (P0.13); factory reset means holding the button while turning the power knob on; unknowns are recorded in the profile and the owner has asked Elecrow for the battery measurement, TCXO voltage, RF switch and P0.21/DIO3. Still open: no schematic; the battery is reported as 255 (unknown); radio bring-up waits for the TCXO voltage; the button mapping on the cased unit must be confirmed; spec 9 cites a user manual that is not in the repository (Q130).

**Q115. Heltec T114 V2: what the schematic leaves open for the board profile.**
- Spec section: 9, 6.11. Heltec_T114_V2.0_Schematic.pdf (one page; no revision or date on the sheet; "V2.0" is only in the file name). Pin readings below come from the drawing and need checking at full size.
- Read from the drawing: user button net USER_SW on P1.10, 10 kΩ to 3.3 V, so active low; green LED4 on net P1.03_Green through 330 Ω, apparently active low; battery sense BAT_ADC on P0.04 through a 390 kΩ / 100 kΩ divider switched by a transistor enabled from ADC_Ctrl on P0.06; GPS header with PPS P1.04, WAKE_UP P1.02, RST_GPS P1.06, TX_GPS P1.05, RX_GPS P1.07 and a switched supply (Vext, enabled by Vext_Ctrl). LED1 and LED2 are addressable RGB parts; LED3 (orange) is the charge light.
- Unclear:
  - The GPS module is not on the schematic (an 8-pin header only). Its model, serial speed, and whether "TX_GPS" is the module's or the processor's output are not stated. No GNSS manual is in the repository, and spec 9 says the module is fitted on every Trail Buddy T114.
  - The Vext_Ctrl pin and the pins of the SX1262 control lines (NSS, SCK, MOSI, MISO, BUSY, DIO1, NRESET) were not read; DIO2 drives the antenna switch and DIO3 the 32 MHz TCXO.
  - LED polarity and the battery conversion need confirming at full size. No datasheet is in the repository, though spec 9 lists one.
- Blocks: the T114 V2 profile and its hardware tests (build step 12).
- Options: Supply the Heltec datasheet and the GNSS module manual. Confirm the pin and polarity readings at full size.

Status (v0.5): PARTLY ANSWERED — new documents confirm user button SW1 = P1.10 and green LED4 = P1.03 (Zephyr board page, matching the schematic), the antenna (checked), and the L76K GNSS module (retailer page; Quectel protocol specification). Still open: no Heltec datasheet in the repository (spec 9 lists one); the GNSS module facts come from a retailer page, not the manufacturer; SX1262 and Vext_Ctrl pins and LED polarity still need reading at full size; no battery conversion formula. The Zephyr page lists the board as "Not actively maintained" (note for the clean team).

**Q116. A phone fix's age is measured from arrival.**
- Spec section: 6.6 (Sending), 6.1, 7.4 SHARE_MY_LOCATION.
- Unclear: For presence from a phone fix, "age" is measured from when SHARE_MY_LOCATION arrived, not from when the phone took the fix. The message carries the fix time, but a radio with an unknown clock can't use it. A delayed message makes the fix look younger than it is.
- Blocks: the NEAR-002 test with a phone source, and the plausibility check (6.6).
- Options: Use the fix time when the clock is known. Refuse phone fixes older than a stated age.

Status (v0.5): ANSWERED — spec 6.1: SHARE_MY_LOCATION carries the fix age when sent, and the radio adds the time since arrival, measured with the uptime counter.


---

## New questions from the v0.5 review

Raised by: review of `docs/SPEC-v0.5.md`, `docs/clean-room/POLICY.md` v0.5 and `docs/clean-room/REQUIREMENTS.md` v0.5, 23 October 2026, with the security review of v0.4 (`Security/SECURITY-REVIEW-v0.4.md`) and the new hardware documents.
Sources used: `CLAUDE.md`, the three v0.5 documents, this file, the security review, `docs/hardware/` (including the seven files added in `00cfed8`), `license/Nordic_5-Clause_License.txt`. Nothing else.

**Cross-check of the documents.**
- Matching: every requirement ID the spec cites exists in REQUIREMENTS, and every ID in REQUIREMENTS is cited, directly or in a range. The phone link has 46 messages, as the 7.4 tables and spec 16 say, and spec 11.7 has a sample for each. The "which document wins" rule is the same in all three. The tiers, 30 riders, 184-byte frame limit, SOS schedule (30 s) and the three boards agree across the spec, POLICY and REQUIREMENTS.
- Recomputed with independent code (not the spec's own figures): HKDF outputs, SOS commitment, all 13 ride frames in 11.2, the three rejection frames with bytes in 11.3 (tampered tag, hops above start, wrong class), the presence frame, the QR bytes, CRC and text, the SHA-256 and the Ed25519 signature in 11.6, the airtime figures in 2.5 (positions, presence, hellos, texts with Delivered, SOS add-on), the join-ID clash chance, and the T114 antenna gain (peak and 915 MHz) from its gain table. All match. The two rejection cases without bytes (sent as you, wrong secret) and the fill column of 2.5 were not recomputed.
- Not matching: the differences are in Q121, Q122 and Q125.

**How v0.5 answers the security review's 21 findings (my check; the reviewer confirms, spec 11.8).**

| Finding | My result |
| --- | --- |
| SR-01 nonce | Answered (join ID, 4.1 and 4.4). Residual: Q123 |
| SR-02 counter storage | Answered (4.4, 6.12). The flash wear design is the clean team's task |
| SR-03 forged SOS end | Partly: owner proof added; gaps in Q119, Q120 |
| SR-04 clock | Answered (2.2, 7.2). Related: Q124 |
| SR-05 blocked Bluetooth | Partly: Q117 |
| SR-06 SOS queue flood | Partly: Q118 |
| SR-07 pairing window | Answered (6.11, 7.2). Detail in Q126 |
| SR-08 unsigned USB route | Answered by a policy decision (7.4, POLICY §12, DFU-001) |
| SR-09 SOS collisions | Answered in the rules; needs simulator evidence. Q125 |
| SR-10 SOS end cases | Answered, except damaged records: Q121 |
| SR-11 replay, key lifetime | Partly: keys and codes expire; reboot replay of an SOS: Q122 |
| SR-12 signature coverage | Answered (7.4, 11.6) |
| SR-13 input checks | Answered (4.8, 7.3, POLICY §18 fuzz tests) |
| SR-14 Bluetooth identity | Answered, one exception: Q129 |
| SR-15 presence privacy | Answered (disclosure, new key per ride) |
| SR-16 fake presence | Answered (6.6, 7.2) |
| SR-17 relay disruption | Answered (hops can only grow for SOS). Detail: Q128 |
| SR-18 key storage | Answered (4.9, POLICY §12) |
| SR-19 amplification | Answered (6.2, 2.3) |
| SR-20 factory reset events | Answered (6.3) |
| SR-21 crypto library | Open clean-team task (spec 14, POLICY §15) |

17 answered or settled, 4 partly (SR-03, SR-05, SR-06, SR-11).

### C. Security and safety gaps in v0.5

**Q117. The two Bluetooth connection places can both be taken by unencrypted connections.**
- Spec section: 7.2 (Connections and time limits). SEC-003, SR-05.
- Unclear: The radio accepts two connections, at most one encrypted, drops an unencrypted one after 5 s and refuses a device address for 30 s after it fails to encrypt. Nothing says that one device may not use both places, and the 30 s refusal is by address while the radio's own rule for strangers is that addresses are private and can change. A stranger's phone (or two) that reconnects as each 5 s deadline passes can keep both places busy. "A paired phone that completes encryption drops other unencrypted connections" only helps once the paired phone is connected. The rider's own phone can't connect while both places are taken, including to send SEND_SOS.
- Blocks: build step 3 (the stranger-holding-a-connection test in POLICY §18) and SEC-003 ("a stranger's phone can't block the rider's own phone").
- Options: Keep one place free for bonded phones (the radio can recognise a bonded phone's private address at connection time). Or limit unencrypted connections to one at a time. Or rate-limit unencrypted connections as a whole, not by address.

**Q118. The SOS schedule limit is counted per sender identity, which a ride-key holder can invent.**
- Spec section: 4.5 (SOS schedule limit), 4.6, 2.7 row 3. SR-06, SOS-002.
- Unclear: Class-0 treatment for relays (own queue part, no suppression, no airtime budget, 1 s channel bound) applies "for each originating sender (rider tag and join ID)", at most 1 alert, 3 SOS frames per 30 s and 6 replies per 60 s. Anyone with the ride key can put any rider tag and join ID in an authentic frame, so a flood of SOS frames from many invented identities stays "within the schedule" for each identity. Every radio would relay all of them at top priority. When the 48-entry relay part is full, "the oldest relayed entry of any kind" is dropped, which can be a genuine rider's SOS. The threat model (2.7) says a ride member must not be able to "make radios drop an SOS".
- Blocks: SOS-002, build step 5 (the SOS-flood scenario has no stated attacker) and the SOS fuzz tests.
- Options: Add a total limit on class-0 relays per minute for all senders together, with the excess treated as class 6. Protect relayed entries from different recent senders from eviction. Say which attacker the simulator's flood scenario uses.

**Q119. Two alerts can share the same owner tag and alert ID, and SOS_ENDED doesn't say which one ended.**
- Spec section: 5 (Message IDs and alert IDs; SOS owner proof), 4.5, 7.4 (SOS_RECEIVED, SOS_ENDED). SEC-002, SOS-004.
- Unclear: "A received SOS whose owner and alert ID match a known alert but whose first-sent time differs is a new alert." A ride member can therefore send an SOS under another rider's tag and the same alert ID, with a different first-sent time and their own commitment, and then an "I'm OK" with their own secret. The receiver holds a commitment for each, and the second "I'm OK" is valid for the forged one. SOS_ENDED carries the alert ID and the owner tag but not the first-sent time, so the app can't tell which of the two ended. The real owner's next repeat would reopen the real alert, so the app could flip between ended and active. The threat model (2.7) says a ride member must not be able to end or alter another rider's SOS.
- Blocks: SOS-004, SEC-002, the forged "I'm OK" protocol test.
- Options: Identify an alert by owner tag, owner join ID and alert ID, and carry the join ID or first-sent time in SOS_ENDED and SOS_RECEIVED. Or ignore a second alert from the same owner tag and alert ID while the first is active.

**Q120. "Sent as you" is detected only when the join ID also matches.**
- Spec section: 4.8 check 7, 6.4. SEC-002, SR-03.
- Unclear: The check applies when the sender tag and join ID are the radio's own. A ride member who forges a frame with the victim's rider tag but a different join ID (the forger chooses it) isn't detected, so the rider's app is not told "someone is sending as you", although SEC-002 and the threat model say it is. Only hellos with a different radio ID cause the clash rule.
- Blocks: SEC-002, the "sent as you" test (POLICY §18).
- Options: Also report a frame with the radio's own rider tag and a different join ID, separately from true echoes. State that a different join ID with the same tag is treated as a clash or as impersonation.

**Q121. An SOS can be left active for ever on other riders' radios, and stale SOS events may follow a radio into its next ride.**
- Spec section: 6.4 (Ending; Damaged SOS record), 6.3 (Stored events), 6.7, 6.12, 7.2 (Connection order step 5). SOS-004, SEC-002.
- Unclear:
  - If the owner's SOS record is damaged (SOS_RESET), the owner secret is lost, so nobody can end that alert. The same happens if the owner's radio is lost or its battery dies. Other radios "keep showing the SOS" and never end it by themselves. The spec says the owner raises a new SOS, but doesn't say how the old one is cleared on other riders' radios or apps.
  - 6.3 and 6.12 keep SOS events in flash until the SOS has ended and 24 hours have passed; 6.7 says they stay "for 24 hours" after End Ride; 7.2 sends SOS_RECEIVED for "every active SOS from others" when a phone connects. A never-ended SOS therefore stays in flash and is sent to the app after End Ride and in the next ride.
- Blocks: SOS tests for End Ride and for reconnecting, and what the app shows riders.
- Options: Say the app (not the radio) may dismiss an SOS locally. Say End Ride clears the active-SOS list. Allow any ride member's End Ride to end only their own copy of an alert.

**Q122. A replayed SOS can appear as a new SOS after a receiver reboots, which SEC-004 forbids.**
- Spec section: 4.4 (Replay after a reboot), 6.4 (To the app). SEC-004.
- Unclear: SEC-004 says recorded traffic replayed later is "shown as old or rejected; it never appears as a new message or a new SOS". Spec 4.4 marks only texts and SOS replies as "old". An SOS is "never discarded for being old" and is shown with its first-sent time. After a receiver reboots, its replay memory and event list are empty, so a recorded SOS from earlier in the same ride is passed to the app as SOS_RECEIVED (a new event). By spec 0 the safer reading for SOS applies (show it), but the documents disagree on the words.
- Blocks: the SEC-004 test and the wording of REQUIREMENTS.
- Options: Change SEC-004 to say an old SOS is shown with its age. Or add "old" to SOS_RECEIVED when the first-sent time is more than a stated age behind a known clock.

**Q123. A radio that rejoins the same ride has a 1-in-4-billion chance of repeating a nonce.**
- Spec section: 2.3 (Join ID), 4.4, 6.11. SEC-001, SR-01 (the reviewer's figure was 2⁻⁴⁰).
- Unclear: The join ID is a random 32-bit number chosen each time a radio joins a ride, and the counter starts at 0. If a radio ends a ride and joins again with the same QR code (allowed within 7 days), its rider tag is the same, so a repeat of the join ID gives repeated nonces. The chance is 1 in 2³² (about 1 in 4.3 billion) per rejoin, 256 times the reviewer's 2⁻⁴⁰ limit. The 4.4 figure of 2⁻⁴⁷ covers two different radios only. 6.11 says "no nonce can repeat" after a reset, which is not exact. SEC-001 says "by design rather than by detection".
- Blocks: SEC-001 and the security reviewer's confirmation.
- Options: Make the join ID a counter kept in flash that never goes back, or add a stored join number to the join ID. Or accept and document the chance, and reword 6.11.

**Q124. A ride can be ended and its key erased because of a wrong clock.**
- Spec section: 2.4 (expiry), 2.2, 6.7. RIDE-005, SEC-005.
- Unclear: When the expiry passes by the radio's clock, the radio ends the ride and erases the key from every flash copy, which can't be undone. A radio with an unknown clock adopts the phone's time from a bonded phone without any check (2.2 rule 2); a GPS fix overrides it only when one arrives. A phone with a wrong date (days ahead) can therefore end a ride at connection. The expiry itself is the creating phone's creation time plus 7 days, so a wrong clock there shifts the real lifetime. The spec doesn't say whether expiry may rely on phone time alone.
- Blocks: the RIDE-005 tests and the flow for a rider with a wrong phone date.
- Options: End a ride on expiry only on a GPS-sourced clock, or after a grace period. Or tell the app to refuse a wrong phone date.

### D. Spec text that conflicts or is unclear

**Q125. The SOS delivery target describes repeats that the SOS schedule doesn't have.**
- Spec section: 4.7 (SOS delivery target), 6.4 (Schedule). SOS-006, SOS-003.
- Unclear: 4.7 says "within 40 s (by the repeats at 10 s and 30 s)". The schedule in 6.4 is first send, +10 s, then 25 s plus 0–5 s after that, so the third send is at 35–40 s, not at 30 s. A third send at 40 s is also at the edge of the 40 s window.
- Blocks: the simulator scenario for SOS-006 (build step 5).
- Options: State the scenario with the real schedule. Or move the target to 45 s.

**Q126. Button rules: a 10 s hold passes through the 2 s press, and it is unclear when the window opens after "forget phones".**
- Spec section: 6.11. SEC-003, BLE-004.
- Unclear: "Press for 2 s: opens the pairing window" and "Hold for 10 s: forgets all paired phones" overlap, so a 10 s hold seems to open the window at 2 s and then forget the phones at 10 s. It isn't said whether the window opens on reaching 2 s or on release, and whether it stays open after "forget". The second row says the window opens when the radio "starts with no phone paired (… or after 'forget phones')", but forgetting happens while running, not at start. The ThinkNode M1 button mapping on the cased unit is also unconfirmed (Q114).
- Blocks: build step 3 (pairing window tests) and the status-light tests.
- Options: State the window opens on release of a press of at least 2 s and under 10 s, and say what happens right after forgetting all phones.

**Q127. A text sent when nobody is within Delivered range.**
- Spec section: 6.3. MSG-002.
- Unclear: In tiers 3 and 4, Delivered travels 2 hops. If every ride member in contact is more than 2 hops away, the count reads "0 of 0", nobody can confirm, and the radio re-sends 3 times and reports NO_CONFIRMATION, although the text may have reached everyone.
- Blocks: MSG-002 tests in tiers 3 and 4.
- Options: Don't re-send or report failure when the second number is 0. Or report a separate state for "sent, no confirmation expected".

**Q128. The SOS "hops can only grow" rule and the duplicate check.**
- Spec section: 4.4 (Duplicate memory), 4.6, 4.8 check 8. SR-17.
- Unclear: Check 8 drops a duplicate, but 4.6 needs a duplicate's hops remaining to be read so a radio can adopt a higher value and relay once. The order isn't stated, and neither is whether a radio that already relayed the frame relays again.
- Blocks: the relay tests for SOS (build step 5, 7).
- Options: Say the duplicate's hops are read before the drop, for SOS-class frames only, and that a frame is relayed at most once.

**Q129. The pairing-window Bluetooth name shows the short ID.**
- Spec section: 7.2 (Advertising), 2.3. SEC-003 ("can't read the radio's identity"), SR-14.
- Unclear: During the pairing window the name is "TrailBuddy " plus the short ID, which is 15 bits of the permanent radio ID. Anyone scanning can read it. The same short ID is shown at every window, so a radio can be recognised from one window to the next until a factory reset. SEC-003 and the threat model (2.7 row 2) say a stranger's phone can't read the radio's identity.
- Blocks: the "nothing identifying before encryption" test.
- Options: Say the short ID in the name is accepted for usability (and update SEC-003), or use a different name per window.

### E. Board documents

**Q130. The spec cites documents that are not in the repository, and a retailer page stands in for a manufacturer document.**
- Spec section: 2.1, 9. POLICY §8, §14 (board profiles come only from the manufacturer's documents). CLAUDE.md.
- Unclear:
  - The ThinkNode M1 user manual is cited for FCC ID 2BDNA-TNMS-ELE, IC 35795-TNM1, the 1 dBi external-antenna note, the −10 to +50 °C range and the names of the case's side buttons. It is not in the repository.
  - Spec 9 lists a "Heltec T114 datasheet (cleaned)", "RAK4631 documentation", the full ThinkNode M1 schematic and the "u-blox M8 receiver description and protocol specification". None is in the repository (the repository has the u-blox ZOE-M8 datasheet only).
  - The T114 GNSS module facts (L76K, 9600 baud, 3.3 V, built-in antenna) come from a Dutch retailer's page, not from Heltec. The T114 antenna specification is from the antenna maker; its header names Heltec as the customer.
  - The Zephyr page for the T114 V2 lists the board as "Not actively maintained".
- Blocks: the three board profiles (Q54) and the lab's certification check.
- Options: The product owner adds the manufacturer documents, or the spec says which facts rest on a retailer page and the profile marks them unconfirmed.

**Q131. Housekeeping in the documents.**
- Spec section: 7.4 (board model enum), 9. POLICY §19, §15.
- Unclear:
  - The board model list holds RAK19007 and POCKET_V3. The RAK19007 datasheet is in `docs/hardware/`, but spec 9 does not list the RAK19007 as a version 1 board or as deferred.
  - `docs/hardware/` holds two byte-identical copies of the T114 schematic (`Heltec_T114_V2.0_Schematic.pdf` and `Heltec_T114_V2.0_Schematic (1).pdf`).
  - POLICY §19 puts licence texts in `docs/licences/`; the Nordic licence is in `license/`.
- Blocks: nothing; it affects traceability and the SOURCE-REGISTER.md entries.
- Options: Say what the RAK19007 is for. Remove the duplicate. Move the licence or correct the path.
