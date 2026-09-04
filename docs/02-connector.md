# 2. Connector Pinout

OpenFarmRack-compliant modules use a standardized 5-pin modular connector:

| Pin | Name | Level | Description                         |
| --- | ---- | ----- | ------------------------------------ |
| 1   | 12V  | Power | Actuator / system power supply       |
| 2   | GND  | -     | Common Ground                        |
| 3   | SDA  | 5V    | I2C Serial Data                      |
| 4   | SCL  | 5V    | I2C Serial Clock                     |
| 5   | EN   | 5V    | Daisy-chain Address Enable           |

Note: there is no dedicated 5V pin. Each module regulates its own 5V logic
supply locally from the 12V rail (see [Electrical Specification](01-electrical.md)).

## Mechanical topology: two connectors per module

OpenFarmRack uses a **vertical stacking** physical form factor. Each module has
**two physical connectors**, not one:

- **Bottom connector** — carries `EN_IN` (input side of the enable chain).
- **Top connector** — carries `EN_OUT` (output side of the enable chain).

Power (`12V`, `GND`) and the I2C bus (`SDA`, `SCL`) are shared straight
through both connectors — every module sits on the same physical bus.
`EN` is the only signal whose meaning differs between the bottom and top
connector of a given module.

This has a direct physical consequence for addressing and fault handling:
because modules are physically stacked, a module can only be removed if
everything above it in the stack is removed first, and — because power and
the I2C bus run straight through the connectors — removing a module
physically disconnects everything stacked above it, not just its `EN`
signal. See [Enumeration & Addressing](03-enumeration.md) for how this
affects hot-plug behavior.
