# pillar-gcode-gen.html

Generates G-code for **cylindrical pillar turning / carving** on a 3-axis CNC with a 4th rotary axis. The tool follows a 2D profile while the rotary axis indexes through 360°.

_Last updated: 2026-06-16 rev 2_

---

## Coordinate System

The app supports two DXF axis conventions, selectable in the UI:

| Option | DXF X | DXF Y |
|--------|-------|-------|
| `DXF X → Z, Y → R` | machine Z (along pillar) | radius |
| `DXF X → R, Y → Z` *(default)* | radius | machine Z (along pillar) |

The default matches the `Pillar.dxf` reference file (AutoCAD R12 / AC1009 format), where the profile is drawn with the pillar axis vertical (Y) and radius horizontal (X, negative for a left-side profile).

**Negative radius** is handled automatically — `abs()` is applied to the radius coordinate after mapping, so profiles drawn on the left side of the centre line (negative X) produce correct positive R values.

---

## Input Parameters

| Parameter | Field ID | Unit | Description |
|-----------|----------|------|-------------|
| DXF content | `dxfInput` | — | Paste full DXF file. Supports LWPOLYLINE and old POLYLINE/VERTEX formats. Arc bulges (group code 42) are tessellated. |
| Axis mapping | `dxfAxisMap` | — | `XR_YZ` (default): DXF X=radius, Y=machine Z. `XZ_YR`: DXF X=machine Z, Y=radius. |
| Feedrate | `feedrate` | mm/min | Cutting feedrate for all G1 moves. |
| Spindle speed | `spindleSpeed` | RPM | Used in `S… M3` at program start. |
| Tool radius | `toolRadius` | mm | Radius of the cutting tool. Used for left-side offset. |
| Clearance | `clearance` | mm | Safety distance added to stock radius for rapids; also used for approach/exit overtravel. |
| Stock radius | `stockRadius` | mm | Radius of the raw stock cylinder. Used to compute safe retract X and preview stock line. |
| Stepover | `stepover` | ° | Angular increment between passes. Total passes = ⌈360 / stepover⌉. |
| 4th axis letter | `rotaryAxis` | — | Letter used in G-code for the rotary axis: A, B, or C. |
| X mode | `xMode` | — | `radius` → X values are radii. `diameter` → X values are doubled (lathe diameter convention). |
| Approach direction | `approach` (radio) | — | `radial` (side) or `axial` (end). |
| Axial entry from | `axialEntry` | — | Only active when approach = axial. `zmin` = enter from low-Z end; `zmax` = enter from high-Z end. |

---

## Tool Offset (Left Side)

Tool centre is displaced from the finished profile by `toolRadius` in the **left-side normal** direction at each vertex.

- Left normal for a segment P1→P2: rotate the unit tangent 90° counter-clockwise → `N = (−dy, dx) / |PQ|`.
- At interior vertices the normal is the average of the two adjacent segment normals, re-normalised.
- This is equivalent to G41 (cutter compensation left) but computed manually — no G41/G42 in output.
- For external turning travelling in +Z: the left side points in +R (outward), so the tool centre sits outside the finished surface. ✓

---

## Approach Modes

### Radial (side)

The tool enters from the side (X direction) at each pass.

```
G0 A{angle}
G0 Z{first_profile_Z}  X{safe_X}    ; position axially
G1 X{first_offset_R}   F{feedrate}  ; plunge radially
G1 Z… X… (follow profile)
G0 X{safe_X}                        ; retract
```

Use when the stock end faces are inaccessible or when the profile doesn't allow clean axial entry (e.g. flanges at both ends).

### Axial (end)

The tool enters from one end of the stock in the Z direction.

```
G0 A{angle}
G0 X{first_offset_R}  Z{entry_Z}    ; position radially, safe Z beyond end
G1 Z{first_profile_Z} F{feedrate}   ; plunge axially
G1 Z… X… (follow profile)
G1 Z{exit_Z}          F{feedrate}   ; exit axially
G0 X{safe_X}                        ; retract
```

`entry_Z` = `Z_start − clearance` (low-Z entry) or `Z_end + clearance` (high-Z entry).

When entering from the high-Z end, the profile is traversed in reverse (last point → first point).

---

## G-code Structure

```gcode
; header / parameters comment block
G21 G90 G94
S{spindleSpeed} M3
G0 X{safe_X} Z{safe_Z} A0.000

; for each pass (angle = 0, stepover, 2×stepover, …):
;   approach move
;   G1 profile points
;   retract

M5
G0 X{safe_X} Z{safe_Z}
M30
```

---

## DXF Parsing

Supported entity types:
- **LWPOLYLINE** (preferred) — group codes 10/20 repeated per vertex; group code 42 per vertex for bulge.
- **POLYLINE / VERTEX / SEQEND** — legacy AutoCAD R12 format; bulge on VERTEX via group code 42.

If multiple polylines exist, the longest one is used.

Closed polylines (flag bit 0 set) have their first point appended as the last point.

### Arc Bulge Tessellation

A DXF bulge value `b` on vertex `i` defines an arc from `vertex[i]` to `vertex[i+1]`:
- `b = tan(θ/4)` where `θ` is the subtended angle.
- `b > 0` → CCW arc; `b < 0` → CW arc; `b = 0` → straight line.
- Arc radius: `R = chord / (2·sin(θ/2))`
- The arc is tessellated into linear segments at ~5° resolution (≤ 5° angular error per segment).
- The stats bar shows original DXF vertex count vs tessellated point count.

The `Pillar.dxf` reference file contains 10 DXF vertices with bulges, which tessellate to ~50+ points.

---

## Preview Canvas

| Element | Colour |
|---------|--------|
| Finished profile | Solid green `#34d399` |
| Tool centre path | Dashed amber `#fbbf24` |
| Stock radius | Dashed grey `#374151` |
| Profile vertices | Green dots |

Axes: Z horizontal, R vertical. Canvas auto-resizes with the window.

---

## Known Limitations

- All profile moves are G1 linear segments. DXF arc bulges are tessellated to ~5° resolution before G-code output — no G2/G3 in output.
- No roughing passes — a single finish pass per angle. Run multiple times with increasing depth, or offset the stock radius inward manually to simulate roughing.
- Assumes a symmetric cylindrical stock (constant radius along Z).
- No collision detection — if the offset profile radius exceeds the stock radius at any point, the generated G-code will air-cut or gouge, depending on direction.
