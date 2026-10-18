# Pytes CAN Protocol

A "reverse engineering" of the CAN protocols spoken by Pytes batteries (V5, E-BOX). Inspired by https://github.com/dfch/BydCanProtocol

The battery speaks **one** inverter protocol at a time. This document covers the two that are relevant for Victron and OpenDTU-OnBattery:

| Protocol | Selection | Identifiers | Content |
| --- | --- | --- | --- |
| [Victron](#victron-protocol-default) | **default** (all DIP switches off) | `0x351`-`0x381` (Victron/Venus OS CAN-bus BMS) | aggregated values of all modules |
| [Pytes](#pytes-protocol-manual-selection) | must be **selected manually** ("PYTES") | `0x400`-`0x41E`, `0x500`+ on request | more detail: capacities in mAh, alarm bit fields, min/max cell with module, balancing, heating, protection thresholds, per-cell data on request |

### Protocol selection

At boot the battery takes its inverter protocol from the 6 DIP switches ("ADD") of the master (switch 1 = bit 0, switch 6 = bit 5, up = on). The **default is Victron** (all switches off).

The DIP selection can be overridden from the service console with `protocol <name>` (e.g. `protocol PYTES`). This stores the protocol in the BMS configuration (address `0x78E`) and switches immediately; while a protocol is stored the **DIP switches are ignored**. Clearing the stored value hands the selection back to the DIP switches. `protocol` without a valid name lists the supported protocols.

To select the **Pytes protocol** via DIP, set **switch 6 up** and all other switches down.

The firmware supports many other inverter protocols on the remaining DIP values; only Victron (default) and Pytes are relevant here, so they are not covered.

## Sources

* **Bus dump** between a Pytes V5a and a Victron Venus OS v3.34 (Victron protocol). All information has been dumped on a Raspberry 4 with Venus OS v3.34 and a waveshare RS485 CAN HAT with `cansniffer -c -t 0 can0`. The example frames in the Victron table are from this dump.
* **Firmware analysis** of `SPBMS16SRPV1.10.28.C16` (device `E-BOX-48100V-D`, built 2025-06-24, GD32F307 MCU), disassembled with Ghidra. All byte layouts, scalings and bit meanings below are taken from the code that builds the frames, unless marked otherwise. Cross-checked against `V1.10.22` and `V1.10.26` to establish the firmware versions below.
* **Comparison with RS485** on 2 x Pytes V5 with CAN and RS485 connected at the same time.
* **Community CAN dump** of a Pytes E-Box 4850-C: https://github.com/helgeerbe/OpenDTU-OnBattery/discussions/541#discussioncomment-9771146

## Firmware versions

The **Victron protocol** (`0x351`-`0x381`) and the **Pytes protocol cyclic frames** (`0x400`-`0x41E`) are present in every firmware observed, on both product lines..

The **per-module data on request** (`0x307` → `0x500`+, and the per-module version `0x19A60`) is the only version-gated part, and the requirement differs **per product line and per BMS flash chip**. The two product lines use separate version schemes, shown in the `0x40A`/`0x35E` manufacturer area (e.g. `SPBMS16SRPV1.10.28` or `SPBMS16SRPV1.5.38.C16`):

| Battery | BMS | Per-module minimum | Verified |
| --- | --- | --- | --- |
| E-BOX 48100**V** (V5°) | `SPBMS16SRP…V1.10.x` | **V1.10.26** (C16 chip) | absent in V1.10.22, present from V1.10.26 and V1.10.28; V1.10.23-25 not checked (not public) |
| E-BOX 48100**R**-C | `SPBMS16SRP…V1.5.x` | **V1.5.38, C16 chip only** | absent in V1.5.25.C16 **and** V1.5.38.**C8**; present in V1.5.38.**C16** |

The `.C8` / `.C16` suffix is the BMS flash chip (8 vs 16 Mbit). The **C8 variant omits the per-module feature even at the same firmware version** (dropped to fit the smaller flash), so the flash chip matters as much as the version number. The cyclic frames and both inverter protocols are unaffected by the chip.

Everything else in this document (frame layouts, scalings, the per-module request/response format) is identical across both product lines and all versions that implement it — only the group numbering differs internally (48100R counts groups from 1, 48100V from 0), which produces the same CAN identifiers on the wire.

## General

* CAN 2.0 at 500 kbit/s, standard 11 bit identifiers (no FD).
* All values little endian.
* The frames are sent every 1000 ms, with a short gap between the frames of one cycle (10 ms Victron, 5 ms Pytes). The battery sends whether or not an inverter is present: the inverter keep-alive (`0x305`) is accepted and counted, but sending does not depend on it.
* What the battery receives on the inverter port:
  * **Victron protocol:** only `0x305` (keep-alive, content ignored). Nothing else is evaluated.
  * **Pytes protocol:** `0x305` (ignored), `0x307` ([per-module cell data request](#per-module-cell-data-on-request)), `0x19A60` extended ([per-module firmware version request](#per-module-firmware-version-on-request)) and `0x19A10`-`0x19A50` extended (firmware update).
* **Battery data is aggregated over all modules** by the master: voltage and temperature are averages, currents and capacities are sums, cycles is the maximum, alarm words are OR-ed over all modules.
* Internally the BMS uses mV, mA, m°C and mAh; the frames scale them to the unit noted per field.

### Cell / module labels

Several frames identify the cell (or temperature sensor) with the minimum / maximum value together with its module:

* **Pytes protocol** (`0x401`, `0x402`): 16 bit, `cell | (module << 8)`, both 0-based.
* **Victron protocol** (`0x374`-`0x377`): 4 ASCII digits `CCMM`, 0-based cell followed by 0-based module, e.g. `"0500"` = cell 6 of module 1.

---

## Victron protocol (default)

The **Hex** / **Ascii** example columns are real frames from the [bus dump](#sources) (a Pytes V5 in Victron mode on Venus OS). The byte layouts, scalings and bit meanings are confirmed against the firmware analysis.

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 351 | 38 02 E8 03 E8 03 C7 01 | 8....... | **Dvcc**: CVL, CCL, DCL, DVL |
|     |                         |          | [00:01] "38 02" (V/10, DeciVolt) 56.8V CVL, charge voltage limit |
|     |                         |          | [02:03] "E8 03" (A/10, DeciAmp) 100.0A CCL, charge current limit |
|     |                         |          | [04:05] "E8 03" (A/10, DeciAmp) 100.0A DCL, discharge current limit |
|     |                         |          | [06:07] "C7 01" (V/10, DeciVolt) 45.5V DVL, discharge voltage limit |
| 355 | 33 00 64 00             | 3.d.     | **StateInfo**: State of Charge, State of Health (DLC 4) |
|     |                         |          | [00:01] "33 00" (%) 51% SoC |
|     |                         |          | [02:03] "64 00" (%) 100% SoH |
| 356 | 8E 14 F9 FF B4 00       | ......   | **BatteryStats**: Voltage, Amps, Temperature (DLC 6) |
|     |                         |          | [00:01] "8E 14" (V/100, CentiVolt) 52.62V average module voltage |
|     |                         |          | [02:03] "F9 FF" (A/10, DeciAmp, signed) -0.7A total current; "-" discharge / "+" charge |
|     |                         |          | [04:05] "B4 00" (°C/10, Deci, signed) 18.0°C average cell temperature |
| 35A | see below               |          | **Alarms and Warnings**, bit field |
| 35E | 50 59 54 45 53          | PYTES    | **ManufacturerInfo** (DLC 5) |
|     |                         |          | [00:04] "50 59 54 45 53" (string) "PYTES" manufacturer identification |
| 35F | 01 00 6E 01 32 00       | ..n.2.   | **BatteryInfo**: Firmware version, Ah available (DLC 6) |
|     |                         |          | [00:01] "01 00" literal constant written by the firmware (byte 0 = 1, byte 1 = 0; not read from any version field) |
|     |                         |          | [02:03] "6E 01" internal build number: the firmware parses a decimal string `N` compiled into the image and sends byte 2 = `N / 100`, byte 3 = `N % 100`. `SPBMS16SRPV1.10.28` has `N = "2801"` → "1C 01" (28.01); the dumped older firmware had `N = 11001` → "6E 01" (110.01). This is an internal build identifier, **not** the `SPBMS16SRP…` version string — here the `28` only coincidentally matches the `.28` patch level. |
|     |                         |          | [04:05] "32 00" (Ah) 50Ah remaining capacity (sum of all modules) |
| 360 | 00                      | .        | **Charge request** (DLC 1) |
|     |                         |          | [00] "FF" = request charging: force charge I or II is active (see Pytes `0x408`) and no module blocks charging; "00" otherwise |
| 372 | 02 00 01 00 01 00 02 00 | ........ | **BankInfo** |
|     |                         |          | [00:01] "02 00" modules online |
|     |                         |          | [02:03] "01 00" modules (of the highest module count seen) without per-module status flag 1, see `0x40C`. Interpreted as "modules blocking charge" |
|     |                         |          | [04:05] "01 00" same for status flag 2, interpreted as "modules blocking discharge" |
|     |                         |          | [06:07] "02 00" modules offline: highest module count seen minus modules online |
| 373 | D8 0C DA 0C 21 01 23 01 | ....!.#. | **CellInfo**: Cell Voltage and Temperature |
|     |                         |          | [00:01] "D8 0C" (mV) 3.288V Lowest Cell Voltage, see 374 |
|     |                         |          | [02:03] "DA 0C" (mV) 3.290V Highest Cell Voltage, see 375 |
|     |                         |          | [04:05] "21 01" (Kelvin) 289K, 16°C Minimum Cell Temperature, see 376 |
|     |                         |          | [06:07] "23 01" (Kelvin) 291K, 18°C Maximum Cell Temperature, see 377 |
| 374 | 30 38 30 30 00 00 00 00 | 0800.... | **Cell name** (string) "0800" with "Lowest Cell Voltage": cell 9 of module 1, see 373 and [labels](#cell--module-labels) |
| 375 | 30 34 30 30 00 00 00 00 | 0400.... | **Cell name** (string) "0400" with "Highest Cell Voltage": cell 5 of module 1, see 373 and [labels](#cell--module-labels)  |
| 376 | 30 32 30 30 00 00 00 00 | 0200.... | **Cell name** (string) "0200" with "Minimum Cell Temperature": sensor 3 of module 1, see 373 and [labels](#cell--module-labels)  |
| 377 | 30 33 30 30 00 00 00 00 | 0300.... | **Cell name** (string) "0300" with "Maximum Cell Temperature": sensor 4 of module 1, see 373 and [labels](#cell--module-labels)  |
| 378 | 40 08 00 00 2B 07 00 00 | @...+... | **History**: Charged / Discharged Energy (sum of all modules) |
|     |                         |          | [00:03] "40 08 00 00" (kWh/10, HectoWattHour) 211.2kWh Charged Energy |
|     |                         |          | [04:07] "2B 07 00 00" (kWh/10, HectoWattHour) 183.5kWh Discharged Energy |
| 379 | 64 00                   | d.       | **BatterySize**: Installed Ah (DLC 2) |
|     |                         |          | [00:01] "64 00" (Ah) 100Ah full charge capacity (sum of all modules) |
| 380 | XX XX XX XX XX XX XX XX | redacted | **Serial number - part 1** (string) |
| 381 | XX XX XX XX XX XX XX XX | redacted | **Serial number - part 2** (string) |

### 0x35A Alarms and Warnings

Every condition is a 2 bit field: `01` = alarm/warning active, `10` = OK, `00` = not supported. Firmware 1.10.28 sends the `10` "OK" pattern, so an all-OK frame is `A8 AA 82 00 A0 AA 02 00` (the older firmware of the bus dump sent all `00`).

Source bits refer to the Pytes alarm words (see [`0x403`](#0x403--0x406-bit-fields) and `0x406`). "while discharging/charging" means the condition is only reported in that battery state.

| Byte | Bits | Alarm / Warning | Source |
| --- | --- | --- | --- |
| 0 | 2-3 | High battery voltage: Alarm | over-voltage protection (bit 0) |
| 0 | 4-5 | Low battery voltage: Alarm | under-voltage protection (bit 4) |
| 0 | 6-7 | High temperature: Alarm | over-temperature (bit 8) while discharging |
| 1 | 0-1 | Low temperature: Alarm | under-temperature (bit 12) while discharging |
| 1 | 2-3 | High charge temperature: Alarm | over-temperature (bit 8) while charging |
| 1 | 4-5 | Low charge temperature: Alarm | under-temperature (bit 12) while charging |
| 1 | 6-7 | High discharge current: Alarm | discharge over-current (bits 17, 18, 24) |
| 2 | 0-1 | High charge current: Alarm | charge over-current (bits 19, 20, 29) |
| 2 | 6-7 | Internal failure: Alarm | module communication timeout (`0x406` bit 9) |
| 3 | 0-1 | Cell imbalance: Alarm | not supported, always `00` |
| 4 | 2-3 | High battery voltage: Warning | not supported, always `00` |
| 4 | 4-5 | Low battery voltage: Warning | low voltage warning (bit 3) |
| 4 | 6-7 | High temperature: Warning | high temperature warning (bit 9) while discharging |
| 5 | 0-1 | Low temperature: Warning | low temperature warning (bit 11) while discharging |
| 5 | 2-3 | High charge temperature: Warning | high temperature warning (bit 9) while charging |
| 5 | 4-5 | Low charge temperature: Warning | low temperature warning (bit 11) while charging |
| 5 | 6-7 | High discharge current: Warning | discharge current warning (bit 21) |
| 6 | 0-1 | High charge current: Warning | charge current warning (bit 22) |
| 6 | 6-7 | Internal failure: Warning | not supported, always `00` |
| 7 | 0-1 | Cell imbalance: Warning | not supported, always `00` |

---

## Pytes protocol (manual selection)

Must be selected manually in the battery settings ("PYTES"). Identifiers `0x400`-`0x40D` and `0x41E` are sent every cycle, `0x40E`-`0x41D` about every 600 cycles, `0x500`+ only on request.

If the batteries are split into groups, every identifier is shifted up by `group index * 0x1000`. The **group index** (0-15) is taken from DIP switches 1-4 (bits 0-3) when switch 6 is up; otherwise it is 0. The default — switch 6 up, switches 1-4 down — is group index **0**, so the offset is 0 and the identifiers are exactly as listed. Group index 1 shifts them by `0x1000` (e.g. `0x400` → `0x1400`, `0x500` → `0x1500`), and so on. (The service console prints the group index + 1, so index 0 is shown as "Group Address: 1".)

No Pytes-protocol bus dump was captured, so the **Hex** / **Ascii** example columns below are *derived* from the same battery state as the Victron bus dump above (to illustrate the layout and let the two tables be read the same way). Fields that state does not cover are shown as `—`. The byte layouts, scalings and bit meanings themselves are from the firmware analysis, not from the examples.

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 400 | 38 02 E8 03 E8 03 C7 01 | 8....... | **Limits** (same as Victron `0x351`) |
|     |   |   | [00:01] (V/10) charge voltage limit |
|     |   |   | [02:03] (A/10) charge current limit |
|     |   |   | [04:05] (A/10) discharge current limit |
|     |   |   | [06:07] (V/10) discharge voltage limit |
| 401 | DA 0C D8 0C 04 00 08 00 | ........ | **Cell voltage min/max** |
|     |   |   | [00:01] "DA 0C" (mV) 3.290V highest cell voltage |
|     |   |   | [02:03] "D8 0C" (mV) 3.288V lowest cell voltage |
|     |   |   | [04:05] "04 00" label of the highest cell: 1-5, see [labels](#cell--module-labels) |
|     |   |   | [06:07] "08 00" label of the lowest cell: 1-9, see [labels](#cell--module-labels) |
| 402 | B4 00 A0 00 03 00 02 00 | ........ | **Cell temperature min/max** |
|     |   |   | [00:01] "B4 00" (°C/10, **signed**) 18.0°C highest cell temperature |
|     |   |   | [02:03] "A0 00" (°C/10, **signed**) 16.0°C lowest cell temperature |
|     |   |   | [04:05] "03 00" label of the sensor with the highest temperature: 1-4, see [labels](#cell--module-labels) |
|     |   |   | [06:07] "02 00" label of the sensor with the lowest temperature: 1-3, see [labels](#cell--module-labels) |
| 403 | 00 00 00 00 00 00 00 00 | ........ | **Alarms and warnings**, see [bit fields](#0x403--0x406-bit-fields) |
|     |   |   | [00:03] "00 00 00 00" protection word, OR over all modules (also carries the battery state); none set in this example |
|     |   |   | [04:07] "00 00 00 00" second alarm word, OR over all modules, same layout (voltage and temperature bits) |
| 404 | 33 00 64 00 02 00 00 00 | 3.d..... | **State** |
|     |   |   | [00:01] "33 00" (%) 51% SoC, integer (use `0x409` for a precise value) |
|     |   |   | [02:03] "64 00" (%) 100% SoH |
|     |   |   | [04:05] "02 00" 2 modules (highest count seen) |
|     |   |   | [06:07] "00 00" cycle count (maximum of all modules) |
| 405 | 8E 14 F9 FF B4 00 20 00 | ...... . | **Battery stats** (as Victron `0x356`) |
|     |   |   | [00:01] "8E 14" (V/100) 52.62V average module voltage |
|     |   |   | [02:03] "F9 FF" (A/10, signed) -0.7A total current, "-" discharge / "+" charge |
|     |   |   | [04:05] "B4 00" (°C/10, signed) 18.0°C average cell temperature |
|     |   |   | [06:07] "20 00" 32 total number of cells (all modules) |
| 406 | 00 00 00 00 00 00 00 00 | ........ | **Failures**, see [bit fields](#0x403--0x406-bit-fields) |
|     |   |   | [00:03] "00 00 00 00" failure word, OR over all modules; none set in this example |
|     |   |   | [04] "00" flag, meaning unclear (1 while every module is charging or has an internal flag set) |
|     |   |   | [05:07] "00 00 00" 0 |
| 407 | — | — | **Status words**, two per-module status words, OR over all modules. Distinct from the `0x403` alarm and `0x406` failure words (they aggregate different per-module fields). The individual bit meanings are not labelled in the firmware's frame builder and have not been decoded. |
|     |   |   | [00:03] per-module status word A (per-module field `[0xe]`) |
|     |   |   | [04:07] per-module status word B (per-module field `[0xd]`) |
| 408 | 01 01 00 00 00 00 00 | ....... | **Charge status** (DLC 7) |
|     |   |   | [00] "01" 1 = charging allowed (charge current limit != 0) |
|     |   |   | [01] "01" 1 = discharging allowed (discharge current limit != 0) |
|     |   |   | [02] "00" 1 = force charge I: raised at SoC <= 8%, cleared at >= 12% |
|     |   |   | [03] "00" 1 = force charge II: raised at SoC < 10%, cleared at > 12% |
|     |   |   | [04] "00" 1 = full charge request: a module stayed below 97% SoC for too long, cleared once all modules reached 97% |
|     |   |   | [05] "00" bitmask of modules blocking charging (bit 0 = module 1, only modules 1-8). Bit 0 is also set if the charge current limit is 0 |
|     |   |   | [06] "00" bitmask of modules blocking discharging, same as [05] |
| 409 | A0 86 01 00 50 C3 00 00 | ....P... | **Capacity** |
|     |   |   | [00:03] "A0 86 01 00" (mAh) 100000 mAh (100.0Ah) full charge capacity (sum of all modules) |
|     |   |   | [04:07] "50 C3 00 00" (mAh) 50000 mAh (50.0Ah) remaining capacity (sum of all modules); SoC = remaining / full |
| 40A | 50 59 54 45 53 | PYTES | **Manufacturer**: "PYTES" (DLC 5) |
| 40B | — | — | **Clock and module count** |
|     |   |   | [00:05] BMS clock: year (20YY), month, day, hour, minute, second |
|     |   |   | [06] modules online |
|     |   |   | [07] modules offline (highest module count seen minus online) |
| 40C | — | — | **Per-module status flags** (bit 0 = module 1), meaning not verified |
|     |   |   | [00:03] modules with status flag 1 set (probably charge enabled), see Victron `0x372` |
|     |   |   | [04:07] modules with status flag 2 set (probably discharge enabled) |
| 40D | — | — | **Heating and balancing** |
|     |   |   | [00:03] bitmask of modules heating (bit 0 = module 1). A module heats while its lowest cell temperature is below a threshold (with hysteresis) |
|     |   |   | [04:07] bitmask of cells being balanced (bit 0 = cell 1, only 16 bits used), OR over all modules, no module information. A cell is balanced if it is >= 30 mV above the lowest cell of its module and >= 3360 mV, while the module is charging or idle. Observed: `0x86E1` = `0x84E1` (module 1) OR `0x0240` (module 2) via RS485 |
| 40E-415 | — | — | **Protection thresholds, set 1**, see [below](#0x40e-0x41d-protection-thresholds) |
| 416-41D | — | — | **Protection thresholds, set 2** |
| 41E | 40 08 00 00 2B 07 00 00 | @...+... | **Energy** (same as Victron `0x378`) |
|     |   |   | [00:03] "40 08 00 00" (kWh/10) 211.2kWh charged energy |
|     |   |   | [04:07] "2B 07 00 00" (kWh/10) 183.5kWh discharged energy |
| 500+ | — | — | **Per-module cell data**, only on request (module 1 = `0x500`-`0x508`, +`0x10` per further module), see [below](#per-module-cell-data-on-request) |

### 0x403 / 0x406 bit fields

Names in quotes are the labels the firmware uses in its debug output.

**`0x403` bytes 0-3** (protection word; bytes 4-7 use the same layout):

| Bit | Meaning |
| --- | --- |
| 0 | cell/pack over-voltage protection |
| 1 | high voltage warning |
| 3 | low voltage warning |
| 4 | cell/pack under-voltage protection |
| 5 | "Sleep" |
| 8 | over-temperature protection |
| 9 | high temperature warning |
| 11 | low temperature warning |
| 12 | under-temperature protection |
| 16 | short circuit protection, "SC" |
| 17 | discharge over-current, "DOC2" |
| 18 | discharge over-current, "DOC" |
| 19 | charge over-current, "COC2" |
| 20 | charge over-current, "COC" |
| 21 | discharge current warning, "DOCAlm" |
| 22 | charge current warning, "COCAlm" |
| 24 | discharge over-current, "DOC1" |
| 25 | state: idle (\|current\| <= 1 A) |
| 26 | state: charging (current > 1 A) |
| 27 | state: discharging (current < -1 A) |
| 28 | "SysError" |
| 29 | charge over-current, "COC1" |

**`0x406` bytes 0-3** (failure word):

| Bit | Meaning |
| --- | --- |
| 2 | current sensor check failed |
| 9 | module communication timeout (no answer from a module for 60 polls). The firmware reports this as "internal failure" in the Victron and Pylon protocols |
| 12 | SRAM check failed |
| 13 | flash check failed |
| 14 | PC / register check failed |
| 15 | coding check failed |
| 17, 18, 19, 24 | set by the firmware, meaning not identified |
| 20 | temperature check failed, or cell temperatures of a module differ by more than 10 °C |

### 0x40E-0x41D Protection thresholds

Two sets of 8 frames with the same layout: set 1 at `0x40E`-`0x415`, set 2 at `0x416`-`0x41D`. Which set is which (e.g. alarm vs. protection, or charge vs. discharge) is not known yet. `p[n]` is the n-th value of the internal threshold table.

| Frame | DLC | Content |
| --- | --- | --- |
| +0 | 8 | (mV) p[0], p[1], p[2], p[3]: cell over-voltage trip / release, alarm / release. Defaults 3650 / 3500, 3630 / 3500 |
| +1 | 8 | (mV) p[6], p[7], p[4], p[5]: cell under-voltage trip / release, alarm / release. Defaults 2800 / 2900, 2900 / 3000 |
| +2 | 8 | (/100) p[9], p[10], p[11], p[12]: pack voltage thresholds (e.g. 570 = 57.0 V) |
| +3 | 8 | (/100) p[15], p[16], p[13], p[14] |
| +4 | 8 | (/100) p[17], p[18], p[19], p[20] |
| +5 | 8 | (/100) p[23], p[24], p[21], p[22] |
| +6 | 6 | (/100) p[25], p[26], p[27] |
| +7 | 6 | (/100) p[28], p[29], p[30] |

Frames +3 to +7 contain current and temperature thresholds; the exact assignment has not been mapped.

### Per-module cell data (on request)

> **Firmware required** (see [Firmware versions](#firmware-versions)): 48100V `>= V1.10.26`, 48100R `>= V1.5.38` and only on the **C16** flash chip. Older/`C8` firmware accepts `0x307` but never answers. A battery that reports its frames with extended identifiers also expects the request as an extended frame.

Request:

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 307 | 01 01 | .. | request cell data of a module. byte 0 = group + 1 (1 without groups), byte 1 = module number (1-based). Example: group 1, module 1 → `01 01` |

Answer (once, in the next cycle): 9 frames. **Module 1 uses `0x500`-`0x508`**; each further module shifts up by `0x10` (module 2 = `0x510`-`0x518`, module 3 = `0x520`-`0x528`, ...), plus the group offset. Shown for module 1:

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 500 | 10 00 00 00 00 00 00 00 | ........ | [00:01] number of cells of the module (16) |
|  |  |  | [02:07] unused |
| 501 | D9 0C D9 0C D9 0C D9 0C | ........ | (mV) cells 1-4, one u16 each (3.289V each here) |
| 502 | DA 0C D9 0C D9 0C D9 0C | ........ | (mV) cells 5-8 (cell 5 = 3.290V, the pack maximum) |
| 503 | D8 0C D9 0C D9 0C D9 0C | ........ | (mV) cells 9-12 (cell 9 = 3.288V, the pack minimum) |
| 504 | D9 0C D9 0C D9 0C D9 0C | ........ | (mV) cells 13-16 |
| 505 | AA 00 AA 00 A0 00 B4 00 | ........ | (°C/10, signed) cells 1-4 (cell 3 = 16.0°C min, cell 4 = 18.0°C max) |
| 506 | AA 00 AA 00 AA 00 AA 00 | ........ | (°C/10, signed) cells 5-8 (17.0°C each here) |
| 507 | AA 00 AA 00 AA 00 AA 00 | ........ | (°C/10, signed) cells 9-12 |
| 508 | AA 00 AA 00 AA 00 AA 00 | ........ | (°C/10, signed) cells 13-16 |

Only one request is handled at a time. Frames for cells a module does not have contain no meaningful data.

### Per-module firmware version (on request)

> **Firmware required:** same as the per-module cell data above (see [Firmware versions](#firmware-versions)).

Uses 29 bit extended identifiers. Request:

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 19A60 | 01 00 | .. | request firmware version of a module. the module number as a 16-bit little-endian value (1-based): module 1 = `0x0001` → bytes `01 00` |

Answer (nothing is sent for an invalid module number):

| Id  | Hex | Ascii | Description |
| --- | ----------------------- | -------- | --- |
| 19A61 | 12 00 — — 01 00 00 00 | ........ | [00:01] length of the version string (18) |
| | | | [02:03] CRC16 of the string (battery-specific, not derivable) |
| | | | [04:05] module number (1-based, little-endian) |
| 19A62 | 53 50 42 4D 53 31 36 53 | SPBMS16S | version string bytes 0-7 |
| 19A63 | 52 50 56 31 2E 31 30 2E | RPV1.10. | bytes 8-15 |
| 19A64 | 32 38 00 00 00 00 00 00 | 28...... | bytes 16-23 (string is `SPBMS16SRPV1.10.28`, 18 bytes, zero padded) |
| 19A65 | 00 00 00 00 00 00 00 00 | ........ | bytes 24-31 (zero padding) |

The identifiers `0x19A10`-`0x19A50` (extended) are used for firmware updates over CAN and are not documented here.

---

### How it looks like in Venus OS

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/c56bf0a1-7ccf-40f4-95c8-f68b0b0bf4fe)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/9696afd2-70c3-4fda-bd3b-fa6a4c7729f2)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/c2f9ffeb-64d4-4fd5-aef6-772cb88352a4)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/19f90cb4-7bab-4eff-af18-c545bc6fae49)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/2838e9da-771d-49ac-8c82-bce841b40821)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/22e2e31e-83d9-4480-aae5-5cd0cdc17085)

![image](https://github.com/AndreasBoehm/PytesCANProtocol/assets/1270749/02e2e040-a7a8-4c99-b8ad-1f65d4425f46)
