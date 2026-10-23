# TrailBuddy Radio Message Spec (Draft v0.3)

22 October 2026 · Spec team · Supersedes Draft v0.2 · Answers OPEN-QUESTIONS.md (Q1–Q56) of 22 October 2026

## 0. How to read this version

- Section 13 maps every open question to the section that answers it.
- Product decisions in this version were made by the product owner on 22 October 2026 and are marked **Decision**.
- Items still open are listed in Section 14. Nothing in this document is final until spec v1.0 is frozen.
- Section 12 lists changes this version makes to `REQUIREMENTS.md` and `POLICY.md`. Updated copies of those files will follow; until then, this spec wins where they differ.

## 1. How it works

Trail Buddy uses its own radio protocol end to end. The app talks to its own radio over Bluetooth; the radios talk to each other over LoRa. Both conversations are defined here.

```
TrailBuddy app  <-- phone link (Bluetooth) -->  TrailBuddy radio  <-- over the air (LoRa) -->  other TrailBuddy radios
```

**Who designs what (Q4):**

- **Over the air:** this spec defines every frame byte for byte (Sections 4 and 5). The clean team implements these layouts exactly. They are fixed binary layouts, not protobuf, so that sizes are exact and test vectors are byte-exact.
- **Phone link:** this spec defines every message's fields, types, units and limits (Section 7). The clean team writes the protobuf schema from those tables, following the rules in Section 7.1.

**Records (Q5):** there is no `docs/PROVENANCE.md`. Hand-overs are recorded in the clean-room log, which the product owner keeps outside the repository, as POLICY §19 says. This spec refers only to files in the repository.

## 2. Foundations

### 2.1 Radio settings and channel plan (Q1, Q3, Q16)

**Decision:** all Trail Buddy radios share one common channel. Rides are kept private by their keys, not by separate frequencies. Presence works because every radio is on the same channel.

Proposed settings. These are subject to confirmation by a compliance test lab before any field use, and the lab's answer may change them:

| Setting | Value |
| --- | --- |
| Frequency | One fixed channel, centre 916.5 MHz, United States only |
| Bandwidth | 500 kHz |
| Spreading factor | 9 |
| Coding rate | 4/5 |
| Preamble | 12 symbols |
| Sync word | 0x12 |
| Header | Explicit, with CRC on |
| Transmit power | 22 dBm at the radio, with an antenna of at most 2 dBi gain |
| Maximum frame | 255 bytes (radio limit); Trail Buddy frames are at most 176 bytes |

**Regulatory route:** the 500 kHz bandwidth is chosen so that the radio can qualify as a digital-modulation device under FCC Part 15, which has no duty-cycle limit in the United States. The lab must confirm this. If it can't be confirmed, the fallback is frequency hopping, which would change Sections 2.1 and 4. REG-001 means: transmit power, antenna gain and bandwidth stay within what the lab approves.

**Airtime budget (Q16):** Trail Buddy's own limit is that normal traffic keeps the channel at or below 30% busy. SOS and SOS replies are exempt from this budget and are never delayed by it. They are still always within the legal limits above.

**Range trade-off:** these settings send each frame quickly, which is what makes 10-second updates possible. Faster settings reach less far than slower ones. Field testing must confirm range against NFR-001.

### 2.2 Time (Q6)

- **Trail Buddy time** is seconds since 2026-01-01 00:00:00 UTC, as an unsigned 32-bit number. The value 0 means "unknown".
- The radio's clock comes from its GPS whenever it has a fix. Otherwise it comes from the phone: HELLO carries the phone's current time, and the radio adopts it if its own clock is unknown or more than 2 seconds different and it has no GPS fix.
- If neither source has ever set the clock since boot, time is unknown and every time field the radio sends is 0. Positions always carry a known fix time, because GPS and phone fixes both include time.
- Intervals, timeouts and ages use the radio's uptime counter, never the clock.
- Presence carries a fix **age** in seconds, not a time, so it works without a clock.

### 2.3 Identities (Q7, Q8)

| ID | Size | What it is | Where it appears |
| --- | --- | --- | --- |
| Radio ID | 8 bytes | Random, created from the hardware random number generator at first boot. Kept through reboots and firmware updates. Changes only on factory reset. | Phone link (WELCOME, rider entries); inside encrypted hello frames only. Never sent in clear. |
| Short ID | 3 characters | The first 15 bits of the radio ID, written as 3 characters from A–Z and 2–7. | App display beside names; default display name. |
| Rider tag | 3 bytes | Ride-scoped: derived from the ride key and the radio ID (Section 4.3). Different in every ride, so outsiders can't link a radio across rides. | Every ride frame envelope; phone link as the rider ID. |
| Anonymous ID | 4 bytes | Random. New at every boot and every 6 hours of operation, but a change is applied only while the radio is stopped, so one encounter never shows one rider as two. | Presence frames only. |

**Privacy limit, documented:** during a ride, an outsider can follow a rider tag for that ride's duration, but can't learn the name, radio ID or messages. A ride member's hello also carries their current anonymous ID, so their own ride's radios can hide them from the Nearby list.

**Rider tag clash:** if a radio receives a hello with its own rider tag but a different radio ID, it re-derives its tag with the next clash counter (Section 4.3) and sends a new hello.

### 2.4 Rides (Q9)

- **Decision:** at most 30 riders per ride.
- A radio holds at most one active ride.
- Every ride frame carries a 2-byte ride tag derived from the ride key, so a radio decrypts only frames that carry its own ride's tag.

### 2.5 Group tiers (Q10)

**Decision:** four tiers, chosen when the ride is created, carried in the QR code, fixed for the ride. Changing tier means ending the ride and joining a new one.

| Tier | Riders | Hop limit | Position interval | Gap threshold (Fill on Return) |
| --- | --- | --- | --- | --- |
| 1 | 2–4 | 3 | 10 s | 60 s |
| 2 | 5–9 | 4 | 45 s | 135 s |
| 3 | 10–19 | 5 | 120 s | 360 s |
| 4 | 20–30 | 7 | 90 s | 270 s |

The gap threshold is 3 × the position interval, with a minimum of 60 s.

**Airtime check, approximate**, with the settings in 2.1. It assumes relay suppression (Section 4.6) keeps each position to at most 4 transmissions on average, and every rider sending presence every 10 s while moving.

| Tier | Positions | Presence | Total busy |
| --- | --- | --- | --- |
| 1 (4 riders, 10 s) | 11.3% | 1.8% | 13.2% |
| 2 (9 riders, 45 s) | 5.7% | 4.1% | 9.8% |
| 3 (19 riders, 120 s) | 4.5% | 8.6% | 13.1% |
| 4 (30 riders, 90 s) | 9.5% | 13.6% | 23.0% |

All four tiers fit the 30% budget on paper. Tier 1 at 10 s must be confirmed in the mesh simulator and then on a 4-rider field ride, with the channel staying usable for an SOS. If it doesn't hold, the fallback is 10 s while moving and 30 s when stopped. Tier 4 needs the simulator to confirm the suppression assumption.

### 2.6 Units on the wire (Q35)

One measurement system on the wire; the app converts for display.

| Quantity | Type | Unit | Unknown value |
| --- | --- | --- | --- |
| Latitude, longitude | signed 32-bit | millionths of a degree | — (a position is never sent without a fix) |
| Accuracy | unsigned 8-bit | metres, 254 = 254 m or worse | 255 |
| Speed | unsigned 8-bit | km/h | 255 |
| Heading | unsigned 8-bit | 1.5° steps (0–239) | 255; always 255 below 3 km/h |
| Fix age (presence) | unsigned 8-bit | seconds, 254 = 254 s or older | 255 |
| Time | unsigned 32-bit | Trail Buddy time (2.2) | 0 |
| Signal strength | signed 8-bit RSSI, and signed 8-bit SNR | dBm; SNR in quarter-dB steps | RSSI 0 |
| Battery | unsigned 8-bit | percent 0–100 | 255 |
| Channel busy | unsigned 8-bit | percent 0–100 | 255 |
| Text and names | UTF-8 bytes | length in bytes, no control characters | — |

## 3. Requirement decisions in this version

| Topic | Decision |
| --- | --- |
| Text to one rider (Q30) | Not in version 1. Text goes to the whole ride only. |
| Maximum text (Q30) | 150 bytes of UTF-8, about 150 characters of plain English. Longer text is refused, not split. |
| Delivered (Q31) | The sender sees a count: "Delivered to 3 of 4". |
| Presence default (Q45) | On by default; always on during the rider's own SOS; back on after End Ride. |
| Battery sharing (Q37) | Not in version 1. No battery field in rider entries. |
| GPS source (Q33) | Radio GPS first, phone GPS as fallback. |
| Who cancels an SOS (Q40) | Only the rider who raised it. |
| Diagnostics (Q53) | Counters stay in release firmware; detailed logging is removed. |
| Lead board (Q54) | RAKwireless WisMesh Pocket V3. |
| Targets (Q2) | At least 12 hours on battery per riding day; a filled track within 60 s of a rider's return; range at least as good as the August field test. Maximum frame size: 176 bytes. |

## 4. Over the air: frames

All multi-byte numbers are little-endian.

### 4.1 Ride frame (encrypted)

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | High 4 bits: protocol version (1). Low 4 bits: frame type (1 = ride frame). |
| 1 | 1 | High 4 bits: hops remaining (0–15). Low 4 bits: priority class (Section 4.5). |
| 2 | 2 | Ride tag |
| 4 | 3 | Sender rider tag |
| 7 | 4 | Frame counter (Section 4.4) |
| 11 | 3 | Destination rider tag; 000000 = the whole ride |
| 14 | n | Encrypted content: 1 kind byte, then the kind's fields (Section 5) |
| 14 + n | 8 | Authentication tag |

Envelope overhead is 22 bytes. The largest frame is 176 bytes (a 150-byte text).

### 4.2 Presence frame (not encrypted, never relayed)

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | High 4 bits: protocol version (1). Low 4 bits: frame type (2 = presence). |
| 1 | 4 | Anonymous ID |
| 5 | 4 | Latitude, rounded to the nearest 0.0001° (about 11 m) |
| 9 | 4 | Longitude, rounded the same way |
| 13 | 1 | Heading |
| 14 | 1 | Speed |
| 15 | 1 | Fix age |

16 bytes in total.

### 4.3 Keys and derived values (Q11)

All values are derived from the 16-byte ride key with HKDF-SHA256 (RFC 5869), no salt, using these info labels:

| Value | Length | Info label |
| --- | --- | --- |
| Encryption key | 16 bytes | `trailbuddy v1 enc` |
| Ride tag | 2 bytes | `trailbuddy v1 ride tag` |
| Rider tag | 3 bytes | `trailbuddy v1 rider tag` + radio ID (8 bytes) + clash counter (1 byte, starts at 0) |
| Key check value | 4 bytes | `trailbuddy v1 check` |

**Encryption:** AES-128-CCM with an 8-byte authentication tag (hardware-accelerated on the nRF52840).

- **Nonce (13 bytes):** sender rider tag (3) + frame counter (4) + ride tag (2) + four zero bytes.
- **Authenticated, not encrypted:** the 14-byte envelope, with the hops-remaining bits set to zero, because relays change them.
- **Encrypted:** the kind byte and its fields.

The choice of algorithm, tag size and counter rule goes to the independent security review before freeze.

### 4.4 Frame counter, duplicates and replays (Q11, Q13)

- Each radio keeps one 32-bit frame counter. It increases by 1 for every ride frame the radio creates, never for relays, and is never reused.
- **Surviving power loss:** the radio stores a counter ceiling in flash, 1024 above the current value. At boot it starts from the stored ceiling and writes a new ceiling 1024 higher before sending anything. A counter can jump after a reboot but never repeats.
- **Duplicates:** a frame's identity is (sender rider tag, frame counter). The radio remembers the last 256 identities for 10 minutes and drops repeats.
- **Replays:** for each sender, the radio keeps the highest counter seen and a 64-frame window below it. Frames below the window, or already marked in it, are dropped. These records are not kept across reboot; this limit is documented for the security review.
- Every SOS repeat is a new frame with a new counter and the same alert ID, so repeats are never dropped as duplicates.

### 4.5 Priority classes and queues (Q15)

| Class | Kinds | Queue length | When full |
| --- | --- | --- | --- |
| 0 | SOS, SOS reply | 8 | Never dropped; a newer frame for the same alert replaces an older queued one |
| 1 | Presence | 1 | The newest replaces the queued one |
| 2 | Position | 1 per rider | The newest position of a rider replaces the queued one |
| 3 | Delivered | 8 | Oldest dropped |
| 4 | Hello, hello request | 4 | A newer hello replaces the queued one |
| 5 | Text | 8 | New text refused with BUSY |
| 6 | Fill ask, fill answer | 8 | Oldest dropped |

Lower class numbers always go first. A transmission already in progress is never cut off. The longest frame lasts about 230 ms on air, so an SOS waits at most that long behind one frame. A presence frame lasts about 45 ms.

### 4.6 Relaying (Q12, Q14, Q17)

A radio relays a ride frame only if all of these hold:
- it carries the radio's own ride tag
- it passes authentication
- it isn't a duplicate
- hops remaining is above 0
- it isn't addressed only to this radio

Radios don't relay other rides' frames. This stops forged floods, because a radio that can't verify a frame never relays it. Presence frames are never relayed.

**Delay before relaying:** the radio waits a random time, then sends the frame with hops remaining reduced by 1.

- **SOS and SOS replies:** a random wait of 0–150 ms. Always relayed.
- **Everything else:** the wait window is 1000 ms, scaled by received signal quality. Frames heard weakly relay sooner, frames heard strongly relay later: scale = (SNR + 10) / 20, clamped to 0.1–1.0. A random 0–20% is added on top.
- **Suppression:** while waiting, if the radio hears the same frame relayed by 2 other radios, it cancels its own relay. This does not apply to SOS or SOS replies.

This combines the distance-based and counter-based schemes from Ni, Tseng, Chen and Sheu, "The Broadcast Storm Problem in a Mobile Ad Hoc Network", MobiCom 1999. The clean team cites it in DECISIONS.md.

**Per-kind rules:**

| Kind | Destination | Starting hops | Class | Acknowledged |
| --- | --- | --- | --- | --- |
| Position | Whole ride | Tier hop limit | 2 | No |
| Hello | Whole ride | Tier hop limit | 4 | No |
| Hello request | Whole ride | Tier hop limit | 4 | Answered by hellos |
| Text | Whole ride | Tier hop limit | 5 | By Delivered |
| Delivered | Original sender | Tier hop limit | 3 | No |
| SOS | Whole ride | 7, whatever the tier | 0 | By SOS replies |
| SOS reply | Whole ride | 7 | 0 | No |
| Fill ask | Whole ride | Tier hop limit | 6 | By fill answer |
| Fill answer | Whole ride | Tier hop limit | 6 | No |
| Presence | Any Trail Buddy radio | Not relayed | 1 | No |

### 4.7 Versions (Q18)

- One protocol version number, 1, covers both links.
- **Over the air:** a radio drops frames with any other version and does not relay them.
- **Phone link:** the version is checked once, in HELLO and WELCOME; it must match exactly. Other phone-link messages don't repeat it.

## 5. Over the air: kinds inside a ride frame

The first encrypted byte is the kind.

| Kind | Byte | Fields (in order) | Size after kind byte |
| --- | --- | --- | --- |
| Position | 0x01 | latitude (4), longitude (4), accuracy (1), fix time (4), source (1: 0 = radio GPS, 1 = phone GPS) | 14 |
| Hello | 0x02 | radio ID (8), anonymous ID (4), flags (1: bit 0 = repeater), name length (1), name (0–24) | 14–38 |
| Hello request | 0x03 | none | 0 |
| Text | 0x04 | message ID (2), text length (1), text (1–150) | 4–153 |
| Delivered | 0x05 | message ID (2), original sender rider tag (3) | 5 |
| SOS | 0x06 | alert ID (2), latitude (4), longitude (4), accuracy (1), fix time (4), time first sent (4), repeat number (1) | 20 |
| SOS reply | 0x07 | alert ID (2), alert owner rider tag (3), reply (1: 1 = copy that, 2 = I'm OK, 3 = ended because the owner left the ride) | 6 |
| Fill ask | 0x08 | rider tag asked about (3), from time (4), to time (4) | 11 |
| Fill answer | 0x09 | rider tag (3), part (1: bit 7 = outline, bits 0–6 = part number), point count (1, at most 12), then per point: latitude (4), longitude (4), fix time (4) | 5–149 |

Message IDs and alert IDs are 16-bit counters kept by the sending radio, starting at 1 and skipping 0.

## 6. Behaviour

### 6.1 Position (POS-001 to POS-003, Q33, Q34)

- The radio owns the schedule and sends a position at the tier interval, with up to 10% random jitter.
- **Source (decision):** the radio's own GPS fix if it is no older than the interval. Otherwise the phone's latest position if it is no older than the interval. Otherwise no position is sent, and STATUS reports "no position".
- The app sends SHARE_MY_LOCATION whenever the phone has a new fix, at most once per second. There is no reply.
- After the phone disconnects, the radio keeps sending positions from its own GPS.
- **GPS duty (POS-003):** the GPS runs continuously while the rider is moving or presence is on. When the rider has been stopped for 5 minutes and presence is off, the GPS samples once per minute. Moving means 5 km/h or more for 10 s; stopped means below 3 km/h for 30 s.
- A board with no working GPS relies on the phone. STATUS reports the GPS as unavailable.

### 6.2 Rider hello (MSG-003, Q36)

- A radio sends a hello when it joins a ride, when its name changes, every 10 minutes, and in reply to a hello request.
- A radio that hears a frame from a rider tag it has no hello for sends one hello request, at most once per minute. Each radio that hears a request replies after a random 0–5 s delay.
- **Display name:** 1–24 bytes of UTF-8 with no control characters. The default is "Rider " plus the short ID. Two riders may share a name; the app shows the short ID beside duplicates.

### 6.3 Messages (MSG-001, MSG-002, Q30–Q32)

- Text goes to the whole ride; 150 bytes maximum.
- Every radio that receives a text stores it and sends Delivered to the sender after a random 0–2 s delay, whether or not its phone is connected.
- **Sender's side:**
  - SEND_RESULT reports "queued", then "sent", then "delivered" with a count each time a new confirmation arrives. The count is out of the ride members heard in the last 10 minutes.
  - With no confirmation after 20 s, the radio resends the text with the same message ID and a new frame counter, up to 3 times.
  - After the third resend with no confirmation, SEND_RESULT reports "failed", reason NO_CONFIRMATION.
- **Stored events:** the radio numbers everything it stores for the app with a 32-bit event number: texts, SOS events, SOS replies and rider changes.
  - It keeps the last 200 events.
  - Events about an SOS stay until that SOS has ended and a further 24 hours have passed; they are never evicted before that.
  - GET_MISSED asks for all events after a given event number. The radio replays them as their normal messages, in order, each carrying the request ID, with the last one flagged.

### 6.4 SOS (SOS-001 to SOS-004, Q39–Q41)

- SEND_SOS starts an alert. The radio assigns the alert ID.
- **Schedule:** sent at once, again 10 s later, then every 5 minutes after the first send, with up to 5 s of random jitter. A missed slot is not caught up.
- Each repeat carries the latest position, the original first-sent time and an increasing repeat number.
- A second SEND_SOS while an alert is active updates its position only; there is one active SOS per radio.
- The radio keeps repeating whether or not the phone is connected.
- **I'm OK:** accepted only when the sender's rider tag matches the alert owner. All ride members hold the same key, so a member could in principle forge one. This limit goes to the security review.
- **Copy that:** informational only; it doesn't stop repeats.
- **Ending:** an SOS ends when the owner sends I'm OK, or when the owner ends the ride. End Ride during the owner's own SOS first sends an SOS reply "ended because the owner left the ride".
  - Radios that never hear the end keep showing the SOS with its last-heard time. It is never ended automatically for lack of signal.
- **To the app:**
  - SOS_RECEIVED is sent for a new alert and for every repeat; the app de-duplicates by alert.
  - SOS_ENDED is sent when an end is heard.
  - On boards with a buzzer, the radio sounds it for a new alert.

### 6.5 Fill on Return (FILL-001, Q38)

Each radio keeps its own position history for the last 2 hours, at least one point every 10 s, and the positions it has heard from ride members for the same period. The rules:

1. **Gap:** a ride member was silent for longer than the tier's gap threshold, is heard again, and has moved more than 50 m from their last known position.
2. **Ask:**
   - Wait a random 2–8 s, then send a fill ask for the missing range, at most 30 minutes.
   - Don't ask if another radio's ask for the same rider and an overlapping range was heard during the wait.
3. **Answer:**
   - Only the returning rider's own radio answers, from its own history. This works whether or not that rider's phone is connected.
   - The answer goes to the whole ride, so every radio with the same gap uses it.
   - Each answer is sent twice, 10 s apart.
   - A range over 15 minutes is sent as an outline first (every 4th point), then the detail.
   - An ask already covered by an answer this radio sent in the last 15 s is skipped.
4. **Retry:**
   - No answer within 30 s means ask again.
   - After 4 unanswered asks, that gap waits for the rider's next return.
5. **Channel:**
   - Fill frames are sent only when channel busy is 25% or less, on a reading under 60 s old. Otherwise they wait.
   - Any queued SOS frame blocks fill.
   - At most one fill frame every 10 s per radio.
   - Blocked answers wait and go when the block ends.

- **Rejoin:** this is what "heard again" means in rule 1.
- **Reporting to the app:** filled points reach the app as TRACK_POINTS only, never as LOCATION_UPDATE.
- **GET_TRACK:** returns what this radio has stored; it doesn't cause a fill ask.

### 6.6 Nearby riders (NEAR-001 to NEAR-008, Q42–Q48)

- **Sending:** presence every 10 s while moving and every 60 s while stopped, with ±1 s of random jitter. This is independent of tier and of whether a ride is active.
- **Not relayed:** a presence frame travels one hop only.
- **Own frames:** a radio ignores presence frames carrying its own anonymous ID.
- **Ride members:** a radio hides presence from its own ride's members, recognised by the anonymous IDs in their hellos.
- **Turning presence off:** this stops sending only; receiving is always on.
  - Presence is forced on during the rider's own SOS.
  - It is on again after End Ride.
- **Limits against fake or flooding presence:**
  - The radio tracks at most 32 anonymous IDs and evicts the oldest.
  - It ignores a report that implies a speed over 200 km/h since the last report from that ID.
  - It accepts at most one report per ID every 2 s.
  - The remaining risk is accepted and documented.
- **Reporting to the app:** at most one NEARBY_RIDER per anonymous ID every 5 s.
- **Alerts:** approaching-rider warnings are made by the app only in version 1. The radio's buzzer is not used for them.
- **Combining with position:** presence and ride position stay separate signals in version 1 (Q48).

### 6.7 End Ride (RIDE-003, Q28)

| Item | After End Ride |
| --- | --- |
| Ride key, name, tier, tags | Erased |
| Own active SOS | Ended. An "owner left the ride" reply is sent first. |
| Queued ride frames | Dropped |
| Stored texts, ride events, track history | Erased |
| Display name, GPS source | Kept |
| Presence sending | On |
| Repeater mode | Off |
| Frame counter | Kept; it never resets |

End Ride affects only the local radio. Other riders notice through silence, and Fill on Return handles the gap if the rider returns.

### 6.8 Repeater mode (RPT-001, Q51)

- **Set-up:** the phone joins the radio to a ride, then turns repeater mode on. The radio keeps the ride and the mode through reboots until the phone ends the ride or turns repeater mode off.
- **What a repeater does:**
  - relays its ride's frames
  - sends a hello every 10 minutes, with the repeater flag set
  - sends no position and no presence
- It still accepts a phone connection at any time.
- REPEATER_STATUS is sent together with STATUS whenever repeater mode is on.

## 7. Phone link

### 7.1 Schema rules (Q5)

- protobuf, `proto3` syntax, package `trailbuddy.v1`, written by the clean team from the tables below.
- Names come from this spec.
- Field numbers are never reused. Removed fields are reserved.
- Every enum starts with an "unspecified" value of 0.
- Every text field and list is bounded by the limits given here.

### 7.2 Bluetooth service (Q19, Q20, Q21)

| Item | Value |
| --- | --- |
| Service UUID | `bc32efc7-02d9-48cb-b66a-9041aa8cfa23` |
| To radio (phone writes, with response) | `85d5f69e-313a-458e-9912-20397554da2f` |
| From radio (radio notifies) | `4dee99ec-354d-451c-a12f-1a0a75cfbe6d` |

- **Framing:** each message is a 2-byte length followed by the protobuf bytes, split across as many Bluetooth packets as needed. The radio asks for the largest packet size the phone allows. The longest message is 1024 bytes.
- **Order and flow:** the phone waits for each write to be confirmed before sending the next. Messages arrive in order.
- **Connections:** one phone at a time. A second phone is refused while one is connected.
- **Pairing and security:**
  - The link must be paired and encrypted (Bluetooth LE Secure Connections) before any message except HELLO and WELCOME is accepted.
  - A radio accepts new pairing only during the 2 minutes after it is powered on.
  - It remembers up to 4 phones.
- **Session:**
  - Before HELLO, everything except HELLO is refused with ERROR NOT_ALLOWED_NOW.
  - There is no idle timeout.
  - After a disconnect, or GOODBYE, the radio carries on relaying, sending positions and presence, repeating its SOS and storing events.
  - A radio in repeater mode accepts phones too.
- **On connect:** after WELCOME, the radio sends RIDE_STATE, then SOS_STATUS if its own SOS is active, then SOS_RECEIVED for every active SOS from others, then STATUS (Q26).

### 7.3 Common header (Q22, Q23)

Every phone-link message carries:
- its **type**
- a **request ID** (unsigned 32-bit), chosen by the app. 0 means unsolicited. A reply carries the request's ID.
- a **last** flag for replies in several parts. Single replies always set it.

Unknown fields are ignored. An unknown type, or a missing required field, is answered with ERROR.

**Error codes**, one list for every message:

| Code | Meaning |
| --- | --- |
| UNKNOWN_TYPE | Message type not recognised |
| BAD_FIELD | Field missing, out of range or too long |
| NOT_ALLOWED_NOW | Not allowed in the current state (for example before HELLO) |
| VERSION_MISMATCH | Protocol version differs |
| NO_RIDE | Needs an active ride |
| RIDE_ACTIVE | A ride is already active; end it first |
| SOS_ACTIVE | Not allowed during the rider's own SOS |
| BUSY | Queue full; try later |
| TOO_LARGE | Text or message over its limit |
| STORAGE_FAILED | The radio couldn't save the change |
| SETTINGS_RESET | Stored settings were damaged and have been reset to defaults |
| LOW_BATTERY | Battery too low for this action |
| UPDATE_REJECTED | Firmware image refused |
| NO_CONFIRMATION | Text not confirmed by anyone after 3 resends |

ERROR carries the request ID and a code, with no free text. Messages that can't otherwise fail silently report problems through ERROR, including SHARE_MY_LOCATION, SOS_REPLY and every GET.

### 7.4 Message list

R = required, O = optional. Types follow Section 2.6. "Rider ID" on the phone link is the 3-byte rider tag.

#### Connect

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| HELLO | App → Radio | protocol version (u32, R); app version (text ≤16, R); phone time (time, O) | BLE-001 |
| WELCOME | Radio → App | protocol version (u32, R); firmware version (three u32: major, minor, patch, R); radio ID (8 bytes, R); short ID (text 3, R); board model (enum: POCKET_V3, RAK4631, T114_V2, THINKNODE_M1, R); last update result (enum: NONE, INSTALLED, ROLLED_BACK, O) | BLE-001, DFU-001 |
| GOODBYE | Either | reason (enum: USER, SHUTDOWN, UPDATING, VERSION_MISMATCH, R) | BLE-001 |

On a version mismatch, the radio replies with WELCOME (so the app learns both versions), then GOODBYE with reason VERSION_MISMATCH, and disconnects.

#### Status

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_STATUS | App → Radio | none | BLE-003 |
| STATUS | Radio → App | battery (R); charging (bool, R); GPS (enum: NO_FIX, FIX, UNAVAILABLE, R); position source in use (enum: RADIO, PHONE, NONE, R); clock known (bool, R); channel busy (R); ride contact (u8, R); events waiting (u32, R); presence sending (bool, R); firmware version (R) | BLE-003, MESH-004 |

- **Link status (Q25):** the phone link itself is known to the app. BLE-003's "link status" is **ride contact**: the number of ride members heard in the last 2 × the position interval.
- **Channel busy:** the share of the last 60 s that the radio spent receiving or detecting LoRa activity, or transmitting. Its own transmissions are included.
- **Events waiting:** stored events not yet sent to the app.
- **When STATUS is sent:**
  - in reply to GET_STATUS
  - on a change of charging, GPS state or ride contact
  - when battery changes by 5 points or more
  - at most once every 30 s

#### Ride

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| JOIN_RIDE | App → Radio | ride name (text 1–24, R); ride key (16 bytes, R); tier (1–4, R) | RIDE-001, RIDE-002 |
| END_RIDE | App → Radio | none | RIDE-003 |
| GET_RIDE | App → Radio | none | BLE-002 |
| RIDE_STATE | Radio → App | ride active (bool, R); ride name (O); tier (O); key check value (4 bytes, O); own rider tag (O) | RIDE-001, BLE-002 |

**JOIN_RIDE rules (Q27):**
- Refused with RIDE_ACTIVE if a ride is active.
- Refused with SOS_ACTIVE during the rider's own SOS.
- Refused with BAD_FIELD for an all-zero key, a wrong key length or a tier outside 1–4.
- The ride is saved in one all-or-nothing write and survives reboot.
- RIDE_STATE replies to JOIN_RIDE, END_RIDE and GET_RIDE, reading back what was actually stored. The key itself is never sent back; the key check value proves it.

#### Messaging

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SEND_TEXT | App → Radio | text (1–150 bytes, R) | MSG-001 |
| SEND_RESULT | Radio → App | message ID (u16, R); state (enum: QUEUED, SENT, DELIVERED, FAILED, R); confirmations (u8, O); ride members (u8, O); error code (O) | MSG-002 |
| TEXT_RECEIVED | Radio → App | event number (u32, R); message ID (u16, R); sender rider ID (R); text (R); time received (time, R); RSSI and SNR (R) | MSG-001 |
| GET_MISSED | App → Radio | after event number (u32, R) | MSG-001 |

#### Location

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SHARE_MY_LOCATION | App → Radio | latitude, longitude (R); accuracy (R); fix time (R); heading (O); speed (O) | POS-001, POS-002 |
| LOCATION_UPDATE | Radio → App | rider ID (R); latitude, longitude (R); accuracy (R); fix time (R); source (R); hops travelled (u8, R) | POS-001, POS-002 |

The app judges freshness by fix time, never by arrival time.

#### Fill on Return

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_TRACK | App → Radio | rider ID (R); from time (R); to time (O, default now) | FILL-001 |
| TRACK_POINTS | Radio → App | rider ID (R); points (list of latitude, longitude, fix time; at most 64 per part, R); filled (bool: true when the points arrived through Fill on Return, R) | FILL-001 |

TRACK_POINTS goes in reply to GET_TRACK, in several parts if needed, and also unsolicited whenever a fill answer is received.

#### SOS

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SEND_SOS | App → Radio | latitude, longitude (O: if absent, the radio uses its own fix); accuracy (O); fix time (O) | SOS-001, SOS-002 |
| SOS_STATUS | Radio → App | alert ID (R); state (enum: ACTIVE, ENDED_OK, ENDED_LEFT_RIDE, R); times sent (u8, R); last sent (time, R); copy-that riders (list of rider IDs, R) | SOS-003 |
| SOS_RECEIVED | Radio → App | event number (R); alert ID (R); rider ID (R); latitude, longitude, accuracy, fix time (R); first sent (time, R); repeat number (R) | SOS-001 |
| SOS_REPLY | App → Radio | alert ID (R); alert owner rider ID (R); reply (enum: COPY_THAT, IM_OK, R) | SOS-004 |
| SOS_REPLY_RECEIVED | Radio → App | event number (R); alert ID (R); alert owner rider ID (R); from rider ID (R); reply (R) | SOS-004 |
| SOS_ENDED | Radio → App | event number (R); alert ID (R); alert owner rider ID (R); reason (enum: IM_OK, LEFT_RIDE, R) | SOS-003, SOS-004 |
| GET_SOS | App → Radio | none | SOS-003 |

- An I'm OK sent for someone else's alert is refused with NOT_ALLOWED_NOW.
- GET_SOS replies with SOS_STATUS for the rider's own active alert, and SOS_RECEIVED for every active alert from others.

#### Riders

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_RIDERS | App → Radio | none | MSG-003 |
| RIDERS_LIST | Radio → App | list (at most 30) of: rider ID; radio ID (O until a hello is heard); short ID (O); display name (O); repeater (bool); last heard (seconds ago, u32); RSSI and SNR (O: direct receptions only); in contact (bool) | MSG-003 |
| RIDER_UPDATE | Radio → App | event number (R) plus the fields of one RIDERS_LIST entry | MSG-003 |

- **RIDER_UPDATE is sent when:** a rider is first heard, a name changes, contact is lost (silent for 3 × the position interval) or contact is regained. At most once per 10 s per rider.
- Riders are never removed during a ride.
- **Last heard:** time since any authenticated frame from that rider.
- **Signal strength:** reported only for frames heard directly, which have travelled 0 hops.

#### Nearby

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| NEARBY_RIDER | Radio → App | anonymous ID (R); latitude, longitude (R); heading (R); speed (R); fix age at receipt (R); received (time, R); RSSI and SNR (R) | NEAR-004 |
| GET_NEARBY | App → Radio | none | NEAR-004 |
| NEARBY_LIST | Radio → App | list (at most 32) of NEARBY_RIDER entries | NEAR-004 |

An empty NEARBY_LIST means "no Trail Buddy riders heard". STATUS tells the app whether the radio has a position and whether presence sending is on, so the app can word the empty view correctly (NEAR-008).

#### Settings

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_SETTINGS | App → Radio | none | BLE-002 |
| SETTINGS | Radio → App | all settings in the table below, as stored | BLE-002 |
| CHANGE_SETTINGS | App → Radio | list of setting ID and new value | BLE-002 |
| SETTINGS_RESULT | Radio → App | OK or error code (R); all settings as now stored (R) | BLE-002 |

| ID | Setting | Type | Default |
| --- | --- | --- | --- |
| 1 | Display name | text 1–24 | "Rider " + short ID |
| 2 | Presence sending | bool | on |
| 3 | Repeater mode | bool | off |
| 4 | GPS source preference | enum: RADIO_FIRST, PHONE_FIRST | RADIO_FIRST |

**Rules (Q49, Q50):**
- A change is written all-or-nothing. If any setting ID is unknown or any value is invalid, nothing changes and the reply is BAD_FIELD.
- A value equal to the stored one is not rewritten.
- SETTINGS_RESULT always returns every setting as stored.
- If stored settings are found damaged at boot, the radio resets them to defaults and reports SETTINGS_RESET on the next connection.
- Ride data is a separate stored record with its own all-or-nothing write.

#### Updates

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| START_UPDATE | App → Radio | new version (R); board model (R); image size (u32, R); image hash (32 bytes SHA-256, R) | DFU-001 |
| UPDATE_CHUNK | App → Radio | offset (u32, R); data (≤200 bytes, R) | DFU-001 |
| UPDATE_PROGRESS | Radio → App | bytes received (u32, R); state (enum: RECEIVING, VERIFYING, INSTALLING, REJECTED, R); error code (O) | DFU-001 |
| ABORT_UPDATE | App → Radio | none | DFU-001 |

**Update path (Q52):**
- Updates arrive through these messages and are installed by a two-slot bootloader. It checks an Ed25519 signature over the image, installs the image, and rolls back automatically if the new firmware doesn't confirm itself within 60 s of starting.
- Each board's UF2 bootloader stays as the recovery route over USB.
- **Refused with UPDATE_REJECTED:** a wrong board model, a wrong signature, or an older version than the one installed.
- **Refused with LOW_BATTERY:** battery under 40%.
- **Refused with SOS_ACTIVE:** during the rider's own SOS.
- UPDATE_PROGRESS is sent every 16 chunks and on any error.
- After an interruption, the app resumes from the bytes-received value.
- The radio reboots after VERIFYING. The result reaches the app in the next WELCOME, as last update result.
- The signing key is held by the product owner, never in the repository. The bootloader and signing tool are chosen by the clean team from the approved libraries and recorded in SOURCE-REGISTER.md.

#### Repeater, errors and diagnostics

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| REPEATER_STATUS | Radio → App | frames relayed since boot (u32); uptime (seconds, u32); battery | RPT-001 |
| ERROR | Radio → App | error code (R) | BLE-001 |
| GET_DIAGNOSTICS | App → Radio | none | DIAG-001 |
| DIAGNOSTICS | Radio → App | per kind: sent, received, relayed (u32 each); drops by reason: duplicate, failed authentication, wrong ride, unknown version, malformed, queue full (u32 each); channel busy for each of the last 60 minutes (60 × u8) | DIAG-001 |
| CLEAR_DIAGNOSTICS | App → Radio | none; the reply is DIAGNOSTICS with every counter at zero | DIAG-001 |

Counters start at zero at boot or on clear. They are 32-bit and wrap to zero.

## 8. Ride QR code (RIDE-001, Q29)

Text form: `trailbuddy://ride/` followed by the bytes below in Base64url, without padding.

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | Format version (1) |
| 1 | 1 | Tier (1–4) |
| 2 | 16 | Ride key |
| 18 | 1 | Ride name length (1–24) |
| 19 | n | Ride name (UTF-8) |
| 19 + n | 2 | CRC-16/CCITT-FALSE over all bytes before it (initial value 0xFFFF) |

The maximum is 45 bytes, or 60 characters after encoding. The app refuses a code with the wrong format version or a bad CRC.

## 9. Hardware (Q54)

- **Lead board (decision):** RAKwireless WisMesh Pocket V3.
- **Board definition:** Zephyr's RAK4631 board definition may be used as the starting point. It is part of the open-source Zephyr project (Apache-2.0), which POLICY §14 allows. It is not another mesh project's board file.
- **Pocket V3 profile:** the RAK19026 baseboard pin details come from RAKwireless documentation, which the product owner will add to `docs/hardware/`.
- **Board features used in version 1:** the buzzer, for a new SOS received (Section 6.4), and the power button for pairing (the 2-minute window after power-on). Screens are not used in version 1.
- **Other boards:** added after the core tests pass on the lead board, each with its own profile.

## 10. Diagnostics (Q53)

- **New requirement DIAG-001:** the radio keeps the counters in Section 7.4 in release firmware, for field testing.
- Detailed logging is still removed from release builds, as POLICY §16 says.
- POLICY §16 is amended to say this.

## 11. Test vectors and sample values (Q55, Q56)

These vectors are exact. The clean team's implementation must reproduce them byte for byte.

### 11.1 Key derivation

| Input or output | Hex |
| --- | --- |
| Ride key | `000102030405060708090a0b0c0d0e0f` |
| Radio ID | `1122334455667788` |
| Encryption key | `91623a7d358eb9fae7a1497546128f3c` |
| Ride tag | `20bb` |
| Rider tag (clash counter 0) | `eea021` |
| Key check value | `715305b1` |

### 11.2 Ride frame: position

Inputs:
- frame counter 42
- hops remaining 3, priority class 2
- whole ride
- latitude 35.349120°, longitude −120.247385°
- accuracy 6 m
- fix time 25,000,000
- source radio GPS

| Item | Hex |
| --- | --- |
| Plaintext (kind + fields) | `0180621b02a72bd5f80640787d0100` |
| Envelope | `113220bbeea0212a000000000000` |
| Authenticated data (hops zeroed) | `110220bbeea0212a000000000000` |
| Nonce | `eea0212a00000020bb00000000` |
| Complete frame (37 bytes) | `113220bbeea0212a0000000000005291589d7f1a84be634834d414601e75c39eeb47854de9` |

### 11.3 Presence frame

Inputs:
- anonymous ID a1b2c3d4
- latitude 35.349120° and longitude −120.247385°, rounded to 35.3491° and −120.2474°
- heading 90° (value 60)
- speed 32 km/h
- fix age 2 s

| Item | Hex |
| --- | --- |
| Complete frame (16 bytes) | `12a1b2c3d46c621b02982bd5f83c2002` |

### 11.4 Ride QR code

Inputs: tier 1, the ride key above, name "Sat Dez Loop".

| Item | Value |
| --- | --- |
| Bytes (33) | `0101000102030405060708090a0b0c0d0e0f0c5361742044657a204c6f6f706b0b` |
| Text | `trailbuddy://ride/AQEAAQIDBAUGBwgJCgsMDQ4PDFNhdCBEZXogTG9vcGsL` |

### 11.5 Phone link

- Phone-link messages are protobuf, so their bytes depend on the schema the clean team writes. The clean team encodes each sample below once with the generated code and saves the input and its bytes together in `spec/vectors/`.
- From then on, those saved bytes are the reference for both the firmware tests and the app tests.
- The spec team checks the decoded values against these samples. It reviews results, never code.

| Message | Sample |
| --- | --- |
| HELLO | protocol version 1; app version "1.0.0"; phone time 25,000,000 |
| JOIN_RIDE | name "Sat Dez Loop"; key as 11.1; tier 1 |
| RIDE_STATE | active; "Sat Dez Loop"; tier 1; check value 715305b1; rider tag eea021 |
| SEND_TEXT | "Waiting at the fork" |
| SEND_RESULT | message ID 7; DELIVERED; 3 confirmations; 4 ride members |
| SHARE_MY_LOCATION | 35.349120°, −120.247385°; accuracy 6 m; fix time 25,000,000; heading 90°; speed 32 km/h |
| SEND_SOS | position as SHARE_MY_LOCATION |
| SOS_REPLY | alert ID 3; owner eea021; COPY_THAT |
| CHANGE_SETTINGS | setting 1 = "Dave"; setting 2 = on |
| ERROR | request ID 12; BAD_FIELD |

Samples for the remaining messages follow in v0.4.

### 11.6 Review

The independent security review of Sections 4.3 to 4.6 must be completed before freeze. The product owner arranges the reviewer.

## 12. Changes to REQUIREMENTS.md and POLICY.md

These take effect with this version. Updated files will follow.

- **MSG-001:** text goes to the whole ride only.
- **MSG-002:** Delivered is reported as a count of confirming riders.
- **RIDE-002:** the tiers are as Section 2.5.
- **SOS-004:** only the rider who raised an SOS can cancel it.
- **NEAR-006:** presence is on by default, forced on during the rider's own SOS, and on again after End Ride.
- **New DIAG-001:** diagnostic counters as Section 7.4, kept in release firmware.
- **REG-001:** "the approved channel plan" means Section 2.1 as confirmed by the compliance lab.
- **POLICY §11:** the spec team defines over-the-air layouts byte for byte; the clean team implements them and writes the phone-link schema.
- **POLICY §16:** detailed logging is removed from release builds; DIAG-001 counters stay.
- **POLICY §8 and §17:** the lead board is the WisMesh Pocket V3. Repeater mode becomes a build step after relaying.

## 13. Where each question is answered

| Questions | Section |
| --- | --- |
| Q1, Q3, Q16 | 2.1 |
| Q2 | 3 |
| Q4, Q5 | 1, 7.1 |
| Q6 | 2.2 |
| Q7, Q8 | 2.3 |
| Q9 | 2.4 |
| Q10 | 2.5 |
| Q11 | 4.3, 4.4 |
| Q12, Q14, Q17 | 4.6 |
| Q13 | 4.4 |
| Q15 | 4.5 |
| Q18 | 4.7 |
| Q19, Q20, Q21 | 7.2 |
| Q22, Q23 | 7.3 |
| Q24 | 7.4 Connect |
| Q25 | 7.4 Status |
| Q26 | 7.2 (on connect), 7.4 Ride and SOS |
| Q27 | 7.4 Ride |
| Q28 | 6.7 |
| Q29 | 8 |
| Q30, Q31, Q32 | 3, 6.3 |
| Q33, Q34 | 6.1 |
| Q35 | 2.6 |
| Q36 | 6.2 |
| Q37 | 3, 7.4 Riders |
| Q38 | 6.5 |
| Q39, Q40, Q41 | 6.4 |
| Q42 to Q48 | 6.6, 4.2 |
| Q49, Q50 | 7.4 Settings, 6.2 |
| Q51 | 6.8 |
| Q52 | 7.4 Updates |
| Q53 | 10 |
| Q54 | 9 |
| Q55, Q56 | 11 |

## 14. Still open

- [ ] Compliance lab: confirm the settings and regulatory route in Section 2.1.
- [ ] Mesh simulator: confirm the airtime figures in Section 2.5, especially tier 1 at 10 s and the suppression assumption for tier 4.
- [ ] Security review: Sections 4.3 to 4.6, including the documented limits: replay records lost at reboot, ride members able to forge I'm OK, and rider tags trackable within a ride.
- [ ] Product owner: add the RAK19026 and Pocket V3 documentation to `docs/hardware/`.
- [ ] Spec team: sample values for the remaining phone-link messages (v0.4).
- [ ] Product owner: what counts as "approaching" for the app's warning (app only; it doesn't block firmware).

## 15. Changes from v0.2

- Answers all 56 open questions (Section 13).
- Adds the radio settings proposal, the time rules, the identities, the tier table with an airtime check, and the units table.
- Defines every over-the-air frame byte for byte, with encryption, counters, priorities and relay rules.
- Restates the five Fill on Return rules; the rider's own radio now answers fills.
- Adds GET_RIDE, RIDE_STATE (replacing RIDE_RESULT), GET_SOS and ABORT_UPDATE; GET_MISSED now uses event numbers. The phone link has 43 messages.
- Adds the Bluetooth service, framing, pairing, session rules, a common header and one error-code list.
- Adds DIAG-001; the lead board is the WisMesh Pocket V3.
- Adds exact test vectors for key derivation, a position frame, a presence frame and the QR code.
