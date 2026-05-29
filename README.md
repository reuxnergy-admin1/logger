# StratusLog

KiCad project for the StratusLog environmental data logger:

- ATmega1284P-A MCU (Arduino-style, 3.3 V)
- DS3231M RTC + CR2032 backup
- microSD storage (Hirose DM3AT)
- LSM100A LoRa/Sigfox modem (SMA antenna)
- SDI-12 transceiver (SN74LVC1G240 + BSS123)
- LT1372HVCS8 5 V→12 V boost for sensor power
- 3× Phoenix MKDS-1-7-3.81 sensor terminals

## Documentation

- **[docs/REVIEW_AND_VALVE_DRIVER.md](docs/REVIEW_AND_VALVE_DRIVER.md)** — schematic review with prioritized findings + valve driver add-on (latching + non-latching).
- **`valves.kicad_sch`** — starter hierarchical sub-sheet for the valve driver block.

## Open priorities (see review doc)

1. Boost converter (U4) — add inductor, Schottky, output cap, FB divider, compensation.
2. 3.3 V regulator — currently missing from the BOM.
3. AVR ISP header — currently no path to flash the bootloader.
4. Crystal Y1 — set frequency (8 MHz @ 3.3 V) and footprint; add load caps.
5. Resistors — change 0201 footprint to 0603 to match HandSolder intent.
