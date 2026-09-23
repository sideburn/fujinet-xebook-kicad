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
assembly house and is hand-soldered. **Correction (2026-08-21): PCBWay did fit it on the
1.0 build** -- see the in-case photos. Treat it as assembled, not hand-soldered, when
reasoning about placement-head clearance.

### ⚠️ IDC pin numbers are NOT Atari SIO pin numbers

Dropping SIO pin 6 (the second GND) shifts every pin above it. Off-by-one from IDC pin 6
upward — get this wrong when building the cable and you will feed +5V into a logic pin.
There are **no per-pin silkscreen labels on the board**; the only silk is the reference
designator (the sole exception is TP1, which prints its net name). This table is the
authoritative mapping:

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
| 10 | SIO_AUDIN | **11** | C3 → R8 → ESP32 IO25 (DAC); also brought out to **TP1** |
| 11 | SIO_INT | **13** | U3 pin 8 (4Y, out) |
| 12 | *not connected* | — | see below |

Deliberately dropped from the original: SIO pin 6 (2nd GND) and SIO pin 12 (+12V, unused
by FujiNet). SIO pin 12's absence is fine; the missing second GND is not ideal — the
board's entire supply current returns through IDC pin 4 alone, on a single 0.2mm trace.
**IDC pin 12 is a free floating pad and should be bridged to pin 4 (GND) to give a second
return path** — do this in copper on the next spin, or on the cable for boards already
built.

### First-build finding (2026-08-18): the bench card's cap map was wrong

The XEBook hardwires J_SIO1 to the 130XE by soldering to the SIO **filter caps**, not the
jack pins. `Docs/XEBook-Cable-Bench-Card.pdf` in the fujinet-hardware checkout had four of
the six 1nF control caps rotated. The correct map — traced from the 130XE C103579-001
schematic (PDF vector geometry, cap riser to signal-label row) and confirmed by continuity
at the jack — is simply sequential:

| Cap | SIO pin | Signal | IDC |
|---|---|---|---|
| C305 | 7 | COMMAND | 6 |
| C306 | 8 | MOTOR CONTROL | 7 |
| C307 | 9 | PROCEED | 8 |
| C308 | 10 | READY +5V | 9 |
| C309 | 11 | SIO AUDIO | 10 |
| C310 | 13 | INTERRUPT | 11 |

The four 27pF clock/data caps (C301 CKOUT, C302 CKIN, C303 DATA IN, C304 DATA OUT) were
already correct.

**Both ground wires come off cap legs, not the jack.** Every one of C301-C310 has its far leg
on the same ground node, so IDC 4 (SIO 4) and IDC 12 (SIO 6) are soldered to the *opposite*
end of whichever cap is most convenient — there is no separate ground point to hunt for, and
the two wires give the parallel return path the single 0.2mm pad-4 trace otherwise lacks. Card is now fixed and also marks the minimum wire set (SIO 3, 4, 5, 7, 10)
and the recommended extras (SIO 6, SIO 9).

**The symptom this caused, worth recognising again:** +5V was landed on C310, which is
INTERRUPT — an open-collector line with a pull-up in the computer. A pull-up reads a perfect
5.00V open-circuit on a 10MΩ meter and collapses to ~1.2V under the board's ~0.5mA idle load
(5 x 3.2k/13.2k). The open-circuit reading looks healthy and is meaningless. CMD had landed
on C308, the real +5V. Diagnostic that settles it in seconds: a known-good FujiNet in the SIO
jack running at a solid 5V while the hardwired node reads 1.2V *at the same instant* proves
the two are different nodes.

Note on the pin-12 advice above: bridging IDC 12 to IDC 4 **on the cable** buys almost
nothing, because pad 12 has no copper — both paths still funnel through pad 4's single 0.2mm
trace. To get a real second return on an already-built board, jumper **pad 12 to the GND via
at (76.60, 57.12)** (~7mm away, the closest one) and run pad 12 out to SIO 6.

#### ⚠ Position 12 must never be the only ground — two independent traps

Both connectors have a "pin 12" and **neither of them is a ground**:

- **J1 SIO jack pin 12 is N/C on the 130XE** — verified on C103579-001, where the pin stub
  carries an explicit ✕. It is the unused +12V pin; only the 400/800 ever drove it. Jack pins
  **4 and 6** are the grounds, tied together at a junction and taken to the ground symbol.
- **J_SIO1 pad 12 on this board has no net** (`unconnected-(J_SIO1-Pin_12-Pad12)`) and **zero
  tracks or vias** touch it — verified against the `.kicad_pcb`; the nearest copper of any kind
  is 2.54mm away at pad 9 (`SIO_5V`). The floating pour on the as-built 1.0 boards doesn't
  reach it, and neither does the GND-assigned pour in 1.1 source, because KiCad carves zone
  copper away from pads that aren't on the zone's net.

So the mapping is IDC 12 ↔ SIO **6**, not SIO 12, and it is inert until pad 12 is jumpered.
Wire "pin 12 to pin 12" as a sole ground and the board floats with no return at all — it will
buzz out fine and look correct. **IDC 4 ← jack pin 4 is the ground that actually works;** the
second one is an addition to it, never a substitute.

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
| 3 | LED_BT | GPIO13 (ESP32 LED3) → R14 (30kΩ) |
| 4 | LED_SIO | GPIO4 (ESP32 LED2) → R15 (10kΩ) |

Current-limiting resistors (R13/R14/R15) live on **this** board, not the LED board — the
daughter board should just carry the 3 LEDs, wired common-anode to pin 1 (3V3), with each
LED's cathode returning to its respective signal pin. The ESP32 GPIO sinks current by
driving the pin low to light that LED.

R13/R14/R15 are each sized for that channel's specific LED color/forward-voltage, not a
single shared value:

- R14/R15 were 1kΩ/1.2kΩ in FN32ROV-1.7.1 (blue BT, ~2.9-3.2V Vf; orange SIO, ~2.0-2.2V
  Vf). They are now **30kΩ/10kΩ**, set by a visual brightness match against the WiFi LED
  (see below).
- R13 was changed from 1kΩ to 2.7kΩ because the LED board's D1 (WiFi) uses a **green**
  LED (Rohm SML-P12PTT86R, ~2.2V Vf) instead of the original white (SunLED
  XZBWR68F5MAV-3, ~2.9V Vf). Green is a **deliberate design choice**, not a forced
  substitution — the white part is still available. Green's lower Vf means more of the
  3.3V rail drops across the resistor, so R13 had to grow to keep the current sane (same
  ~0.4mA target as the original white-LED design).

If the LED board's color choices change again, revisit R13/R14/R15 here to keep currents
sane per color.

### Brightness matching — R14 → 30k, R15 → 10k (2026-09-18)

On the first working build (2026-08-18) the channels were visibly mismatched: WiFi (R13
2.7k, green, ~0.41mA) looked about half as bright as BT (R14 1k, blue) and SIO (R15 1.2k,
orange, ~1.0mA). The user is **happy with WiFi's brightness**, so WiFi is the reference and
BT/SIO were dimmed to match it. (An earlier plan here brightened WiFi instead — R13 → 1k —
and is superseded.) A strip of "50% dimming" tape over BT/SIO made them match, which
was used as a clue only; tape is **not** the fix.

**Final values came from a bench match, not calculation.** The LED board was powered off
the machine at 3.3V on pin 1, with WiFi through 2.7k (its real value) and BT/SIO through
trial resistors to GND, compared by eye:

| Channel | Old | New | LCSC | Approx. current |
|---|---|---|---|---|
| WiFi (R13, green) | 2.7k | 2.7k (unchanged) | C413089 | ~0.41mA |
| BT (R14, blue) | 1k | **30k** | C25776 | tens of µA |
| SIO (R15, orange) | 1.2k | **10K** | C25744 | ~0.12mA |

The visual match was 9.2k on SIO. 10K was chosen deliberately (slightly dimmer, and
already a basic part used on this board). Calculated estimates (assuming the tape was a
true 50%) came out around 2.4k for both and were far off — **trust the bench match**.
Blue at 30k runs very close to its knee, so its brightness is sensitive to the actual 3V3
rail. The bench match was done at **3.33V**, close enough to the 3V3 rail that the values
should carry over.

Changed in the schematic, PCB, and `Fab/Gerbers-FN32ROV-XEBook-1.1/` BOM + pick-and-place
(values are not on silkscreen, so gerbers are unaffected). **The as-built 1.0 board still
has 1k/1.2k** — hand-rework it: all three are 0402s in the open row at y=67 — R14 (46.01),
R13 (49.51), R15 (53.51), 3.5mm apart.

Not a concern if R13 is ever changed: GPIO2 is an ESP32 strapping pin and the LED forms a weak
pull-up on it, but the LED clamps that node at 3.3-Vf ~ 1.1V **regardless of the resistor
value**. Only the available current changes, so download-mode strapping behaviour is
unaffected by the swap.

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

- **65 stitching vias added** on a 3mm grid (λ/20 at 2.4GHz on FR4) — **GND vias went
  4 → 69**, and **total vias 112 → 177**. (Verified 2026-08-21 against both the board file
  and the drill files: `as-built-1.0/…-PTH.drl` has 112 holes at 0.300mm, the 1.1 package
  has 177. The boards physically in hand are the 112-via 1.0 build and do **not** carry the
  stitching.) Placement method: take the zone's filled polygon per layer, deflate by
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
| F.Silkscreen | **differs since the 2026-09-18 re-export** — D4-D8 cathode bars, J_SIO1 vertical-header outline |
| B.Silkscreen, F/B Mask, F/B Paste, Edge.Cuts | **byte-identical** |

**Re-exported 2026-09-18** from current source (same commands as below, `--subtract-soldermask`
on). Copper and drill came out identical to the 2026-08-01 1.1 export, confirming the
post-autoroute copper restore is exact; only F.Silkscreen changed. Pick-and-place positions
and rotations are identical to 1.0; only values changed (U1 -E, D7/D8 BEA, R14 30k, R15 10K)
plus J_SIO1's footprint name. The plain BOM is now exported from the schematic
(`kicad-cli sch export bom`, grouped by Value+Footprint).

**`FN32ROV-XEBook-KiCad-BOM-PCBWay.xlsx`/`.csv` is the file to send PCBWay.** It is
hand-mapped (the schematic carries no MPN/LCSC fields) and was verified against the
schematic BOM and pick-and-place (60 placements, 28 lines). It bakes in everything from the
1.0 quote round-trips so they aren't re-queried: same-part designators on one row (PCBWay's
explicit request), the actual purchased MPNs, the higher-voltage caps they substituted
(10V/16V/16V, described as such), J_SIO1 as BOOMELE `2.54-2*6P` (LCSC C9136, vertical), and
R13 as its own line. The JLCPCB BOM/CPL are kept for reference.

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

### TP1 — SIO audio take-off for PokeyMax

Added 2026-09-22. A 2.0mm through-hole pad (1.0mm drill,
`TestPoint:TestPoint_THTPad_D2.0mm_Drill1.0mm`) on `SIO_AUDIN` at **(80.475, 56.475)** —
in the open area right of J_SIO1, centred in a GND stitching-via cell, above the USB-C
connector. Routed back to J_SIO1 pad 10 with three F.Cu segments. Silkscreens `SIO_AUDIN`
(the reference is set not to print). Marked *exclude from BOM* and *exclude from position
files*, so the BOM stays at 30 rows and the pick-and-place at 60 placements — both are
byte-identical to the 1.1 export, and the PCBWay BOM needs no revision.

**Why it exists:** on a PokeyMax-equipped XEBook the amp takes L/R from PokeyMax J2 pins
2/3, bypassing the motherboard mixer, so FujiNet's S.A.M. disk-swap speech only reaches the
speakers if SIO audio is wired into PokeyMax J2 pin 4. Previously that meant tacking a wire
onto the back of the IDC header pin. Note the trap this pad avoids: the wire goes to **IDC
pin 10**, not pin 11 — pin 11 is SIO_INT, and wiring it there gives crackling and popping
with no speech, while a finger-on-the-wire hum test still passes.

Clearances: 0.82mm to the four nearest GND vias, 1.1mm from each via column along the
route. DRC after the change is clean for TP1 (0 violations involving it, 0 unconnected,
0 schematic-parity).

### Deliberate deviations from FN32ROV-1.7.1 (all verified correct)

- **D7/D8** are `PMEG2010BEA,115` in SOD-323, not the original's `PMEG2010ER,115` in
  SOD-123F. Equivalent 20V/1A Schottky; PCBWay's actual-purchase column confirms the
  SOD-323 part, matching the footprint.
- **U2's exposed pad (pad 25) is now grounded**; the original left it floating.
- **All four SD1 shield tabs are grounded**; the original grounded one.
- Dropped: SIO +12V, SIO's 2nd GND, J3 pass-through receptacle, R29 (was NOSTUFF anyway).
- PCBWay substituted higher-voltage caps (10V for C1-C4/C7/C10/C11, 16V for C5/C6/C8/C9
  and C12) — confirmed acceptable on the quote sheet, more headroom, no functional change.

### Build notes (things that will bite at first power-up)

- **DO NOT autoroute this board without explicitly including `/GND`.** On 2026-08-21 a
  rip-up-and-autoroute (Freerouting via the `app_freerouting_kicad-plugin`) silently
  destroyed the entire ground network: **GND track went 392.9mm -> 0.0mm and all 69 GND
  stitching vias were deleted**, while 2529mm of signal routing came back fine. The DSN
  export presents GND as plane-covered, so the router assumes the pour handles it and skips
  the net entirely. Symptom: DRC shows ~33 unconnected items, *all* `/GND`, with every
  signal net clean.
  - **The pour alone is NOT sufficient on this board** and never was. The F.Cu pour fills as
    ~24 islands that do not touch each other; they are tied together by 393mm of GND track
    plus 69 vias down to the B.Cu plane. Re-pouring cannot fix a missing ground net -- a
    re-zone + refill reproduced byte-identical results (1242.5mm2 B.Cu, 28 islands, 33
    unconnected).
  - **Stitching vias cannot rescue an autorouted board either** (tested): restoring all 69
    original via positions gave 42 new errors and *raised* unconnected to 34, because the
    autorouter's B.Cu traces occupy those spots and have carved away the plane underneath.
    Placing only the 51 that still fit moved unconnected 33 -> 32 and produced `via_dangling`.
  - **If you must autoroute:** delete the GND zone first so Freerouting sees `/GND` as an
    ordinary unrouted net, keep existing traces (they import as pre-routed and are skipped),
    then re-create the zone after the SES import. Also set a board-edge clearance in
    Freerouting -- without it the router laid `/EN` tracks against the outline and produced
    two `copper_edge_clearance` errors.
  - **Recovery used:** the copper was restored wholesale from a pre-rip-up snapshot and the
    non-copper fixes reapplied on top (U1 value, D4-D8 cathode bars, J_SIO1 footprint). Back
    to 392.9mm GND track / 69 GND vias / 177 vias total / 24 F.Cu + 2 B.Cu islands /
    **0 unconnected**. J_SIO1 and C12 went back to their original coordinates as part of
    this, which is why the C12 courtyard overlap below is present again.
- **C12 <-> J_SIO1 courtyards overlap by 1.21 x 1.01mm** -- a real `courtyards_overlap`
  DRC error, and the only error on the board. It was *hidden* until the J_SIO1 footprint
  was corrected (the old latch geometry pointed the wrong way and collided with MH2
  instead). **Bodies still clear by 0.25mm** in Y (J_SIO1 F.Fab ends y=55.10, C12 F.Fab
  starts y=55.35), and PCBWay assembled 1.0 without a placement failure, so this is a
  margin problem rather than a fit problem -- deliberately left alone rather than spend
  re-routing on an otherwise-final board. To fix: shift **C12 +1.01mm in Y** (or -1.21mm in
  X); the nearest other X-overlapping neighbours (U5, R16, R25) sit 2.39-2.62mm away, so
  there is room. Two `silk_overlap` warnings between J_SIO1 and C12 come from the same
  tight corner.
- **Silkscreen fixes made after PCBWay's rev-1.0 DFM queries** (both cost email round-trips
  during the 1.0 build; fixed in source, so re-export gerbers before the next order):
  - **D4/D5/D6 (and D7/D8) cathode bars widened 0.12mm → 0.30mm.** The parts were
    assembled correctly, but only after PCBWay queried orientation
    (`Fab/as-built-1.0-.../Edits to PCBway/T-1D22W845207A/位号：D4,D5,D6,确认方向.png`).
    Polarity was never *wrong* — on all five, the bar sits on pad 1 and pad 1 is the
    cathode (D4/D5/D6 are ESD5Z5.0T1G with cathode to USB_D+/USB_D-/USB_5V and anode to
    GND; D7/D8 are PMEG2010BEA OR-ing Schottkys with cathode to `PWR_SW_VCC`). It was
    simply invisible: 0.12mm is under PCBWay's ~0.15mm silk minimum, and it read as one
    side of a body outline rather than a band. Only the cathode-side vertical segment was
    widened; the two long sides stay 0.12mm. Clearance to pad 1 is still 0.17mm. The bar
    was also **extended past the body outline** (SOD-523 y +/-0.60 -> +/-0.85, SOD-323
    +/-0.85 -> +/-1.10) so it reads as a polarity band crossing the outline rather than as
    the corner of a rectangle -- PCBWay's reply was explicitly "the polarity of the
    component we cannot determined", and a hairline flush with the outline was the cause.
  - **J_SIO1 footprint corrected to `Connector_IDC:IDC-Header_2x06_P2.54mm_Vertical`.**
    The fitted part is a **vertical shrouded box header**, not the right-angle latch header
    the old `LocalOverrides:IDC-Header_2x06_P2.54mm_Latch_Horizontal` modelled -- that one
    drew an 18.4 x 34.9mm outline (3.5mm of it past the board edge at x=85) for a connector
    whose pads span 2.5 x 12.7mm. **Pad positions are identical between the two footprints**
    (p1 0,0 / p2 2.54,0 / p11 0,12.7 / p12 2.54,12.7), so the swap cost zero routing. Body
    is now 8.9 x 22.9mm, courtyard 9.9 x 23.9mm, all silk on-board. Updated in both the
    `.kicad_pcb` and the schematic's Footprint field (they must match or parity flags it).
    This also cleared the old spurious `courtyards_overlap` against MH2.
  - Remaining board-wide: **all other F.SilkS lines are still KiCad's default 0.12mm**,
    marginally under PCBWay's minimum. Designators printed fine on 1.0, so this has not
    been changed globally.
- **D4-D8 raise `lib_footprint_mismatch` — intentional**, from the cathode-bar edit above.
  Expect 5 of these in every DRC run and in the GUI. Do **not** "fix" them by reloading
  from library; that reverts the bars and re-opens the PCBWay query. (Note: the width
  change alone did not trip the check — KiCad appears to compare shape endpoints but not
  stroke width — only extending the bar's endpoints did. So a width-only silk tweak is
  invisible to this check, which is worth knowing before relying on it.) If the warnings
  ever become noise, the fix is to copy the two footprints into
  `libraries/LocalOverrides.pretty` as modified variants and re-point D4-D8 at them,
  matching how J_SIO1's IDC header is already handled — at the cost of changed footprint
  names in the BOM/CPL footprint column.
- **U1 antenna — boards fabbed before 2026-08-21 are ESP32-WROVER-**IE**-N16R8** (LCSC
  `C701352`) and have **no working PCB antenna**; a U.FL/IPEX pigtail is mandatory or WiFi
  is unusably weak (the meander is etched on the module but not connected). The BOM now
  specifies **ESP32-WROVER-E-N16R8** (LCSC `C529589`), matching FN32ROV-1.7.1. The board
  already carries an `antenna keepout` rule area (F.Cu+B.Cu, x 41.25–59.25, y 22.56–28.86,
  copper/tracks/vias/pads/footprints all barred) plus an Edge.Cuts notch under the module's
  antenna end, so **no layout change was required** — only U1's value and the LCSC number.
- **Converting an existing -IE board to its PCB antenna is a 0402 move, not a jumper.** Per
  the ESP32-WROVER-E/-IE datasheet reference designs (pp. 38–39), `R15` (ANT1 → `PCB_ANT`)
  and `R14` (ANT2 → `J39`, the IPEX connector) are the antenna select: the -E fits R15 with
  R14 `0(NC)`, the -IE the reverse. On the module, R14 is the populated 0Ω nearest the U.FL
  and R15 the empty pad toward the meander. **Move the resistor — never bridge both**, or
  the two antennas sit in parallel on one feed and the match is wrecked. The rest of the RF
  chain (`L5` 2.0nH, `C15`/`L4`/`C14`) is identical in both reference designs, and both
  carry the same note that those values "vary with the actual PCB board" — so the sometimes-
  repeated claim that the pi network differs between -E and -IE is not supported by
  Espressif's own docs. Use a fine-tip iron, not hot air (J39's plastic body deforms).
  Voids the module's FCC/CE/NCC marks.
- **J_PWR1 must be switched or jumpered or the board is dead.** R22 (10k) holds U5's CE
  low; with J_PWR1 open the 3V3 regulator never enables. This is the original's S5 slide
  switch moved off-board, not a defect.
- Track widths are 0.2mm everywhere including the 5V/3V3/GND power path — about 0.7A at
  10°C rise on 1oz outer copper, adequate for FujiNet's ~350mA peak but with no margin.
- Verify pin-1 orientation of U2 (QFN-24), D7/D8 and C12 (tantalum) against silkscreen when
  the boards arrive; the CPL exports raw KiCad rotations.
- LED brightness was matched on 2026-09-18: **R14 → 30k, R15 → 10K**, R13 stays 2.7k. The
  1.0 boards still carry 1k/1.2k and need hand-rework; see "External LED board interface
  (J_LED1)", "Brightness matching".

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
