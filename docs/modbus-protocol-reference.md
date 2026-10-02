# Inverter Modbus Protocol — vendor reference (translated)

This is an English reference distilled from the manufacturer's own Modbus
protocol document for this inverter family. It is **not** a full
translation — it covers the sections relevant to this repo (link
parameters, function codes used, and the register/bit/event tables) rather
than every section of the original (RTU framing internals, CRC algorithm
detail, write-function examples, etc., which are standard Modbus and
covered by the Modbus spec itself).

- **Document:** 逆变器 MODBUS 通讯协议 V1.0 ("Inverter MODBUS Communication
  Protocol V1.0")
- **Manufacturer:** 佛山市金广源电源科技有限公司 (Foshan Jinguangyuan
  Power Technology Co., Ltd.)
- **Date:** 2018-08
- Machine-translated (Gemini) from the original Chinese PDF, then checked
  against this repo's own live register captures where possible — see the
  "Field verification" column below and [FINDINGS.md](../FINDINGS.md) for
  the test methodology.

The original PDF reference [modbus-protocol-original-zh.pdf](modbus-protocol-original-zh.pdf).

## Link parameters

Interface: RS232 or RS485. **9600 baud, 8N1 (8 data bits, no parity, 1
stop bit), slave address 1.** Modbus RTU framing, standard CRC-16 (poly
0xA001). Matches what both this repo and the upstream HT-12212 repo
independently discovered.

## Function codes used by this device

| Code | Name | Used for |
|---|---|---|
| 0x03 | Read Holding Registers | User settings (not used by this repo's YAML currently) |
| 0x04 | Read Input Registers | All live telemetry sensors in this repo |

(The document also defines 0x01, 0x02, 0x05, 0x06, 0x0F, 0x10 for coil/
holding-register read-write, standard Modbus — not used by this repo.)

## Table 3.1.1 — Input register map (whole-unit data)

All registers are function `0x04`, 2 bytes each, high-byte-first on the
wire (standard Modbus byte order).

| Address | Field (translated) | Range | Unit | Field verification on HT-800W12V |
|---|---|---|---|---|
| 000 | AC Input Voltage | 0–5000 | 0.1V | **Confirmed working** |
| 001 | Input Frequency | 0–5000 | 0.1Hz | **Confirmed working** |
| 002 | Output Voltage | 0–5000 | 0.1V *(doc lists "0.1A" here — a typo in the original; field name and every comparable register confirm this is volts)* | **Confirmed working** |
| 003 | Output Frequency | 0–5000 | 0.1Hz | **Confirmed working** |
| 004 | Output Current | 0–20000 | 0.1A | **Static/unimplemented on this unit** — corrects an earlier assumption (from the upstream HT-12212 repo) that this register was Load % |
| 005 | Reserved | — | — | **Static 0, matches doc** — corrects an earlier assumption that this was Output Power |
| 006 | Output Load Rate | 0–300 | % | **Confirmed working** — this is what this repo calls "Load Percentage." Note the official range tops out at 300%, not 100% — an overload condition can read above 100. |
| 007 | Battery Voltage | 0–5000 | 0.1V | **Confirmed working** |
| 008 | Reserved | — | — | Static 0, matches doc |
| 009 | Battery Capacity Rate | 0–100 | % | **Confirmed working** |
| 010 | Reserved | — | — | Static 0, matches doc |
| 011 | Reserved | — | — | Static 0, matches doc |
| 012 | DC Bus Current | 0–65535 | 0.1A | Untested on this unit — not yet added to the YAML |
| 013 | Internal Temperature | 0–2000 | 0.1°C | **Static 0 on this unit**, despite the doc listing it as a real field — likely unimplemented on this lower-wattage SKU |
| 014 | Ambient Temperature | 0–2000 | 0.1°C | **Static 0 on this unit**, same as above |
| 015–030 | Reserved | — | — | Static 0, matches doc |
| 031 | Setting Status Bits | — | — | See Table 3.1.2 below. Not yet added to the YAML. |
| 032 | Running Status Bits | — | — | **Confirmed working** — see Table 3.1.2 and this repo's `binary_sensor` block |
| 033 | Warning Status Bits | — | — | Untested — bit definitions not given in the source document |
| 034 | Error Status Bits | — | — | Untested — bit definitions not given in the source document |
| 035 | Event Code | — | — | **Confirmed working** — see Table 3.1.3 below |

## Table 3.1.2 — Status bit definitions

**Setting Status Bits (register 031):** all 16 bits reserved/undefined in
the source document. Not implemented in this repo.

**Running Status Bits (register 032)** — this is the register this repo
calls "Operating Status Word":

| Bit | Meaning (translated) | This repo's sensor |
|---|---|---|
| 0 | 市电正常 — Grid Normal | `Grid Normal` binary_sensor |
| 1 | 充电运行中 — Charging in progress | `Battery Charging` binary_sensor |
| 2 | 逆变运行中 — Inverting in progress | `Inverter Active` binary_sensor |
| 3–7 | Reserved | — |
| 8 | 输出开启 — Output enabled | `Output Enabled` binary_sensor |
| 9–15 | Reserved | — |

All four bits have been field-verified on this HT-800W12V against a real
grid-loss/grid-restore event — see FINDINGS.md.

## Table 3.1.3 — Event code table (register 035)

| Code | Event (translated) |
|---|---|
| 00 | (No event) |
| 01 | Internal overcurrent protection |
| 02 | Output short circuit protection |
| 03 | Output overload protection |
| 04 | System overtemperature protection |
| 05 | High battery voltage protection |
| 06 | Low battery voltage protection |
| 07 | Phase sequence error warning |
| 08 | Low output voltage protection |
| 09 | ECO mode active |

Only code 0 has been observed on this unit so far (normal operation). The
fault/warning codes are implemented in this repo's `text_sensor` decode but
not field-tested — none of these conditions have been deliberately
triggered.
