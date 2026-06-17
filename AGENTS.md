# CNC Project — Agent Instructions

This document tells any Claude agent how to work in this project. Read it before making changes.

---

## Project Purpose

This is a CNC / G-code tooling workspace. Tools here are single-file HTML apps that run in the browser — no build step, no dependencies, no server. The target machine is a **3-axis CNC router or mill with a 4th rotary axis (A/B/C)**.

All G-code is metric (G21), absolute (G90), feed per minute (G94).

---

## Files

| File | Purpose |
|------|---------|
| `pillar-gcode-gen.html` | Cylindrical pillar G-code generator — see `docs/pillar-gcode-gen.md` |
| `iGoldenCNC.pp` | Vectric post-processor for iGoldenCNC machine — extended with A-axis support |
| `Pillar.dxf` | Reference pillar profile (AC1009 / AutoCAD R12, POLYLINE with arc bulges) |
| `AGENTS.md` | This file |
| `docs/` | Per-tool documentation |

---

## Conventions

### G-code
- Always metric (G21), absolute positioning (G90), feed per minute (G94).
- X axis = radial distance from pillar centre (radius or diameter — selectable).
- Z axis = along the pillar / workpiece length.
- A/B/C = rotary axis (user-selectable letter).
- Safe retract is always a **rapid (G0)** to `stock_radius + clearance` before any rotation.
- Spindle on with `S… M3`, off with `M5`, program end `M30`.
- Comments use `;` (semicolon).

### DXF parsing
- Accept LWPOLYLINE (preferred) and old POLYLINE/VERTEX (AC1009) format.
- Axis mapping is **user-selectable**: `XR_YZ` (default, matches `Pillar.dxf`) or `XZ_YR`. Always expose this as a UI dropdown.
- Take `abs()` of the radius coordinate — profiles drawn on the left side of the centreline have negative X.
- **Arc bulges** (DXF group code 42): parsed on both entity formats. Never silently drop them.
- Use the longest polyline found if multiple exist.

### Arc segments (G2/G3)
- Output arc bulges as G2/G3, not tessellated G1 moves.
- `G18` (XZ plane) must appear in the G-code header before any arc moves.
- Arc direction: computed from cross-product of (centre→start) × (centre→end) in machine XZ space. This automatically corrects for the axis-mapping reflection (abs of radius reverses DXF CCW↔CW).
- Tool offset on arcs: same centre, R ± toolRadius (+ for G3/CCW, − for G2/CW). Degenerate arcs (R ≤ 0 after offset) are dropped.
- `I` = center_R − start_R; `K` = center_Z − start_Z. `I` is always radius, even in diameter mode.

### Tool radius compensation
- Computed **manually** (no G41/G42 in output) — tool centre is offset from the finished profile.
- Offset direction: **left side of the direction of travel** (equivalent to G41).
- Left normal at each vertex is the average of adjacent segment left normals, normalised.

### UI / code style
- Single HTML file per tool — inline CSS and JS, no external dependencies.
- Dark theme: background `#111827`, surface `#1f2937`, accent `#34d399`.
- Inputs: feedrate, spindle speed, tool radius, clearance, stock radius, stepover, rotary axis letter, X mode (radius/diameter).
- Always include a canvas preview showing: finished profile (green solid), tool centre path (amber dashed), stock radius line (grey dashed).
- Include a "Load sample" shortcut so the tool can be tested without a real DXF.

### Rotation modes (pillar-gcode-gen)
- **Indexed** (default): A indexes to fixed angles; full Z traverse at each angle. Controlled by `stepover` (°).
- **Continuous**: A rotates without stopping; Z traverses boustrophedon while depth spirals inward.
  - `engagementStart` (mm): total radial depth above finish profile at start.
  - `feedPerRev` (mm): radial depth removed per 360° of A.
  - `angularStep = min(5°, 2·arcsin(toolR/minR)·0.9)` — ensures 10% tool overlap at tightest point.
  - `nPassesPerRev = ceil(360/angularStep)` — Z traverses per revolution; exact so each rev ends on 360° boundary.
  - Depth interpolated continuously within each Z traverse (not stepped): smooth helical spiral.
  - Profile tessellated to ~0.5 mm G1 lines (arcs cannot be used with simultaneous A motion).
  - No mid-path retracts; one radial approach at start, one retract at end.

### Roughing operation (pillar-gcode-gen)
- Enabled by `roughingEnabled` checkbox; controlled by `passDepth` and `finishAllowance`.
- Pass count `N = max(0, ceil((stockRadius − minProfileR − finishAllowance) / passDepth))`.
- `minProfileR` = minimum R across all segment endpoints — the globally deepest cut.
- Roughing passes emitted outermost-first (k=1..N), each using `offsetSegments(segs, toolRadius + finishAllowance + (N−k)*passDepth)`.
- Finishing pass follows if `includeFinishing` is true (it can be disabled to leave material for a separate tool pass).
- Each roughing pass uses the same approach/retract sequence as the finishing pass.

### Documentation
- Every tool has a corresponding `docs/<tool-name>.md`.
- **Update `docs/<tool-name>.md` and this file whenever the tool changes.**

---

### Post-processor (iGoldenCNC.pp)
- Vectric `.pp` format. Variables follow `VAR NAME = [letter|mode|output|format]`.
- Mode `A` = always output; `C` = conditional (only when changed).
- The A-axis variable is `[A|C|A|1.3]` (conditional, letter A, 3 decimal places).
- A-axis home is `[AH|A|A|1.3]` (always output in HEADER/TOOLCHANGE).
- All move blocks (RAPID_MOVE, FIRST_FEED_MOVE, FEED_MOVE) include `[A]`.
- Arc move blocks (CW/CCW) do **not** include `[A]` — arcs are planar XY operations.

## How to add a new tool

1. Create `<tool-name>.html` in the project root following the conventions above.
2. Create `docs/<tool-name>.md` using the template in `docs/pillar-gcode-gen.md` as a reference.
3. Add a row to the Files table in this file.

---

## How to modify an existing tool

1. Edit the HTML file.
2. Update `docs/<tool-name>.md` — parameters section, behaviour section, G-code structure, known limitations.
3. Bump the revision note at the top of the doc.
