# Trail Buddy radio: product requirements

Version 0.2 · 22 October 2026 · Source: Clean-Room Firmware Development Specification, Part 2 Section 9, plus the Nearby riders requirements from the message spec (Draft v0.2).

Every requirement has an ID. Every protocol decision in the spec cites the requirement it serves, a rationale and a test.

## Position

| ID | Requirement |
| --- | --- |
| POS-001 | Broadcast own position on the interval set by the ride's group tier. |
| POS-002 | Every position carries its GPS fix time. Receivers judge freshness by the fix time, never by when the packet arrived. |
| POS-003 | GPS duty cycle follows the group tier, to save battery. |

## Messages

| ID | Requirement |
| --- | --- |
| MSG-001 | Group text messages to everyone in the ride. |
| MSG-002 | Delivery acknowledgement that names the message it confirms. |
| MSG-003 | Rider name and short ID announced to the ride, including to radios that join later. |

## SOS

| ID | Requirement |
| --- | --- |
| SOS-001 | SOS carries location and send time. |
| SOS-002 | SOS goes ahead of every other queued transmission and is never dropped when queues are full. |
| SOS-003 | SOS is sent twice, 10 seconds apart, then repeated every 5 minutes until "I'm OK" or End Ride. |
| SOS-004 | "Copy that" replies reach the sender and the group; "I'm OK" cancels the SOS for everyone. |

## Fill on Return

| ID | Requirement |
| --- | --- |
| FILL-001 | After a rider rejoins, missing positions are requested, answered from stored history, retried, and held back when the channel is busy. The spec team restates the full five rules in the spec. |

## Ride

| ID | Requirement |
| --- | --- |
| RIDE-001 | A ride is a private group with its own key, created on a phone and shared by QR code. |
| RIDE-002 | Four group tiers, each setting a hop limit and a position interval. |
| RIDE-003 | End Ride returns the radio to a known default state. |

## Mesh

| ID | Requirement |
| --- | --- |
| MESH-001 | Every radio relays ride traffic, with a hop limit. |
| MESH-002 | Duplicate packets are suppressed. |
| MESH-003 | Rebroadcasts wait a randomized delay. The spec team chooses any weighting from published literature and cites it in DECISIONS.md. |
| MESH-004 | The radio reports a channel-busy measure the phone can read. |

## Repeater

| ID | Requirement |
| --- | --- |
| RPT-001 | Repeater mode: a radio with no phone relays ride traffic. |

## Nearby riders

Goal: help riders avoid trail collisions by seeing any Trail Buddy rider nearby, not just their own ride, and warning when one is approaching.

| ID | Requirement |
| --- | --- |
| NEAR-001 | Every Trail Buddy radio sends a short presence signal that any Trail Buddy radio can hear, whether or not they share a ride. |
| NEAR-002 | The presence signal carries position, heading, speed and GPS fix time, plus an anonymous rider ID that changes at least daily. It never carries names, messages or ride details. |
| NEAR-003 | Presence is sent often while moving and less often when stopped. The intervals are set by the spec team against the airtime budget. |
| NEAR-004 | The radio passes every presence signal it hears to the app. |
| NEAR-005 | The app warns the rider when another rider is approaching, using sound and vibration as well as the screen. Each rider's app decides this independently, so both riders are warned. |
| NEAR-006 | The rider can turn presence off, for example when parked. Whether it is on by default is a product decision. |
| NEAR-007 | Presence never delays an SOS. |
| NEAR-008 | An empty "Nearby" view must never be presented as a clear trail. The app states that it shows Trail Buddy riders only. |

## Phone link

| ID | Requirement |
| --- | --- |
| BLE-001 | Trail Buddy's own Bluetooth Low Energy service, with a versioned message set and a protocol-version check on connect. |
| BLE-002 | Each settings change is one atomic write. The radio checks whether it already holds the value and reads every stored value back to the phone. No settings write can leave the radio unable to boot. |
| BLE-003 | The radio reports battery, firmware version and link status. |

## Updates, regulatory and targets

| ID | Requirement |
| --- | --- |
| DFU-001 | Firmware updates use signed images and roll back if an update fails. |
| REG-001 | Transmit power, dwell time and duty cycle stay within the approved channel plan. |
| NFR-001 | Targets for range, battery hours per riding day, maximum packet size and return-to-filled-track time, set by the product owner. |
