# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## What this repository is

A from-scratch KiCad 9 rebuild of the FujiNet FN32ROV WiFi/SIO adapter (originally a
DipTrace design in `FujiNetWIFI/fujinet-hardware`), adapted for permanent internal
mounting inside a custom "XEBook" laptop build. See `README.md` for the full description
and what differs from the original FN32ROV-1.7.1.

Status: work in progress — schematic and PCB layout built, not yet fabricated.

## Working conventions

- The user does manual placement/routing/symbol cleanup themselves in the KiCad GUI;
  prioritize correctness over polish when editing files directly.
- KiCad ref/value labels for R/C go beside the body, not above/below. Net-label stub
  length must scale with the label's name length.
- Verification is two-tier and both tiers are required before calling the board correct:
  1. `kicad-cli pcb drc --schematic-parity` — PCB vs. our schematic
  2. Netlist diff of our schematic against `Docs/netlistExport.net` (the original
     FujiNet DipTrace netlist export) — schematic vs. original design intent
- Files are edited via direct text-surgery on the `.kicad_sch`/`.kicad_pcb` s-expression
  files when doing bulk/scripted changes, to preserve the user's manual GUI edits. Always
  re-verify with both tiers above after doing so.

## External LED board interface (J_LED1)

The three status LEDs (WiFi, Bluetooth, SIO) are not on this board — they connect off-board
via a JST pigtail to a separate LED daughter board, built in its own repo:
[FN32ROV-XEBook-LED-KiCad](../FN32ROV-XEBook-LED-KiCad). This is the interface contract
between the two boards; if it changes here, it must change there too.

**Connector:** J_LED1, `Connector_JST:JST_PH_S4B-PH-K_1x04_P2.00mm_Horizontal` — 4-pin JST
PH, 2.0mm pitch (matches the other JST connectors on this board).

**Pinout:**

| Pin | Net | Driven by |
|---|---|---|
| 1 | 3V3 | common anode supply |
| 2 | LED_WIFI | GPIO2 (ESP32 LED1) → R13 (2.7kΩ) |
| 3 | LED_BT | GPIO13 (ESP32 LED3) → R14 (1kΩ) |
| 4 | LED_SIO | GPIO4 (ESP32 LED2) → R15 (1.2kΩ) |

Current-limiting resistors (R13/R14/R15) live on **this** board, not the LED board — the
daughter board should just carry the 3 LEDs, wired common-anode to pin 1 (3V3), with each
LED's cathode returning to its respective signal pin. The ESP32 GPIO sinks current by
driving the pin low to light that LED.

R13/R14/R15 are each sized for that channel's specific LED color/forward-voltage, not a
single shared value:

- R14/R15 (1kΩ/1.2kΩ) size the original FN32ROV-1.7.1 LEDs: blue (BT, ~2.9-3.2V Vf) and
  orange (SIO, ~2.0-2.2V Vf).
- R13 was changed from 1kΩ to 2.7kΩ because the LED board's D1 (WiFi) uses a **green**
  LED (Rohm SML-P12PTT86R, ~2.2V Vf) instead of the original white (SunLED
  XZBWR68F5MAV-3, ~2.9V Vf, now obsolete) — green's lower Vf means more of the 3.3V rail
  drops across the resistor, so it needed to grow to keep the current sane (same
  ~0.4mA target as the original white-LED design).

If the LED board's color choices change again, revisit R13/R14/R15 here to keep currents
sane per color.
