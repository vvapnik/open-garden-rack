# OpenGardenRack Protocol Specification

**Version:** 0.1 (WIP)
**Status:** MVP Standard — actively evolving
**Architecture:** Master–Slave (I2C), vertical daisy-chain
**Target use case:** indoor vertical home garden systems
**License:** Open Source

OpenFarmRack Protocol is a hardware/software specification for interconnecting
modular gardening electronics (sensors, actuators, plant-anchor modules) over
a shared I2C bus, with automatic address assignment based on physical
stacking order.

## Reading the spec

The specification is split into focused documents so each part can evolve
and be reviewed independently:

| # | Document | Covers |
|---|----------|--------|
| 1 | [Electrical Specification](docs/01-electrical.md) | Voltage rails, logic levels, bus speed |
| 2 | [Connector Pinout](docs/02-connector.md) | 5-pin modular connector, mechanical topology |
| 3 | [Enumeration & Addressing](docs/03-enumeration.md) | Daisy-chain boot sequence, hot-plug, fault handling, restart |
| 4 | [Register Map](docs/04-register-map.md) | System block, `EP_SELECT` paging, descriptors, data window |
| 5 | [Frame Format](docs/05-frame-format.md) | Byte order, transaction structure, CRC/PEC integrity, retry policy |
| 6 | [Standard CLASS IDs](docs/06-class-ids.md) | Module class identifiers |
| 7 | [Standard Type IDs](docs/07-type-ids.md) | Endpoint type/format identifiers |
| 8 | [Reference Implementations](docs/08-libraries.md) | Public libraries implementing this spec |

A zoomable map of the spec and the libraries around it is in
[`ARCHITECTURE.takesix.json`](ARCHITECTURE.takesix.json) (a TakeSix
diagram).

## Contributing

Issues and PRs should target the specific `docs/*.md` file(s) they affect.
Any change to enumeration behavior, register layout, or addressing must be
accompanied by an ADR in `decisions/` explaining the rationale and any
alternatives considered.
