# pillar-gcode-gen.html

Generates G-code for **cylindrical pillar turning / carving** on a 3-axis CNC with a 4th rotary axis. The tool follows a 2D profile while the rotary axis indexes through 360°.

_Last updated: 2026-06-16 rev 5_

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
| ~~Stepover~~ | ~~`stepover`~~ | — | ~~See Rotation mode above.~~ |
| 4th axis letter | `rotaryAxis` | — | Letter used in G-code for the rotary axis: A, B, or C. |
| X mode | `xMode` | — | `radius` → X values are radii. `diameter` → X values are doubled (lathe diameter convention). |
| Rotation mode | `rotMode` (radio) | — | `indexed` (stepover, default) or `continuous`. |
| Stepover | `stepover` | ° | Angular increment between indexed passes. Total passes = ⌈360/stepover⌉. Hidden in continuous mode. |
| Engagement start | `engagementStart` | mm | Continuous mode only. Radial depth above the finish profile at the very start of rotation. |
| Feed / rev | `feedPerRev` | mm | Continuous mode only. Radial material removed per full 360° revolution. |
| Include finishing (cont.) | `contIncludeFinishing` | — | Continuous mode only. After all roughing revolutions, do one final full revolution at the finish profile. |
| Approach direction | `approach` (radio) | — | `radial` (side) or `axial` (end). Continuous mode always uses radial approach. |
| Axial entry from | `axialEntry` | — | Only active when approach = axial. `zmin` = enter from low-Z end; `zmax` = enter from high-Z end. |
| Enable roughing | `roughingEnabled` | — | When checked, emits multiple radial passes per angle before the finish pass. |
| Pass depth | `passDepth` | mm | Radial depth of each roughing pass. |
| Finish allowance | `finishAllowance` | mm | Material left on the profile after all roughing passes. |
| Include finishing | `includeFinishing` | — | When checked (default), a final pass at the true offset profile follows the roughing passes. |

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

## Rotation Modes

### Indexed (default)

A axis indexes to fixed angles (`0, stepover, 2×stepover, …`). At each angle the tool makes a full Z traverse (plus any roughing passes, see below), then retracts and rotates to the next angle. This is the traditional approach — the machine is stationary rotationally during each cut.

### Continuous

A rotates continuously while Z traverses the profile. The two parameters drive a **spiral-in** cut:

| Parameter | Meaning |
|-----------|---------|
| `engagementStart` | Radial depth above the finish profile at first contact (total roughing depth). |
| `feedPerRev` | Material removed per 360° of A rotation. |

**Algorithm:**

1. `nRoughRevs = ⌈engagementStart / feedPerRev⌉` — number of full A revolutions to reach the finish profile.
2. `angularStep = 2·arcsin(toolRadius / minR) × 0.9` — step between Z traverses, ensures 10% overlap at the tightest profile point.
3. `nPassesPerRev = ⌈360 / angularStep⌉` — Z traverses per revolution (always a whole number so each revolution lands on a full 360° boundary).
4. The tool makes `nRoughRevs × nPassesPerRev` Z traverses (boustrophedon — alternating forward/reverse). Depth decreases **continuously** within each traverse: `depth(t) = depthStart + t × (depthEnd − depthStart)`, creating a smooth helical spiral.
5. Optionally one finishing revolution at depth = 0.

**G-code structure:**

```gcode
G0 Z{zStart} X{safeX}          ; position axially
G1 X{profile + engagementStart} ; radial plunge
; --- For each Z traverse ---
G1 Z{z} X{profile(z) + depth(t)} A{a} F{feedrate}
; (no retracts between traverses — continuous motion)
M5
G0 X{safeX}
M30
```

Because arcs cannot be used for this coupled Z-X-A motion, the profile is tessellated into fine (≈0.5 mm) G1 segments.

---

## Roughing Operation

When roughing is enabled the generator computes:

```
maxDepth = stockRadius − minProfileR − finishAllowance
N = max(0, ceil(maxDepth / passDepth))
```

where `minProfileR` is the smallest radius value across all profile segments (the deepest cut point).

For each A-axis angle, the passes are emitted in order:

1. Roughing pass k=1 — outermost: offset = `toolRadius + finishAllowance + (N−1)×passDepth`
2. Roughing pass k=2: offset = `toolRadius + finishAllowance + (N−2)×passDepth`
3. …
4. Roughing pass k=N — innermost: offset = `toolRadius + finishAllowance`
5. Finishing pass (if **Include finishing** checked): offset = `toolRadius`

Each pass uses the same approach/retract sequence (radial or axial) as the finishing pass.

---

## G-code Structure

```gcode
; header / parameters comment block
G21          ; metric
G90          ; absolute
G94          ; feed per minute
G18          ; XZ plane — required for G2/G3 arc moves
S{spindleSpeed} M3
G0 X{safe_X} Z{safe_Z} A0.000

; for each angle (0, stepover, 2×stepover, …):
;   G0 A{angle}
;   [roughing passes — approach, profile at offset, retract]  ← only when roughing enabled
;   [finishing pass  — approach, profile at toolRadius offset, retract]
;   (blank line)

M5
G0 X{safe_X} Z{safe_Z}
M30
```

### Arc move format (G18 XZ plane)

```gcode
G2 Z{end_Z} X{end_R} I{center_R - start_R} K{center_Z - start_Z} F{feedrate}
G3 Z{end_Z} X{end_R} I{center_R - start_R} K{center_Z - start_Z} F{feedrate}
```

- `G2` = CW arc (viewed from +Y); `G3` = CCW arc.
- `I` = X-axis offset from start to arc centre (always in radius units, even in diameter mode).
- `K` = Z-axis offset from start to arc centre.
- Direction determined from the cross-product of (centre→start) × (centre→end) in machine XZ space, so reflections from the abs(radius) mapping are handled automatically.

---

## DXF Parsing

Supported entity types:
- **LWPOLYLINE** (preferred) — group codes 10/20 repeated per vertex; group code 42 per vertex for bulge.
- **POLYLINE / VERTEX / SEQEND** — legacy AutoCAD R12 format; bulge on VERTEX via group code 42.

If multiple polylines exist, the longest one is used.

Closed polylines (flag bit 0 set) have their first point appended as the last point.

### Arc Bulge Conversion

A DXF bulge value `b` on vertex `i` defines an arc from `vertex[i]` to `vertex[i+1]`:
- `b = tan(θ/4)` where `θ` is the subtended angle.
- `b > 0` → CCW in DXF space; `b < 0` → CW; `b = 0` → straight line (segment omitted).
- Arc radius: `R = chord / (2·sin(θ/2))`

Each arc segment is output as a single **G2/G3** move — no tessellation. The arc direction (G2 vs G3) is computed from the cross-product of the arc centre vectors in machine XZ space, which correctly handles the axis-mapping reflection (abs of radius reverses DXF CCW/CW).

**Tool radius offset on arcs:** the arc centre is unchanged; only the radius is adjusted:
- G3 (CCW) arc: `R_offset = R + toolRadius` (tool on outside)
- G2 (CW) arc: `R_offset = R − toolRadius` (tool on inside)

If `R_offset ≤ 0` the arc is degenerate and is dropped.

The stats bar shows: `N DXF pts → L lines + A arcs`.

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

- Non-tangent transitions between line and arc segments produce small position discontinuities at junctions. CAD profiles designed with tangent continuity (as `Pillar.dxf` is) are not affected.
- G2/G3 arcs require the machine controller to support **G18** (XZ plane). Most FANUC-compatible and Haas controllers do; verify before running.
- Roughing pass count is computed from `minProfileR` (the globally shallowest radius on the profile). If the profile is non-uniform, some areas may see more air passes than necessary; this is conservative and safe.
- Assumes a symmetric cylindrical stock (constant radius along Z).
- No collision detection — if the offset profile radius exceeds the stock radius at any point, the generated G-code will air-cut or gouge, depending on direction.
