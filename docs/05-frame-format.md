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

### Write validation (node-side, immediate NACK)

The node computes the running CRC as each byte arrives. On receiving the
final (PEC) byte, it immediately compares its own computed value against
the received PEC:

- **Match:** ACK the PEC byte as normal. The write is applied.
- **Mismatch:** **NACK the PEC byte** instead of ACKing it, and discard
  the entire write (registers retain their previous values). This gives
  the master immediate, in-transaction feedback — no follow-up status
  read is needed to discover a failed write.

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

On a detected integrity failure — a NACKed PEC on write, or a mismatched
PEC on read — the master retries the **entire transaction** (not just the
failed byte) from a fresh `START`, up to **3 attempts total**.

If all 3 attempts fail:

- The transaction is abandoned.
- This counts as a failed attempt against that module in the
  [health-check cycle](03-enumeration.md#33-bus-scanning-cycles) — 3
  consecutive fully-failed transactions (i.e. `3×3 = 9` raw attempts) push
  the module to `PROBLEM` state under the existing state model, exactly
  as a response timeout would. No separate error-counting mechanism is
  introduced — CRC failures feed the same state machine as any other
  communication fault.

This retry count is deliberately small: on a properly-terminated bus
within the specified [electrical limits](01-electrical.md#bus-length--node-count-limits),
CRC failures should be rare (transient noise), not systemic — a module
that keeps failing after 3 clean retries has a real fault worth surfacing
as `PROBLEM` rather than one worth retrying indefinitely.

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
`0xNN`, it NACKs the PEC byte and the restart is **not** applied — the
master sees the NACK and retries per §5.4.
