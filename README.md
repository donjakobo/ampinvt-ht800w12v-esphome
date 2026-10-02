# AMPINVT HT-800W12V → ESPHome → Home Assistant

Monitor an **[AMPINVT HT-800W12V UPS/inverter-charger](https://www.amazon.com/dp/B098QL2VBZ)** over its RS485 Modbus
port using an ESP32 running [ESPHome](https://esphome.io), reporting directly
into [Home Assistant](https://www.home-assistant.io) via ESPHome's native API
(no MQTT broker required).

This started as a fork of the excellent groundwork in
[Bgilsing/ampinvt-ht12212-modbus](https://github.com/Bgilsing/ampinvt-ht12212-modbus),
which reverse-engineered the Modbus RTU protocol on the **HT-1200W12V
(HT-12212)**. This repo documents what changes — and what doesn't work — on
the smaller **HT-800W12V** unit, plus a full ESPHome build instead of a
Raspberry Pi + Modbus TCP gateway. Findings here are now additionally
backed by the manufacturer's own Modbus protocol document — see
[`docs/modbus-protocol-reference.md`](docs/modbus-protocol-reference.md).

If you own any Ampinvt HT/HTS/FT/TG-series inverter, testing this against
your unit and reporting back (here or upstream) helps map the whole product
line. See [FINDINGS.md](FINDINGS.md) for the full register sweep this repo
is based on, and the "Verified on" table below.

---

## How it fits together

```
AMPINVT HT-800W12V  --RS485 (RJ45, pins 1/2/8)-->  HiLetgo TTL-RS485 module
                                                            |
                                                     UART (GPIO16/17)
                                                            |
                                                          ESP32
                                                     (ESPHome firmware)
                                                            |
                                                      WiFi, native API
                                                            |
                                                     Home Assistant
```

- **ESPHome** owns the RS485/Modbus polling, does the scaling math, and
  exposes each field as a Home Assistant sensor entity automatically —
  no YAML on the Home Assistant side, no MQTT broker, no separate gateway
  process. The ESP32 polls the inverter every 5 seconds and pushes updates
  over its encrypted native API connection.
- **Home Assistant** just needs the built-in ESPHome integration, which
  auto-discovers the device via mDNS once it's on your network.

## Hardware used

| Part | Specific hardware |
|---|---|
| UPS / inverter-charger | [AMPINVT HT-800W12V](https://www.amazon.com/dp/B098QL2VBZ) (800W, 12V DC / 120V AC), RS485 on RJ45 |
| Microcontroller | ESP32 dev board, silkscreened "NodeMCU-32S" (ESP32-WROOM-32 module) |
| RS485 transceiver | [HiLetgo TTL to RS485 UART module](https://www.amazon.com/dp/B082Y19KV9), automatic flow control (auto direction, no DE/RE pin needed) |

See [`docs/wiring-diagram.svg`](docs/wiring-diagram.svg) for the full pinout
reference. Real photos of the physical build are welcome as a follow-up PR —
none are included here to avoid using vendor/marketplace product photography.

### Wiring summary

| From | To |
|---|---|
| RJ45 pin 1 (A+) | RS485 module A+ |
| RJ45 pin 2 (B−) | RS485 module B− |
| RJ45 pin 8 (GND / 接大地) | RS485 module GND |
| RJ45 pin 6 | **Do not connect.** Carries ~18.5V on this unit — will destroy a transceiver. |
| RS485 module VCC | ESP32 3.3V |
| RS485 module GND | ESP32 GND |
| RS485 module TXD | ESP32 GPIO17 |
| RS485 module RXD | ESP32 GPIO16 |

This particular auto-flow RS485 module needed **straight, non-crossed**
TX/RX wiring to work reliably — the opposite of the usual UART crossover
convention. If you wire the conventional way and get complete silence
(Modbus timeouts, device shows offline), try swapping GPIO16/17 before
assuming anything else is wrong.

Link parameters: **9600 baud, 8N1, slave address 1**, Modbus RTU — confirmed
by both the upstream repo's reverse-engineering and the manufacturer's own
protocol document (see `docs/modbus-protocol-reference.md`).

## Sensors and registers (confirmed working on the HT-800W12V)

All via function `0x04` (read input registers), read-only:

| Register | Field | Raw scale | Unit | Notes |
|---|---|---|---|---|
| 0 | AC Input Voltage | ÷10 | V | |
| 1 | AC Input Frequency | ÷10 | Hz | |
| 2 | AC Output Voltage | ÷10 | V | |
| 3 | AC Output Frequency | ÷10 | Hz | |
| 6 | Load Percentage | ×1 (no scaling) | % | Officially "Output Load Rate" per the vendor doc (0–300% range — can exceed 100 during overload). Confirmed by adding a known ~95W load and watching the raw value track it in real time. |
| 7 | Battery Voltage | ÷10 | V | |
| 9 | Battery Capacity | ×1 | % | Voltage-curve estimate, not a true coulomb-counted SoC — expect it to swing with load |
| 32 | Operating Status Word (raw) | — | — | Bitfield, vendor-documented (Table 3.1.2) and field-verified against a real grid-loss/grid-restore event. Decoded into four binary sensors — see below. |
| 35 | Event Code (raw) | — | — | Vendor-documented 0–9 fault/warning table (Table 3.1.3). Decoded into a text sensor — see below. Only code 0 (Normal) observed so far. |

### Derived and decoded sensors

In addition to the register-backed sensors above, several sensors are
computed or decoded entirely on the ESP32:

| Sensor | Source | Notes |
|---|---|---|
| Calculated Load | Load Percentage (reg 6) | `load_percentage × 8.0` watts, recalculated the instant Load Percentage updates (via an `on_value` trigger, not an independent timer — the two were briefly out of sync in early testing before this fix). Assumes a linear 800W rated scale; not a substitute for a real power meter. |
| Grid Normal, Battery Charging, Inverter Active, Output Enabled | Operating Status Word (reg 32), bits 0/1/2/8 | Vendor-documented bit assignments, field-verified across all three real states: grid-on, grid-cut (running on battery), and grid-restored. |
| Inverter Event and Alarm State | Event Code (reg 35) | Vendor-documented 0–9 text mapping. |

### What was removed, and why

An earlier version of this YAML included an **"Inverter Operating Mode"**
sensor, decoded from holding register `0x0007`, sourced from another
HT-12212 owner's GitHub issue rather than the vendor document. On this
unit, its value ("Inverter Mode (Battery)") directly contradicted the
vendor-confirmed status bits read at the same moment. It has been removed
from the YAML entirely — see FINDINGS.md for the full comparison.

## What doesn't work on the HT-800W12V

Unlike the HT-1200W12V this protocol was originally reverse-engineered
against, the **HT-800W12V does not expose Output Current (reg 4),
Temperature (regs 13/14), or DC Bus Current (reg 12)** over Modbus, despite
all being named, documented fields in the manufacturer's own protocol
spec — they read static/zero on this unit regardless of real conditions.
(Register 5 is officially marked "Reserved" in the vendor doc, so its
static-zero reading isn't a gap, it's expected.) See
[FINDINGS.md](FINDINGS.md) for the full methodology, including
cross-checking against a real ~200W load measured independently with a
smart plug, and a second confirmation from deliberately adding a ~95W load.

A "Calculated Load" sensor (see the derived-sensor table above) provides a
rough wattage estimate from Load %, but for real, measured power draw, use
an external smart plug upstream of the unit — the inverter itself won't
give you an actual watt reading over Modbus.

## Verified on

| Model | Internal model | Wattage | Verified by | Notes |
|---|---|---|---|---|
| HT-1200W12V | HT-12212 | 1200W | [Bgilsing](https://github.com/Bgilsing/ampinvt-ht12212-modbus) | Original protocol reverse-engineering |
| HT-800W12V | — | 800W | this repo | AC V/Hz, Load %, Battery V/%, status word (all 4 bits), event code all confirmed working and vendor-documented. Output Current, Temperature, DC Bus Current confirmed unimplemented on this unit despite being documented fields. |

If you test this against another model in the line, please open an issue or
PR with your findings — a register-by-register confirmation table like the
one above is exactly what makes this useful across the product line.

## Setup

1. Wire everything per the table above and `docs/wiring-diagram.svg`.
2. Copy `config/secrets.yaml.example` to `config/secrets.yaml` and fill in
   your WiFi credentials. (Or let the ESPHome dashboard generate an API key
   for you when you add the device — either works.)
3. Add `config/ampinvt-ups.yaml` as a new device in your ESPHome dashboard,
   or compile/flash it directly with the ESPHome CLI.
4. First flash requires USB; every flash after that can go over WiFi (OTA).
5. Add the device in Home Assistant via Settings → Devices & Services → it
   should auto-discover via mDNS as `ampinvt-ups.local`.

## Credits

- [Bgilsing/ampinvt-ht12212-modbus](https://github.com/Bgilsing/ampinvt-ht12212-modbus) —
  the original Modbus RTU reverse-engineering this repo builds on, including
  ruling out Ampinvt's separate APC charge-controller protocol and finding
  the RJ45 pin 6 danger voltage.
- Manufacturer protocol document: 逆变器 MODBUS 通讯协议 V1.0, 佛山市金广源电源科技有限公司
  (Foshan Jinguangyuan Power Technology Co., Ltd.), 2018-08 — see
  [`docs/modbus-protocol-reference.md`](docs/modbus-protocol-reference.md)
  for the translated reference this repo uses.
- Not affiliated with or endorsed by Ampinvt or Foshan Jinguangyuan Power
  Technology.

## License

MIT — see [LICENSE](LICENSE).
