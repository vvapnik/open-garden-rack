# 5. Frame Format

## 5.1 Byte Order

All multi-byte values (`PLANT_UID`, `BP_UID`, and the `EP_SELECT` data
window) are transmitted **little-endian** (least significant byte first),
matching the native representation on the AVR/STM32/ESP-class MCUs this
protocol targets — no byte-swapping required in firmware on either side.

## 5.2 Transaction Structure

Registers are accessed with the standard "register address, then data"
I2C pattern.

**Write:**

```
[START] [NODE_ADDR+W] [REG_ADDR] [DATA_0] [DATA_1] ... [PEC] [STOP]
```

**Read** (register address set first, then a repeated start for the read
phase):

```
[START] [NODE_ADDR+W] [REG_ADDR] [REPEATED START] [NODE_ADDR+R] [DATA_0] [DATA_1] ... [PEC] [STOP]
```

`DATA_N` count depends on the target register (1 byte for most System
Block registers, up to 4 bytes for the `EP_SELECT` data window).

## 5.3 Data Integrity — CRC/PEC

Every transaction — read and write alike — carries one trailing **PEC**
(Packet Error Check) byte: **CRC-8/SMBUS** (polynomial `0x07`, initial
value `0x00`, no input/output reflection, no final XOR).

The PEC is computed over every byte that physically appears on the bus
during the transaction, in order, **excluding the PEC byte itself and all
ACK/NACK bits**:

- Write: `NODE_ADDR+W`, `REG_ADDR`, `DATA_0..DATA_N`
- Read: `NODE_ADDR+W`, `REG_ADDR`, `NODE_ADDR+R`, `DATA_0..DATA_N`
  (both address-phase bytes are included, matching SMBus PEC convention)

### Write validation (node-side)

The node buffers the incoming write and computes CRC-8/SMBUS over
`NODE_ADDR+W, REG_ADDR, DATA_0..DATA_N` as the bytes arrive, the same way
the master does. Every byte is **ACKed as normal** — the node never
signals a CRC failure via the ACK/NACK bit. This is a deliberate
portability choice: it keeps the node implementable on top of a plain
buffered receive callback (the common shape of I2C slave APIs/drivers
across the AVR/RISC-V/Xtensa target range), with no dependency on a
driver exposing per-byte ACK/NACK control.

Once the PEC byte has been received, the node compares it against its own
computed value:

- **Match:** the write is applied.
- **Mismatch:** the write is discarded (registers retain their previous
  values) and `STATUS` is set to `1 (Error)` — the same mechanism already
  used for an invalid `EP_SELECT` write (see
  [Register Map §4.2](04-register-map.md#42-endpoint-access--ep_select-paging)).

Because the node gives no in-band signal at write time, the master does
not learn about a failed write immediately. It finds out the way it finds
out about any other node-side fault: `STATUS` reads `Error` next time it's
checked — during the routine
[health-check cycle](03-enumeration.md#33-bus-scanning-cycles), or right
away if the master chooses to read `STATUS` (or read back the register it
just wrote) immediately after a write it considers critical (e.g.
`SET_ADDR`, `RESTART`, `PLANT_UID`). That verification is a master-side
policy choice — the protocol does not require it for every write.

### Read validation (master-side)

The master cannot signal a data fault via ACK/NACK on a read (the
master's ACK/NACK during a read only tells the node whether to keep
sending bytes, per the I2C spec — it carries no data-validity meaning).
Instead, the master independently computes the expected PEC over
everything it saw on the bus and compares it to the received PEC byte
after the transaction completes:

- **Match:** the read data is accepted.
- **Mismatch:** the master discards the result and retries the entire
  transaction from a fresh `START` (see §5.4).

## 5.4 Retry Policy

Retry behavior differs between reads and writes, because only the master
ever gets an in-transaction signal of a PEC failure (§5.3).

**Reads:** on a mismatched PEC, the master retries the **entire
transaction** (not just the failed byte) from a fresh `START`, up to **3
attempts total**. If all 3 attempts fail, the transaction is abandoned and
counts as a failed attempt against that module in the
[health-check cycle](03-enumeration.md#33-bus-scanning-cycles) — 3
consecutive fully-failed transactions (i.e. `3×3 = 9` raw attempts) push
the module to `PROBLEM` state under the existing state model, exactly as a
response timeout would.

**Writes:** there is no immediate retry loop — the node doesn't NACK a bad
PEC (§5.3), so the master has nothing to react to at write time. A write
that fails its PEC check is silently discarded by the node and surfaces as
`STATUS = Error` on the next read of `STATUS`, feeding the same `PROBLEM`
classification as any other node-side fault. A master wanting a stronger
delivery guarantee for a specific write is free to read back and re-issue
it itself; the protocol does not require this for every write.

The 3-attempt read retry count is deliberately small: on a
properly-terminated bus within the specified
[electrical limits](01-electrical.md#bus-length--node-count-limits), CRC
failures should be rare (transient noise), not systemic — a module that
keeps failing after 3 clean retries has a real fault worth surfacing as
`PROBLEM` rather than one worth retrying indefinitely. No separate
error-counting mechanism is introduced anywhere in this section — CRC
failures feed the same state machine as any other communication fault.

## 5.5 Example

Writing `0x01` (Soft Restart) to `RESTART` (`0x06`) on a module at address
`0x2A`:

```
Bytes on wire (excluding START/STOP/ACK bits):
  0x54  (0x2A << 1 | W)   -- NODE_ADDR+W
  0x06                    -- REG_ADDR (RESTART)
  0x01                    -- DATA (soft restart)
  0xNN                    -- PEC = CRC8-SMBUS(0x54, 0x06, 0x01)
```

If the node's own computed CRC over `{0x54, 0x06, 0x01}` doesn't match
`0xNN`, it ACKs the PEC byte as normal but discards the write and sets
`STATUS = 1 (Error)` — the restart is **not** applied. The master only
finds out if/when it reads `STATUS` (§5.3).
