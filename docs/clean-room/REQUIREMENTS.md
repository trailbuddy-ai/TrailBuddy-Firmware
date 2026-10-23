# Trail Buddy radio: product requirements

Version 0.4 · 23 October 2026 · Issued together with message spec Draft v0.4 (`docs/SPEC-v0.4.md`) and POLICY v0.4. Where a requirement points to a spec section, that section holds the detail.

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
| MSG-002 | Delivery acknowledgement that names the message it confirms. The sender sees a count, for example "Delivered to 3 of 4" (spec 6.3). |
| MSG-003 | Rider name and short ID announced to the ride, including to radios that join later. |

## SOS

| ID | Requirement |
| --- | --- |
| SOS-001 | SOS needs an active ride. It carries the best location available and its send time, and is never held back for lack of a GPS fix (spec 6.4). |
| SOS-002 | SOS goes ahead of every other queued transmission. An SOS or an "I'm OK" is never dropped when queues are full; a "Copy that" may be (spec 4.5). |
| SOS-003 | SOS is sent twice, 10 seconds apart, then repeated every 5 minutes until "I'm OK" or End Ride. An active SOS survives a reboot or battery swap and resumes (spec 6.4). |
| SOS-004 | "Copy that" replies reach the sender and the group; "I'm OK" cancels the SOS for everyone. Only the rider who raised the SOS can cancel it. |
| SOS-005 | The app alerts the rider to an SOS with a banner on the Map page and an SOS message with GPS coordinates on the Messages page. The radio makes no SOS light or sound. |

## Fill on Return

| ID | Requirement |
| --- | --- |
| FILL-001 | After a rider rejoins, missing positions are requested, answered from stored history, retried, and held back when the channel is busy. The returning rider's own radio answers. "Filled" means the outline of a gap of up to 30 minutes reaches the app within 60 s of the return, and the full detail within 3 minutes. The rules are in spec 6.5. |

## Ride

| ID | Requirement |
| --- | --- |
| RIDE-001 | A ride is a private group with its own key, created on a phone and shared by QR code. |
| RIDE-002 | Four group tiers, each setting a hop limit and a position interval: 2–4 riders, 3 hops, 10 s; 5–9 riders, 4 hops, 45 s; 10–19 riders, 5 hops, 120 s; 20–30 riders, 7 hops, 90 s (spec 2.5). |
| RIDE-003 | End Ride returns the radio to a known default state (spec 6.7). |
| RIDE-004 | At most 30 riders per ride, repeaters included, and one active ride per radio. |

## Mesh

| ID | Requirement |
| --- | --- |
| MESH-001 | Every radio relays ride traffic, with a hop limit. |
| MESH-002 | Duplicate packets are suppressed. |
| MESH-003 | Rebroadcasts wait a randomized delay, using the weighting in spec 4.6, which cites its published source. |
| MESH-004 | The radio reports a channel-busy measure the phone can read (spec 6.10). |
| MESH-005 | Every radio checks the channel before transmitting. An SOS never waits for a clear channel (spec 4.7). |

## Repeater

| ID | Requirement |
| --- | --- |
| RPT-001 | Repeater mode: a radio with no phone relays ride traffic (spec 6.8). |

## Nearby riders

Goal: help riders avoid trail collisions by seeing any Trail Buddy rider nearby, not just their own ride, and warning when one is approaching.

| ID | Requirement |
| --- | --- |
| NEAR-001 | Every Trail Buddy radio sends a short presence signal that any Trail Buddy radio can hear, whether or not they share a ride. |
| NEAR-002 | The presence signal carries position, heading, speed and the age of the GPS fix, plus an anonymous rider ID that changes at least once every 24 hours. It never carries names, messages or ride details. |
| NEAR-003 | Presence is sent often while moving and less often when stopped. The intervals are set by the spec team against the airtime budget. |
| NEAR-004 | The radio passes to the app the presence signals it hears from radios outside its own ride, leaving out its own signal, repeats and implausible reports (spec 6.6). |
| NEAR-005 | The app warns the rider when another rider is approaching, using sound and vibration as well as the screen. Each rider's app decides this independently, so both riders are warned. The definition of "approaching" is deferred by the product owner; it doesn't affect the firmware. |
| NEAR-006 | The rider can turn presence off, for example when parked. Presence is on by default, always on during the rider's own SOS, and on again after End Ride. |
| NEAR-007 | Presence never delays an SOS. |
| NEAR-008 | An empty "Nearby" view must never be presented as a clear trail. The app states that it shows Trail Buddy riders only. |

## Phone link

| ID | Requirement |
| --- | --- |
| BLE-001 | Trail Buddy's own Bluetooth Low Energy service, with a versioned message set and a protocol-version check on connect. |
| BLE-002 | Each settings change is one atomic write. The radio checks whether it already holds the value and reads every stored value back to the phone. No settings write can leave the radio unable to boot. |
| BLE-003 | The radio reports battery, firmware version and link status. |

## Hardware

| ID | Requirement |
| --- | --- |
| HW-001 | Version 1 runs on three boards from one firmware: the RAK19026 VC base board with the RAK4631 core, the Heltec Mesh Node T114 V2 and the Elecrow ThinkNode M1. Riders on any mix of these boards can share a ride. Each board's status light shows that it is powered and when it is rebooting (spec 9). |

## Updates, regulatory and targets

| ID | Requirement |
| --- | --- |
| DFU-001 | Firmware updates use signed images and roll back if an update fails. |
| REG-001 | Version 1 is for the United States only. Transmit power, antenna gain and bandwidth stay within the channel plan in spec 2.1, as confirmed by the compliance lab. |
| NFR-001 | Battery: at least 12 hours with tier 1, presence on, moving half the time, the board's standard battery, at around 20 °C. Fill: as FILL-001. Range: on the August field-test route with two riders, total blackout time and the longest blackout (43 minutes in that test) no worse than the August result. Frames of at most 176 bytes. Normal traffic keeps the channel at most 30% busy (spec 6.10). |

## Diagnostics

| ID | Requirement |
| --- | --- |
| DIAG-001 | The radio keeps diagnostic counters (spec 7.4) in release firmware, for field testing. Detailed logging is not included in release builds. |

## Changes from v0.3

| ID | Change |
| --- | --- |
| Header | "Which document wins" rule added. |
| SOS-001 | Needs a ride; never held back for lack of a fix. |
| SOS-002 | "Never dropped" now covers SOS and "I'm OK"; a "Copy that" may be dropped. |
| SOS-003 | Survives reboot. |
| SOS-005 | New: the app alerts the rider (Map banner, Messages entry with coordinates); no SOS light or sound on the radio. |
| FILL-001 | "Filled" defined. |
| RIDE-004 | Repeaters count towards 30. |
| MESH-003 | Points to the chosen weighting in spec 4.6. |
| MESH-005 | New: listen before talking. |
| NEAR-002 | Fix age instead of fix time; ID change "at least once every 24 hours". |
| NEAR-004 | Own ride, own signal, repeats and implausible reports left out. |
| NEAR-005 | "Approaching" deferred by the product owner. |
| REG-001 | United States only. |
| HW-001 | New: the three version 1 boards. |
| NFR-001 | Battery conditions, fill target and range target defined. |
