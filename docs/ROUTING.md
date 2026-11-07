# AIR MOUSE PCB: how the routing works

This guide explains every route on the board: where it goes, which layer it uses, and why it takes that path.

To follow along, open `AIR_MOUSE.kicad_pcb` in KiCad. Click any track and press `` ` `` (backtick) to highlight its whole net.

| Picture | What it shows |
|---|---|
| [route-map.png](route-map.png) | Red = F.Cu tracks. Blue = B.Cu tracks. Yellow = F.Cu pads. Cyan = B.Cu pads (U4). Faded colours = GND copper. |
| [board-top.png](board-top.png) | 3D render with the silkscreen labels. The ESP32 has no 3D model, so you see its pads. |

## The ideas behind the layout

1. **Top layer for routes, bottom layer for ground.** Almost everything runs on F.Cu. B.Cu is a nearly unbroken GND plane, which also gives D+/D− a solid reference.
2. **Blue means a hop.** A B.Cu section exists only to pass under other tracks. Every hop is listed in [All B.Cu hops](#all-bcu-hops).
3. **Wider tracks carry more current.** Power tracks are 0.5 mm, 3.3 V is 0.4 mm, signals are 0.25 mm. The 0.2 mm tracks are the USB pair and the U4 escapes.
4. **The USB pair splits the board.** D+/D− run from J1 at the bottom to U6, so the left and right halves can only connect through a hop. This explains most of the hops.
5. **U4 is on the back.** The LSM6DSL sits on B.Cu, so each used pin has a short B.Cu escape and a via up to F.Cu.

Coordinates below are in mm, as shown in KiCad. A dead zone at the top of the board (y < 104.5) is the antenna keepout: no copper there on either layer.

## Net names

KiCad's automatic names are not very readable, so the silkscreen and this guide use these:

| Name used here | KiCad net | Meaning |
|---|---|---|
| VBUS | `Net-(D1-A)` | 5 V from USB-C |
| VBAT | `Net-(J2-Pin_1)` | Battery positive |
| VBAT_SW | `Net-(SW1-B)` | Battery after the power switch |
| 3V3 | `3.3_power` | 3.3 V regulated rail |
| LEFT | `Net-(U6-IO7)` | Left mouse button, SW5 |
| RIGHT | `Net-(U6-IO8)` | Right mouse button, SW4 |
| MID | `Net-(U6-IO9)` | Middle mouse button, SW3 |
| BOOT | `Net-(U6-IO0)` | Boot button, SW2 |
| EN | `Net-(SW6-C)` | ESP32 enable / reset, SW6 |
| CC1, CC2 | `Net-(J1-CC1)`, `Net-(J1-CC2)` | USB-C configuration pins |
| PROG | `Net-(U1-PROG)` | Sets the charge current |
| STAT | `Net-(U1-STAT)` | Charger status output |
| LED | `Net-(D1-K)` | Link between the LED and its resistor |
| SCL, SDA, INT1, D+, D−, GND | same | Same name in KiCad |

## Power path

Follow the energy: USB-C → charger → battery → switch → regulator → everything else.

```
J1 ── VBUS ──┬── C1
             ├── D1 ── LED ── R4 ── STAT ── U1 pin 1
             └── U1 pin 4 (charger input)
                  U1 pin 5 ── PROG ── R3 ── GND
                  U1 pin 3 ── VBAT ──┬── C2
                                     ├── J2 (battery)
                                     └── SW1 pin 1
                                          SW1 pin 2 ── VBAT_SW ──┬── C3
                                                                 └── U3 pins 1, 3
                                                                      U3 pin 5 ── 3V3 ── rest of board
```

### VBUS (0.5 mm)
- **Start:** J1's two VBUS pin groups each go straight to a via, at (117.55, 166.35) and (122.45, 166.35). The vias are joined on B.Cu, because the USB signal pins sit between them.
- **Up to C1:** from the left via, a straight F.Cu track runs up x = 117.55 to y = 157.
- **Along y = 157:** a straight line links C1 pin 1 on the right to D1's anode on the left.
- **To the charger:** at x = 115.6 a branch runs up to U1 pin 4.

### VBAT (0.5 mm)
- **Battery to charger:** J2 pin 1 runs diagonally to y = 150.25, then right to the vertical at x = 112.36. That vertical joins U1 pin 3 and C2 pin 1.
- **Charger to switch:** from U1 pin 3 it passes under U1's body, between pins 4 and 5, then along y = 148. It hops under D+/D− between (121.3, 148) and (123.15, 148).
- **Final stretch:** up x = 123.15, along y = 142.7 above C3, then diagonally into SW1 pin 1.

### VBAT_SW (0.5 mm)
- From SW1 pin 2, straight left along y = 146, then along y = 145.45 through the gap between C3 and U3.
- Short branches drop to C3 pin 1 and U3 pin 1.
- A loop down x = 125.2 reaches U3 pin 3. U3's pins 1 (input) and 3 (enable) are tied together so the regulator is on whenever the switch is on.

### 3V3 (0.4 mm)
The rail starts at U3 pin 5 and has one main trunk. Read it in this order:

1. **U3 to C4:** from pin 5, right to x = 129.9, down, and along y = 150.5 to C4 pin 1.
2. **Up to the trunk:** a hop from (129.9, 147.05) to (129.9, 141.7) passes under VBAT_SW and VBAT.
3. **The trunk:** up x = 128.2, in the gap between D+ and SW4, to y = 119.65.
4. **To R5:** the trunk continues diagonally to x = 130.45, then up to R5 pin 1 (the EN pull-up).
5. **Into the middle:** a hop at y = 125.1 passes under D+/D− to a via between R6 and R7. A short vertical joins both pull-ups.
6. **To U4:** from R7 along y = 127.3 to U4's via at (120.5, 126.9), then round the south side on y = 128 to the via at (118, 125.75).
7. **To C7/C8:** from (118, 125.75) along y = 125.1, between the two capacitors' GND pads. A vertical at x = 114.625 joins both capacitors.
8. **To the left side:** a hop from (109.6, 125.1) to (104.175, 125.1) passes under MID, RIGHT and LEFT.
9. **The pull-up column:** a straight vertical at x = 104.175 joins pin 1 of R11, R10, R9 and R8, and carries on through the middle of SW2.
10. **To the ESP32:** a hop from (105.5, 111.2) to (105.5, 109.6) passes under BOOT, then runs diagonally to C6 pin 1 and on to U6 pin 3.

U4's third 3V3 pin (pin 12) joins through a short B.Cu link from via (118.8, 122.9) to via (118, 125.75).

### Charger support
- **PROG (0.25 mm):** one diagonal from U1 pin 5 to R3 pin 1. R3 (2 kΩ) sets the charge current.
- **STAT (0.25 mm):** left from U1 pin 1, down x = 110, a hop under VBAT from (110, 149.4) to (110, 151), then down x = 110.9 to R4 pin 1.
- **LED (0.25 mm):** one short diagonal from D1's cathode to R4 pin 2.

## USB

### D+ and D− (0.2 mm, side by side)
The pair stays on F.Cu all the way, with no vias. D+ is always the right-hand track when looking up the board.

1. **At J1:** USB-C has each signal twice so the plug works both ways round. A7 links to B7 below the pads, and A6 links to B6 above them.
2. **Past C1:** diagonal to x = 122 / 122.45, passing to the right of C1.
3. **Middle of the board:** straight up to y ≈ 141.7.
4. **Between the buttons:** diagonal to x = 126.6 / 127.05, then straight up between SW3 and SW4, to the right of R6/R7.
5. **Into the ESP32:** left along y = 120.75 (D−) and y = 120.3 (D+), under the module's bottom row, ending at U6 pin 23 (D−) and pin 24 (D+).

The pair goes around SW3, not between its pads, because SW3's two MID pads need a link across the switch.

### CC1 and CC2 (0.25 mm)
These tell the USB host that this board is a device.
- **CC1:** from J1 A5, a hop under VBUS from (118.3, 165.5) to (116.7, 165.5), then left along y = 165.5 to R1 pin 1.
- **CC2:** from J1 B5, right along y = 165.05 to R2 pin 1.

R1 and R2 are 5.1 kΩ to GND.

## IMU bus: SCL, SDA, INT1 (0.25 mm)

These three start on U6's left side and end around U4. They must get past the button tracks leaving U6 just below them, so each takes a hop right at the module.

| Net | U6 pin | Hop on B.Cu | Then on F.Cu | Ends at |
|---|---|---|---|---|
| SCL | 8 | (111.85, 112.3) → (110.55, 118.6) | diagonal, along y = 121.75 | U4 via (120, 122.7), then y = 123.6 to R6 pin 1 |
| SDA | 9 | (111.85, 113.15) → (111.2, 118) | diagonal, along y = 121.25 | Branch at x = 121.2 to U4 via (121.2, 122.9); carries on to x = 124.8, between R6's pads, to R7 pin 1 |
| INT1 | 10 | (111.85, 114) → (111.85, 117.4) | around the west side of the other two, along y = 122.6, down x = 117.8, along y = 124.75 | U4 via (122, 125.75) |

R6 and R7 (4.7 kΩ) are the I²C pull-ups to 3V3.

## Buttons (0.25 mm)

Each button line does two jobs: it connects to a 10 kΩ pull-up and to its switch. Each switch has two pads for the same pin, so a straight link joins them.

| Label | U6 pin | Pull-up | Route |
|---|---|---|---|
| BOOT | 4 | R8 | Left to SW2. A straight link at y = 110.4 joins SW2's two pads. A branch runs down x = 106.6, with a hop under LEFT, to R8 pin 2. |
| LEFT | 11 | R11 | Left to x = 105.05, down the gap between the two columns of R8–R10 pads to R11 pin 2, then straight down x = 105.825 to the SW5 link at y = 131. |
| RIGHT | 12 | R10 | Left to (109.85, 115.7), a hop to (107.75, 118.1). A branch goes to R10 pin 2. The main line runs down x = 107.75, diagonally, then along y = 134.9 through SW3. A second hop passes under D+/D− and the 3V3 trunk, ending directly on SW4's left pad on B.Cu. An F.Cu link at y = 131 joins SW4's pads. |
| MID | 13 | R9 | Left along y = 116.55. A branch goes to R9 pin 2. The main line runs down x = 108.85 and diagonally to SW3's left pad. An F.Cu link at y = 133.5 joins SW3's pads. |

RIGHT has the most complicated path because its switch is on the far side of the USB pair.

On the silkscreen, SW3's route is labelled `MID` because `MIDDLE` doesn't fit.

## Reset (EN, 0.25 mm)
From U6 pin 45, right along y = 105.75 above R5, to R5 pin 2. Straight down x = 132.425 to C5 pin 2, then diagonally to SW6 pin 1.

## Ground

There are few GND tracks because the pours on both layers carry the ground.

- **Pours:** F.Cu and B.Cu are both filled with GND across the whole board (the antenna area stays empty).
- **Small pads:** each GND SMD pad has a short stub and a via down to the B.Cu plane.
- **ESP32 centre:** the 9 thermal pads are joined, with 4 vias in the free space around the block.
- **ESP32 top row:** a bus along y = 105.3 joins all the GND pins, with 3 vias just inside it.
- **U1 pin 2:** its stub runs left to a via at (110.95, 148).

## All B.Cu hops

| Net | From | To | Passes under |
|---|---|---|---|
| VBUS | (117.55, 166.35) | (122.45, 166.35) | USB signal pins and D+/D− |
| CC1 | (118.3, 165.5) | (116.7, 165.5) | VBUS |
| VBAT | (121.3, 148) | (123.15, 148) | D+, D− |
| STAT | (110, 149.4) | (110, 151) | VBAT |
| 3V3 | (129.9, 147.05) | (129.9, 141.7) | VBAT_SW, VBAT |
| 3V3 | (125.625, 125.1) | (128.2, 125.1) | D−, D+ |
| 3V3 | (109.6, 125.1) | (104.175, 125.1) | MID, RIGHT, LEFT |
| 3V3 | (105.5, 111.2) | (105.5, 109.6) | BOOT |
| 3V3 | (118.8, 122.9) | (118, 125.75) | Nothing; it joins U4's 3V3 pins beside the chip |
| RIGHT | (125.95, 132.8) | SW4 pad (130, 131) | D−, D+, 3V3 trunk |
| RIGHT | (109.85, 115.7) | (107.75, 118.1) | MID |
| BOOT | (106.6, 115.2) | (106.6, 116.85) | LEFT |
| SCL | (111.85, 112.3) | (110.55, 118.6) | LEFT, RIGHT, MID |
| SDA | (111.85, 113.15) | (111.2, 118) | LEFT, RIGHT, MID |
| INT1 | (111.85, 114) | (111.85, 117.4) | LEFT, RIGHT, MID |

## Parts of the routing that were kept on purpose

- **J1 and U4 escapes** are fine-pitch and deliberate. Don't change them without re-checking clearances.
- **U4 escape vias:** 8 vias surround the chip. The custom DRC rule allows 0.15 mm clearance for items touching U4's courtyard.

## DRC warnings you will see

These are expected and are not errors:

| Message | Where | Why |
|---|---|---|
| Missing footprint U9 | Schematic parity | The schematic has a stray XIAO-ESP32-S3 with no footprint. |
| Silkscreen clipped by board edge | U6, J1 | Their outlines overhang the board edge. |
| Via connected on only one layer | 3V3 via at (118.8, 122.9) | It is the end of U4 pin 12's escape, joined on B.Cu. INT1 and SCL enclose it on F.Cu. |
| Footprint does not match library | U4 | Already there before this routing work. |
