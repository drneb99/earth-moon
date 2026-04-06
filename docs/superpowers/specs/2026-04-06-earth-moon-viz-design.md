# Earth-Moon Orbital Visualizer — Design Spec
_Date: 2026-04-06_

## Vision

A browser-native, scientifically accurate Earth-Moon observatory. The primary differentiator from tools like Solar System Scope is **focus**: every detail is tuned to the Earth-Moon relationship, with tidal locking as the central educational feature. Loads live to the current moment in time.

## Core Use Cases

1. **Live snapshot** — Open the app and immediately see where the Moon is right now, which face points at Earth, and its current distance/phase.
2. **Tidal locking demo** — Accelerate time and watch the Moon orbit Earth while its same face stays locked toward us. The "aha moment" for understanding synchronous rotation.
3. **Trajectory intuition** — Scrub time forward/back to understand where the Moon will be for events like Artemis 2, full moons, perigee/apogee.

## Architecture

A single `index.html` file. No build tools, no frameworks. Three.js loaded via CDN. Internally organized into three logical layers:

### 1. Astronomy Engine (`astronomyEngine` object)
Pure functions. Stateless. Takes a JavaScript `Date`, returns:
- Moon ecliptic longitude/latitude (Meeus Ch. 47 truncated series, ~0.1° accuracy)
- Moon distance from Earth center (km)
- Moon phase angle (0–360°) and named phase string
- Moon orbital speed (km/s)
- Moon rotation angle (tidally locked = orbital angle)
- Libration in longitude (optical, dominant eccentricity term, max ~7.9°)
- Earth's subsolar point direction (for day/night terminator)

### 2. Time Controller (`timeController` object)
Manages simulation time. On load, pins to `Date.now()` at 1x speed (LIVE mode). State:
- `simTime`: current simulation timestamp (ms)
- `speedMultiplier`: 1 | 100 | 1000 | 10000 | 100000
- `isLive`: boolean — true when pinned to wall clock
- `lastWallTime`: used to advance simTime each frame

Methods: `tick()`, `setSpeed(n)`, `goLive()`, `pause()`

### 3. Three.js Renderer
Consumes astronomy engine output each animation frame. Scene contents:

- **Starfield**: ~3,000 points on a large fixed sphere
- **Sun light**: DirectionalLight positioned at correct ecliptic angle for current sim time (drives Earth terminator)
- **Earth**: textured sphere, 23.44° axial tilt, rotates once per 24 sim-hours, day/night terminator from sun direction
- **Moon**: textured sphere, tidally locked (rotation angle = orbital angle + libration offset), correct orbital inclination (5.14°), eccentricity (0.0549)
- **Orbit ring**: 3D ellipse at correct inclination, faint line
- **Camera**: Three.js OrbitControls — click-drag to rotate, scroll to zoom, touch supported

## Astronomical Model

Based on Jean Meeus, _Astronomical Algorithms_, 2nd ed., Ch. 47 (truncated lunar theory).

| Parameter | Value |
|-----------|-------|
| Moon mean orbital period | 27.321661 days |
| Orbital eccentricity | 0.0549 |
| Orbital inclination to ecliptic | 5.145° |
| Earth axial tilt | 23.44° |
| Tidal lock | Moon rotation angle = mean anomaly + equation of center |
| Libration in longitude | `arctan(e·sin(M) / (1 - e·cos(M)))` — max ±7.9° |
| Accuracy | ~0.1° for visualization purposes |

## HUD Layout

**Top-left panel** (semi-transparent dark card):
- Sim date and time (UTC)
- LIVE badge (pulsing green dot) when in live mode
- Moon distance from Earth (km)
- Moon phase: icon + name (New / Waxing Crescent / First Quarter / Waxing Gibbous / Full / Waning Gibbous / Last Quarter / Waning Crescent)
- Moon orbital speed (km/s)
- Libration angle (°)

**Bottom-center controls**:
```
[◀◀ -1000x]  [◀ -100x]  [‖ Pause]  [▶ 100x]  [▶▶ 1000x]  [▶▶▶ 100000x]  [● LIVE]
```
Speed label shown above controls. LIVE button re-pins simTime to wall clock.

## Visual Style

- Dark space aesthetic
- Earth: NASA Blue Marble texture (threejs.org CDN)
- Moon: NASA lunar surface texture (threejs.org CDN)
- Orbit ring: faint white, low opacity
- HUD: dark background `rgba(0,0,0,0.6)`, monospace font, subtle border
- Stars: white points, varying sizes

## Responsive Behavior

- Canvas fills 100vw × 100vh
- HUD panels reflow on small screens (smaller font, compact layout)
- OrbitControls supports touch (pinch zoom, one-finger rotate)

## Deliverable

Single file: `earth-moon-viz/index.html`
- All JS inline in `<script>` tag
- All CSS inline in `<style>` tag
- No external dependencies except Three.js CDN and texture CDNs
- Deployable by opening the file directly in a browser or dropping on any static host
