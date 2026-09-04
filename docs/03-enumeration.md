# 3. Enumeration & Addressing (Daisy-Chain)

OpenFarmRack uses a physical "pass-through" mechanism to handle module positioning
and dynamic addressing. Positioning in the physical stack and order of
registration on the bus are the same thing — the bus address itself carries
no positional information, so the enumeration order is the **only** source
of truth for where a module physically sits in the chain. This has
consequences throughout this document, particularly for hot-plug recovery.

## 3.1 Initial Boot Sequence

1. **Boot State:** All Slave modules pull their `EN_OUT` to LOW.
2. **Activation:** The Master asserts HIGH on the `EN` line feeding the
   bottom connector of the first module.
3. **Registration:**
   - The module detecting HIGH on its `EN_IN` responds to the **Default
     Address (`0x08`)**.
   - Master reads `CLASS` and assigns a unique 7-bit I2C address by
     writing to the `SET_ADDR` register.
   - Upon successful address assignment, the module pulls its `EN_OUT`
     (feeding the connector above it) to HIGH.
4. **Chain Reaction:** The HIGH signal from module *N*'s top connector
   enables `EN_IN` on module *N+1*'s bottom connector.
5. **Termination:** The Master stops when no device responds to address
   `0x08`.

## 3.2 Module State Model

The master tracks each module in one of four states:

| State | How it's detected | Master action |
|---|---|---|
| `UNASSIGNED` | Responds on `0x08`, no address assigned yet | Runs registration (3.1) |
| `OK` | Responds on its assigned address, `STATUS = 0` | Polled normally |
| `PROBLEM` | Responds (ACK) on its assigned address but `STATUS ≠ 0`, or fails to respond within `N` consecutive attempts within timeout `T` | Kept in the module table; recovery action (e.g. `RESTART`) is left to master's discretion |
| `REMOVED` | NACK on the address phase for `K` consecutive attempts | Removed from the module table; address freed |

The distinction between `PROBLEM` and `REMOVED` is exactly the I2C-level
distinction between "ACK received, but something is wrong with the
content/response" and "NACK on the address itself — device is not
electrically present on the bus."

## 3.3 Bus Scanning Cycles

The master runs three independent, differently-scoped scanning cycles.
Together they cover the full module lifecycle (join, run, degrade,
physically removed, physically restored) without special-casing any of
those transitions in master logic.

### Health-check cycle

Runs frequently, against all **currently registered** addresses.

- Read `STATUS` (or another lightweight register) from each known address.
- NACK, `K` times in a row → mark `REMOVED`, free the address.
- ACK but `STATUS = Error`, or response timeout despite ACK → mark
  `PROBLEM`, keep in the table. Exhausted CRC/PEC retries (see
  [Frame Format §5.4](05-frame-format.md#54-retry-policy)) are treated the
  same as a timeout here — no separate error path.

### Scan-for-new cycle

Runs frequently, checking **only** the default address `0x08`.

- Anything answering on `0x08` is a genuinely new hot-plugged
  module  `UNASSIGNED` (e.g. due to a cascading restart — see 3.5).

### Full-range scan cycle

Runs less frequently (more expensive — probes the whole 7-bit address
space, minus `0x08` and the reserved `0x78–0x7F` block), checking for
**orphaned modules**: devices that already hold an assigned address but are
absent from the master's table. This happens when:

- A module was previously marked `REMOVED` but is actually still present
  and responsive (e.g. it was transiently unreachable).
- The master itself restarted (e.g. after a power loss) while slave
  modules stayed powered long enough (backed by local capacitors) to
  retain their assigned addresses — the master's table is empty, but the
  bus already has fully-addressed modules on it.

This cycle **must always run once, unconditionally, on master startup /
after master power is restored** — this is exactly the scenario where the
master's table is guaranteed to be stale or empty.

Finding an orphaned address does **not** mean adopting it directly (see
3.5) — an address with no known physical position is not useful to a
protocol whose whole addressing model is built on physical stacking order.

## 3.4 `RESTART` semantics

Writing to `RESTART` (soft or hard) instructs a module to forget its
assigned address and re-enter the boot sequence (3.1) from scratch: pull
`EN_OUT` back to LOW, and begin listening on `EN_IN` again.

Because `EN_OUT` of a restarted module is the same signal that holds
`EN_IN` HIGH for whatever is stacked immediately above it, **restarting any
module that is not at the very top of the physical stack necessarily drops
everything stacked above it back to `UNASSIGNED`.** This is not a fault
condition — it is an unavoidable, documented consequence of the daisy-chain
enable mechanism.
This is intentionally **not** treated as a special case by the master:
modules that cascade back to `UNASSIGNED` simply reappear on `0x08` and are
picked up by the ordinary scan-for-new cycle. `RESTART` is, from the
master's point of view, the software equivalent of physically unplugging
and replugging a module.

## 3.5 Recovering an Orphaned Module

When the full-range scan (3.3) finds an address `X` not present in the
master's table, the module at `X` is **not** contacted or adopted directly
at that address. Instead:

1. The master begins issuing `RESTART` to modules already in its table,
   **starting from the topmost (most recently registered) known module**
   and working downward.
2. After each `RESTART`, the master re-checks whether `X` still responds.
3. Because power and the I2C bus run straight through the stack (see
   [Connector Pinout](02-connector.md)), restarting a module drops
   `EN_IN` for everything physically above it — including `X`, once the
   master has restarted the module immediately below `X`'s actual
   position.
4. As soon as `X` stops responding at its old address, the master **stops**
   issuing further restarts — this is the signal that `X` has fallen back
   to `UNASSIGNED`. Restarting modules further down the stack is
   unnecessary and should be avoided.
5. Standard registration (3.1) then proceeds normally. `X` (and any other
   module that was cascaded back to `UNASSIGNED` along the way) re-joins
   the chain at its correct physical position, because registration order
   is, by construction, physical order.

In-place "adoption" of an orphaned address (reading its registers directly
and inserting it into the table without forcing it through the EN cascade)
was considered and explicitly rejected — see
[ADR-0004](../decisions/0004-orphan-module-recovery.md) for the rationale.

## 3.6 Known limitation

A module that is fully hung (firmware crashed, not merely
`STATUS = Error`) may hold the bus in a state indistinguishable from
`REMOVED` — a `RESTART` command cannot reach a device that isn't
responding to its address at all.
