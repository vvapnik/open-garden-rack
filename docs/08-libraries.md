# 8. Reference Implementations

Public libraries implementing this spec. All of them are independent
repositories (not part of this spec repo) — this page just indexes them.

## ⚠️ AI-Generated

> **These libraries were written by an AI coding assistant.** Each one has
> been exercised against real hardware (enumeration, endpoint read/write,
> hot-plug/restart behavior) and the specific bugs found that way are
> documented in their own commit history — but **none of this code has
> gone through a full human review** yet. Treat it as a working reference
> implementation, not an audited one: read it before you trust it in
> anything that matters, and **use it at your own risk**.

| Library | Role | Target(s) | Repository |
|---|---|---|---|
| `ogr-node-core` | Node protocol engine (platform-independent) | Any (used by the Node HAL packages below) | [github.com/vvapnik/ogr-node-core](https://github.com/vvapnik/ogr-node-core) |
| `ogr-node-hal-avr` | Node HAL | ATmega328-class (hardware TWI), ATtiny85-class (USI) | [github.com/vvapnik/ogr-node-hal-avr](https://github.com/vvapnik/ogr-node-hal-avr) |
| `ogr-node-hal-esp32-arduino` | Node HAL | ESP32 / ESP32-S3 / ESP32-C3, on Arduino's `Wire` | [github.com/vvapnik/ogr-node-hal-esp32-arduino](https://github.com/vvapnik/ogr-node-hal-esp32-arduino) |
| `ogr-main` | Master (head module) engine + ESP32 I2C-master transport | ESP32 / ESP32-S3 / ESP32-C3 | [github.com/vvapnik/ogr-main](https://github.com/vvapnik/ogr-main) |

## Human Reviewed

*(none yet)*

| Library | Role | Target(s) | Repository | Reviewed by |
|---|---|---|---|---|
