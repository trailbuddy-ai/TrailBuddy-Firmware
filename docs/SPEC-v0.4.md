# TrailBuddy Radio Message Spec (Draft v0.4)

23 October 2026 · Spec team · Supersedes Draft v0.3 · Issued together with REQUIREMENTS v0.4 and POLICY v0.4 · Answers OPEN-QUESTIONS.md Q57–Q98 and the 11 partly answered questions from the v0.3 review

## 0. How to read this version

**Which document wins (Q62).** REQUIREMENTS says what the product must do, POLICY says how the clean team works, and this SPEC says exactly how the radio behaves. The three are issued together under one version number. If they disagree, none of them wins: the implementer writes a question in OPEN-QUESTIONS.md and builds nothing that depends on that point until the spec team answers. Until then, for anything affecting SOS, the reading that is safer for the rider applies.

- Section 15 maps every question to the section that answers it.
- Decisions made by the product owner are marked **Decision**. Those made by the policy owner are marked **Policy**.
- Section 16 lists every change from v0.3. Section 14 lists what is still open.
- Nothing here is final until spec v1.0 is frozen.

## 1. How it works

Trail Buddy uses its own radio protocol end to end. The app talks to its own radio over Bluetooth; the radios talk to each other over LoRa. Both conversations are defined here.

```
TrailBuddy app  <-- phone link (Bluetooth) -->  TrailBuddy radio  <-- over the air (LoRa) -->  other TrailBuddy radios
```

**Who designs what:**
- **Over the air:** this spec defines every frame byte for byte (Sections 4 and 5). The clean team implements these layouts exactly. They are fixed binary layouts, not protobuf.
- **Phone link:** this spec defines every message's fields, types, units and limits (Section 7). The clean team writes the protobuf schema from those tables, following the rules in Section 7.1, and hands the schema file to the product owner for the app team.

**Records:** hand-overs are recorded in the clean-room log, which the product owner keeps outside the repository. This spec refers only to files in the repository.

**Market (Decision):** version 1 is for the United States only.

## 2. Foundations

### 2.1 Radio settings and channel plan

**Decision:** all Trail Buddy radios share one common channel. Rides are kept private by their keys, not by separate frequencies.

Proposed settings. These are subject to confirmation by a compliance test lab before any field use:

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

**Regulatory route:** the 500 kHz bandwidth is chosen so that the radio can qualify as a digital-modulation device under FCC Part 15 (47 CFR §15.247), which sets no duty-cycle limit. The lab must confirm this. If it can't, the fallback is frequency hopping, which would change Sections 2.1 and 4. Duty-cycle limits would apply only in other regions, which version 1 doesn't serve (**Policy**).

**Airtime budget:** Trail Buddy's own limit is that normal traffic keeps the channel at or below 30% busy. SOS and SOS replies are exempt and are never delayed by it. Section 6.10 says what a radio does when the channel is busier than that.

**Range trade-off:** these settings send each frame quickly, which is what makes 10-second updates possible. Faster settings reach less far than slower ones. Field testing checks range against NFR-001.

### 2.2 Time (Q75)

- **Trail Buddy time** is seconds since 2026-01-01 00:00:00 UTC, as an unsigned 32-bit number. The value 0 means "unknown". It follows UTC, so GPS time must be converted to UTC using the leap-second offset the GPS module reports. A module that already reports UTC needs no conversion.
- **Setting the clock**, in order:
  1. Whenever the radio has a GPS fix, it sets its clock from GPS if the two differ by more than 1 s.
  2. If the radio has had no GPS fix for 10 minutes, it adopts the phone's time when the clock is unknown or differs from the phone's by more than 2 s. The phone's time comes from HELLO, and from the fix time of each SHARE_MY_LOCATION.
  3. Otherwise the clock runs freely. The radio's crystal keeps it within 2 s per hour.
- If neither source has set the clock since boot, time is unknown and every time field the radio sends is 0. Items stored while the clock was unknown keep 0; they are not filled in later.
- **The clock may step backwards** when corrected. Intervals, timeouts and ages always use the radio's uptime counter, never the clock, so a step doesn't affect them.
- **Presence** carries a fix age in seconds, not a time, so it works without a clock.

### 2.3 Identities (Q71, Q72, Q73)

| ID | Size | What it is | Where it appears |
| --- | --- | --- | --- |
| Radio ID | 8 bytes | Random, from the hardware random number generator at first boot. Kept through reboots and firmware updates. A new one is made only by factory reset (Section 6.11). | Phone link (WELCOME, rider entries); inside encrypted hello frames only. Never sent in clear. |
| Short ID | 3 characters | See below. | App display beside names; default display name. |
| Rider tag | 3 bytes | Ride-scoped, derived from the ride key and the radio ID (Section 4.3). | Every ride frame envelope; phone link as the rider ID. |
| Anonymous ID | 4 bytes | Random. See Section 6.6 for when it changes. | Presence frames only. |

**Short ID:**
- Take the first two bytes of the radio ID, most significant bit of the first byte first, and keep the first 15 bits.
- Split them into three 5-bit groups, first group first.
- Write each group as one character from `ABCDEFGHIJKLMNOPQRSTUVWXYZ234567`, where A = 0 and 7 = 31. This is the Base32 alphabet of RFC 4648 §6.
- Radio ID `1122334455667788` gives `CER` (vector in 11.1).

**Rider tag rules:**
- The value 000000 is reserved for "whole ride". A radio whose derived tag is 000000 moves to the next clash counter.
- **Clash:** if a radio receives a hello with its own rider tag but a different radio ID, it re-derives its tag with the next clash counter and sends a new hello at once. The clash counter is stored with the ride record and resets to 0 at End Ride.
- **Merging:** receiving radios key their riders list on the radio ID once a hello has been heard. When a known radio ID appears under a new tag, its entry moves to the new tag, so one rider never appears twice.

**Privacy limit, documented:** during a ride, an outsider can follow a rider tag for that ride's duration, but can't learn the name, radio ID or messages.

### 2.4 Rides (RIDE-004, Q72)

- **Decision:** at most 30 riders per ride. Repeaters count towards the 30.
- A radio holds at most one active ride.
- Every ride frame carries a 2-byte ride tag, so a radio decrypts only frames for its own ride.
- **A 31st radio** with the same QR code still works: radios process and relay its frames, and an SOS from it is handled like any other. The riders list holds 30 entries. RIDE_STATE reports "ride over limit", so the host can be told.

### 2.5 Group tiers

**Decision:** four tiers, chosen when the ride is created, carried in the QR code, fixed for the ride.

| Tier | Riders | Hop limit | Position interval | Presence while moving | Delivered hop limit | Gap threshold (Fill) |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 2–4 | 3 | 10 s | every 10 s | 3 | 60 s |
| 2 | 5–9 | 4 | 45 s | every 10 s | 4 | 135 s |
| 3 | 10–19 | 5 | 120 s | every 20 s | 2 | 360 s |
| 4 | 20–30 | 7 | 90 s | every 20 s | 2 | 270 s |

- A radio with no ride sends presence every 10 s while moving.
- Every radio sends presence every 60 s while stopped.

**Airtime check (Q66), approximate.** It uses the settings in 2.1, and these assumptions:
- relay suppression (4.6) keeps each relayed frame to at most 4 transmissions on average
- one hello per rider every 10 minutes
- one 60-byte text per minute per ride, confirmed by every other rider
- one Fill on Return episode every 10 minutes per ride: an outline and its detail for a 30-minute gap

| Tier | Positions | Presence | Hellos | Texts + Delivered | Fill | One ride, total |
| --- | --- | --- | --- | --- | --- | --- |
| 1 (4 riders) | 11.3% | 1.8% | 0.2% | 2.1% | 0.7% | 16.2% |
| 2 (9 riders) | 5.7% | 4.1% | 0.5% | 4.1% | 0.7% | 15.1% |
| 3 (19 riders) | 4.5% | 4.3% | 1.1% | 4.5% | 0.7% | 15.1% |
| 4 (30 riders) | 9.5% | 6.8% | 1.7% | 6.7% | 0.7% | 25.4% |

- **One ride on its own** fits the 30% budget in every tier. Without the slower presence and the 2-hop Delivered limit in tiers 3 and 4, tier 4 would reach about 38%.
- **Two rides in range plus 10 other Trail Buddy riders**, for example at a trailhead, would go over 30% in every tier. That is the case Section 6.10 handles: radios slow their own presence first, then their positions, until the channel recovers.
- The mesh simulator must confirm these figures (build step 5), and a 4-rider field ride must confirm tier 1 at 10 s.

### 2.6 Units on the wire

| Quantity | Type | Unit | Unknown value |
| --- | --- | --- | --- |
| Latitude, longitude | signed 32-bit | millionths of a degree, rounded half away from zero | 0x7FFFFFFF for both, only in an SOS with no position (6.4) |
| Accuracy | unsigned 8-bit | metres, 254 = 254 m or worse | 255 |
| Speed | unsigned 8-bit | km/h, rounded to nearest | 255 |
| Heading | unsigned 8-bit | 1.5° steps (0–239), rounded to nearest step; 360° becomes 0 | 255; always 255 below 3 km/h |
| Fix age (presence) | unsigned 8-bit | seconds, 254 = 254 s or older | 255 |
| Time | unsigned 32-bit | Trail Buddy time (2.2) | 0 |
| Signal strength | signed 8-bit RSSI, signed 8-bit SNR | dBm; SNR in quarter-dB steps | RSSI 0 |
| Battery | unsigned 8-bit | percent 0–100 | 255 |
| Channel busy | unsigned 8-bit | percent 0–100 | 255 |
| Text and names | UTF-8 (RFC 3629) | length in bytes; no control characters (U+0000–U+001F, U+007F) | — |

All names, including the hello name, the display name and the ride name, are 1–24 bytes (Q62).

## 3. Product decisions

| Topic | Decision |
| --- | --- |
| Text to one rider | Not in version 1. Text goes to the whole ride only. |
| Maximum text | 150 bytes of UTF-8. Longer text is refused, not split. |
| Delivered | The sender sees a count: "Delivered to 3 of 4". The count is defined in 6.3. |
| Presence default | On by default; always on during the rider's own SOS; back on after End Ride. |
| Battery sharing | Not in version 1. |
| GPS source | Radio GPS first, phone GPS as fallback. The rider can't change this; there is no setting for it (Q61). |
| Who cancels an SOS | Only the rider who raised it. |
| Diagnostics | Counters stay in release firmware; detailed logging is removed. |
| SOS with no ride (Q79) | An SOS needs a ride in version 1. SEND_SOS with no ride is refused with NO_RIDE, and the app says "Create or join a ride to use SOS". |
| SOS with no GPS fix (Q80) | An SOS is never held back. See 6.4. |
| SOS across a reboot (Q81) | An active SOS survives a reboot or battery swap and resumes. See 6.4. |
| "Left the ride" reply (Q83) | Sent twice, 10 s apart. See 6.4. |
| "Never dropped" (Q59) | An SOS and an "I'm OK" are never dropped. Under extreme load a "Copy that" may be. See 4.5. |
| Fill on Return target (Q58) | "Filled" means the outline of the whole gap, up to 30 minutes, reaches the app within 60 s of the rider's return, and the full detail within 3 minutes. |
| Battery target conditions (Q2) | At least 12 hours with: tier 1, presence on, moving half the time, the board's standard battery, around 20 °C. |
| Range target (Q2) | On the August field-test route with two riders, total blackout time and the longest blackout must be no worse than that test's result. The longest blackout was 43 minutes. |
| Version 1 boards | Three boards: the RAK19026 VC base board with the RAK4631 core (lead board, built on first), the Heltec Mesh Node T114 V2 and the Elecrow ThinkNode M1. The WisMesh Pocket V3 is added when it ships. See 9. |
| "Approaching" definition | Deferred by the product owner. App only; doesn't block firmware. |

## 4. Over the air: frames

All multi-byte numbers are little-endian.

### 4.1 Ride frame (encrypted)

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | High 4 bits: protocol version (1). Low 4 bits: frame type (1 = ride frame). |
| 1 | 1 | High 4 bits: hops remaining (0–7 in version 1). Low 4 bits: priority class (4.5). |
| 2 | 2 | Ride tag |
| 4 | 3 | Sender rider tag |
| 7 | 4 | Frame counter (4.4) |
| 11 | 3 | Destination rider tag; 000000 = the whole ride |
| 14 | n | Encrypted content: 1 kind byte, then the kind's fields (5) |
| 14 + n | 8 | Authentication tag |

Envelope overhead is 22 bytes. The smallest valid ride frame is 23 bytes; the largest is 176 bytes.

### 4.2 Presence frame (not encrypted, never relayed)

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | High 4 bits: protocol version (1). Low 4 bits: frame type (2 = presence). |
| 1 | 4 | Anonymous ID |
| 5 | 4 | Latitude, rounded to the nearest 0.0001° (about 11 m), half away from zero |
| 9 | 4 | Longitude, rounded the same way |
| 13 | 1 | Heading |
| 14 | 1 | Speed |
| 15 | 1 | Fix age |

Exactly 16 bytes.

### 4.3 Keys and derived values

All values are derived from the 16-byte ride key with HKDF-SHA256 (RFC 5869; SHA-256 is FIPS 180-4), no salt, using these info labels:

| Value | Length | Info label |
| --- | --- | --- |
| Encryption key | 16 bytes | `trailbuddy v1 enc` |
| Ride tag | 2 bytes | `trailbuddy v1 ride tag` |
| Rider tag | 3 bytes | `trailbuddy v1 rider tag` + radio ID (8 bytes) + clash counter (1 byte, starts at 0) |
| Key check value | 4 bytes | `trailbuddy v1 check` |

**Encryption:** AES-128 (FIPS 197) in CCM mode (NIST SP 800-38C, RFC 3610), with a 13-byte nonce and an 8-byte authentication tag.
- **Nonce:** sender rider tag (3) + frame counter (4) + ride tag (2) + four zero bytes.
- **Authenticated, not encrypted:** the 14-byte envelope with the hops-remaining bits set to zero.
- **Encrypted:** the kind byte and its fields.

**Policy:** these choices are decided for building, and only the independent security review before freeze can change them. Any change it makes comes as a new spec version.

### 4.4 Frame counter, duplicates and replays (Q70, Q71)

- Each radio keeps one 32-bit frame counter. It increases by 1 for every ride frame the radio creates, never for relays, and never repeats.
- **The counter ceiling:**
  - The radio stores a ceiling in flash, 1024 above the current counter, in two copies, each with a CRC.
  - At boot it starts from the stored ceiling and writes a new one 1024 higher before sending anything.
  - While running, when the counter comes within 64 of the ceiling, the radio writes the next ceiling before it sends another frame.
- **If a ceiling write fails,** the radio stops sending ride frames, sets "storage failed" in STATUS and sends ERROR STORAGE_FAILED. It keeps receiving and keeps sending presence. If both stored copies are damaged at boot, it does the same. The only recovery is End Ride and joining a new ride, because a new ride key makes reused counters harmless.
- **Duplicate memory:** a frame's identity is (sender rider tag, frame counter). The radio remembers up to 256 identities; each expires after 10 minutes, and the oldest is evicted when full.
  - Its own frames are entered too, so their echoes are never relayed.
  - A duplicate is counted for relay suppression (4.6) before it is dropped.
- **Replay window:** for each sender, the radio keeps the highest counter seen and a 64-frame window below it, for up to 32 senders, evicting the least recently heard. Frames below the window, or already marked in it, are dropped. The window protects against replays; the duplicate memory serves relaying. Neither is kept across reboot; this limit is documented for the security review.
- Every SOS repeat is a new frame with a new counter.

### 4.5 Priority classes and queues (Q59, Q69)

| Class | Kinds | Queue length (own / relay) | When full |
| --- | --- | --- | --- |
| 0 | SOS, SOS reply | 64 shared | See below |
| 1 | Presence | 1 own | The newest replaces the queued one |
| 2 | Position | 1 per rider tag, own and relayed, up to 31 | The newest position of a rider replaces that rider's queued one |
| 3 | Delivered | 4 / 8 | Oldest relayed entry dropped first, then oldest own |
| 4 | Hello, hello request | 2 / 4 | A newer hello from the same sender replaces the queued one; otherwise oldest relayed dropped |
| 5 | Text | 4 / 8 | A relayed text is dropped. An own text is refused with BUSY. |
| 6 | Fill ask, fill answer | 4 / 8 | Oldest relayed entry dropped first, then oldest own |

- Own frames and relayed frames share each queue, but each class keeps part of its space for the radio's own frames, as shown.
- **Class 0 (Decision: an SOS and an "I'm OK" are never dropped):**
  - A newer repeat of the same alert replaces the queued older one.
  - Replies are keyed on alert owner, alert ID, sender and reply value, so two riders' "Copy that" never replace each other.
  - When the queue is full, the oldest "Copy that" is dropped. SOS frames, "I'm OK" and "left the ride" replies are never dropped.
  - 64 entries hold the worst case of 30 riders.
- A relay cancelled by suppression leaves its queue.
- Every drop because a queue is full is counted (DIAGNOSTICS, "queue full").
- Lower class numbers always go first. A transmission already in progress is never cut off. The longest frame lasts about 230 ms on air.

### 4.6 Relaying (Q68)

A radio relays a ride frame only if all of these hold:
- it carries the radio's own ride tag
- it passes authentication
- it isn't a duplicate
- hops remaining is above 0
- it isn't addressed only to this radio

Radios don't relay other rides' frames. Presence frames are never relayed.

**Delay before relaying:**
- **SOS and SOS replies:** uniform random 20–150 ms. Always relayed; never suppressed.
- **Everything else:**
  - Delay = 20 ms + s × 1000 ms × u + 200 ms × v, where u and v are independent uniform random numbers from 0 to 1.
  - s is set by the SNR of the first copy received, in dB (the wire value divided by 4): s = (SNR + 10) / 20, clamped to 0.1–1.0.
  - So frames heard weakly relay sooner than frames heard strongly.
- **Suppression:** while waiting, each further reception of the same identity counts one, whatever its hops remaining; the first copy received doesn't count. At 2, the radio cancels its relay.
- The relayed frame is identical except that hops remaining is reduced by 1. Because those bits aren't authenticated, a receiver drops any frame whose hops remaining is above that kind's starting value.

This combines the distance-based and counter-based schemes from Ni, Tseng, Chen and Sheu, "The Broadcast Storm Problem in a Mobile Ad Hoc Network", MobiCom 1999.

**Per-kind rules:**

| Kind | Destination | Starting hops | Class | Acknowledged |
| --- | --- | --- | --- | --- |
| Position | Whole ride | Tier hop limit | 2 | No |
| Hello | Whole ride | Tier hop limit | 4 | No |
| Hello request | Whole ride | Tier hop limit | 4 | Answered by hellos |
| Text | Whole ride | Tier hop limit | 5 | By Delivered |
| Delivered | Original sender | Delivered hop limit (2.5) | 3 | No |
| SOS | Whole ride | 7 | 0 | By SOS replies |
| SOS reply | Whole ride | 7 | 0 | No |
| Fill ask | Whole ride | Tier hop limit | 6 | By fill answer |
| Fill answer | Whole ride | Tier hop limit | 6 | No |
| Presence | Any Trail Buddy radio | Not relayed | 1 | No |

### 4.7 Sending a frame: listen before talking (Q64)

Every transmission follows these steps. A relay goes through them after its relay delay.

1. Take the queued frame with the lowest class number; within a class, the oldest. A frame that arrives in a lower class before the transmission starts goes first.
2. **Channel check:** run the radio's channel activity detection (SX1262 CAD) once.
3. **Clear:** transmit at once.
4. **Busy, normal frame:**
   - Wait a uniform random time from 0 to 50 ms × 2^k, with k starting at 1 and rising by 1 after each busy check, up to k = 5 (up to 1.6 s).
   - Then go back to step 1.
   - After 8 busy checks for the same frame, drop it and count it under "gave up".
5. **Busy, SOS or SOS reply:** wait a uniform random 0–100 ms once, then transmit whether or not the channel is clear. An SOS never waits for a clear channel, so it is delayed by at most one frame in progress plus 100 ms.
6. After transmitting, the radio returns to receiving at once.

### 4.8 Receive checks (Q67)

Any frame that fails a check is dropped, not relayed, and counted under the reason shown. Checks run in this order:

| # | Check | Drop reason |
| --- | --- | --- |
| 1 | Protocol version is 1 | Unknown version |
| 2 | Frame type is 1 or 2 | Malformed |
| 3 | Length: presence exactly 16; ride frame at least 23 | Malformed |
| 4 | Ride frame: ride tag is this radio's ride | Wrong ride (no further checks) |
| 5 | Authentication passes | Failed authentication |
| 6 | Not a duplicate, and inside the replay window | Duplicate |
| 7 | Kind is 0x01–0x09; any other kind is dropped | Malformed |
| 8 | Content length is exactly what the kind's fields require, including length bytes and point counts (fill points 1–11, rider count 1–8) | Malformed |
| 9 | Hops remaining is not above the kind's starting hops | Malformed |
| 10 | Latitude within ±90°, longitude within ±180°, except the SOS no-position marker | Malformed |
| 11 | Fix time not 0 (except an SOS with no position), and when the clock is known, no more than 60 s in the future | Malformed |
| 12 | Reserved bits are 0 (hello flags bits 1–7; fill part byte as defined) | Malformed |
| 13 | Text and names are valid UTF-8 with no control characters, 1 byte or longer | Malformed |
| 14 | Destination is the whole ride or this radio's tag, or the frame is to be relayed | Not an error; addressed to another rider |

Presence frames go through checks 1–3 and 10, then the plausibility rules in 6.6.

## 5. Over the air: kinds inside a ride frame

The first encrypted byte is the kind.

| Kind | Byte | Fields (in order) | Size after kind byte |
| --- | --- | --- | --- |
| Position | 0x01 | latitude (4), longitude (4), accuracy (1), fix time (4), source (1: 0 = radio GPS, 1 = phone GPS) | 14 |
| Hello | 0x02 | radio ID (8), anonymous ID (4), flags (1: bit 0 = repeater; bits 1–7 = 0), name length (1, 1–24), name | 15–38 |
| Hello request | 0x03 | none | 0 |
| Text | 0x04 | message ID (2), text length (1, 1–150), text | 4–153 |
| Delivered | 0x05 | message ID (2), original sender rider tag (3) | 5 |
| SOS | 0x06 | alert ID (2), latitude (4), longitude (4), accuracy (1), fix time (4), time first sent (4), repeat number (1, stops at 255) | 20 |
| SOS reply | 0x07 | alert ID (2), alert owner rider tag (3), reply (1: 1 = copy that, 2 = I'm OK, 3 = ended because the owner left the ride) | 6 |
| Fill ask | 0x08 | rider count (1, 1–8), then per rider: rider tag (3), from time (4), to time (4) | 12–89 |
| Fill answer | 0x09 | rider tag (3), range from (4), range to (4), part (1: bit 7 = outline, bits 0–6 = part number from 0), total parts in this pass (1), point count (1, 1–11), then per point: latitude (4), longitude (4), fix time (4) | 26–146 |

**Message IDs and alert IDs (Q74):**
- 16-bit counters kept by the sending radio and stored in flash, written 16 ahead like the frame counter, so they survive reboots.
- They start at 1, skip 0, and wrap from 65535 to 1.

## 6. Behaviour

### 6.1 Position (POS-001 to POS-003, Q57, Q77)

- **Schedule:** the radio sends a position at the tier interval, with up to 10% random jitter added.
- **Source (Decision: radio first):**
  1. The radio's own GPS fix, if it is no older than 2 × the tier interval.
  2. Otherwise the phone's latest position, if it is no older than 2 × the tier interval.
  3. Otherwise the newer of the two, if it is no older than 10 minutes.
  4. Otherwise nothing is sent, and STATUS reports "no position".

  The position always carries its true fix time. Receivers judge freshness by it.
- The app sends SHARE_MY_LOCATION whenever the phone has a new fix, at most once per second. There is no reply.
- After the phone disconnects, the radio keeps sending positions from its own GPS.
- **Moving or stopped:**
  - Moving means 5 km/h or more for 10 s. Stopped means below 3 km/h for 30 s.
  - Between those speeds the previous state holds. At boot, and whenever speed is unknown, the state is "moving".
  - GPS duty and presence use the same state.
- **GPS duty (POS-003):** the GPS runs continuously while moving or while presence is on. When the rider has been stopped for 5 minutes with presence off, it samples once per minute. A stopped rider keeps sending positions at the tier interval, using the latest sample (rule 3 above), so they never go silent and never trigger Fill on Return.

### 6.2 Rider hello (MSG-003, Q76)

- A radio sends a hello:
  - when it joins a ride
  - when its name changes
  - when its rider tag changes
  - when its anonymous ID changes
  - every 10 minutes
  - in reply to a hello request
- A radio that hears a frame from a rider tag it has no hello for sends one hello request, at most once per minute. Each radio that hears a request replies after a random 0–5 s delay.
- **Display name:** 1–24 bytes of UTF-8 with no control characters. The default is "Rider " plus the short ID. Two riders may share a name; the app shows the short ID beside duplicates.

### 6.3 Messages (MSG-001, MSG-002, Q60, Q74, Q86)

**In contact (one definition, used everywhere):** a ride member is in contact if any authenticated frame from them has been heard within 3 × the tier's position interval, with a minimum of 60 s. STATUS ride contact, RIDERS_LIST "in contact" and the Delivered total all use it.

**Texts:**
- Text goes to the whole ride, 150 bytes maximum.
- Every rider radio that receives a text stores it and sends Delivered to the sender after a random 0–2 s delay, whether or not its phone is connected. Repeaters don't send Delivered.
- **The Delivered count** ("Delivered to 3 of 4"):
  - The first number is distinct riders who confirmed.
  - The second is the ride members in contact when the text was first sent, not counting the sender or repeaters.
  - A rider's confirmation counts once, however many arrive.
- **Sender's side:**
  - SEND_RESULT reports "queued", then "sent", then "delivered" with the count each time a new rider confirms.
  - With no confirmation after 20 s, the radio resends the text with the same message ID and a new frame counter, up to 3 times. After the third resend with none, SEND_RESULT reports "failed", reason NO_CONFIRMATION.
- **Receiver's side:**
  - A radio remembers (sender rider tag, message ID) for 30 minutes.
  - A resent text it already holds creates no new event and isn't passed to the app again, but it sends Delivered again, because the first may have been lost.

**Stored events:**
- The radio numbers everything it stores for the app with a 32-bit event number: texts, SOS events, SOS replies and rider changes. Event numbers only go up; the next number is stored in flash, like the counters in 5, so it survives reboots and End Ride.
- **Storage:**
  - The radio keeps the last 200 events in memory.
  - SOS events are also kept in flash, and stay until that SOS has ended and a further 24 hours have passed.
  - An SOS's repeats update its existing event rather than creating new ones.
- **An event counts as sent** once it has been written to a connected app.
- **GET_MISSED:**
  - The app sends the last event number it holds.
  - The radio answers with EVENTS_INFO first, giving the oldest event number it still holds and the newest. If the oldest is above the app's number plus 1, events were lost and the app can say so.
  - It then replays the events after the app's number as their normal messages, in order, each carrying the request ID, with the last one flagged.
- **Events waiting** in STATUS counts stored events not yet sent to the app.

### 6.4 SOS (SOS-001 to SOS-004, Q79–Q83, Q90)

- **Needs a ride (Decision):** SEND_SOS with no active ride is refused with NO_RIDE.
- SEND_SOS starts an alert, and the radio assigns the alert ID. The radio's reply is SOS_STATUS with the request ID.
- **Position (Decision: an SOS is never held back):**
  - The SOS goes at once with the best position available: the source rules in 6.1 first. If those give nothing, the last known position of any age, with its true fix time, so the app can show how old it is.
  - If the radio has never had a position since it was switched on, the SOS goes with the "no position" marker: latitude and longitude 0x7FFFFFFF, accuracy 255, fix time 0.
  - Each repeat carries the latest position.
- **Schedule:**
  - The first send is at time t0.
  - Then t0 + 10 s.
  - Then t0 + 300 s × n for n = 1, 2, 3 and so on, each with a random 0–5 s added (never subtracted).
  - The first send and the 10 s repeat have no added jitter. A missed slot is not caught up.
- **Repeat number:** 0 on the first send, then +1 for each repeat. It stops at 255; it doesn't start again from 0.
- **Survives reboot (Decision):**
  - The radio stores the alert ID, first-sent time, repeat number and latest position in flash when the SOS starts and at every repeat.
  - After a reboot or battery swap, it sends a repeat as soon as it is running, then carries on. If the clock is known, it continues the schedule from t0; if not, from the moment it rebooted.
- A second SEND_SOS while an alert is active updates its position only; there is one active SOS per radio.
- The radio keeps repeating whether or not the phone is connected.
- **I'm OK:** accepted only from the alert's owner. Every ride member holds the same key, so a member could in principle forge one. This limit goes to the security review.
- **Copy that:** informational only; it doesn't stop repeats.
- **Ending:**
  - An SOS ends when the owner sends I'm OK, or when the owner ends the ride.
  - **End Ride during the owner's own SOS (Decision):** the radio sends the "owner left the ride" reply twice, 10 s apart. It keeps the ride key until both copies have been sent, or 60 s have passed. Then it erases the ride and reports RIDE_STATE (not active).
  - Radios that never hear the end keep showing the SOS with its last-heard time. It is never ended automatically for lack of signal.
- **SOS_STATUS** (the owner's own alert) is sent:
  - in reply to SEND_SOS
  - after every send and repeat
  - after each new Copy that
  - when the alert ends

  Its Copy-that list holds the distinct riders who replied Copy that to this alert, and clears when the alert ends. "Last sent" is 0 when the clock is unknown.
- **To the app:**
  - SOS_RECEIVED is sent for a new alert and for every repeat, with the alert's original event number; the app de-duplicates by alert.
  - SOS_ENDED is sent when an end is heard.
- **Alert on the radio (Q90, Decision):** none. The radio makes no SOS light or sound, because a rider wouldn't see or hear it. The rider is alerted by the app: an SOS banner on the Map page and an SOS message with GPS coordinates on the Messages page, both driven by SOS_RECEIVED.

### 6.5 Fill on Return (FILL-001, Q58, Q84, Q38)

**Storage:** each radio keeps its own position history for the last 2 hours, one point every 10 s, and the positions it has heard from ride members (6.12).

**Definitions:**
- **Return:** the moment the first authenticated frame, direct or relayed, is heard from a ride member after a silence longer than the tier's gap threshold.
- **Gap:** from the fix time of that rider's last position before the silence to the fix time of their first position after it. If that is more than 30 minutes, the gap is the latest 30 minutes before the return.
- **Filled (Decision):** the outline of the whole gap has reached the app within 60 s of the return, and the full detail within 3 minutes.

**The rules:**
1. **Gap:** a ride member returns, and has moved more than 50 m, measured between their last position before the silence and their first position after it.
2. **Ask:**
   - Wait a random 2–8 s.
   - Send one fill ask listing every rider who returned, up to 8 per ask, each with their gap.
   - Leave out any rider for whom another radio's ask with an overlapping gap was heard during the wait.
   - A radio whose clock is unknown waits until it is known before asking.
3. **Answer:**
   - Only the returning rider's own radio answers, from its own history, for its own tag. This works whether or not its phone is connected.
   - It answers each asked gap in one pass:
     - First the **outline:** one point per 60 s of the gap, up to 30 points, in up to 3 parts.
     - Then the **detail:** every stored point in the gap, at 10 s spacing, in up to 17 parts. The detail repeats the outline's points, so receivers simply drop points they already hold.
   - Each part names the range it covers, its part number and the total parts in its pass (outline or detail), so receivers know when a pass is complete.
   - The answer goes to the whole ride, so every radio with the same gap uses it.
   - An ask whose gap was fully covered by an answer this radio sent in the last 15 s is skipped.
4. **Retry:**
   - A gap is **answered** when every part of the detail pass covering it has been received.
   - If any part is missing 30 s after the last part was heard, or after the ask if nothing came, the radio asks again for just the missing sub-range.
   - After 4 asks for the same gap, it waits for the rider's next return.
5. **Channel:**
   - Fill frames are sent only on a channel-busy reading under 60 s old (6.10).
   - The rate depends on that reading:
     - below 15%: up to 1 fill frame every 3 s
     - from 15% to 25%: 1 every 10 s
     - above 25%: none
   - Any queued SOS frame blocks fill. Blocked frames wait and go when the block ends.

**Timing:**
- At one frame every 3 s, a 30-minute gap's outline (3 parts) takes about 10 s, and its detail (17 parts) about 50 s more. That meets the 60 s and 3-minute targets with a margin for the ask and relays.
- At 1 every 10 s, the outline takes about 30 s and the detail about 3 minutes.

**To the app:**
- Received fill points are added to the radio's stored history of that rider. Points already held, same rider and fix time, are dropped.
- New points reach the app as TRACK_POINTS with "filled" set, sorted by fix time, outline first.
- GET_TRACK returns stored history only; it doesn't cause a fill ask.

### 6.6 Nearby riders (NEAR-001 to NEAR-008, Q76, Q78, Q47)

- **Sending:**
  - Presence goes at the interval in 2.5, with ±1 s of random jitter.
  - It uses the radio's GPS fix if it is no older than 10 s, otherwise the phone's latest fix if it is no older than 10 s. Otherwise no presence is sent.
  - The fix age is measured with the uptime counter from when the fix was taken, or for a phone fix, from when SHARE_MY_LOCATION arrived. That works with an unknown clock.
  - Heading and speed come from the same source; 255 when unknown.
- **Not relayed:** a presence frame travels one hop only.
- **Anonymous ID changes:**
  - A new one is made at every boot, and when 6 hours of uptime have passed since the last change.
  - It is applied while the rider is stopped, so one encounter doesn't show a rider twice. But it is always applied within 24 hours of the last change, even while moving.
  - The radio sends a hello as soon as its ID changes, so its ride mates can keep hiding it.
- **What reaches the app (NEAR-004):** the radio passes every presence frame it receives to the app, except:
  - its own
  - those from members of its own ride, recognised by the anonymous IDs in their hellos
  - reports that fail the plausibility check
  - repeat reports from the same anonymous ID within 5 s

  It tracks up to 32 anonymous IDs, evicting the one heard least recently, and an entry expires 5 minutes after its last accepted report.
- **Plausibility:**
  - The radio compares each report with the last accepted report from that ID. The distance between them is divided by the time between receipts, adjusted by the difference in their fix ages.
  - Above 200 km/h, the report is rejected and doesn't replace the last accepted one.
  - The first report from an ID is always accepted.
  - The remaining risk of fake presence is accepted and documented.
- **Turning presence off** stops sending only; receiving is always on. Presence is forced on during the rider's own SOS, and on again after End Ride.
- **Alerts:** approaching-rider warnings are made by the app only in version 1.
- **Combining with position:** presence and ride position stay separate signals in version 1.

### 6.7 End Ride (RIDE-003, Q83, Q87)

| Item | After End Ride |
| --- | --- |
| Ride key, name, tier, tags, clash counter | Erased, after any "left the ride" replies have been sent (6.4) |
| Own active SOS | Ended, with the "left the ride" reply sent twice first |
| Queued ride frames | Dropped |
| Stored texts, ride events, track history, known riders | Erased (SOS events stay in flash for 24 hours) |
| Display name | Kept |
| Presence sending | On |
| Repeater mode | Off |
| Frame counter, message and alert ID counters, event number | Kept; they never reset |
| Diagnostics counters | Kept |

If any write fails, RIDE_STATE carries the error code STORAGE_FAILED. End Ride affects only the local radio.

### 6.8 Repeater mode (RPT-001, Q89)

- **Set-up:** the phone joins the radio to a ride, then turns repeater mode on. CHANGE_SETTINGS returns NO_RIDE with no ride, and SOS_ACTIVE during the rider's own SOS. The radio keeps the ride and the mode through reboots until the phone ends the ride or turns repeater mode off.

| A repeater… | |
| --- | --- |
| Relays its ride's frames | Yes |
| Sends a hello every 10 minutes, with the repeater flag set, and answers hello requests | Yes |
| Sends positions or presence | No |
| Sends Delivered | No; it isn't counted in Delivered totals or ride contact |
| Counts towards the 30-rider limit and appears in RIDERS_LIST, flagged | Yes |
| Answers fill asks | No; it has no position history. It relays fill frames. |
| Keeps stored events and passes nearby presence to a connected phone | Yes |
| Accepts a phone and opens the pairing window like any radio | Yes |

REPEATER_STATUS is sent together with STATUS whenever repeater mode is on.

### 6.9 Settings rules (Q87, Q88)

- A change is written all-or-nothing. If any setting ID is unknown or any value is invalid, nothing changes and the reply is BAD_FIELD.
- A value equal to the stored one is not rewritten.
- SETTINGS_RESULT always returns every setting as stored.
- **Presence turned off during the rider's own SOS:** stored, but it takes effect only when the SOS ends. During the SOS, SETTINGS shows the stored value and STATUS shows presence sending as on.
- Each stored record (Section 6.12) has two copies and a CRC.

| Record damaged in both copies at boot | What the radio does | Reported (sent right after WELCOME, before RIDE_STATE) |
| --- | --- | --- |
| Settings | Resets them to defaults; repeater mode off | ERROR SETTINGS_RESET |
| Ride | Erases the ride | ERROR RIDE_RESET, then RIDE_STATE not active |
| Active SOS | The SOS ends locally; it can't be resumed | ERROR SOS_RESET |
| Counter ceiling | Stops sending ride frames (4.4) | ERROR STORAGE_FAILED |

### 6.10 Channel busy and the airtime budget (Q65, Q66)

**Measuring:**
- Every second the radio adds up the time it spent:
  - transmitting
  - receiving any LoRa frame with a valid header on the channel, from any ride, presence, or otherwise, worked out from the frame's length with the standard LoRa time-on-air formula
  - 10 ms for each channel check that found activity without a frame following
- **Channel busy** is that total over the last 60 s, as a percentage, updated every second.
- For the first 60 s after boot, it is computed over the time so far. During the first 10 s it is reported as 255 (unknown).

**Above the budget (runtime rule):**
- If channel busy stays above 30% for 60 s, the radio slows its own traffic in steps. It takes the next step after another 60 s if the channel is still above 30%:
  1. presence interval doubled
  2. position interval × 1.5
  3. fill frames stopped
- When busy stays below 20% for 60 s, it undoes one step at a time, in reverse order.
- SOS and SOS replies are never slowed.
- STATUS reports the current step, 0–3, so the app can say updates are slower because the channel is busy.

### 6.11 Factory reset, pairing and the button (Q71, Q94)

Every version 1 board has one **user button**, named for each board in Section 9. The status light (9) doesn't show pairing; the app shows whether a phone is paired.

| Action | Result |
| --- | --- |
| Press the button for 2 s while running | Opens the pairing window for 2 minutes |
| The first 2 minutes after the radio starts | Pairing window open |
| Hold the button for 10 s while running | Forgets all paired phones |
| Hold the button while the radio starts (power-on or reset), for 10 s | Factory reset |

**Factory reset erases:**
- the radio ID, and makes a new one
- settings
- the ride
- paired phones
- stored events and history
- the counter ceiling, which restarts at 0
- the message and alert ID counters
- the event number

It is safe because a new radio ID gives new rider tags, so no nonce can repeat.

### 6.12 What is stored, and what survives a reboot (Q85)

| Record | Where | Survives reboot | Capacity and eviction | Erased by End Ride |
| --- | --- | --- | --- | --- |
| Radio ID | Flash | Yes | 1 | No (factory reset only) |
| Settings | Flash, 2 copies + CRC | Yes | 1 | No |
| Ride (key, name, tier, clash counter) | Flash, 2 copies + CRC | Yes | 1 | Yes |
| Frame counter ceiling | Flash, 2 copies + CRC | Yes | 1 | No |
| Message ID, alert ID, event number counters | Flash, written 16 ahead | Yes | 1 each | No |
| Active SOS state | Flash, 2 copies + CRC | Yes | 1 | Yes, after the "left the ride" replies |
| SOS events | Flash | Yes | All, until 24 h after the SOS ends | No |
| Paired phones | Flash | Yes | 4; a fifth replaces the least recently used | No |
| Own position history | RAM | No | 720 points (2 h at 10 s) | Yes |
| Heard position history | RAM | No | 4096 points shared; when full, points older than 30 minutes are thinned to one per 30 s, then the oldest are dropped | Yes |
| Other stored events | RAM | No | 200 | Yes |
| Known riders (names, radio IDs, anonymous IDs) | RAM | No; rebuilt from hellos and hello requests | 30 (+1 over-limit entry) | Yes |
| Pending fill asks | RAM | No | 8 | Yes |
| Duplicate memory, replay windows | RAM | No | 4.4 | No |
| Nearby table | RAM | No | 32 | No |
| Diagnostics counters | RAM | No | 7.4 | No |

## 7. Phone link

### 7.1 Schema rules (Q5, Q92)

- protobuf (Google's proto3 language specification), package `trailbuddy.v1`, written by the clean team from the tables below. The clean team hands the `.proto` file to the product owner for the app team.
- **Names:** message names in CamelCase from this spec (HELLO becomes `Hello`, SEND_TEXT becomes `SendText`); field names in snake_case.
- **One wrapper message,** `Envelope`, carries every message:
  - `request_id` (field 1)
  - `last` (field 2)
  - a `oneof body` with one field per message type, numbered from 10 in the order the messages appear in Section 7.4
- **Field numbering:** in table order, starting at 1. Numbers are never reused; removed fields are reserved.
- **Presence tracking:** every scalar field is marked `optional`, so the receiver can tell "missing" from 0. The receiver checks the R and O markings in 7.4 and answers a missing required field with BAD_FIELD.
- Every enum starts with an "unspecified" value of 0.
- Every text field and list is bounded by the limits given here.

### 7.2 Bluetooth service (Q93, Q94)

| Item | Value |
| --- | --- |
| Service UUID | `bc32efc7-02d9-48cb-b66a-9041aa8cfa23` |
| To radio (phone writes, with response) | `85d5f69e-313a-458e-9912-20397554da2f` |
| From radio (radio notifies) | `4dee99ec-354d-451c-a12f-1a0a75cfbe6d` |

**Framing:**
- Each message is a 2-byte little-endian length followed by the encoded `Envelope`, split across as many Bluetooth packets as needed.
- A length of 0 or over 1024 is answered with ERROR TOO_LARGE, and the receiver discards everything buffered.
- A message not completed within 2 s of its last byte is discarded.
- The phone waits for each write to be confirmed before sending the next.

**List sizes per part:**

| List | Entries per part |
| --- | --- |
| RIDERS_LIST | 10 |
| NEARBY_LIST | 16 |
| TRACK_POINTS | 64 points |
| GET_MISSED replays | One event per message |

Every part carries the request ID; the last part sets `last`.

**Connection order:**
1. Connect.
2. HELLO and WELCOME.
3. The phone starts encryption: it pairs if this is a new phone, or encrypts with the stored bond if not.
4. Every other message.

**Pairing and security:**
- Bluetooth LE Secure Connections with bonding, Just Works.
- A new phone can pair only while the pairing window is open (6.11). That requires someone at the radio, which is the confirmation, since screens are not used in version 1.
- A phone that has already paired can reconnect at any time.
- Up to 4 phones are remembered.
- One phone may be connected at a time. A second is refused while one is connected.
- Before encryption, everything except HELLO and WELCOME is refused with NOT_ALLOWED_NOW.

**Session:** there is no idle timeout. After a disconnect, the radio carries on relaying, sending positions and presence, repeating its SOS and storing events. A radio in repeater mode accepts phones too.

**On connect, after encryption, the radio sends in this order:**
1. any ERROR from 6.9
2. RIDE_STATE
3. SOS_STATUS, if its own SOS is active
4. SOS_RECEIVED for every active SOS from others
5. STATUS

### 7.3 Common rules and error codes (Q95)

- Every message carries a request ID (0 means unsolicited) and the `last` flag.
- Unknown fields are ignored.
- An unknown type, or a missing required field, is answered with ERROR.

| Code | Meaning |
| --- | --- |
| UNKNOWN_TYPE | Message type not recognised |
| BAD_FIELD | Field missing, empty when it mustn't be, out of range, or a name over 24 bytes |
| NOT_ALLOWED_NOW | Not allowed in the current state (before HELLO, before encryption, or I'm OK for someone else's alert) |
| VERSION_MISMATCH | Protocol version differs |
| NO_RIDE | Needs an active ride |
| RIDE_ACTIVE | A ride is already active; end it first |
| SOS_ACTIVE | Not allowed during the rider's own SOS |
| BUSY | Queue full; try later |
| TOO_LARGE | Text over 150 bytes, or a message over 1024 bytes |
| STORAGE_FAILED | The radio couldn't save something |
| SETTINGS_RESET | Stored settings were damaged and reset to defaults |
| RIDE_RESET | The stored ride was damaged and erased |
| SOS_RESET | A stored active SOS was damaged and could not be resumed |
| LOW_BATTERY | Battery too low for this action |
| UPDATE_REJECTED | Firmware image refused |
| NO_CONFIRMATION | Text not confirmed by anyone after 3 resends |

**When several apply,** the first in this order is returned: NOT_ALLOWED_NOW, VERSION_MISMATCH, UNKNOWN_TYPE, TOO_LARGE, BAD_FIELD, NO_RIDE, SOS_ACTIVE, RIDE_ACTIVE, LOW_BATTERY, BUSY, STORAGE_FAILED.

**Examples:**

| Case | Code |
| --- | --- |
| A 151-byte text | TOO_LARGE |
| An empty text | BAD_FIELD |
| A 25-byte name | BAD_FIELD |
| An update chunk at the wrong offset | BAD_FIELD, followed by UPDATE_PROGRESS giving the offset the radio expects |

### 7.4 Message list

R = required, O = optional. Types follow Section 2.6. "Rider ID" on the phone link is the 3-byte rider tag. The phone link has 44 messages.

#### Connect

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| HELLO | App → Radio | protocol version (u32, R); app version (text ≤16, R); phone time (time, O) | BLE-001 |
| WELCOME | Radio → App | protocol version (R); firmware version (three u32: major, minor, patch, R); radio ID (8 bytes, R); short ID (text 3, R); board model (enum: RAK19026_VC, RAK19007, POCKET_V3, T114_V2, THINKNODE_M1, R); last update result (enum: NONE, INSTALLED, ROLLED_BACK, O) | BLE-001, DFU-001 |
| GOODBYE | Either | reason (enum: USER, SHUTDOWN, UPDATING, VERSION_MISMATCH, R) | BLE-001 |

On a version mismatch, the radio replies with WELCOME, then GOODBYE with reason VERSION_MISMATCH, and disconnects.

#### Status

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_STATUS | App → Radio | none | BLE-003 |
| STATUS | Radio → App | battery (R); charging (bool, R); GPS (enum: NO_FIX, FIX, UNAVAILABLE, R); position source in use (enum: RADIO, PHONE, NONE, R); clock known (bool, R); channel busy (R); slow-down step (u8 0–3, R); ride contact (u8, R); events waiting (u32, R); presence sending (bool, R); storage failed (bool, R); firmware version (R) | BLE-003, MESH-004 |

- **Ride contact:** the number of ride members in contact (6.3), not counting this radio.
- **STATUS is sent:**
  - in reply to GET_STATUS
  - on a change of charging, GPS state, ride contact, slow-down step or storage failed
  - when battery changes by 5 points or more
  - at most once every 30 s, except a change of storage failed, which goes at once

#### Ride

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| JOIN_RIDE | App → Radio | ride name (text 1–24, R); ride key (16 bytes, R); tier (1–4, R) | RIDE-001, RIDE-002 |
| END_RIDE | App → Radio | none | RIDE-003 |
| GET_RIDE | App → Radio | none | BLE-002 |
| RIDE_STATE | Radio → App | ride active (bool, R); ride name (O); tier (O); key check value (4 bytes, O); own rider tag (O); over rider limit (bool, O); error code (O) | RIDE-001, RIDE-004, BLE-002 |

**JOIN_RIDE rules:**
- Refused with RIDE_ACTIVE if a ride is active.
- Refused with SOS_ACTIVE during the rider's own SOS.
- Refused with BAD_FIELD for an all-zero key, a wrong key length or a tier outside 1–4.
- The ride is saved in one all-or-nothing write and survives reboot.
- RIDE_STATE replies to JOIN_RIDE, END_RIDE and GET_RIDE, reading back what was stored. The key itself is never sent back; the check value proves it.

#### Messaging

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SEND_TEXT | App → Radio | text (1–150 bytes, R) | MSG-001 |
| SEND_RESULT | Radio → App | message ID (u16, R); state (enum: QUEUED, SENT, DELIVERED, FAILED, R); confirmations (u8, O); in contact when sent (u8, O); error code (O) | MSG-002 |
| TEXT_RECEIVED | Radio → App | event number (u32, R); message ID (u16, R); sender rider ID (R); text (R); time received (time, R); RSSI and SNR (R) | MSG-001 |
| GET_MISSED | App → Radio | after event number (u32, R) | MSG-001 |
| EVENTS_INFO | Radio → App | oldest event number held (u32, R); newest event number (u32, R) | MSG-001 |

#### Location

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SHARE_MY_LOCATION | App → Radio | latitude, longitude (R); accuracy (R); fix time (R); heading (O); speed (O) | POS-001, POS-002 |
| LOCATION_UPDATE | Radio → App | rider ID (R); latitude, longitude (R); accuracy (R); fix time (R); source (R); hops travelled (u8, R) | POS-001, POS-002 |

**LOCATION_UPDATE rules (Q91):**
- Each new authenticated position from a ride member is reported once.
- A position older, by fix time, than one already held for that rider is stored in history but not reported.
- **Hops travelled:** the kind's starting hops minus hops remaining at receipt.
- **Own position:** the radio also reports the position it sends, with its own rider tag, when the source is radio GPS.

#### Fill on Return

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_TRACK | App → Radio | rider ID (R); from time (R); to time (O, default now) | FILL-001 |
| TRACK_POINTS | Radio → App | rider ID (R); points (list of latitude, longitude, fix time; at most 64 per part, sorted by fix time, R); filled (bool, R); outline (bool, R) | FILL-001 |

#### SOS

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| SEND_SOS | App → Radio | latitude, longitude (O); accuracy (O); fix time (O). If absent, the radio uses 6.4. | SOS-001, SOS-002 |
| SOS_STATUS | Radio → App | alert ID (R); state (enum: ACTIVE, ENDED_OK, ENDED_LEFT_RIDE, R); times sent (u16, R); last sent (time, R); position sent (latitude, longitude, fix time, or no-position flag, R); copy-that riders (list of rider IDs, R) | SOS-003 |
| SOS_RECEIVED | Radio → App | event number (R); alert ID (R); rider ID (R); latitude, longitude, accuracy, fix time, or no-position flag (R); first sent (time, R); repeat number (R) | SOS-001 |
| SOS_REPLY | App → Radio | alert ID (R); alert owner rider ID (R); reply (enum: COPY_THAT, IM_OK, R) | SOS-004 |
| SOS_REPLY_RECEIVED | Radio → App | event number (R); alert ID (R); alert owner rider ID (R); from rider ID (R); reply (R) | SOS-004 |
| SOS_ENDED | Radio → App | event number (R); alert ID (R); alert owner rider ID (R); reason (enum: IM_OK, LEFT_RIDE, R) | SOS-003, SOS-004 |
| GET_SOS | App → Radio | none | SOS-003 |

#### Riders

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_RIDERS | App → Radio | none | MSG-003 |
| RIDERS_LIST | Radio → App | list (at most 30, 10 per part) of: rider ID; radio ID (O until a hello is heard); short ID (O); display name (O); repeater (bool); last heard (seconds ago, u32); RSSI and SNR (O: direct receptions only); in contact (bool) | MSG-003 |
| RIDER_UPDATE | Radio → App | event number (R) plus the fields of one RIDERS_LIST entry | MSG-003 |

RIDER_UPDATE is sent at most once per 10 s per rider, when:
- a rider is first heard
- a name changes
- a rider's tag changes (the entry moves to the new tag)
- contact is lost or regained (6.3)

Riders are never removed during a ride.

#### Nearby

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| NEARBY_RIDER | Radio → App | anonymous ID (R); latitude, longitude (R); heading (R); speed (R); fix age at receipt (R); received (time, R); RSSI and SNR (R) | NEAR-004 |
| GET_NEARBY | App → Radio | none | NEAR-004 |
| NEARBY_LIST | Radio → App | list (at most 32, 16 per part) of NEARBY_RIDER entries not yet expired | NEAR-004 |

#### Settings

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| GET_SETTINGS | App → Radio | none | BLE-002 |
| SETTINGS | Radio → App | all settings, as stored | BLE-002 |
| CHANGE_SETTINGS | App → Radio | list of setting ID and new value | BLE-002 |
| SETTINGS_RESULT | Radio → App | OK or error code (R); all settings as now stored (R) | BLE-002 |

| ID | Setting | Type | Default |
| --- | --- | --- | --- |
| 1 | Display name | text 1–24 | "Rider " + short ID |
| 2 | Presence sending | bool | on |
| 3 | Repeater mode | bool | off |
| 4 | Removed in v0.4 (was GPS source preference). Never reused. | — | — |

#### Updates

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| START_UPDATE | App → Radio | new version (three u32, R); board model (R); image size (u32, R); image hash (32 bytes SHA-256 of the whole image, R); signature (64 bytes Ed25519 over the image hash, R) | DFU-001 |
| UPDATE_CHUNK | App → Radio | offset (u32, R); data (≤200 bytes, R) | DFU-001 |
| UPDATE_PROGRESS | Radio → App | bytes received (u32, R); state (enum: RECEIVING, VERIFYING, INSTALLING, REJECTED, ABORTED, R); error code (O) | DFU-001 |
| ABORT_UPDATE | App → Radio | none | DFU-001 |

**Update rules (Q97):**
- **Signing:** Ed25519 (RFC 8032), over the SHA-256 of the image, carried in START_UPDATE. The bootloader checks it before installing.
- **Refused with UPDATE_REJECTED:**
  - a wrong board model
  - a bad signature or hash
  - a version equal to or older than the installed one
  - an image larger than the update slot
- **Refused with LOW_BATTERY:** battery under 40%.
- **Refused with SOS_ACTIVE:** during the rider's own SOS.
- **Refused with NOT_ALLOWED_NOW:** START_UPDATE while an update is already in progress.
- A chunk at the wrong offset gets BAD_FIELD and an UPDATE_PROGRESS with the expected offset.
- With no chunk for 60 s, the update is aborted (state ABORTED).
- **While receiving,** the radio keeps relaying and repeating any SOS. SEND_SOS during an update aborts the update and starts the SOS.
- **After VERIFYING,** the radio reboots into the new image. It is offline for at most 90 s.
- **Confirming:** the new firmware confirms itself only after its radio, Bluetooth and flash checks pass, within 60 s of starting. Otherwise the bootloader rolls back.
- The result reaches the app in the next WELCOME.
- **USB recovery:** the UF2 bootloader is the recovery route over USB. It doesn't check signatures. It is accepted as outside DFU-001, for recovery by the product owner with the radio in hand. The clean team records the flash layout of both bootloaders in DECISIONS.md.
- The signing key is held by the product owner, never in the repository.

#### Repeater, errors and diagnostics

| Message | Direction | Fields | Req |
| --- | --- | --- | --- |
| REPEATER_STATUS | Radio → App | frames relayed since boot (u32); uptime (seconds, u32); battery | RPT-001 |
| ERROR | Radio → App | error code (R) | BLE-001 |
| GET_DIAGNOSTICS | App → Radio | none | DIAG-001 |
| DIAGNOSTICS | Radio → App | see below | DIAG-001 |
| CLEAR_DIAGNOSTICS | App → Radio | none; the reply is DIAGNOSTICS with every counter at zero | DIAG-001 |

**DIAGNOSTICS counters (Q96)**, all u32, wrapping to 0. They start at zero at boot or on clear; End Ride doesn't clear them.
- **For each of the 9 ride kinds and presence:**
  - **sent:** own frames transmitted
  - **received:** frames passing the LoRa CRC, before any other check
  - **relayed:** relays actually transmitted
  - **suppressed:** relays cancelled by suppression
  - **gave up:** frames dropped after 8 busy checks
- **Drops by reason,** each frame counted once, under the first check it fails in 4.8: unknown version, malformed, wrong ride, failed authentication, duplicate. Plus queue full (4.5).
- **Channel busy history:** the last 60 complete minutes, oldest first. The minute in progress isn't included.

## 8. Ride QR code (RIDE-001, Q98)

Text form: `trailbuddy://ride/` followed by the bytes below in Base64url (RFC 4648 §5): no padding, only the characters A–Z, a–z, 0–9, "-" and "_", with letter case significant.

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | Format version (1) |
| 1 | 1 | Tier (1–4) |
| 2 | 16 | Ride key |
| 18 | 1 | Ride name length (1–24) |
| 19 | n | Ride name (UTF-8) |
| 19 + n | 2 | CRC, low byte first |

**The CRC** is CRC-16 with polynomial 0x1021, start value 0xFFFF, no bit reflection and no final XOR. This is known as CRC-16/CCITT-FALSE; the parameters here are complete, so no other document is needed. It covers every byte before it.

The maximum is 45 bytes, or 60 characters. The app refuses a code with:
- a wrong format version
- a bad CRC
- a wrong length for its name
- bytes after the CRC
- any character outside the set above

## 9. Hardware

**Version 1 boards (Decision).** All use a Nordic nRF52840 and a Semtech SX1262, and one firmware serves them all, with a board profile for each.

| Board | Role | Starting point | Profile source in `docs/hardware/` |
| --- | --- | --- | --- |
| RAKwireless RAK19026 VC base board with RAK4631 core | Lead board: every build step runs here first | Zephyr's RAK4631 board definition | RAK19026 VC datasheet; RAK4631 documentation |
| Heltec Mesh Node T114 V2 | Version 1 board | Zephyr's Heltec T114 V2 board definition | Heltec T114 V2 datasheet and schematic |
| Elecrow ThinkNode M1 | Version 1 board | No Zephyr definition; the clean team writes one | Elecrow ThinkNode M1 datasheet and schematic |

Zephyr board definitions are part of the open-source Zephyr project, not another mesh project's board files. Manufacturer documents go into `docs/hardware/` only as copies cleaned by the product owner, because some mention other firmware.

**What version 1 uses on each board.** The clean team confirms every item, pin and polarity from the manufacturer documents and records it in the board profile. If a document doesn't confirm an item, it goes to OPEN-QUESTIONS.md.

| Role | RAK19026 VC | T114 V2 | ThinkNode M1 |
| --- | --- | --- | --- |
| User button (6.11) | SW1 | User button | Function button |
| GNSS | Integrated | GPS module, fitted on every Trail Buddy T114 | Integrated, with a GPS on/off switch |
| Status light (9) | LED 1 | Green LED | Blue status LED |
| Not used in version 1 | LED 2 | RGB LEDs, display | Buzzer, e-paper display and front light, page button |

- **GNSS switched off or failed:** if the ThinkNode M1's GPS switch is off, or any board's GNSS doesn't respond, STATUS reports GPS UNAVAILABLE and the radio uses phone positions (6.1, 6.6). It never fails because GNSS is missing.
- **Power and reset controls** (reset buttons, the ThinkNode M1's power switch, the RAK19026's battery switch) keep their hardware function. "Starting" in 6.11 means power-on or reset by whatever means the board provides; each profile records how.
- **Battery measurement** comes from each board's documented method and is recorded in its profile.
- **Status light (Decision):** each board's status light shows only that the radio is powered and which mode it is in:

  | State | Status light |
  | --- | --- |
  | Firmware starting (power-on or reboot) | On steadily for 2 s |
  | Running normally | One short blink (50 ms) every 5 s, so the rider can see the radio is on |
  | Rebooting to install an update (7.4, Updates) | Fast blink (5 times a second) until the new firmware starts |
  | Factory reset under way (6.11) | On steadily until the reset finishes |
  | Off, or battery flat | Off |

  The UF2 recovery bootloader drives the light in its own way; the firmware doesn't change that. Lights wired to the charger in hardware keep their own function.
- **Buzzers and screens** are not used in version 1. The firmware doesn't drive them.
- **The WisMesh Pocket V3** gets its own profile once the board and its documentation are available.

## 10. Diagnostics

- **DIAG-001:** the radio keeps the counters in 7.4 in release firmware, for field testing.
- Detailed logging is removed from release builds.

## 11. Test vectors and sample values (Q56, Q73)

These vectors are exact. Implementations must reproduce them byte for byte.

### 11.1 Key derivation and IDs

| Input or output | Hex |
| --- | --- |
| Ride key | `000102030405060708090a0b0c0d0e0f` |
| Radio ID | `1122334455667788` |
| Encryption key | `91623a7d358eb9fae7a1497546128f3c` |
| Ride tag | `20bb` |
| Rider tag, clash counter 0 | `eea021` |
| Rider tag, clash counter 1 | `c4b67b` |
| Key check value | `715305b1` |
| Short ID | `CER` |

### 11.2 Ride frames

All use the key and radio ID above, clash counter 0 and whole-ride destination. The position is latitude 35.349120°, longitude −120.247385°, accuracy 6 m, fix time 25,000,000.

| Frame | Inputs | Complete frame (hex) |
| --- | --- | --- |
| Position | counter 42, hops 3, class 2, source radio GPS | `113220bbeea0212a0000000000005291589d7f1a84be634834d414601e75c39eeb47854de9` |
| Same position, relayed once | identical except hops 2 | `112220bbeea0212a0000000000005291589d7f1a84be634834d414601e75c39eeb47854de9` |
| Hello | counter 43, hops 3, class 4; radio ID above, anonymous ID a1b2c3d4, flags 0, name "Dave" | `113420bbeea0212b000000000000b3b53fc6ae74758e35bf9422248e69dd174810377376ffc1b44f5e` |
| Text | counter 44, hops 3, class 5; message ID 7, "Waiting at the fork" | `113520bbeea0212c0000000000007a9931d68a3b6f70bf038b212ea3d5bb5c744705502f0634349e8d1af30b4b` |
| SOS | counter 45, hops 7, class 0; alert ID 3, the position above, first sent 25,000,100, repeat 0 | `117020bbeea0212d0000000000008687dfca43391ec98197a76c5e14e68462a38293c78ef0190906514923` |
| SOS, no position | counter 46, hops 7, class 0; alert ID 4, latitude and longitude 0x7FFFFFFF, accuracy 255, fix time 0, first sent 25,000,200, repeat 0 | `117020bbeea0212e0000000000005fc416a586644488eac3bf9733c66624232a4f64c961442fa05699ae15` |
| Fill ask | counter 47, hops 3, class 6; 1 rider: tag 5a6b7c, from 24,998,200 to 25,000,000 | `113620bbeea0212f000000000000628e483fe1ad751e442551d064613d546b824347a0` |
| Fill answer | counter 48, hops 3, class 6; own tag eea021, range 24,998,200 to 25,000,000, outline part 0 of 1, 3 points: the position above plus i × 50 to latitude and minus i × 50 to longitude, fix time 24,998,200 + i × 60, for i = 0, 1, 2 | `113620bbeea02130000000000000779c985079ba5e343250748b00320ba52779e5d68f2f1a32f0a0688849781cf0d2fed88cc63060e840f38d8e6ca03b946dd5401728c1e1d6b58918` |

### 11.3 Presence frame

Inputs:
- anonymous ID a1b2c3d4
- the position above, rounded to 35.3491° and −120.2474°
- heading 90° (value 60)
- speed 32 km/h
- fix age 2 s

Frame: `12a1b2c3d46c621b02982bd5f83c2002`

### 11.4 Ride QR code

Inputs: tier 1, the ride key above, name "Sat Dez Loop". The CRC is 0x0B6B, sent low byte first as `6b0b`.

| Item | Value |
| --- | --- |
| Bytes (33) | `0101000102030405060708090a0b0c0d0e0f0c5361742044657a204c6f6f706b0b` |
| Text | `trailbuddy://ride/AQEAAQIDBAUGBwgJCgsMDQ4PDFNhdCBEZXogTG9vcGsL` |

### 11.5 Phone-link samples (Q55)

The clean team encodes each sample once with the generated code and saves the input and its bytes together in `spec/vectors/`. Those saved bytes then become the reference for both the firmware and app tests. The spec team checks the decoded values against these samples.

| Message | Sample |
| --- | --- |
| HELLO | protocol version 1; app version "1.0.0"; phone time 25,000,000 |
| WELCOME | version 1; firmware 1.0.0; radio ID 1122334455667788; short ID "CER"; RAK19026_VC; last update NONE |
| GOODBYE | USER |
| GET_STATUS | (no fields) |
| STATUS | battery 84; not charging; FIX; RADIO; clock known; busy 14; step 0; contact 3; events waiting 2; presence on; storage OK; firmware 1.0.0 |
| JOIN_RIDE | "Sat Dez Loop"; key as 11.1; tier 1 |
| END_RIDE | (no fields) |
| GET_RIDE | (no fields) |
| RIDE_STATE | active; "Sat Dez Loop"; tier 1; check value 715305b1; rider tag eea021; not over limit |
| SEND_TEXT | "Waiting at the fork" |
| SEND_RESULT | message ID 7; DELIVERED; 3 confirmations; 3 in contact when sent |
| TEXT_RECEIVED | event 101; message ID 7; sender c4b67b; "Waiting at the fork"; time 25,000,050; RSSI −92; SNR 26 (6.5 dB) |
| GET_MISSED | after event 100 |
| EVENTS_INFO | oldest 1; newest 105 |
| SHARE_MY_LOCATION | 35.349120°, −120.247385°; accuracy 6 m; fix time 25,000,000; heading 90°; speed 32 km/h |
| LOCATION_UPDATE | rider c4b67b; 35.349170°, −120.247435°; accuracy 8 m; fix time 25,000,010; RADIO; hops travelled 1 |
| GET_TRACK | rider c4b67b; from 24,998,200; to 25,000,000 |
| TRACK_POINTS | rider eea021; the three points of the fill answer in 11.2; filled; outline |
| SEND_SOS | position as SHARE_MY_LOCATION |
| SOS_STATUS | alert 3; ACTIVE; times sent 2; last sent 25,000,110; position as above; copy-that riders [c4b67b] |
| SOS_RECEIVED | event 102; alert 3; rider eea021; position as above; first sent 25,000,100; repeat 1 |
| SOS_REPLY | alert ID 3; owner eea021; COPY_THAT |
| SOS_REPLY_RECEIVED | event 103; alert 3; owner eea021; from c4b67b; COPY_THAT |
| SOS_ENDED | event 104; alert 3; owner eea021; IM_OK |
| GET_SOS | (no fields) |
| GET_RIDERS | (no fields) |
| RIDERS_LIST | one entry: c4b67b; radio ID 8877665544332211; short ID from it; "Dave"; not repeater; last heard 4 s; RSSI −92; SNR 26; in contact; last part |
| RIDER_UPDATE | event 105 plus the entry above |
| NEARBY_RIDER | anonymous ID a1b2c3d4; 35.3491°, −120.2474°; heading 90°; speed 32 km/h; fix age 2 s; received 25,000,020; RSSI −101; SNR 8 (2 dB) |
| GET_NEARBY | (no fields) |
| NEARBY_LIST | one entry as NEARBY_RIDER; last part |
| GET_SETTINGS | (no fields) |
| SETTINGS | 1 = "Dave"; 2 = on; 3 = off |
| CHANGE_SETTINGS | 1 = "Dave"; 2 = on |
| SETTINGS_RESULT | OK; 1 = "Dave"; 2 = on; 3 = off |
| START_UPDATE | version 1.0.1; RAK19026_VC; size 262,144; hash 32 bytes of 0x11; signature 64 bytes of 0x22 (encoding test only, not a valid signature) |
| UPDATE_CHUNK | offset 0; 200 bytes of 0x33 |
| UPDATE_PROGRESS | 200 bytes received; RECEIVING |
| ABORT_UPDATE | (no fields) |
| REPEATER_STATUS | relayed 1,234; uptime 3,600 s; battery 77 |
| ERROR | request ID 12; BAD_FIELD |
| GET_DIAGNOSTICS | (no fields) |
| DIAGNOSTICS | every counter 0 except position sent 5, received 12, relayed 3; busy history 60 × 14 |
| CLEAR_DIAGNOSTICS | (no fields) |

### 11.6 Review

The independent security review of Sections 4.3 to 4.8 and 7.2 must be completed before freeze. The product owner arranges the reviewer.

## 12. Requirement and policy changes in this version

Issued in REQUIREMENTS v0.4 and POLICY v0.4. Each has its own change table.

- **REQUIREMENTS:**
  - NEAR-002 and NEAR-004 reworded to match 6.6 (Policy).
  - SOS-001: an SOS needs a ride and is never held back for lack of a fix.
  - SOS-002: an SOS and an "I'm OK" are never dropped; a "Copy that" may be.
  - SOS-003: an active SOS survives a reboot.
  - FILL-001 and NFR-001: the filled target, battery conditions and range target are defined.
  - RIDE-004: repeaters count towards 30.
  - REG-001: version 1 is for the United States only.
- **POLICY:**
  - the "which document wins" rule (Section 0)
  - §8: version 1 boards (RAK19026 VC lead, T114 V2, ThinkNode M1), button and status light; no buzzer or screen
  - §10 and §18: channel check and airtime budget
  - §12: decided for building, review can change; Ed25519 named
  - §14 and §19: approved standards and `docs/standards/`

## 13. Approved standards (Q63)

The spec team approves these as sources. The product owner adds the official documents to `docs/standards/` and logs each in SOURCE-REGISTER.md; sessions never fetch them from the web.

| Standard | Defining document |
| --- | --- |
| HKDF | RFC 5869 |
| SHA-256 | FIPS 180-4 |
| AES | FIPS 197 |
| CCM mode | NIST SP 800-38C and RFC 3610 |
| Ed25519 | RFC 8032 |
| Base64url, Base32 alphabet | RFC 4648 (§5 and §6) |
| UTF-8 | RFC 3629 |
| Protocol Buffers | Google's proto3 language specification |
| UTC | ITU-R TF.460 |
| GPS time | IS-GPS-200 |
| QR code | ISO/IEC 18004. For the app team only; the firmware never makes or reads QR codes. |
| CRC-16/CCITT-FALSE | No document; fully defined in Section 8 |
| Bluetooth LE, 128-bit UUIDs | Bluetooth Core Specification (already allowed) |
| FCC rules | 47 CFR Part 15, §15.247 (already allowed) |
| UF2 | Microsoft's published UF2 format, with the board's UF2 bootloader documentation |

## 14. Still open

- [ ] Compliance lab: confirm the settings and regulatory route in Section 2.1.
- [ ] Mesh simulator: confirm the airtime figures in 2.5 and the slow-down steps in 6.10.
- [ ] Security review: Sections 4.3 to 4.8 and 7.2, including the documented limits:
  - replay records lost at reboot
  - ride members able to forge I'm OK
  - rider tags trackable within a ride
  - Just Works pairing protected only by the pairing window
- [ ] Product owner: confirm, before build step 2, that the chosen crypto library supports AES-128-CCM, HKDF-SHA256 and Ed25519.
- [ ] Product owner: add the standards in Section 13 to `docs/standards/`.
- [ ] Product owner: add cleaned manufacturer documents for the T114 V2 and the ThinkNode M1 to `docs/hardware/`.
- [ ] Product owner: the WisMesh Pocket V3 documents, when the board ships.
- [ ] "Approaching" definition: deferred by the product owner.

## 15. Where each question is answered

| Questions | Section |
| --- | --- |
| Q2 | 3 |
| Q5, Q92 | 7.1 |
| Q7, Q71, Q72, Q73 | 2.3, 4.4, 6.11 |
| Q15, Q59, Q69 | 4.5 |
| Q25 | 6.3, 6.10, 7.4 Status |
| Q38, Q58, Q84 | 6.5 |
| Q39, Q81, Q82 | 6.4 |
| Q47 | 6.6, 7.2 |
| Q54 | 9 |
| Q55 | 11.5 |
| Q56 | 11.1 to 11.4 |
| Q57, Q77 | 6.1 |
| Q60 | 6.3 |
| Q61 | 3, 7.4 Settings |
| Q62 | 0, 2.6, 12, and REQUIREMENTS and POLICY v0.4 |
| Q63 | 13 |
| Q64 | 4.7 |
| Q65, Q66 | 6.10, 2.5 |
| Q67 | 4.8 |
| Q68 | 4.6 |
| Q70 | 4.4 |
| Q74 | 5, 6.3 |
| Q75 | 2.2 |
| Q76, Q78 | 6.6 |
| Q79, Q80, Q83 | 3, 6.4 |
| Q85 | 6.12 |
| Q86 | 6.3 |
| Q87, Q88 | 6.9 |
| Q89 | 6.8 |
| Q90 | 6.4, 9 |
| Q91 | 7.4 Location |
| Q93, Q94 | 7.2, 6.11 |
| Q95 | 7.3 |
| Q96 | 7.4 Diagnostics |
| Q97 | 7.4 Updates |
| Q98 | 8 |

## 16. Changes from v0.3

- **Precedence:** none of the three documents wins; a conflict stops work until answered (Section 0).
- **United States only.** Version 1 boards are the RAK19026 VC base board with RAK4631 core (lead), the Heltec T114 V2 and the Elecrow ThinkNode M1. The user button is mapped per board (9); the status light shows power and mode only; no buzzer or screen. SOS alerts are shown by the app only.
- **SOS:**
  - needs a ride
  - never held back, with a no-position marker
  - survives reboot
  - "left the ride" reply sent twice
  - class 0 queue of 64, never dropping an SOS or I'm OK
  - exact schedule
  - no light or buzzer alert on the radio; the app alerts the rider
- **Fill on Return:**
  - "filled" defined
  - multi-rider asks
  - answers carry range, part number and total
  - faster rate on a quiet channel
- **New sections:**
  - listen before talking (4.7)
  - receive checks (4.8)
  - channel-busy measurement and slow-down steps (6.10)
  - factory reset and button actions (6.11)
  - storage table (6.12)
  - error-code precedence (7.3)
- **Tiers:** tiers 3 and 4 send presence every 20 s while moving and limit Delivered to 2 hops. The airtime table now includes hellos, texts and fill.
- **Clock,** short ID, rider tag reserved value and clash merging, message and alert IDs, duplicate memory, relay delay and suppression are all defined exactly.
- **Phone link:**
  - wrapper message and numbering rule
  - presence tracking
  - framing errors and part sizes
  - pairing window
  - EVENTS_INFO added (44 messages)
  - GPS source setting removed
  - new codes RIDE_RESET and SOS_RESET
- **Updates:** signature carried in START_UPDATE; refusal, timeout and confirmation rules; USB recovery documented.
- **QR code:** CRC low byte first, parameters stated in full, strict text rules.
- **Test vectors:** added for hello, text, SOS, SOS without a position, a relayed frame, fill ask and answer, rider tag clash 1 and the short ID. Samples for all 44 phone-link messages.
