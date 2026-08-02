# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## What this repository is

A from-scratch KiCad 9 rebuild of the FujiNet FN32ROV WiFi/SIO adapter (originally a
DipTrace design in `FujiNetWIFI/fujinet-hardware`), adapted for permanent internal
mounting inside a custom "XEBook" laptop build. See `README.md` for the full description
and what differs from the original FN32ROV-1.7.1.

Status: **ordered** — rev 1.0 fabricated and assembled by **PCBWay** (order
`T-1D22W845207A`, 5 units, 2026-07-31) from `Fab/Gerbers-FN32ROV-XEBook-1.0/`.

⚠️ **The three boards went to two different fab houses.** This board is the only one at
PCBWay. The **LED and SD daughter boards were ordered separately at JLCPCB and have already
shipped** — they are past the point of any file revision, and their rev 1.1 fab packages are
for a *future* order only. Only this main board is potentially still updatable, and only if
PCBWay's change window is open.

See "Design review" below for what the post-order audit found; the source files have since
diverged from the as-ordered Fab package on purpose.

## Board family

This board is the hub of a small family of boards, all derived from or built to interface
with the original FujiNet FN32ROV design. Each lives in its own sibling folder under
`Development/KiCad/`, with its own git repo, README, and (where noted) CLAUDE.md:

- **`FN32ROV-XEBook-KiCad`** (this repo) — the main WiFi/SIO adapter board, mounted
  internally in the XEBook laptop.
- **[`FN32ROV-XEBook-LED-KiCad`](../FN32ROV-XEBook-LED-KiCad)** — daughter board carrying
  the 3 status LEDs, off-board via J_LED1. See "External LED board interface" below.
- **[`FN32ROV-XEBook-SD-KiCad`](../FN32ROV-XEBook-SD-KiCad)** — daughter board carrying a
  relocatable full-size microSD socket, off-board via J_SD1. See "External SD card board
  interface" below.
- **[`XEBookButtonBar`](../XEBookButtonBar)** — panel-mount board carrying the tactile
  buttons (volume up/down, plus the three FujiNet control buttons: swap, BT, reset),
  off-board via this board's J_BTN1. See "External button bar interface" below. This is
  an older board (pre-dates the LED/SD daughter-board pattern and the two-tier
  verification workflow). Its J2 connector has been upgraded from 3 pins to a proper
  4-pin GND-inclusive header matching J_BTN1 (previously an outstanding mismatch, now
  fixed — see that board's own CLAUDE.md for details); the new pin layout still needs
  copper routed by hand before the next fab run.

The original DipTrace design this whole family descends from
(FN32ROV-1.7.1, officially OSHWA-certified, UID US000651) lives in a separate,
unrelated repo: the "Atari 8-Bit XE Book PCB's" / `fujinet-hardware` checkout
(`ATARI/FN32ROV-1.7.1/`). That repo's own CLAUDE.md documents it as a hardware-only,
DipTrace-source repository with no relationship info about this KiCad family — treat this
file as the authoritative source for how the derived boards relate to each other.

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

## Host SIO interface (J_SIO1)

The original FN32ROV-1.7.1 carried two full Atari SIO connectors — J1 (plug,
`7-745288-2`) and J3 (receptacle, `AT60-202-2031`) wired in parallel for daisy-chaining
external SIO peripherals. Both are gone here, replaced by a single low-profile 12-pin
right-angle IDC header hardwired to the XEBook's motherboard.

**Connector:** J_SIO1, `LocalOverrides:IDC-Header_2x06_P2.54mm_Latch_Horizontal` — 2x6,
2.54mm pitch, through-hole. No LCSC part number in the BOM; it is **not** placed by the
assembly house and is hand-soldered.

### ⚠️ IDC pin numbers are NOT Atari SIO pin numbers

Dropping SIO pin 6 (the second GND) shifts every pin above it. Off-by-one from IDC pin 6
upward — get this wrong when building the cable and you will feed +5V into a logic pin.
There are **no per-pin silkscreen labels on the board**; the only silk is the reference
designator. This table is the authoritative mapping:

| IDC pin | Net | Atari SIO pin | Direction / notes |
|---|---|---|---|
| 1 | SIO_CKIN | 1 | U4 pin 2 (74LS07 1Y, open-collector out) |
| 2 | SIO_CKOUT | 2 | U4 pin 3 (2A, in) |
| 3 | SIO_DATAIN | 3 | U4 pin 6 (3Y, out); R16 4.7k pull-up to SIO_5V |
| 4 | GND | 4 | **the only ground pin** |
| 5 | SIO_DATAOUT | 5 | U4 pin 9 (4A, in) |
| 6 | SIO_CMD | **7** | U3 pin 1 (1A, in); R12 10k pull-up on the IO39 side |
| 7 | SIO_MCTL | **8** | U3 pin 3 (2A, in); R24 2k pull-down |
| 8 | SIO_PROC | **9** | U3 pin 6 (3Y, out) |
| 9 | SIO_5V | **10** | board supply in — C12 47uF bulk, D8 OR-ing diode, R4, R16 |
| 10 | SIO_AUDIN | **11** | C3 → R8 → ESP32 IO25 (DAC) |
| 11 | SIO_INT | **13** | U3 pin 8 (4Y, out) |
| 12 | *not connected* | — | see below |

Deliberately dropped from the original: SIO pin 6 (2nd GND) and SIO pin 12 (+12V, unused
by FujiNet). SIO pin 12's absence is fine; the missing second GND is not ideal — the
board's entire supply current returns through IDC pin 4 alone, on a single 0.2mm trace.
**IDC pin 12 is a free floating pad and should be bridged to pin 4 (GND) to give a second
return path** — do this in copper on the next spin, or on the cable for boards already
built.

### Mechanical

J_SIO1's courtyard overlaps mounting hole MH2 (this is the board's one DRC error, and it
is expected). The plastic shroud itself clears MH2 — it's the ejector-latch arms, which
sit above the board, that sweep over it. A flat M2 screw head at MH2 is fine; a standoff
is not. The connector body also overhangs the board's right edge by about 3.4mm (footprint
extends to x≈88.4mm against an 85mm edge), which is normal for an edge-mounted right-angle
header but matters for case fit.

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

## External button bar interface (J_BTN1)

Panel-mount tactile buttons are not on this board — they connect off-board via a cable to
[XEBookButtonBar](../XEBookButtonBar), an older board that predates the LED/SD
daughter-board pattern above. This is the interface contract between the two boards; if
it changes here, it must change there too. Unlike J_LED1/J_SD1, the far end of this cable
isn't a mating JST-PH housing — XEBookButtonBar's J2 is a plain 4-pin through-hole header
with the wires soldered directly to the pins.

**Connector:** J_BTN1, `Connector_JST:JST_PH_S4B-PH-K_1x04_P2.00mm_Horizontal` — 4-pin
JST PH, 2.0mm pitch (matches the other JST connectors on this board).

**Pinout:**

| Pin | Net | ESP32 pad (U1) | Notes |
|---|---|---|---|
| 1 | GND | | |
| 2 | BTN_SWAP | pad 25 (IO0) | 10kΩ pull-up (R6). IO0 is a boot-strapping pin — Q1 (UMH3NTN) sits on this net, likely to isolate the button from strapping during power-up; verify in schematic before changing this net. |
| 3 | BTN_BT | pad 6 (IO34) | 10kΩ pull-up (R1). IO34 is input-only (no internal pull-up on the silicon), hence the external one. |
| 4 | BTN_RST | pad 13 (IO14) | 10kΩ pull-up (R3). |

All three buttons are active-low (pulled up on this board, switch on the button bar pulls
to GND).

### Resolved: XEBookButtonBar's J2 upgraded to a 4-pin GND-inclusive connector

XEBookButtonBar used to carry the 3 FujiNet buttons (reset, swap, BT) on **J2**
(schematic value "fuji") as a 3-pin `PinHeader_1x03_P2.00mm_Vertical` with **no GND
pin** — those buttons borrowed their ground return through J1's cable instead of
carrying their own, which only worked as long as J1 stayed connected, and didn't match
J_BTN1's own GND pin here.

This has been fixed: J2 is now a 4-pin `PinHeader_1x04_P2.00mm_Vertical` (pin 1 = GND,
pin 2 = BTN_SWAP, pin 3 = BTN_BT, pin 4 = BTN_RST), matching J_BTN1's pinout 1:1 for a
straight, non-crossed cable. A plain pin header was used rather than a mating JST-PH
connector (unlike J_LED1/J_SD1) — the user solders cable wires directly to the pins on
both this board and XEBookButtonBar, so no mating housing is needed on the small board.
See [XEBookButtonBar](../XEBookButtonBar)'s own `CLAUDE.md` for implementation details
and gotchas hit doing this (notably: button-to-pin identity had to be confirmed against
the PCB's silkscreen labels, since it didn't match the naive pin-order assumption).

ERC and `kicad-cli pcb drc --schematic-parity` are clean (no shorts) on that board, but
the copper routing for the new 4th pin still needs to be finished by hand in the KiCad
GUI before the next fab run — J1 ("vol") was unaffected and needed no changes.

## Design review (2026-07-31, post-order)

A full audit was run against the original DipTrace design after the boards were ordered.
Method — both tiers, per "Working conventions":

1. `kicad-cli sch export netlist` on this schematic, diffed node-by-node against
   `reference/FN32ROV-1.7.1_DipTrace_netlist.net` (identical to `Docs/netlistExport.net`
   in the fujinet-hardware checkout; DipTrace 5.2.0.4 export of `FN32ROV-1.7.1.dch`).
2. `kicad-cli pcb drc --schematic-parity`, plus a byte-diff of regenerated gerbers against
   the as-ordered `Fab/` package.

**Result: the netlist is a faithful, pin-for-pin copy of FN32ROV-1.7.1.** All 60
carried-over components sit on identical nets — every ESP32 pad, all 12 74LS07 SIO buffer
gates (correct direction and pull-up each way), the Q1 DTR/RTS auto-reset circuit, the
Q2/Q3 VCC_BUF high-side switch, D7/D8 OR-ing orientation, C12 tantalum polarity, USB-C
D+/D- and CC1/CC2, and the SD bus. The as-ordered gerbers match the `.kicad_pcb` they were
generated from (copper differed by two zone-fill vertices, i.e. refill nondeterminism).

### Fixed in source, NOT in the ordered boards: the copper pour had no net

The single zone spanning `F.Cu`/`B.Cu` was `(net 0) (net_name "")` — a **floating** pour,
not a ground plane: 1548mm² top (42% of board) and 2201mm² bottom (60%), confirmed present
in the fabricated gerbers (98 + 23 `G36` regions). GND was carried entirely by 0.2mm
traces. KiCad's `isolated_copper` rule only fires on orphaned islands of a *netted* zone,
so nothing flagged it.

The zone is now assigned to `/GND` and refilled (done via the bundled `pcbnew` module, not
text surgery, so the fill is real).

#### Follow-up completed 2026-08-01 — plane is now finished in source

The initial net assignment left 81 DRC violations. All the pour-related ones are now
resolved; the board is **DRC-clean apart from silkscreen warnings and the known MH2
courtyard overlap** (81 → 8 violations, 0 unconnected).

- **65 stitching vias added** on a 3mm grid (λ/20 at 2.4GHz on FR4) — GND vias went
  **4 → 177**. Placement method: take the zone's filled polygon per layer, deflate by
  0.75mm (via radius 0.3 + clearance 0.2 + 0.25 safety), intersect F.Cu with B.Cu, and grid
  points inside the result. The filled pour is *already* carved back from every track, pad
  and board edge by DRC clearance, so anything inside the deflated polygon is inherently
  clearance-safe — this is why zero new clearance errors appeared. Also enforced 1.6mm
  minimum centre-to-centre against existing vias. Script: scratchpad `stitch.py`.
- **`island_removal_mode` set to "Always"** — this, not the vias, is what cleared the 67
  `isolated_copper` warnings. Deflating by 0.75mm shrinks a small island to nothing, so no
  via candidate ever lands in one; they had to be deleted rather than connected. Pour went
  1760 → 1624mm² F.Cu and 2269 → 2095mm² B.Cu; that delta is floating scrap copper. 0
  unconnected items confirms nothing load-bearing was removed.
- **6 `starved_thermal` errors cleared.** All read `min spoke count 2; actual 1` — the pads
  were connected, just by one spoke. Fixed two ways: `min_resolved_spokes` lowered 2 → 1 in
  `.kicad_pro` (KiCad's default of 2 targets 4-layer plane designs; one 0.5mm spoke carries
  ~1.5A), **and** `D4` pad 2 and `C4` pad 1 set to solid zone connection, since thermal-relief
  inductance is actively counterproductive on an ESD diode and a decoupling cap.
- Do **not** tighten `thermal_gap` to fix starved thermals: tested at 0.3mm and it
  introduced 4 new `hole_clearance` errors while still leaving 4 starved thermals. Leave
  it at 0.5mm.

Nothing moved to achieve any of this — verified after each step: all 64 footprint positions
byte-identical, reference list identical, track segments 894 → 894, board outline
66.050 × 56.050mm unchanged to 0.1µm. The mounting brackets were already fabricated, so
component positions and board size are **hard-frozen**; any future pour work must stay
additive in the same way.

Note on testing zone changes: run DRC on the file **in place**, not on a copy in a scratch
directory. A copy outside the project dir doesn't load `.kicad_pro` severity overrides or
the project fp-lib-table, which spuriously surfaces 4 `hole_clearance` errors (internal to
the J2 USB-C footprint — its own GND pads vs its own NPTH pegs, 0.185mm) plus
`lib_footprint_issues`, and silently drops schematic-parity checking.

**The ordered boards are electrically complete — the floating pour is not a defect that
stops them working.** Re-verified 2026-08-02 against the as-built state: **0 unconnected
items**, all **59 GND pads** tied together by **188 track/via segments**. Because the zone
was net-less during layout, KiCad's ratsnest forced GND to be fully routed in copper, so
the pour was never load-bearing. The same holds for the shipped SD boards (0 unconnected,
9 GND pads, 31 segments). What 1.1 buys is EMI/return-path quality, not function.

(The as-built gerbers were confirmed to match git `HEAD`'s board file: the F.Cu geometry
differs by exactly 2 coordinate lines out of 11346, i.e. zone-fill nondeterminism. So `HEAD`
can be trusted as the as-built reference without re-deriving it from the gerbers.)

A manual GND rework on the 5 ordered boards is therefore **optional**. The closest bridge points are GND
vias sitting exactly 0.80mm from the pour edge (zone clearance + annulus, i.e. as close as
the geometry allows) on **both** layers from the same via: **(76.60, 57.12)** — about
8.8mm from C12, 9.6mm from D5. Three more GND vias are available at (22.26, 55.15) near
R27/R28, and (51.21, 30.00) and (51.21, 39.81) alongside U1. Note a single-point tie only
removes the floating-conductor problem; it does not create a return-current plane.

### Fab folder layout (restructured 2026-08-01)

```
Fab/
  as-built-1.0-PCBWay-T-1D22W845207A/   <- FROZEN. Exactly what PCBWay was sent.
    Gerbers-FN32ROV-XEBook-1.0/         Never regenerate or edit anything in here.
    FN32ROV-XEBook-1.0-Gerbers.zip
    Edits to PCBway/                    PCBWay quote + part-substitution correspondence
  Gerbers-FN32ROV-XEBook-1.1/           <- new revision, grounded plane
  FN32ROV-XEBook-1.1-Gerbers.zip        <- 12 files, same set as the 1.0 zip
```

The 1.0 tree is the as-built record and no longer matches the source files; that is
intentional. The SD board's `Fab/` follows the same pattern but archived under
`as-built-1.0-JLCPCB/` (different fab house, no PCBWay order number applies) plus
`Gerbers-FN32ROV-XEBook-SD-1.1/`. The LED board was **not** revised — its source is
unchanged, so its `Fab/` still matches and stays flat.

**All three 1.1 packages are now future-order-only.** PCBWay answered on **2026-08-02** that
the 1.0 boards were already in manufacturing and the change window had closed, so the swap
did not happen. The SD and LED rev 1.0 boards had already shipped from JLCPCB. Nothing in
`Gerbers-FN32ROV-XEBook-1.1/` has ever been fabricated — do not describe it as built.

**1.1 vs 1.0, verified layer by layer (timestamps ignored):**

| Layer | Result |
|---|---|
| F.Cu / B.Cu | **differ** — repoured GND plane, island removal, +65 vias |
| PTH drill | **differ** — 145 → 210 hits (exactly +65) |
| NPTH drill | identical (6) |
| F/B Mask, F/B Paste, F/B Silkscreen, Edge.Cuts | **byte-identical** |

Copper and drill only. Everything else is bit-for-bit what PCBWay already has, which is the
argument to make to them: this is a bare-board copper revision, not an assembly change.
**BOM and CPL are unchanged** — they were copied verbatim from the 1.0 folder rather than
regenerated, deliberately, because PCBWay already approved those exact files including their
part substitutions.

#### Export settings — the two boards are NOT the same, verify before trusting a re-export

`kicad-cli pcb export gerbers` defaults match 1.0 for protel extensions, X2, netlist
attributes, absolute origin (no aux) and precision 6. **But soldermask subtraction differs
per board**, and the boards' stored `pcbplotparams` do not reflect what was actually
exported (`usegerberextensions no` is stored on the main board, yet 1.0 shipped protel
`.gtl/.gbl` — so do **not** use `--board-plot-params`):

- **Main board: pass `--subtract-soldermask`.** 1.0 used it — its silkscreen has `%LPC`
  clear-polarity with 276 pad flashes knocking silk out from over pads.
- **SD board: do NOT pass it.** 1.0 has `%LPC`=0, no pad flashes.

Getting this wrong changes the silkscreen layers while leaving everything else correct, and
it is easy to miss. The check: `grep -c '%LPC'` on the F_Silkscreen file — 1 means
subtraction was used, 0 means it wasn't. Always diff a fresh export against the as-built 1.0
files (filtering `CreationDate` and `Created by KiCad` lines) and confirm only F.Cu, B.Cu
and the PTH drill differ.

Full commands used for 1.1:

```
L="F.Cu,B.Cu,F.Paste,B.Paste,F.Silkscreen,B.Silkscreen,F.Mask,B.Mask,Edge.Cuts"
kicad-cli pcb export gerbers -o <out>/ -l "$L" [--subtract-soldermask] <board>.kicad_pcb
kicad-cli pcb export drill   -o <out>/ --format excellon --excellon-separate-th \
                             --generate-map --map-format gerberx2 <board>.kicad_pcb
```

The drill map files (`*-drl_map.gbr`) are generated for reference but **excluded from the
zip** — the 1.0 zip contained 12 files and didn't include them.

### Deliberate deviations from FN32ROV-1.7.1 (all verified correct)

- **U1 is ESP32-WROVER-**IE**-N16R8** (LCSC `C701352`), not the original's WROVER-E. Same
  pads, but the IE has **no PCB antenna** — a U.FL/IPEX pigtail antenna is mandatory or
  there is no WiFi.
- **D7/D8** are `PMEG2010BEA,115` in SOD-323, not the original's `PMEG2010ER,115` in
  SOD-123F. Equivalent 20V/1A Schottky; PCBWay's actual-purchase column confirms the
  SOD-323 part, matching the footprint.
- **U2's exposed pad (pad 25) is now grounded**; the original left it floating.
- **All four SD1 shield tabs are grounded**; the original grounded one.
- Dropped: SIO +12V, SIO's 2nd GND, J3 pass-through receptacle, R29 (was NOSTUFF anyway).
- PCBWay substituted higher-voltage caps (10V for C1-C4/C7/C10/C11, 16V for C5/C6/C8/C9
  and C12) — confirmed acceptable on the quote sheet, more headroom, no functional change.

### Build notes (things that will bite at first power-up)

- **J_PWR1 must be switched or jumpered or the board is dead.** R22 (10k) holds U5's CE
  low; with J_PWR1 open the 3V3 regulator never enables. This is the original's S5 slide
  switch moved off-board, not a defect.
- Track widths are 0.2mm everywhere including the 5V/3V3/GND power path — about 0.7A at
  10°C rise on 1oz outer copper, adequate for FujiNet's ~350mA peak but with no margin.
- Verify pin-1 orientation of U2 (QFN-24), D7/D8 and C12 (tantalum) against silkscreen when
  the boards arrive; the CPL exports raw KiCad rotations.
- LED currents are ~0.4-1.1mA by design (matching the original), which is dim. If the
  status LEDs are hard to see through the case light pipes, drop R13/R14/R15 — they are
  0402s in an accessible row at y=67 on this board.

### Known non-issues (checked, don't re-investigate)

- **SD1 card detect**: pad numbers differ between tools — DipTrace's DM3D symbol numbers
  the switch terminals `SW_B`=9 / `SW_A`=14 with `CASE` on 11-13, KiCad's `DM3D-SF`
  footprint numbers them 9/10 with shield on 11. Topology is identical either way (one
  terminal to GND, the other pulled up through R23 to IO15) because the detect switch is a
  plain mechanical SPST — orientation is irrelevant.
- **U3/U4 are drawn with the `74xx:74LCX07` symbol** while the fitted part is SN74LS07.
  Pinouts are identical (1A/1Y/2A/2Y/3A/3Y/GND/4Y/4A/5Y/5A/6Y/6A/VCC); cosmetic only.
- **ERC `label_dangling` on `IO12/TDI`** — U1 pad 14 is a single-node net in the original
  design too. Matches upstream; leave it.
- **ERC `lib_symbol_mismatch` on `Q_NMOS`/`Q_PMOS`** — cached pin numbering verified as
  G=1/S=2/D=3, correct for SOT-23. Cosmetic.
