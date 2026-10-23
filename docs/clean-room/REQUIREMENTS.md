# Trail Buddy radio: product requirements

Version 0.5 · 23 October 2026 · Issued together with message spec Draft v0.5 (`docs/SPEC-v0.5.md`) and POLICY v0.5. Where a requirement points to a spec section, that section holds the detail.

**Which document wins.** REQUIREMENTS says what the product must do, POLICY says how the clean team works, and the SPEC says exactly how the radio behaves. If they disagree, none of them wins: write the conflict in OPEN-QUESTIONS.md and build nothing that depends on it until the spec team answers. Until then, for anything affecting SOS, the reading that is safer for the rider applies.

Every requirement has an ID. Every protocol decision in the spec cites the requirement it serves, a rationale and a test.

## Position

| ID | Requirement |
| --- | --- |
| POS-001 | Broadcast own position on the interval set by the ride's group tier. |
| POS-002 | Every position carries its GPS fix time. Receivers judge freshness by the fix time, never by when the packet arrived. |
| POS-003 | GPS duty cycle follows the rider's movement and presence setting, to save battery (spec 6.1). |

## Messages

| ID | Requirement |
| --- | --- |
| MSG-001 | Group text messages to everyone in the ride, up to 150 bytes. Text to a single rider is not in version 1. |
| MSG-002 | Delivery acknowledgement that names the message it confirms. The sender sees a count, for example "Delivered to 3 of 4", counting only riders within confirmation range (spec 6.3). |
| MSG-003 | Rider name and short ID announced to the ride, including to radios that join later. |

## SOS

| ID | Requirement |
| --- | --- |
| SOS-001 | SOS needs an active ride. It carries the best location available and its send time, and is never held back for lack of a GPS fix (spec 6.4). |
| SOS-002 | SOS goes ahead of every other queued transmission. The radio's own SOS and own replies always have queue space and are never dropped; relayed SOS traffic yields to them (spec 4.5). |
| SOS-003 | SOS is sent twice, 10 seconds apart, then repeated at least every 30 seconds until "I'm OK" or End Ride. An active SOS survives a reboot or battery swap and resumes (spec 6.4). |
| SOS-004 | "Copy that" replies reach the sender and the group. "I'm OK" ends the SOS for everyone. Only the rider who raised the SOS can end it, and no other ride member can fake that (spec 5, 6.4). |
| SOS-005 | The app alerts the rider to an SOS with a banner on the Map page and an SOS message with GPS coordinates on the Messages page, showing how long ago the SOS was first sent. The radio makes no SOS light or sound. |
| SOS-006 | In a 30-rider ride bunched together, an SOS reaches every radio within the hop limit within 10 seconds with probability at least 0.95, and within 40 seconds with probability at least 0.999, shown by the mesh simulator (spec 4.7). |

## Fill on Return

| ID | Requirement |
| --- | --- |
| FILL-001 | After a rider rejoins, missing positions are requested, answered from stored history, retried, and held back when the channel is busy. The returning rider's own radio answers, or says it has no history. "Filled" means the outline of a gap of up to 30 minutes reaches the app within 60 s of the return, and the full detail within 3 minutes, while the channel is below 15% busy. The rules are in spec 6.5. |

## Ride

| ID | Requirement |
| --- | --- |
| RIDE-001 | A ride is a private group with its own key, created on a phone and shared by QR code. |
| RIDE-002 | Four group tiers, each setting a hop limit and a position interval: 2–4 riders, 3 hops, 10 s; 5–9 riders, 4 hops, 45 s; 10–19 riders, 5 hops, 120 s; 20–30 riders, 7 hops, 90 s (spec 2.5). |
| RIDE-003 | End Ride returns the radio to a known default state (spec 6.7). |
| RIDE-004 | At most 30 riders per ride, repeaters included, and one active ride per radio. |
| RIDE-005 | The app makes a new ride key for every ride, from the phone's cryptographic random number generator. Ride codes expire 7 days after they are made; an expired code can't be joined, and a ride ends when its code expires, except during the rider's own SOS (spec 2.4, 8). |

## Mesh

| ID | Requirement |
| --- | --- |
| MESH-001 | Every radio relays ride traffic, with a hop limit. |
| MESH-002 | Duplicate packets are suppressed. |
| MESH-003 | Rebroadcasts wait a randomized delay, using the weighting in spec 4.6, which cites its published source. |
| MESH-004 | The radio reports a channel-busy measure the phone can read (spec 6.10). |
| MESH-005 | Every radio checks the channel before transmitting, with random backoff. An SOS is never dropped, and channel access never delays it by more than 1 second (spec 4.7). |

## Repeater

| ID | Requirement |
| --- | --- |
| RPT-001 | Repeater mode: a radio with no phone relays ride traffic (spec 6.8). |

## Nearby riders

Goal: help riders avoid trail collisions by seeing any Trail Buddy rider nearby, not just their own ride, and warning when one is approaching.

| ID | Requirement |
| --- | --- |
| NEAR-001 | Every Trail Buddy radio sends a short presence signal that any Trail Buddy radio can hear, whether or not they share a ride. |
| NEAR-002 | The presence signal carries position, heading, speed and the age of the GPS fix, plus an anonymous rider ID that changes at least once every 24 hours. It never carries names, messages or ride details. While presence is on, anyone nearby with LoRa equipment can see the rider's approximate position, speed and heading; the ID change stops casual linking between days, not tracking during a ride. |
| NEAR-003 | Presence is sent often while moving and less often when stopped. The intervals are set by the spec team against the airtime budget. |
| NEAR-004 | The radio passes to the app the presence signals it hears from radios outside its own ride, leaving out its own signal, repeats and implausible reports. Fake signals can't push real nearby riders out of view (spec 6.6). |
| NEAR-005 | The app warns the rider when another rider is approaching, using sound and vibration as well as the screen. Each rider's app decides this independently, so both riders are warned. The definition of "approaching" is deferred by the product owner; it doesn't affect the firmware. The aim is to show an unknown radio as far away as possible. |
| NEAR-006 | The rider can turn presence off, for example when parked. Presence is on by default, always on during the rider's own SOS, and on again after End Ride. |
| NEAR-007 | Presence never delays an SOS. |
| NEAR-008 | An empty "Nearby" view must never be presented as a clear trail. The app states that it shows Trail Buddy riders only, and says when the view may be incomplete. |
| NEAR-009 | At set-up, the app tells the rider in plain words what presence reveals (NEAR-002) and that they can turn it off. |

## Phone link

| ID | Requirement |
| --- | --- |
| BLE-001 | Trail Buddy's own Bluetooth Low Energy service, with a versioned message set and a protocol-version check on connect. |
| BLE-002 | Each settings change is one atomic write. The radio checks whether it already holds the value and reads every stored value back to the phone. No settings write can leave the radio unable to boot. |
| BLE-003 | The radio reports battery, firmware version and link status. |
| BLE-004 | Pairings are kept through reboots and updates and lost only by "forget phones" or factory reset. When an iPhone refuses to connect because the radio no longer holds its pairing, the app tells the rider exactly how to "Forget This Device" in iPhone settings and reconnect (spec 7.2). |

## Security

The attackers version 1 must resist are listed in spec 2.7 (Decision).

| ID | Requirement |
| --- | --- |
| SEC-001 | No encryption nonce is ever used twice under one ride key, by one radio or by two radios, by design rather than by detection (spec 4.4). |
| SEC-002 | Only the rider who raised an SOS can end it. A radio never drops, ends or changes an SOS because of a frame's time or a frame sent under its own identity, and tells the rider when someone sends under their identity (spec 4.8, 6.4). |
| SEC-003 | A new phone can pair only after someone presses the radio's button, or at first set-up; one new phone per pairing window; already-paired phones are told of new pairings. A stranger's phone can't block the rider's own phone, change the radio's clock or read the radio's identity (spec 6.11, 7.2). |
| SEC-004 | Recorded traffic replayed later is shown as old or rejected; it never appears as a new message or a new SOS (spec 4.4). |
| SEC-005 | Keys are made from cryptographic random sources, never logged or sent back, and erased from every copy at End Ride, expiry and factory reset. Release radios lock their debug port. The app offers a "ride compromised" step: everyone ends the ride and joins a new one (spec 4.9). |

## Hardware

| ID | Requirement |
| --- | --- |
| HW-001 | Version 1 runs on three boards from one firmware: the RAK19026 VC base board with the RAK4631 core, the Heltec Mesh Node T114 V2 and the Elecrow ThinkNode M1. Riders on any mix of these boards can share a ride. Each board's status light shows that it is powered, when it is rebooting, and when its pairing window is open (spec 9). |

## Updates, regulatory and targets

| ID | Requirement |
| --- | --- |
| DFU-001 | Firmware updates install over Bluetooth or USB, and both routes accept only signed images; the signature covers the version, board and size. The bootloader checks the signature before every start-up. A failed update rolls back to the previous confirmed image. Shipped radios have no unsigned route (spec 7.4). |
| REG-001 | Version 1 is for the United States only. Transmit power, antenna gain and bandwidth stay within the channel plan in spec 2.1, as confirmed by the compliance lab with each board's supplied antenna. |
| NFR-001 | Battery: at least 12 hours with tier 1, presence on, moving half the time, the board's standard battery, at around 20 °C. Fill: as FILL-001. Range: on the August field-test route with two riders, total blackout time and the longest blackout (43 minutes in that test) no worse than the August result. Frames of at most 184 bytes. Normal traffic keeps the channel at most 30% busy (spec 6.10). |

## Diagnostics

| ID | Requirement |
| --- | --- |
| DIAG-001 | The radio keeps diagnostic counters (spec 7.4) in release firmware, for field testing. Detailed logging is not included in release builds, and no log or counter holds key material. |

## Changes from v0.4

| ID | Change |
| --- | --- |
| MSG-002 | Count only riders within confirmation range. |
| SOS-002 | Own SOS and own replies always have queue space; relayed SOS traffic yields to them. |
| SOS-003 | Repeated at least every 30 seconds (was every 5 minutes). |
| SOS-004 | Only the owner can end an SOS, and no other member can fake it. |
| SOS-005 | App shows how long ago the SOS was first sent. |
| SOS (scope) | Raising an SOS from the radio without a phone is not in version 1 (decision). |
| SOS-006 | New: SOS delivery target. |
| FILL-001 | "No history" answer; filled target applies below 15% busy. |
| RIDE-005 | New: new key every ride; codes expire after 7 days. |
| MESH-005 | SOS listens before talking, but channel access never delays it more than 1 s. |
| NEAR-002 | States what presence reveals. |
| NEAR-004 | Fake signals can't push real riders out of view. |
| NEAR-008 | App says when the view may be incomplete. |
| NEAR-009 | New: the app tells the rider at set-up. |
| BLE-004 | New: pairings kept; iPhone "forget this device" help. |
| SEC-001 to SEC-005 | New: security requirements from the security review. |
| HW-001 | Status light also shows the pairing window. |
| DFU-001 | Signed over Bluetooth and USB; signature covers version, board and size; checked before every start-up; no unsigned route on shipped radios. |
| REG-001 | Lab tests each board with its supplied antenna. |
| NFR-001 | Frames of at most 184 bytes (was 176). |
| DIAG-001 | No key material in logs or counters. |

Edits made in v0.4 and first listed here (Q111): MESH-003 (points to spec 4.6), MESH-004 (points to spec 6.10), RPT-001 (points to spec 6.8), NEAR-005 ("approaching" deferred).
