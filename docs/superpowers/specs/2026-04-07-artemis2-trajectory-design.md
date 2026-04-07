# Artemis II Trajectory Overlay — Design Spec
_Date: 2026-04-07_

## Overview

A toggleable overlay that brings the Artemis II mission to life inside the Earth-Moon Observatory. When activated, the sim jumps to launch time and plays the mission forward, showing the spacecraft's real path around the Moon with accurate positional timing anchored to published mission data.

## Mission Data (Anchors)

| Milestone | UTC Timestamp | Notes |
|-----------|--------------|-------|
| Launch | 2026-04-01T22:24:00Z | Kennedy Space Center, LC-39B |
| Lunar closest approach | 2026-04-06T23:02:00Z | 4,070 mi (6,550 km) altitude above surface |
| Splashdown | 2026-04-11T00:07:00Z | Off San Diego coast (April 10, 8:07 PM EDT) — T+217h 43min |
| Crew | Reid Wiseman (CDR), Victor Glover (PLT), Christina Koch, Jeremy Hansen (CSA) | |

## UI

### Toggle Button
- Label: `🚀 Artemis II`
- Added to `#btn-row` alongside existing controls
- Styled as `.ctrl-btn` — green highlight when active (same pattern as orbit toggle)
- Independent of speed/live buttons — does not get cleared by `setActiveBtn`

### Mission Info Card
- Positioned below `#focus-indicator` (top-right)
- Visible only when Artemis II overlay is active
- Shows:
  - `ARTEMIS II` title
  - Mission elapsed time (MET) — `FD 5 / 14:22:07` format (flight day + HH:MM:SS)
  - Current phase label (see Phases below)
  - Crew names (static, one line each)
- Same dark card style as HUD and focus indicator

### Spacecraft Dot
- Small pulsing orange dot rendered as a `THREE.Mesh` (sphere, radius ~0.08 scene units)
- Orange emissive material so it glows against the dark background
- Pulse achieved via scale animation in the render loop (sine wave, ~1.5s period)
- Hidden (`.visible = false`) when sim time is before launch or after splashdown

### Trajectory Line
- Rendered as `THREE.Line` with ~500 sample points
- Color: faint orange, low opacity (rgba ~0.4), `depthWrite: false`
- Full path drawn at all times while overlay is active (not just past positions)
- Computed once at toggle-on; does not rebuild each frame

## Behavior

### Toggle ON
1. Save current `simTime` and `multiplier` so they can be restored on toggle-off
2. Jump sim to launch time: `2026-04-01T22:24:00Z`
3. Set speed to `1000×` forward
4. Compute and render trajectory line
5. Show spacecraft dot and mission card
6. Camera focus stays on Earth (user can click Moon to switch as normal)

### Toggle OFF
1. Hide trajectory line, spacecraft dot, mission card
2. Restore sim time and speed to saved values from before toggle-on

### Spacecraft Position
- Spacecraft position is interpolated along the pre-computed trajectory array based on current sim time
- Before launch time: dot hidden
- After splashdown: dot hidden (mission complete)
- Between launch and splashdown: dot placed at the trajectory point corresponding to elapsed mission time

## Trajectory Computation

The trajectory is a spline through five real waypoints computed at toggle-on using the astronomy engine:

| Waypoint | Time | Position |
|----------|------|----------|
| 0 — Launch | T+0h | Earth origin (0,0,0) |
| 1 — Translunar midpoint | T+60h | Midpoint of Earth→Moon vector at that time, offset slightly toward Moon |
| 2 — Closest approach | T+121h (Apr 6 23:02 UTC) | Moon scene position + 6,550 km outward from Moon center toward Earth |
| 3 — Return midpoint | T+169h | Midpoint of Moon→Earth vector at that time |
| 4 — Splashdown | T+218h (Apr 11 00:07 UTC) | Earth origin (0,0,0) |

The spline is a **Catmull-Rom** curve through these 5 points (`THREE.CatmullRomCurve3`), sampled at 500 points. This produces a smooth, physically plausible free-return arc without requiring full N-body integration.

The outbound leg naturally curves past the Moon; the return leg curves back to Earth. Because waypoints 1 and 3 are derived from actual Moon positions at those timestamps, the trajectory reflects the true geometry of the mission.

## Mission Phases

Phase is determined by elapsed mission time:

| Phase | MET Range | Label |
|-------|-----------|-------|
| Pre-launch | before T+0 | — (dot hidden) |
| Translunar Injection | T+0h to T+6h | `Trans-Lunar Injection` |
| Translunar Coast | T+6h to T+114h | `Translunar Coast` |
| Lunar Flyby | T+114h to T+128h | `Lunar Flyby` |
| Return Coast | T+128h to T+218h | `Return Coast` |
| Post-splashdown | after T+218h | — (dot hidden) |

## File Structure

All new code lives inline in `index.html` following the existing pattern:

- **Trajectory computation function** `buildArtemisTrajectory()` — called once at toggle-on, returns `{ curve, points, totalMs }`
- **Spacecraft mesh** created at scene init, hidden by default
- **Mission card HTML** added to `<body>`, hidden by default via CSS `display: none`
- **Artemis state object** `artemisState` tracks `{ active, savedSimTime, savedMultiplier, trajectory }`
- **Button wiring** follows same pattern as orbit toggle

## Accuracy Notes

This is a scientifically grounded approximation, not a frame-perfect simulation. The trajectory shape and timing are anchored to real published NASA data. The intermediate path is a smooth interpolation, not full orbital mechanics. The mission card is labeled accordingly: no claim to centimeter-level precision.
