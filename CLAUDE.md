# Galactic View — CLAUDE.md

## Project

Single-file browser astronomy visualization (`galactic-interactive.html`). No build step, no dependencies, no frameworks. Open directly in a browser.

## File Layout

```
galactic-interactive.html   — entire app: HTML + CSS + JS (~900 lines)
README.md
gitignore
```

## Code Structure

The `<script>` block is divided into sections marked with `═══` banners:

| Section | Lines | What it does |
|---|---|---|
| VERSION | ~326 | Single `const VERSION` string; shown in header via `versionDisplay` element |
| 3D MATH | ~332–371 | Pure math utilities (no state) |
| GEOMETRY DEFINITIONS | ~373–532 | Geometry factories and season data |
| RENDER STATE | ~534–546 | Mutable globals: `rotX`, `rotY`, `zoom`, `dragging`, `seasonVisible[]` |
| DRAW | ~548–797 | `render()` function |
| INTERACTION | ~799–897 | Event listeners; calls `render()` |

## Versioning

Update only `const VERSION = "x.y.z"` near the top of `<script>`. Scheme:
- **Major** — significant new features or UI overhaul
- **Minor** — a few new features or major bug fixes
- **Patch** — small bug fixes

## Coordinate System

All geometry is defined in galactic coordinates:
- `+Y` = galactic north
- `+X` = anti-core (away from Sagittarius A*)
- `+Z` = toward viewer in default pose

`EARTH_POS = SUN_POS = [0.58, 0, 0]` — coincident at galactic scale (58% from core toward anti-core).

## Key Constants (in `render()`)

```js
GAL_R   = 1.15   // galactic disk radius (normalized units)
CEQ_R   = 0.17   // celestial equator radius
ORBIT_R = 0.15   // Earth orbit / ecliptic radius
AXIS_L  = 0.62   // galactic N/S axis half-length
ROT_L   = 0.24   // Earth rotation axis half-length
ARROW_L = 0.52   // seasonal view arrow length
```

All geometry is scaled by `SCALE = Math.min(W, H) * 0.38 * zoom` inside `render()`. Geometry functions return unit-space points; `sv(v)` scales them to canvas space.

## 3D Pipeline

1. Define geometry in galactic coordinates (unit space)
2. `applyRot(v, rotX, rotY)` — applies `rotateY` then `rotateX`
3. `project(v, scale, ox, oy)` — simple perspective with `fov = 900`
4. `proj(pt)` = `applyRot` + `project` in one call

## Rendering: Painter's Algorithm

`render()` builds a `drawList` array of `{depth, fn}` objects, sorts by depth (ascending = back-to-front), then calls each `fn`. Depth is computed via `polyDepth()` — the average rotated Z of a polygon's vertices.

Draw primitives:
- `drawPoly(pts3, strokeColor, fillColor, lw, dash)` — closed polygon
- `drawLine(p3a, p3b, color, lw, dash)` — line segment
- `drawArrow(p3from, p3to, color, lineW, headSize)` — line + arrowhead (fixed angle `0.38 rad`)
- `dot(p3, color, r)` — filled circle
- `label(p3, text, color, offx, offy, align, size)` — canvas text in `Space Mono`

All primitives call `proj()` internally. Canvas is resized to `offsetWidth × offsetHeight` at the start of each `render()` call.

## Seasons

`SEASONS` array (4 entries): Summer, Autumn, Winter, Spring.

Each entry:
```js
{
  name, color,
  orbitAngle,   // Earth's angle on the ecliptic orbit (radians)
  galBias,      // [x,y,z] added to anti-sun direction before normalizing arrow
  quadrant,     // display string e.g. "Down / In"
}
```

`nightViewArrow(earthPos, galBias, arrowLen)` blends the raw anti-sun direction with `galBias` (scaled by 0.5 in `render()`), normalizes, and returns the arrow tip.

`seasonVisible[i]` boolean array controls visibility; toggled by sidebar buttons.

## Interaction

| Input | Handler | Effect |
|---|---|---|
| Mouse drag | `onMouseMove` | `rotY += dx*0.007`, `rotX += dy*0.007`; `rotX` clamped to `±π/2` |
| Scroll wheel | `onWheel` | `zoom *= 1.08` or `0.93`; clamped to `[0.3, 4]` |
| Pinch (touch) | `onTouchMove` | `zoom *= dist/lastTouchDist` |
| Reset button | click | `rotX=0.38`, `rotY=-0.55`, `zoom=1.0` |
| Season toggles | click | Flip `seasonVisible[i]`, toggle `.active` class and opacity |

Canvas resize is handled by a `ResizeObserver` with `requestAnimationFrame` debounce.

## Geometry Functions

| Function | Description |
|---|---|
| `makeDisc(cx,cy,cz, rx,rz, n)` | Ellipse in the XZ plane (galactic disk) |
| `makeEclipticDisc(centre, r, n)` | Circle tilted by `ECL_TILT` around X axis |
| `makeCelestialEquator(centre, r, n)` | Circle perpendicular to `rotAxisN` via cross product |
| `makeOrbit(r, n)` | Calls `makeEclipticDisc` centred on `SUN_POS` |
| `earthOnOrbit(angle, orbitR)` | Earth's 3D position at a given orbit angle |

## Style

- Fonts: `Cormorant Garamond` (header, info box) + `Space Mono` (all labels, UI)
- CSS variables: `--void #03040c`, `--panel rgba(5,8,18,0.92)`, `--border rgba(255,255,255,0.08)`, `--label #8a9ab8`, `--bright #e8eef8`
- Background filled explicitly in `render()` as `#03040c` (so no white flash in iframes)

## Working Conventions

- **Single file** — keep entire app in `galactic-interactive.html`; do not split into multiple files
- **No frameworks** — vanilla HTML, CSS, and JavaScript only; no external libraries beyond Google Fonts
- **No build step** — file must open directly in a browser with no compilation or bundling
- **Complete file output** — when generating updates, always output the complete file, not diffs or partial snippets
- **Versioning** — bump `const VERSION` as the final step of each working session before push
- **Observer** — primary use case is 42°N latitude (Newton, MA); all seasonal logic should reflect this
- **iPad workflow** — file is edited and deployed from iPad via Working Copy; keep file transfer-friendly
- **Coordinate system** — do not change the galactic coordinate convention (+Y north, +X anti-core, +Z toward viewer)
- **Style** — maintain the existing dark void aesthetic; Cormorant Garamond + Space Mono font pairing is intentional
- **Comments** — preserve the ═══ section banner style for new code sections