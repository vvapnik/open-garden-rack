# 1. Electrical Specification

To ensure stability, noise immunity, and compatibility with modern
microcontrollers, OpenFarmRack utilizes a dual-voltage rail system.

- **Logic Level:** 5V DC. All `SDA`, `SCL`, and `EN` lines must operate at
  5V logic.
- **Logic Power:** 5V DC, used to power microcontrollers and sensors.
  **This rail is not carried on the connector** — each module derives 5V
  locally from the 12V System Power rail via an on-board regulator. See
  [Connector Pinout](02-connector.md) for the resulting pin list.
- **System Power:** 12V DC. Dedicated high-current rail for actuators
  (pumps, fans, LED arrays).
- **Common Ground:** All modules MUST share a common GND reference.
- **Bus Speed:** Fixed at 100 kbps (Standard Mode) for maximum reliability
  over physical connectors.

## Forward Compatibility Note

OpenFarmRack targets indoor vertical garden installations. A realistic
maximum installation height is **3 meters** — taller is rarely practical
to access, water, or maintain indoors regardless of ceiling height. All
bus limits below are derived for a single physical chain spanning up to
3m.

### Capacitance budget

I2C Standard-mode (100 kbps) limits total bus capacitance to **400 pF**
(`Cb_max`). Exceeding it degrades signal rise time below spec and causes
unreliable communication — this is a hard physical constraint, not a
tunable parameter.

Assumptions (validate against the actual selected cable/connector
datasheet once finalized — figures below are representative, not
guaranteed):

| Contributor | Assumed value |
|---|---|
| Cable capacitance (SDA/SCL pair) | ~60 pF/m |
| Per-module stub (MCU pin + trace + connector) | ~15 pF |
| Master's own bus capacitance | ~10 pF |

At the maximum 3m run:

| Quantity | Value |
|---|---|
| Cable capacitance over 3m | 180 pF |
| Remaining budget (400 − 180 − 10) | 210 pF |
| **Max modules by capacitance alone** | **~14** |


### Specified limit

**Maximum 12 modules per physical chain**, in addition to the 3m maximum
height. Twelve was chosen because it stays safely under the capacitance
budget even at the representative 60 pF/m cable estimate (370 pF total,
comfortably under 400 pF with margin for cable/connector variance), while
still accommodating the common case of 20–25 cm module spacing at full 3m
height. Installations wanting a denser or taller chain must split it
across a bus buffer/repeater (see Best Practices) — this limit applies per
electrical segment, not per installation.

### Pull-up resistor sizing

For 5V logic, `Rp_min ≈ 1.5 kΩ` (from `Iol_max = 3 mA`, `Vol_max = 0.4V`).
`Rp_max` depends on actual bus capacitance, which varies with populated
module count:

| Modules | Bus capacitance | `Rp_max` |
|---|---|---|
| 10 | 340 pF | 3.47 kΩ |
| 12 | 370 pF | 3.19 kΩ |

**Recommended pull-up: 2.2 kΩ**, placed only at the master (see Best
Practices) — this sits comfortably within range across the full 0–12
module count without needing to be re-selected as the chain is populated.

