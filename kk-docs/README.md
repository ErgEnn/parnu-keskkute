# kk – heating plant controller: functional documentation

This document was reverse-engineered from `kk.vlp` (Unitronics VisiLogic). It was derived from the `vlpdump --dir` output in [`../kk/`](../kk/) using the `visilogic-plc-dump` skill.

> **Sensitive:** the project holds plain-text passwords (see finding 15). Treat `kk/` and these docs as confidential.

## The plant in one paragraph

A Unitronics **Vision V-570** (320x240 colour touch panel) controls a domestic heating plant. The I/O modules are:
- IO-PT4x: PT100 temperatures
- IO-ATC8: analog/thermocouple inputs
- IO-AO6X: analog outputs (valves, pump speed, heat-pump set point)
- IO-DI16: digital inputs
- IO-TO16: transistor outputs

The plant is made of these parts:
- **Air/water heat pump (SP):** the main heat source. It switches between space heating and hot water with a 3-way valve and has defrost and booster-heater handling plus water-flow supervision.
- **Radiator circuit (KK):** a weather-compensated PID mixing valve with room-temperature correction and 2+6 weekly clocks.
- **Floor-heating circuit (PK):** a curve plus PID mixing valve, clocks, timers, clock on/off, and an optional electric floor heater.
- **Domestic hot water (TV):** three set points (two clocks and one manual), a heat-pump time limit, and a weekly/periodic **Legionella** disinfection.
- **Buffer tank (MP):** switches into the circuits by temperature offsets.
- **Wood boiler:** its pump is controlled by chimney temperature, with an air damper.

Every function can be switched on or off from the panel. Errors are reported on a pop-up screen and on an auxiliary display.

## Chapters

| # | File | Content |
|---|---|---|
| 1 | [01-overview-hardware-structure.md](01-overview-hardware-structure.md) | Plant summary, metadata, hardware, complete I/O map, subroutine tree, system services, Estonian glossary |
| 2 | [02-radiator-heating-KK.md](02-radiator-heating-KK.md) | Radiator circuit: weather curve, room correction, PID valve, clocks, pump |
| 3 | [03-floor-heating-PK.md](03-floor-heating-PK.md) | Floor circuit: curve, PID, clocks and timers, clock switching, electric floor heating |
| 4 | [04-hot-water-TV-and-legionella.md](04-hot-water-TV-and-legionella.md) | Hot water: set-point selection, differential, heat-pump time limit, Legionella cycle |
| 5 | [05-heat-pump-SP.md](05-heat-pump-SP.md) | Heat pump: demand arbitration KK/PK vs TV, set-point output, defrost/booster, flow alarm, pump speed |
| 6 | [06-buffer-tank-and-boiler.md](06-buffer-tank-and-boiler.md) | Buffer tank switching, wood-boiler pump and damper, overview diagram animation |
| 7 | [07-operator-interface-HMI.md](07-operator-interface-HMI.md) | 35 screens, navigation graph, screen catalogue, full operator parameter table |

Each chapter has the same sections:
- purpose
- subroutines
- signals
- logic, with net references
- timers
- Mermaid state diagrams
- operator parameters
- open questions

## How to read these documents

- **Net references** such as `K.K.seade/0014` mean `kk/ladder/! Main Module/K.K.seade/0014.txt`. Open that file to see the exact rung.
- **Operand references** use `MB` / `MI` / `ML` / `DW` (memory bit / int / long / double word), `SB` / `SI` (system), `I` / `O` (I/O) and `T` (timer). `kk/index.txt` lists every reader and writer of each operand.
- **Units:**
  - Temperatures are usually integers in **0.1 °C** (for example `MI198` = 453 → 45.3 °C). Where the program uses whole degrees, the chapter says so.
  - Timer presets are in **10 ms** units (12000 = 120 s).
- **Estonian terms** are explained in the glossary in chapter 1.8, for example KK = keskküte (radiator heating), PK = põrandaküte (floor heating), TV = tarbevesi (domestic hot water), SP = soojuspump (heat pump), MP = mahupaak (buffer tank), seade = set point, kell = clock, sees/väljas = on/off.
- **Certainty:** statements are taken from the ladder. Where a meaning was inferred, for example from an operand name or a text held in an undecoded blob, the text says "inferred". The behaviour of `kk/hmi` texts stored in binary blobs was not decoded.

## Key findings (probable bugs and inconsistencies)

These are ordered roughly by impact. The details and evidence are in the chapter sections given in brackets.

1. **`MB91` has three coils** (`2-5.KK/0005`, `2-6.KK/0005`, `2-7.KK/0005`). The last rung that executes wins, so KK temperature-2 clocks 1–4 (F2-5 and F2-6) have no effect while F2-7's clocks are off. Fix: OR all six clock bits in one rung. [2.8]
2. **`MB304` has two coils in parallel branches of one rung** (`K.K.seade/0014`). The "−" branch overwrites the "+" branch, so the room-correction limit only works in one direction. [2.8]
3. **The PK button uses the wrong hide bit.** `3-3.PK/0002` writes `MB1221` (the F2-3 hide bit) instead of `MB1321`. The PK selection-1 MIINUS button never hides, and it hides the KK button instead. [3.8, 7.5]
4. **PID runs only after an auto-tune.** The KK and PK PID function blocks are enabled only after auto-tune completes. Their sample-time registers `MI318` / `MI418` are never written. After a fresh download, the mixing valves are not regulated until someone presses "Start AUTO-TUNE". [2.8, 3.8]
5. **Unconfirmed: is the `MB125` buffer-tank block writable at all?** No ladder rung or touch operand writes it. However, its F6-2 VAR uses the same settings blob (`sha1:034177461852`) as the six plant on/off switches (`MB1120`…`MB1125`), which have no other writer either. So the image VAR itself most likely toggles the bit. Chapter 6.8 assumes it does not, so confirm this in VisiLogic. [6.8, 7.5]
6. **The `I82` label is the inverse of its use.** It is labelled "HP 3-way valve open towards KK/PK", but the ladder treats `I82` = 1 as the hot-water position. [4.8, 5.8]
7. **The set-point limiter is labelled 60 °C but clamps at 56 °C.** [5.8]
8. **Hot-water set points are not clamped before scaling.** `MI500` and `MI512` go through `LINEAR` (10–60 °C) unclamped, so a value above 60 °C produces more than 20 mA. [5.8]
9. **Defrost texts and timer values disagree.** The minutes shown on F5-2 do not match the timer presets used in the ladder. [5.8]
10. **The Legionella-pause hourglass `MI57` is never shown.** Both hourglasses on F5-2 link `MI55`. [5.8, 7.5]
11. **`MI203` (chimney temperature) is keypad-enabled** on two screens with no limits. It is overwritten every scan, so this is harmless but confusing. [6.8, 7.5]
12. **The error-return logic depends on `SI252`** ("current HMI display number"), which is documented as Standard Vision only. On a V-570 the "back to faulty screen" button and the error-code pages' Back buttons may not work. [7.5]
13. **`MB1143` is never set**, so the "back to faulty screen" button is always visible. [7.5]
14. **Clock subroutines are called twice per scan** while their screen is displayed: once from the main routine and once as the display's linked routine. This is probably harmless because of the edge contacts, but it is unverified. [7.5]
15. **Plain-text passwords:** the info-mode password is in `MI27`, and the Ethernet/remote-access password is in `MI4087`–`MI4090` (power-up values in `kk/operands.txt`). Change them if this file has ever left the site.
16. **Stale metadata.** `project.txt` says the panel is 128x64, but all screens are laid out for 320x240. [1.9, 7.5]

## Not verified / limits

- The texts of list VARs, binary texts and messages are stored in undecoded blobs. Their meanings are inferred from operand names and from the ladder.
- Hydraulic roles of some inputs (`I81`/`I82`, the hand valves) are inferred from names and drawings, not from a P&ID.
- The display numbers used by `SI252` and the start-up display are not present in the dump.
- None of this was tested on a PLC. Everything is static analysis of the program.
