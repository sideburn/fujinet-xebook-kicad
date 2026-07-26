# FN32ROV XEBook KiCad

A KiCad rebuild of the [FujiNet](https://fujinet.online/) FN32ROV WiFi/SIO adapter for
Atari 8-bit computers, adapted for permanent internal mounting inside an XEBook laptop
build.

This project is derived from the FN32ROV-1.7.1 release in
[FujiNetWIFI/fujinet-hardware](https://github.com/FujiNetWIFI/fujinet-hardware), which is
officially certified open source hardware by [OSHWA](https://oshwa.org) (UID
[US000651](https://certification.oshwa.org/us000651.html)). The original design is in
DipTrace; this project is a from-scratch rebuild in KiCad, since DipTrace's binary file
format has no viable automated conversion path into KiCad and this revision changes
enough of the board that preserving the original layout wasn't a goal.

## Status

Work in progress — schematic and PCB layout are still being built out. Not yet fabricated
or tested.

## What's different from FN32ROV-1.7.1

- The two external Atari SIO connectors (SIO plug + SIO receptacle pass-through) are
  replaced with a single low-profile right-angle 12-pin IDC header, hardwired directly to
  the host motherboard rather than daisy-chaining external SIO peripherals.
- The three status LEDs and three tactile buttons move off the PCB onto JST pigtails, so
  they can be panel-mounted separately from the board.
- The onboard microSD socket is retained, with an added parallel breakout header so the
  card can be relocated to an external board later without a further redesign.

Everything else (ESP32-WROVER-E, CP2102N USB-serial bridge, power path, SIO level
shifters) is carried over from the original design.

## Tooling

Built with KiCad 9. `kicad-cli sch erc` / `kicad-cli pcb drc` are used to verify the
design as it's built.

## License

[CERN Open Hardware Licence Version 2 - Strongly Reciprocal](LICENSE) (CERN-OHL-S-2.0),
matching the copyleft, share-alike spirit of the original OSHWA-certified design.
