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

### Gotchas hit building the LED daughter board (apply to any new board built the same way)

- A `.kicad_sch`'s cached `lib_symbols` entries must use the fully-qualified name (e.g.
  `"Device:LED"`, `"Connector_Generic:Conn_01x04"`), not the bare name from the raw
  global library file (`"LED"`, `"Conn_01x04"`). A mismatch here doesn't error —
  `kicad-cli sch erc` segfaults instead, with no useful message. If ERC segfaults on a
  hand-authored schematic, check this first.
- For a symbol placed at rotation 0 (no mirror), pin Y is negated relative to the
  placement point: local pin `(at x y 0)` maps to global `(symbol_x + x, symbol_y - y)`,
  not `+y`. This bit a hand-wired connector where pins 1 and 3 ended up swapped and pin 4
  landed in the wrong place entirely — verify computed pin coordinates against what
  `kicad-cli sch erc`/`pcb drc` actually reports before trusting hand-derived positions,
  don't just trust the formula.
- Symbol `Reference`/`Value` text `(at X Y)` positions are absolute sheet coordinates, not
  relative to the symbol. If 2-pin parts sit on a tight row pitch, a standard ~2.54mm
  above/below text offset can land exactly on the *next* part — ERC won't catch this,
  only a visual check will.
- For physical/mechanical fit checks (e.g. a case light-pipe clearing a connector), use
  the footprint's **courtyard** extent (the plastic housing edge), not the copper pad
  edge — pad-to-pad spacing is only relevant for electrical shorts. Getting this wrong
  gave a false "0.5mm clearance" result when the real housings were actually overlapping.
- A through-hole part's copper plates through **both** layers no matter which side the
  body is mounted on. If a footprint needs to exist on only one face (e.g. to avoid
  landing under a case light-pipe), it needs to be genuinely SMD, not just flipped.
- PCBs for a new board are built via KiCad's bundled `pcbnew` Python scripting
  (`/Applications/KiCad/KiCad.app/Contents/Frameworks/Python.framework/Versions/3.9/bin/python3.9`,
  which has a working `pcbnew` module, unlike the system `python3`) rather than
  hand-written — it reliably handles footprint mirroring
  (`footprint.Flip(center, pcbnew.FLIP_DIRECTION_LEFT_RIGHT)`), net/pad assignment, and
  board-outline geometry that are easy to get subtly wrong by hand. Full regeneration
  from a script wipes any routing/copper-pour the user has added in the GUI since the
  last regen — warn before doing it, and prefer targeted `Edit`-tool text-surgery over
  full regeneration once the user has started routing.

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

## External SD card board interface (J_SD1)

The onboard microSD socket (SD1, Hirose DM3D-SF) is retained on this board, but J_SD1
breaks the same SPI bus out in parallel to a JST pigtail, so the card can be relocated to
a separate daughter board without a further redesign. This is the interface contract
between the two boards; if it changes here, it must change there too. The daughter board
now lives in its own repo, [FN32ROV-XEBook-SD-KiCad](../FN32ROV-XEBook-SD-KiCad) — full-size
push-push SD, side-mounted in the XEBook case — following the same pattern as
[FN32ROV-XEBook-LED-KiCad](../FN32ROV-XEBook-LED-KiCad) (own git repo, own README/CLAUDE.md,
schematic hand-authored, PCB built via `pcbnew` Python scripting — see that project's
CLAUDE.md "Working conventions" for the specific gotchas hit building it, most of which
apply to any new board built the same way).

**Connector:** J_SD1, `Connector_JST:JST_PH_S7B-PH-K_1x07_P2.00mm_Horizontal` — 7-pin JST
PH, 2.0mm pitch (matches the other JST connectors on this board).

**Pinout:**

| Pin | Net | Notes |
|---|---|---|
| 1 | 3V3 | card power |
| 2 | GND | |
| 3 | IO18/SPI_CLK | |
| 4 | IO23/SPI_MOSI | pulled up 10kΩ (R18) on this board |
| 5 | IO19/SPI_MISO | pulled up 10kΩ (R17) on this board |
| 6 | IO5/SPI_CS | pulled up 10kΩ (R19) on this board |
| 7 | IO15/TDO | card-detect (ESP32 GPIO15 doubles as JTAG TDO). Confirmed via netlist trace: this net reaches SD1 pin 9 (`DET_B`), pulled up to 3V3 through R23; `DET_A` (SD1 pin 10) ties to GND. Active low — reads low when a card is present, closing the mechanical switch between `DET_A`/`DET_B`. |

Pull-up resistors (R17/R18/R19, all 10kΩ) live on **this** board, not the SD daughter
board — same pattern as the LED board's current-limiting resistors (R13/R14/R15). The
daughter board should just carry the microSD socket and a matching JST connector, wired
straight through pin-for-pin — no pull-ups needed there.

J_SD1 is wired in parallel to the onboard SD1 socket, not through a mux/switch — only one
card should be inserted at a time. Inserting cards in both sockets simultaneously would
short the SPI bus between two cards.
