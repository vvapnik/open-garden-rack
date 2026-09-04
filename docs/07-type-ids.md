# 7. Standard Type IDs

> Note: the v0.3 draft's table had a formatting error — the Moisture row
> was missing its ID (`0x40`). Fixed below.

## Sensors (0x40 – 0x7F)

| ID | Name | Format | Description |
| --- | ---- | ------ | ----------- |
| 0x40 | Moisture | `uint16` | Raw ADC / capacitance reading (0–65535). Master maps 0–100% via dry/wet calibration points. |
| 0x41 | Temp | `int16` | `0.1 °C` |
| 0x42 | Humidity | `uint8` | `1 %` |
| 0x43 | Light | `uint32` | `1 Lux` or Raw |
| 0x44 | Weight | `uint32` | Raw / `1 g` |
| 0x45 | Distance | `uint16` | `1 mm` |
| 0x46 | Water Level | `uint16` | `1 %` |

## Actuators (0x80 – 0xBF)

| ID | Name | Format | Description |
| --- | ---- | ------ | ----------- |
| 0x80 | Relay | `0 / 1` | Generic binary switch (Off / On) |
| 0x81 | Dimmer | `0-255` | Generic PWM intensity control |
| 0x82 | Water Valve | `0 / 1` | Solenoid / motorized valve control (0: Closed, 1: Open) |
| 0x83 | Light Switch | `0 / 1` | Dedicated lighting binary toggle (0: Off, 1: On) |
| 0x84 | Water Pump | `0-255` | Variable speed pump intensity (0: Off, 255: Max Speed) |
| 0x85 | Vibro | `0-255` | Vibration motor intensity / frequency |

## Widgets & External Devices (0xC0 – 0xFF)

| ID | Name | Format | Description |
| --- | ---- | ------ | ----------- |
| 0xC0 | Camera | `uint8` | External streaming module flag. Communicates out-of-band (0: Offline, 1: Active). Transport for the out-of-band stream itself (Wi-Fi / BLE / other) is outside the scope of this I2C spec. |

All formats above map directly onto the `EP_SELECT` data window described
in [Register Map §4.2](04-register-map.md#42-endpoint-access--ep_select-paging);
values wider than the endpoint's natural size occupy the low-order bytes
of the 4-byte window.
