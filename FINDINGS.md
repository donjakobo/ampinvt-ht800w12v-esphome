# Findings — AMPINVT HT-800W12V register sweep

This documents the testing that led to the sensor list in
[`config/ampinvt-ups.yaml`](config/ampinvt-ups.yaml). Everything here was
measured on one physical **HT-800W12V** unit; treat it the same way the
upstream repo treats its own findings — confidence-rated, not gospel, and
worth checking against your own unit if you can.

## Starting point

[Bgilsing/ampinvt-ht12212-modbus](https://github.com/Bgilsing/ampinvt-ht12212-modbus)
had already solved the link layer for the HT-series: 9600 8N1, slave address
1, Modbus RTU, CRC-16 (poly 0xA001). That part transferred over to the 800W
unit with zero changes — same baud, same address, same CRC, immediate valid
replies.

## Wiring troubleshooting

Initial wiring (module TXD→GPIO16, RXD→GPIO17 — matching the conventional
UART crossover) produced complete silence: every poll timed out
(`Stop waiting for response`), the modbus_controller marked the device
offline, and no amount of register-address changes helped. This turned out
to be specific to the particular HiLetgo auto-flow-control module used here:
swapping to **straight** wiring (module TXD→GPIO17, RXD→GPIO16) — the
opposite of the expected crossover — immediately produced valid replies. An
RXD LED on the module blinking in sync with poll attempts was a useful
sanity check that the ESP32 side was transmitting correctly even while
still debugging this.

Lesson: if you build this and get total silence with correctly wired RJ45
pins 1/2/8 and correct A/B polarity, try swapping the module's TX/RX pins
before assuming anything else is wrong.

## Register sweep — input registers (function 0x04)

Registers 0–14 and 32 are the range the upstream repo validated on the
HT-12212. On the HT-800W12V:

| Register | Upstream (HT-12212) | This unit (HT-800W12V) |
|---|---|---|
| 0 | AC Input Voltage — confirmed | **Confirmed working**, matches AC meter |
| 1 | AC Input Frequency — confirmed | **Confirmed working** |
| 2 | AC Output Voltage — confirmed | **Confirmed working** |
| 3 | AC Output Frequency — confirmed | **Confirmed working** |
| 4 | Load % — good confidence | **Static, stuck at 1%** regardless of real load — appears unimplemented on this unit |
| 5 | Output Power — good confidence | **Static 0**, even at ~200W real load |
| 6 | Charge Current — medium confidence | **Actually Load Percentage**, not charge current — see correction below |
| 7 | Battery Voltage — confirmed | **Confirmed working** |
| 8 | unknown | Static 0 |
| 9 | Battery Capacity — confirmed | **Confirmed working** (tracked 76%→100% over a charge cycle) |
| 10–12 | unknown | Static 0 |
| 13 | Temperature 1 — medium confidence | **Static 0** (reads as exactly 0.0°C regardless of real ambient temp — a basement at ~15–18°C should never read 0°C) |
| 14 | Temperature 2 — medium confidence | **Static 0** |
| 15–31 | reads zero on HT-12212 too | Swept individually, **all static 0** |
| 32 | Status word, uninterpreted, read `0x0103` (259) with AC present + charging | **Reads identically: `0x0103` (259)** in the same state — strong evidence the addressing/CRC/framing is identical across models even where field implementation differs |

## Register sweep — holding registers (function 0x03)

The upstream repo only validated holding registers as *settings*, not live
telemetry. On the chance the 800W unit exposed load/power/temp there
instead, a sample was swept during a real ~200W load:

| Register | Value observed | Interpretation |
|---|---|---|
| 4 | 120 (static) | AC output voltage setting — matches upstream's own reading of reg 4 as a setting |
| 5 | 60 (static) | AC output frequency setting |
| 8 | 1 (static) | Unknown setting/index |
| 9 | 0 (static) | — |
| 10 | 0 (static) | — |
| 15 | 3 (static) | Unknown setting/index |
| 16 | 0 (static) | — |
| 22 | 0 (static) | — |
| 24 | 3 (static) | Unknown setting/index |
| 26 | 0 (static) | — |

All static across dozens of polls while real load and AC state visibly
changed. No live telemetry found in the holding bank either.

## Correction: register 6 is Load Percentage, not Charge Current

Initial testing labeled register 6 "Charge Current" (÷10 scaling → values
like 1.9A–2.3A) because the numbers moved and looked amp-like, and it was
the closest match to the upstream repo's medium-confidence guess for that
address. That was wrong.

A follow-up test added a known load (a box fan drawing ~95W) directly to
the UPS output and watched the raw register value in real time:

- Baseline (just the existing ~200W household load): raw value bounced
  19–23, matching the LCD's 19–23% load display and the ~25% expected from
  200W of 800W capacity.
- With the fan added (~295W total, ~37% of 800W): raw value climbed to the
  low-30s and continued bouncing in step with the LCD, which also moved to
  roughly 33–37%.

The raw integer **is** the load percentage directly — no ÷10 scaling
needed. What looked like "1.9A → 2.3A jitter" was actually "19% → 23%
jitter" misread as amps. The `config/ampinvt-ups.yaml` sensor has been
renamed to "Load Percentage" with the scaling filter removed accordingly.

This also resolves an earlier, unexplained anomaly: register 6 stayed
nonzero (reading ~19–20) even when AC input was completely cut and the unit
was running off battery — which never made sense for *charge* current (there
was nothing to charge from) but makes complete sense as *load* percentage
(the household load was still being served from the battery through the
inverter).

**Net effect:** this means the HT-800W12V does not appear to expose actual
charge current anywhere in the registers tested — it joins Output Power and
Temperature as fields this unit's Modbus implementation doesn't report.

## Ground-truth check

To rule out "maybe the load just happened to be near zero," actual
consumption was measured independently with a smart plug positioned
**upstream of the UPS**, reading a steady **201.9W** — roughly 25% of the
unit's 800W rating — while input register 5 (Output Power) continued to
read a flat 0 and register 4 (Load %) sat at 1%. The front-panel LCD, for
comparison, was independently observed bouncing in the 19–23% range during
the same load, consistent with the smart plug's reading and inconsistent
with what registers 4/5 reported.

## Calculated Load — a derived power estimate

Since this unit doesn't report Output Power over Modbus, `ampinvt-ups.yaml`
includes a template sensor, "Calculated Load," computed on the ESP32:

```
watts = load_percentage × 8.0
```

The multiplier comes from the unit's 800W rating (800 ÷ 100 = 8W per
percentage point). This is a linear approximation, not a measurement, and
it inherits two open assumptions:

1. **Linearity is unverified across the full range.** It's only been
   cross-checked at two points — baseline (~20%, ~200W measured via smart
   plug) and with an added ~95W load (~33–37%, consistent with ~295W). Both
   points are consistent with a linear 8W/% relationship, but low-end
   (near 0%) and high-end (near 100%) behavior hasn't been tested.
2. **It compounds whatever error exists in Load % itself.** If the
   register's percentage reporting has its own rounding or noise, the
   watt estimate inherits it, scaled by 8×.

Treat this sensor as a convenience for dashboards and automations where a
rough watt figure is more useful than a bare percentage — not as a
replacement for an actual power meter. A real smart plug upstream of the
UPS remains the only trustworthy watt reading in this setup.

## Conclusion

The HT-800W12V's Modbus implementation appears to be a **strict subset** of
the HT-12212's, but not a simple one — some fields are missing, and one
(register 6) reports something different than upstream's own confidence
rating suggested. Confirmed working: AC input/output voltage+frequency
(regs 0–3), Load Percentage (reg 6 — not register 4, which is unimplemented
on this unit), Battery Voltage and Capacity (regs 7, 9), and the raw status
word (reg 32, byte-for-byte identical to the HT-12212 in the same state).
Not available on this unit: Output Power, Temperature (both sensors), and
Charge Current.

No further register range remains untested in the ranges the upstream repo
identified as relevant (0–32 input, spot-checked holding). If someone finds
these values elsewhere (a different function code, a vendor-specific
extended range, or via the IM-WR4 WiFi box's own protocol rather than plain
Modbus), a PR updating this file and the sensor YAML would be very welcome.
