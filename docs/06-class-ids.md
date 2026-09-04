# 6. Standard CLASS IDs

| Class ID | Name | Description | Typical Endpoints |
| -------- | ---- | ------------ | ------------------ |
| `0x01` | **Plant Container** | Modules tied directly to an individual plant slot/pot/hydroponic container. Usually designated as Anchor modules (`IS_ANCHOR = 1`). | Soil moisture, root zone temp, local drip valve, status LED |
| `0x02` | **Light** | Modules controlling plant illumination (shared or individual light arrays). | PWM dimmers, spectrum channels (PAR/UV/IR), light sensors |
| `0x03` | **Water Supply** | Central irrigation, nutrient dosing, and water management modules. | Pumps, main solenoids, water level sensors, EC/pH sensors |
| `0x04` | **Climate** | Environmental monitoring and microclimate control units. | Ambient temp/humidity, CO2 sensors, intake/exhaust fans, heaters |
| `0xFF` | **Custom / Unspecified** | Vendor-specific or experimental modules not fitting standard categories. | Defined by endpoint descriptors or Blueprint |
