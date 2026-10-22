# TrailBuddy Radio Message Spec (Draft v0.2)

Oct 22, 2026 · David · Supersedes Draft v0.1

## How it works

Trail Buddy uses its own radio protocol end to end. The app talks to its own radio over Bluetooth; the radios talk to each other over LoRa. Both conversations are defined here, and both are Trail Buddy's own design.

```
TrailBuddy app  <-- phone link (Bluetooth) -->  TrailBuddy radio  <-- over the air (LoRa) -->  other TrailBuddy radios
(on the phone)                                  (Trail Buddy firmware)
```

- **Hardware under test:** Heltec Mesh Node T114 V2, Elecrow ThinkNode M1, RAKwireless RAK4631 (WisBlock) and RAKwireless WisMesh Pocket V3. All four use a Nordic nRF52840 microcontroller (Bluetooth 5.0) and a Semtech SX1262 LoRa radio. The Pocket V3 is the IP67 waterproof version of the Pocket V2, on the RAK19026 baseboard, with GPS and a notification buzzer. One firmware serves all boards, with a separate board profile for each (pins, GNSS, display, buttons, battery measurement, buzzer). Version 1 is built and tested on one lead board first; the others are added once the core tests pass on it.
- **Board profiles come only from the manufacturer:** datasheets, schematics and the manufacturer's own board support. Never from any other mesh project's board files.
- **Phone link:** Bluetooth Low Energy, one Trail Buddy service on the radio.
- **Every phone-link message carries:** its type (the names below), a request ID so each reply matches its request, and the protocol version.
- **Encoding:** our own protobuf schema, written from scratch by the clean team from this spec (see "How to Create Trail Buddy's Protocol Buffers"). Field names come from this document.
- **Direction:** App → Radio, Radio → App, or Either.
- **Requirement IDs** refer to Section 9 of the Clean-Room Firmware Development Specification, plus the new NEAR requirements below.

**Clean-room note:** written from Trail Buddy's product requirements only. No other project's message files, names, layouts or values were used. Log this version in docs/PROVENANCE.md.

## New requirements: Nearby riders

Goal: help riders avoid trail collisions by seeing any Trail Buddy rider nearby, not just their own ride, and warning when one is approaching.

| ID | Requirement |
| --- | --- |
| NEAR-001 | Every Trail Buddy radio sends a short presence signal that any Trail Buddy radio can hear, whether or not they share a ride. |
| NEAR-002 | The presence signal carries position, heading, speed and GPS fix time, plus an anonymous rider ID that changes at least daily. It never carries names, messages or ride details. |
| NEAR-003 | Presence is sent often while moving and less often when stopped. The intervals are set by the spec team against the airtime budget. |
| NEAR-004 | The radio passes every presence signal it hears to the app. |
| NEAR-005 | The app warns the rider when another rider is approaching, using sound and vibration as well as the screen. Each rider's app decides this independently, so both riders are warned. |
| NEAR-006 | The rider can turn presence off, for example when parked. Whether it is on by default is a product decision (see open questions). |
| NEAR-007 | Presence never delays an SOS. |
| NEAR-008 | An empty "Nearby" view must never be presented as a clear trail. The app states that it shows Trail Buddy riders only. |

## Phone link message list

Version 1 has 40 messages in 14 areas. Every reply carries the request ID of the message it answers.

| Area | Message | Direction | What it's for | Key fields | Req |
| --- | --- | --- | --- | --- | --- |
| Connect | HELLO | App → Radio | Starts a session after Bluetooth connects | app version, protocol version | BLE-001 |
| Connect | WELCOME | Radio → App | Confirms the session; a protocol version mismatch is refused with a clear reason | firmware version, protocol version, radio ID, board model (T114 V2, ThinkNode M1, RAK4631, WisMesh Pocket V3) | BLE-001 |
| Connect | GOODBYE | Either | Ends the session cleanly | reason | BLE-001 |
| Status | GET_STATUS | App → Radio | Asks for current device health | none | BLE-003 |
| Status | STATUS | Radio → App | Device health, on request or when it changes | battery %, charging yes/no, GPS fix yes/no, channel busy (% of airtime in use), messages waiting, firmware version | BLE-003, MESH-004 |
| Ride | JOIN_RIDE | App → Radio | Loads a ride from its QR code onto the radio | ride name, ride key, group tier | RIDE-001, RIDE-002 |
| Ride | END_RIDE | App → Radio | Ends the ride and returns the radio to its default state | none | RIDE-003 |
| Ride | RIDE_RESULT | Radio → App | Confirms a ride change by reading back what the radio actually stored | request ID, OK or error, stored ride name, stored tier, ride active yes/no | RIDE-001, BLE-002 |
| Messaging | SEND_TEXT | App → Radio | Sends a text to everyone in the ride or to one rider | request ID, recipient (ride or rider ID), text | MSG-001 |
| Messaging | SEND_RESULT | Radio → App | Tracks a sent message | request ID, state (queued, sent, delivered, failed), failure reason | MSG-002 |
| Messaging | TEXT_RECEIVED | Radio → App | A message arrived | message ID, sender ID, recipient, text, time received, signal strength | MSG-001 |
| Messaging | GET_MISSED | App → Radio | Fetches messages stored while the phone was disconnected | since (time or last message ID) | MSG-001 |
| Location | SHARE_MY_LOCATION | App → Radio | Sends the rider's position to the ride | latitude, longitude, accuracy (m), GPS fix time, source (phone or radio GPS) | POS-001, POS-002 |
| Location | LOCATION_UPDATE | Radio → App | A ride member's position arrived; the app judges freshness by the GPS fix time, never by arrival time | rider ID, latitude, longitude, accuracy (m), GPS fix time | POS-001, POS-002 |
| Fill on Return | GET_TRACK | App → Radio | Asks for a rider's stored positions, for example after the phone was away | rider ID, since (time) | FILL-001 |
| Fill on Return | TRACK_POINTS | Radio → App | Positions filled in after a gap, sent in batches | rider ID, list of (latitude, longitude, GPS fix time), more to come yes/no | FILL-001 |
| SOS | SEND_SOS | App → Radio | Raises an SOS for this rider | latitude, longitude, accuracy (m), GPS fix time, time sent | SOS-001, SOS-002 |
| SOS | SOS_STATUS | Radio → App | Tracks this rider's own SOS | alert ID, state (sending, repeating, cancelled), times sent, last sent, replies heard | SOS-003 |
| SOS | SOS_RECEIVED | Radio → App | Another rider raised an SOS, or repeated one | alert ID, rider ID, latitude, longitude, accuracy (m), GPS fix time, time sent | SOS-001 |
| SOS | SOS_REPLY | App → Radio | Answers an SOS: "Copy that" from others, or "I'm OK" from the rider who raised it | alert ID, kind (copy that, I'm OK) | SOS-004 |
| SOS | SOS_REPLY_RECEIVED | Radio → App | A reply to an SOS arrived | alert ID, rider ID, kind | SOS-004 |
| SOS | SOS_ENDED | Radio → App | An SOS is over (I'm OK received, or the ride ended) | alert ID, reason | SOS-003, SOS-004 |
| Riders | GET_RIDERS | App → Radio | Asks for the ride members the radio has heard | none | MSG-003 |
| Riders | RIDERS_LIST | Radio → App | Reply to GET_RIDERS | per rider: ID, display name, last heard, signal strength, battery % if shared | MSG-003 |
| Riders | RIDER_UPDATE | Radio → App | One ride member appeared or changed | same fields as one RIDERS_LIST entry | MSG-003 |
| Nearby | NEARBY_RIDER | Radio → App | A presence signal was heard from any Trail Buddy rider | anonymous rider ID, latitude, longitude, heading, speed, GPS fix time, signal strength | NEAR-004 |
| Nearby | GET_NEARBY | App → Radio | Asks for every presence heard recently, for example after reconnecting | since (time) | NEAR-004 |
| Nearby | NEARBY_LIST | Radio → App | Reply to GET_NEARBY | list of NEARBY_RIDER entries | NEAR-004 |
| Settings | GET_SETTINGS | App → Radio | Reads device settings | none | BLE-002 |
| Settings | SETTINGS | Radio → App | Current settings | display name, presence on/off, repeater mode on/off, GPS source | BLE-002, NEAR-006, RPT-001 |
| Settings | CHANGE_SETTINGS | App → Radio | Changes one or more settings, applied by the radio as one all-or-nothing write; a value the radio already holds is not rewritten | request ID, setting names and new values | BLE-002 |
| Settings | SETTINGS_RESULT | Radio → App | Confirms a settings change by reading back the values actually stored | request ID, OK or error, every changed setting with its stored value | BLE-002 |
| Updates | START_UPDATE | App → Radio | Begins a firmware update | new version, image size, signature | DFU-001 |
| Updates | UPDATE_CHUNK | App → Radio | One piece of the update | offset, data | DFU-001 |
| Updates | UPDATE_PROGRESS | Radio → App | Reports progress and outcome | bytes received, state (receiving, verifying, installed, rolled back, failed), reason | DFU-001 |
| Repeater | REPEATER_STATUS | Radio → App | Repeater activity, when repeater mode is on | messages relayed, uptime, battery % | RPT-001 |
| Errors | ERROR | Radio → App | Something the radio couldn't do | request ID, error code, short description | BLE-001 |
| Diagnostics | GET_DIAGNOSTICS | App → Radio | Asks for recent radio activity, for field testing | since (time) | BLE-003 |
| Diagnostics | DIAGNOSTICS | Radio → App | Counts of what was actually sent, received, relayed and dropped | per message kind: sent, received, relayed, dropped; channel busy history | BLE-003 |
| Diagnostics | CLEAR_DIAGNOSTICS | App → Radio | Resets the counters | none | BLE-003 |

## Over-the-air messages (radio to radio)

These are what radios send each other. The spec team defines their contents and limits; the clean team designs the layout from scratch.

| Kind | Who can read it | What it carries | Req |
| --- | --- | --- | --- |
| Ride position | Ride members only (ride key) | rider ID, position, accuracy, GPS fix time | POS-001, POS-002 |
| Rider hello | Ride members only | rider ID, display name | MSG-003 |
| Text | Ride members only | message ID, recipient, text | MSG-001 |
| Delivered | Ride members only | message ID being confirmed | MSG-002 |
| SOS | Ride members only | alert ID, rider ID, position, GPS fix time, time sent | SOS-001 to SOS-003 |
| SOS reply | Ride members only | alert ID, kind (copy that, I'm OK) | SOS-004 |
| Fill ask | Ride members only | rider ID, missing time range | FILL-001 |
| Fill answer | Ride members only | rider ID, batch of positions with fix times | FILL-001 |
| Presence | Any Trail Buddy radio (no ride key) | anonymous rider ID, position, heading, speed, GPS fix time | NEAR-001, NEAR-002 |

**The envelope around every over-the-air message** must provide: protocol version, a packet identity for duplicate suppression, sender radio ID, destination (ride, one rider, or any Trail Buddy radio), hops remaining, and priority class (MESH-001, MESH-002). Ride messages use authenticated encryption with the ride key and a replay check (Section 12 of the Development Specification).

**Priority, highest first:** SOS and SOS replies, presence, ride positions, delivered confirmations, text, fill answers. SOS is never dropped when queues are full (SOS-002).

## Ride QR code

Created on the ride leader's phone and scanned by each rider (RIDE-001). It carries: format version, ride name, ride key, group tier. Nothing else.

## Not in version 1

- USB cable connection
- Maps, waypoints and trail routes on the radio
- Sensor readings (temperature, altitude)
- Changing settings on other riders' radios
- Seeing radios from other systems; the Nearby view shows Trail Buddy riders only

## Before this spec can be frozen

Every message, on both the phone link and over the air, needs:

- [ ] which fields are required and which optional
- [ ] a size limit for each text field and list, and a byte budget per over-the-air message
- [ ] units and precision for every number (for example position precision, speed in km/h or mph)
- [ ] at least one set of sample values
- [ ] the requirement ID it serves (done in this draft)

## Open questions

- [ ] Product owner: is this the complete list of features riders need in version 1?
- [ ] Product owner: is presence on or off by default? If a rider turns it off, should it switch back on automatically during their own SOS?
- [ ] Product owner: what counts as "approaching" for the warning (distance, closing speed, time until meeting)? Should the warning differ on narrow trail versus open ground?
- [ ] Spec team: presence must be heard by every Trail Buddy radio, but rides may use different frequencies. Do all Trail Buddy radios share one common channel, or does presence get its own channel and timing?
- [ ] Spec team: can ride positions and presence be one signal (presence plus an encrypted ride part) to save airtime?
- [ ] Spec team: how is a fake presence signal prevented or limited, given that presence has no ride key?
- [ ] Spec team: presence and position intervals per tier, checked against the airtime budget.
- [ ] What is the maximum length of a text message?
- [ ] Should location come from the phone's GPS or the radio's own GPS, and when does it switch?
- [ ] How many missed messages and track points should the radio store, and for how long?
- [ ] Firmware updates: through these phone-link messages, or through the board's own update mechanism?
- [ ] Product owner: which board leads version 1? The RAK4631 already has a board definition in Zephyr, the open-source system the firmware is built on, so the clean team wouldn't need to write one from schematics. The Pocket V3 builds on the same RAK family and is waterproof, with a buzzer for alerts, which makes it the strongest rider-ready candidate. The T114 V2 is the current plan's choice.
- [ ] Spec team: obtain the RAK19026 baseboard schematic and pin details from RAKwireless for the Pocket V3 board profile, and check how much of the RAK4631 board definition in Zephyr applies to it.
- [ ] Spec team: per-board needs. The ThinkNode M1 has an e-paper screen and several buttons; the Pocket V3 has a buzzer, useful for approaching-rider alerts (NEAR-005). Which board features does version 1 use?
- [ ] IP attorney: confirm the clean-room process before any release outside the company.

## Changes from v0.1

- Replaced the translation-layer architecture with Trail Buddy's own radio protocol end to end.
- Added Nearby riders (NEAR-001 to NEAR-008) and its messages.
- Added SOS, Fill on Return, firmware updates, repeater status and diagnostics.
- Replaced Groups with Ride (JOIN_RIDE, END_RIDE, RIDE_RESULT) and defined the ride QR code.
- Added GPS fix time and position source to location messages.
- Settings and ride changes now read back stored values; settings are written all-or-nothing.
- Added the over-the-air message list, envelope and priority order.
- Hardware: four boards under test, one firmware with a profile per board, built on one lead board first. Encoding set to our own protobuf schema.
- Added requirement IDs, the freeze checklist and new open questions.
