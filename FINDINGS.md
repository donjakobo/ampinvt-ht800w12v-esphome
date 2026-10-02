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
| 4 | Load % — good confidence | **Static, stuck at 1%** regardless of real load — appears unimplemented on this unit. (Later corrected: the vendor doc identifies this register as Output Current, not Load %, so upstream's own label was a guess that didn't hold up — see "Vendor document" section below.) |
| 5 | Output Power — good confidence | **Static 0**, even at ~200W real load. (Later corrected: vendor doc marks this register Reserved — not a gap, expected.) |
| 6 | Charge Current — medium confidence | **Actually Load Percentage**, not charge current — see correction below, since vendor-confirmed as "Output Load Rate" |
| 7 | Battery Voltage — confirmed | **Confirmed working** |
| 8 | unknown | Static 0 — matches vendor doc (Reserved) |
| 9 | Battery Capacity — confirmed | **Confirmed working** (tracked 76%→100% over a charge cycle) |
| 10–12 | unknown | Static 0 — matches vendor doc (Reserved/untested) |
| 13 | Temperature 1 — medium confidence | **Static 0** despite vendor doc listing this as a real field (Internal Temperature) — appears genuinely unimplemented on this lower-wattage SKU, not just undocumented |
| 14 | Temperature 2 — medium confidence | **Static 0**, same situation (vendor doc: Ambient Temperature) |
| 15–31 | reads zero on HT-12212 too | Swept individually, **all static 0** — matches vendor doc (Reserved) |
| 32 | Status word, uninterpreted, read `0x0103` (259) with AC present + charging | **Reads identically: `0x0103` (259)** in the same state — now fully decoded, see below |

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

**Net effect (later superseded by the vendor document):** this originally
led to the conclusion that the HT-800W12V doesn't expose actual charge
current anywhere tested. The vendor document (below) confirms register 6 is
officially "Output Load Rate," and clarifies that register 4 — which
upstream had guessed was Load % — is actually Output Current, and register 5
is Reserved, not Output Power. Neither 4 nor 5 works on this unit regardless
of what they're called, but the vendor doc means we now know *why* they're
named what they're named, rather than carrying forward another repo's guess.

## Ground-truth check

To rule out "maybe the load just happened to be near zero," actual
consumption was measured independently with a smart plug positioned
**upstream of the UPS**, reading a steady **201.9W** — roughly 25% of the
unit's 800W rating — while input register 5 (Output Power, per upstream's
original guess) continued to read a flat 0 and register 4 (Load %, per
upstream's original guess) sat at 1%. The front-panel LCD, for comparison,
was independently observed bouncing in the 19–23% range during the same
load, consistent with the smart plug's reading and consistent with what we
now know register 6 (the real Load %) was reporting.

## Calculated Load — a derived power estimate, and a sync bug

Since this unit doesn't report Output Power over Modbus, `ampinvt-ups.yaml`
includes a template sensor, "Calculated Load," computed on the ESP32:

```
watts = load_percentage × 8.0
```

The multiplier comes from the unit's 800W rating (800 ÷ 100 = 8W per
percentage point). This is a linear approximation, not a measurement, and
it inherits two open assumptions:

1. **Linearity is unverified across the full range.** It's only been
   cross-checked at two real-load points (~20%, ~200W measured via smart
   plug; and ~33–37% with an added ~95W load). The vendor document's
   0–300% range for this register implies overload conditions are possible
   and would presumably still scale linearly, but that hasn't been tested.
2. **It compounds whatever error exists in Load % itself.**

**Sync bug found and fixed:** the template sensor originally ran on its own
independent `update_interval: 5s`, rather than recalculating the instant
Load Percentage changed. Because its timer drifted out of phase with the
Modbus poll cycle, dashboard snapshots sometimes showed clearly mismatched
values — e.g. 23% Load Percentage alongside 208W or 152W Calculated Load,
neither of which is 23% × 8 = 184W. Fixed by triggering the template
sensor's `sensor.template.publish` directly from Load Percentage's
`on_value`, so both update in the same instant. Confirmed fixed: a
follow-up capture showed 23% / 184W together correctly.

Treat this sensor as a convenience for dashboards and automations where a
rough watt figure is more useful than a bare percentage — not as a
replacement for an actual power meter. A real smart plug upstream of the
UPS remains the only trustworthy watt reading in this setup.

## Vendor document: full protocol confirmation

A first-party manufacturer Modbus protocol document was located: **逆变器
MODBUS 通讯协议 V1.0** ("Inverter MODBUS Communication Protocol V1.0"),
published by **佛山市金广源电源科技有限公司** (Foshan Jinguangyuan Power
Technology Co., Ltd.), dated 2018-08. A translated reference distilled from
it lives at
[`docs/modbus-protocol-reference.md`](docs/modbus-protocol-reference.md).

This resolved several open questions at once:

- **Confirmed, word-for-word:** link parameters (9600 8N1, address 1),
  register 6 as Output Load Rate (Load %, 0–300% range), and the full
  register map for 0–9.
- **Corrected a repo-wide assumption:** the manufacturer name in this
  repo's Credits section was previously guessed as "Ampinvt / Foshan Top
  One Power Technology" — the real manufacturer, per this document, is
  Foshan Jinguangyuan Power Technology.
- **Corrected register 4's identity:** officially "Output Current," not
  "Load %" as upstream had guessed. Doesn't change behavior (still
  unimplemented on this unit either way) but fixes the label.
- **Corrected register 5's identity:** officially "Reserved," not "Output
  Power." This means its static-zero reading isn't a mystery gap — it's
  expected per spec.
- **Newly documented: registers 12 (DC Bus Current), 31 (Setting Status
  Bits), 33 (Warning Status Bits), 34 (Error Status Bits).** None of these
  are implemented in the current YAML. 31/33/34 have no bit definitions
  given in the source document (all marked reserved/undefined), so there's
  nothing to decode yet even if they're added.
- **Fully confirmed the register 32 bitfield** this repo had already
  implemented via community sourcing (see next section): Bit0 = Grid
  Normal, Bit1 = Battery Charging, Bit2 = Inverter Active, Bit8 = Output
  Enabled.
- **Fully confirmed register 35's event code table**, word-for-word
  matching what had already been implemented from a community source
  (GitHub issue #2 on the upstream repo).

## Status word (register 32) — vendor bitfield, field-verified

With the vendor document confirming the bit assignments, a real grid-loss
and grid-restore event was used to validate them against actual physical
state changes rather than a single static snapshot.

**Sequence observed (continuous uptime, one boot session):**

| Time | Event | Status word | Decoded bits | AC Input |
|---|---|---|---|---|
| Baseline | Grid present, charging | 259 (`0x103`) | Grid Normal + Battery Charging + Output Enabled | 120.5V |
| Grid cut | Running on battery | 260 (`0x104`) | Inverter Active + Output Enabled (Grid Normal, Charging both correctly dropped) | 0.0V |
| Grid restored (transient) | ~5s transition | 261 (`0x105`) | Grid Normal + Inverter Active + Output Enabled (Charging not yet set) | 121.0V |
| Grid restored (steady-state) | Back to charging | 259 (`0x103`) | Grid Normal + Battery Charging + Output Enabled | 121.0V |

Every individual bit flipped exactly as the vendor document predicts,
simultaneous with AC Input dropping to 0V on grid loss and AC Output
holding steady — correct UPS failover behavior.

**Correction to an earlier community-sourced assumption:** another HT-12212
owner's GitHub issue (issue #2 on the upstream repo) labeled status word
261 as a sustained operating state, "Inverter Mode." What was actually
captured here is **261 as a one-poll-cycle transient** during grid
hand-off — not a steady mode. It appeared for about 5 seconds while the
unit had detected grid return but hadn't yet resumed charging, then settled
back to 259. This is a more plausible physical interpretation than a
sustained "Inverter Mode" state that simultaneously claims Grid Normal is
true — worth flagging back to that issue thread as a refinement, not a flat
contradiction (different unit, and it may behave differently as a sustained
state on the HT-12212).

## Removed: "Inverter Operating Mode" (holding register 0x0007)

An earlier version of the YAML included a text sensor decoding holding
register `0x0007` into an operating-mode string (0 = "Inverter Mode
(Battery)", 1 = "Bypass Mode (Grid)", 2 = "AC Charging", 3 = "Fault Mode"),
sourced from the same community GitHub issue, not the vendor document (the
vendor PDF has no holding-register table at all — it ends at the input
register and bitfield tables above, section 3.1.3, with nothing like a 3.2
holding-register section despite the function-code table implying one
exists).

On this unit, it read "Inverter Mode (Battery)" at the exact same moment
the vendor-confirmed status bits showed Grid Normal = true and Battery
Charging = true — a direct contradiction, since those bits mean the unit
was on-grid and charging, not running on battery/inverter. This sensor has
been **removed from the YAML entirely** rather than kept with a caveat,
since there's no vendor documentation to fall back on and the one data
point available actively contradicts it.

## Conclusion

The HT-800W12V's Modbus implementation is now understood with much higher
confidence than before, backed by the manufacturer's own document rather
than cross-model guessing:

**Confirmed working and vendor-documented:** AC input/output
voltage+frequency (regs 0–3), Load Percentage (reg 6 — officially "Output
Load Rate"), Battery Voltage and Capacity (regs 7, 9), the full Operating
Status Word bitfield (reg 32, all 4 bits field-verified against a real
grid-loss/restore event), and the Event Code table (reg 35).

**Confirmed absent, despite being documented fields:** Output Current (reg
4), Internal and Ambient Temperature (regs 13/14), DC Bus Current (reg 12,
untested but likely absent given the pattern). Register 5 (Reserved) was
never expected to report anything.

**Removed due to contradiction:** the holding-register-based "Inverter
Operating Mode" sensor, which had no vendor documentation and conflicted
with confirmed data.

**Still untested:** registers 12, 31, 33, 34. 31/33/34 have no bit
definitions in the source document, so even if added they'd currently
decode to nothing meaningful.

If someone finds Output Current, Temperature, or DC Bus Current working via
a different function code, a vendor-specific extended range, or the IM-WR4
WiFi box's own protocol, a PR updating this file and the sensor YAML would
be very welcome.
