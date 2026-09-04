# 4. Register Map

All communication is performed by reading or writing to 8-bit registers.

## 4.1 System Block (0x00 – 0x1F)

| Reg | Access | Name | Format | Description |
| --- | ------ | ---- | ------ | ----------- |
| 0x00 | R | `PROT_VER` | `uint8` | Protocol version (0x10 = v1.0) |
| 0x01 | R | `STATUS` | `uint8` | Device status (0: OK, 1: Error, 2: Busy) |
| 0x02 | R | `CLASS` | `uint8` | Device Class (see [Standard CLASS IDs](06-class-ids.md)) |
| 0x03 | R | `EP_COUNT` | `uint8` | Total number of functional endpoints |
| 0x04 | R | `SYS_FLAGS` | `uint8` | System Flags (Bit 0: `IS_ANCHOR`, Bit 1: `HAS_BLUEPRINT`) |
| 0x05 | W | `SET_ADDR` | `uint8` | Set assigned 7-bit I2C address |
| 0x06 | W | `RESTART` | `uint8` | Force module restart (0x01: Soft, 0x02: Hard) — see [Enumeration §3.4](03-enumeration.md#34-restart-semantics) |
| 0x07 | R/W | `PLANT_UID_0` | `uint8` | Plant UID Byte 0 (LSB) |
| 0x08 | R/W | `PLANT_UID_1` | `uint8` | Plant UID Byte 1 |
| 0x09 | R/W | `PLANT_UID_2` | `uint8` | Plant UID Byte 2 |
| 0x0A | R/W | `PLANT_UID_3` | `uint8` | Plant UID Byte 3 (MSB) |
| 0x0B | R | `BP_UID_0` | `uint8` | Blueprint UID Byte 0 (LSB) |
| 0x0C | R | `BP_UID_1` | `uint8` | Blueprint UID Byte 1 |
| 0x0D | R | `BP_UID_2` | `uint8` | Blueprint UID Byte 2 |
| 0x0E | R | `BP_UID_3` | `uint8` | Blueprint UID Byte 3 (MSB) |
| 0x0F | R/W | `EP_SELECT` | `uint8` | Index of the currently selected endpoint (0-based). See §4.2. |
| 0x10..0x1F | - | `RESERVED` | - | Reserved for future system extensions (16 bytes) |

### System Flags (`SYS_FLAGS`, 0x04)

- **Bit 0 (`IS_ANCHOR`):**
  - `0`: Non-anchor module (e.g. lighting, pump, sensors).
  - `1`: Anchor module tied directly to a physical plant position (e.g.
    container, pot, hydroponic slot).
- **Bit 1 (`HAS_BLUEPRINT`):**
  - `0`: Generic device — endpoint descriptors are read dynamically via
    `EP_SELECT` (§4.2).
  - `1`: Blueprint UID supported (`0x0B`–`0x0E`).
- **Bits 2–7:** Reserved for future use.

### Plant Unique Identifier (`PLANT_UID`, 0x07 – 0x0A)

Holds a 32-bit (`uint32`) ID of the plant instance associated with the
module.

- **Sentinel Value (`NO_UID`):** `0xFFFFFFFF` indicates no plant assigned
  or unsupported feature.

**Anchor modules (`IS_ANCHOR == 1`):** mandatory support. MUST store and
persist `PLANT_UID` in non-volatile memory (NVS/EEPROM).

- Read returns the stored 32-bit Plant UID (or `0xFFFFFFFF` if
  unassigned).
- Write persists the new Plant UID into NVS/EEPROM.

**Non-anchor modules (`IS_ANCHOR == 0`):** optional support.

- Modules MAY store and return `PLANT_UID` if useful/available.
- Modules that do NOT support UID storage MUST return `0xFFFFFFFF`
  (`NO_UID`) on reads and silently ignore writes (No-Op).

### Blueprint Unique Identifier (`BP_UID`, 0x0B – 0x0E)

Holds a 32-bit (`uint32`) ID of the module's blueprint template.

- **Sentinel Value (`NO_BLUEPRINT`):** `0x00000000` or `0xFFFFFFFF`
  indicates a generic module without an assigned blueprint (requires
  dynamic descriptor parsing via `EP_SELECT`).

## 4.2 Endpoint Access — `EP_SELECT` Paging

### `EP_SELECT` (0x0F)

- **Indexing:** 0-based (the first endpoint is index `0x00`).
- **Default value:** `0x00`, both immediately after address assignment and
  after any `RESTART`. A master can therefore read endpoint 0's data
  immediately after enumeration without an explicit select.
- **Invalid index:** if a value ≥ `EP_COUNT` is written, the module MUST
  ignore the write (No-Op — the previous `EP_SELECT` value is retained)
  and set `STATUS = 1 (Error)`. This is deliberately not a silent clamp:
  the master must be able to detect the mistake via `STATUS` rather than
  silently reading the wrong endpoint.
- Selecting an endpoint is "sticky": repeated reads/writes to the data
  window (0x40–0x43) without an intervening write to `EP_SELECT` all
  target the same endpoint. This lets a master poll one endpoint
  repeatedly (e.g. a monitoring loop) without paying the cost of a select
  on every read.
- `EP_SELECT` MUST NOT be changed by a module mid-transaction — a
  multi-byte read/write of the data window must complete before the
  selected endpoint changes.

### Capability Descriptor window (0x20 – 0x22)

Describes the **currently selected** endpoint (per `EP_SELECT`) as a
3-byte tuple:

- **Byte 0 (0x20):** Type ID (see [Standard Type IDs](07-type-ids.md)).
- **Byte 1 (0x21):** Mode (0x01: Read, 0x02: Write, 0x03: R/W).
- **Byte 2 (0x22):** Data Format (0x01: Binary, 0x05: Percent, 0x0A: Raw16).

### Data window (0x40 – 0x43)

Holds the value of the **currently selected** endpoint, 4 bytes wide. This
width was chosen specifically to fit `uint32` sensor types (e.g. Weight,
Light) without a special case. Narrower formats use the low-order bytes;
unused high-order bytes read as `0x00` and MUST be ignored on write.

`0x23–0x3F` and `0x44–0xBF`, previously reserved for additional
direct-addressed descriptors/data under the old fixed-address model, are
now freed and reserved for future use.

### Example transaction sequence

To read endpoint index 5:

```
1. I2C Write: [NODE_ADDR] [REG=0x0F] [VALUE=5] [PEC]   // select EP 5
2. I2C Read:  [NODE_ADDR] [REG=0x40] -> N bytes [PEC]  // read EP 5's data
```

Reading the same endpoint again immediately afterward does not require
repeating step 1. Every transaction here carries a trailing PEC byte per
[Frame Format §5.3](05-frame-format.md#53-data-integrity--crcpec).

## 4.3 Frame format & data integrity

The exact byte-level structure of a full I2C request/response, including
byte order and the CRC/PEC integrity mechanism, is specified in
[Frame Format](05-frame-format.md).
