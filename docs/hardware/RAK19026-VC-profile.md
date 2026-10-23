# Hardware profile: RAK19026 VC WisMesh Base Board

Draft, 22 October 2026. Source: `docs/hardware/RAK19026_VC_WisMesh_Base_Board_Datasheet_cleaned.pdf` (RAKwireless, board version VC), page numbers below. Nothing else was used. Supports POS-001 to POS-003 (GNSS), BLE-003 (battery) and NFR-001 (battery life); this is documentation, so it has no test of its own.

Pin names are as printed on the RAK4630 schematic symbol (page 18, Figure 14): the board net name, then the nRF52840 pin. Where the schematic marks a part "0/NC", the part may not be fitted. Pin readings came from the schematic images and must be checked against the full-size pages before any code relies on them.

**This is not the Pocket V3 profile.** Spec 9 (line 631) says the Pocket V3 uses the RAK19026 baseboard. No manufacturer document in the repository says so. See "Open gaps".

## Main parts (pages 1–4, 8, 18)
| Part | Detail |
| --- | --- |
| Processor and radio | RAK4630 module: Nordic nRF52840 and Semtech SX1262 (page 2) |
| GNSS | u-blox ZOE-M8Q: GPS, GLONASS, QZSS, BeiDou; serial and I2C; 10 Hz update; cold start 29 s, hot start 1 s (page 8) |
| Accelerometer | ST LIS3DH on I2C (page 8). Its interrupt pin is not recorded here. |
| USB | USB-C straight to the module's USB port; programming, serial and battery charging (pages 1, 5) |
| Display | 4-pin header for an I2C OLED (J2); version 1 does not use screens (page 5) |

## Pins used by Trail Buddy (page 18)
| Function | Net | Module pin / nRF pin | Note |
| --- | --- | --- | --- |
| GNSS receive (from GNSS) | RXD1 | 6 / P0.15 (UART2_RX) | |
| GNSS transmit (to GNSS) | TXD1 | 7 / P0.16 (UART2_TX) | through a diode (D8) |
| GNSS on I2C | I2C1_SDA, I2C1_SCL | 4 / P0.13, 5 / P0.14 | 0-ohm links R36, R37 |
| GNSS standby | IO4 | 41 / P0.04 (AIN2) | link R44 marked "0/NC" |
| GNSS reset | IO6 | 12 / P0.10 (named NFC2) | link R45 marked "0/NC" |
| GNSS time pulse | IO3 | 11 / P0.21 (UART1_DE) | link R39 marked "0/NC" |
| User button | SW1 | 25 / P1.01 | "User Defined Button" (page 13) |
| LED 1 | LED1 | 27 / P1.03 | page 13 calls it "LED for battery charge indicator", but page 8 says the red charge LED is wired to the charger chip and the green and blue LEDs are the ones the processor controls. Which of LED1 and LED2 is which colour was not determined. |
| LED 2 | LED2 | 28 / P1.04 | "LED for custom used" |
| LED 3 | LED3 | not read | |
| Reset button | RESET | 17 (NRF_RESET) | resets the processor (page 8) |
| Battery voltage reading | ADC_VBAT, on net AIN0 | 40 / P0.05 (AIN3) | through link R33 (0 ohm) |
| Sensor-power switch | IO2 | 26 / P1.02 (SW2) | IO2 turns the switched 3.3 V supply (3V3_S) on and off (page 13) |

## Power (pages 7, 13–15, 17)
- **Battery:** rechargeable LiPo, 3.7 V nominal, 3.3–4.3 V range; 500 mA discharge or more; EU/UK boards need an NTC (temperature sensor) wire. Do not use non-rechargeable cells (pages 7, 14).
- **Charging:** USB 4.5–5.5 V. The datasheet table says 350 mA (page 14); the schematic notes say "VCHG=4.20V, ICHG=440mA" (page 17). **The two differ.**
- **Battery reading:** the schematic shows a resistor divider from the switched battery rail to ADC_VBAT, which looks like 1 MΩ over 1.5 MΩ (page 17, to be confirmed at full size). The datasheet text gives no conversion formula.
- **Power key (K1):** a slide switch between VBAT and the switched rail (VBAT_S). It cuts power to the board. Software cannot read it (pages 1, 17).
- **Current:** under 2 µA with no modules; under 10 µA asleep with core and sensor; up to 130 mA while the LoRa module transmits (page 13).
- **Temperature:** operating −35 to +75 °C; extended −40 to +80 °C (page 17).
- **Solar:** 5 V panels only (page 7).

## Open gaps
1. **Pocket V3.** No document exists yet. Nothing here confirms it uses this board. Its profile, buzzer and power button are on hold (see `TODO.md`).
2. **Buzzer.** Not in this datasheet.
3. **Power button.** This board has a battery slide switch and a user button, but no button that software could treat as a power button.
4. **Radio wiring.** The SX1262's control pins are inside the RAK4630 module and are not on this schematic. The module documentation is not in the repository. On this sheet, the module's LoRa antenna pin (37) and Bluetooth antenna pin (15) are marked as not connected.
5. **LEDs and the accelerometer interrupt pin.** The LED1 description on page 13 and page 8 disagree; LED3's pin and the accelerometer interrupt pin are not recorded.
6. **Charging current.** 350 mA against 440 mA, as above.
7. **Battery divider.** Resistor values and the conversion to volts need confirming.
8. **Schematic readings.** All pin entries above need checking at full size.
