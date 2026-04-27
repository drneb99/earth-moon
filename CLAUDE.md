# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file 3D Earth–Moon observatory. Everything lives in `index.html` — HTML, CSS, and JS are all inline. There is no build system, no bundler, no npm. Three.js r162 is loaded via an importmap CDN.

Local assets: `earth_night.jpg` (NASA Black Marble city-lights texture), `moon_4k.jpg` (4K Moon surface). Remote textures (Earth day/normal/specular) are fetched from `threejs.org/examples/textures/`.

## Running

Serve with any static HTTP server — the file uses ES modules so it cannot open as `file://`:

```bash
python3 -m http.server 8181
# then open http://localhost:8181
```

There are no tests, no linter, and no build step.

## Git workflow

Remote: `https://github.com/drneb99/earth-moon.git` (branch: `master`)

**Always ask the user for confirmation before pushing.** Commit freely during a session, but only run `git push` after the user explicitly approves. When pushing:

```bash
git push origin master
```

On a new machine, clone with:
```bash
git clone https://github.com/drneb99/earth-moon.git
```

## Architecture

### Coordinate system

- **1 scene unit = 10,000 km**
- Earth is fixed at the origin, radius `2.0` units (= 20,000 km scene-space, representing 6,371 km real)
- Moon radius `0.545` units
- The ecliptic plane is the **XZ plane**; Y is ecliptic north
- Moon positions are computed in geocentric ecliptic coords and converted to scene-space `Vector3`

### Core objects

**`astronomyEngine`** — all orbital math (Meeus Ch. 47 series). The main entry point is:

```js
astronomyEngine.getMoonState(date)
// returns: { scenePos, distance (km), speed (km/s), libration (°),
//            phaseName, phaseIcon, lon, lat, sunDir (Vector3), trueElongation }
```

`sunDir` and `trueElongation` are computed as a byproduct here — don't call `getSunDirection()` separately in the same frame.

**`timeController`** — manages simulated time:
- `tick()` → advances `simTime` by `wallDelta × multiplier`, returns current `Date`
- `setSpeed(mult)` / `goLive()` / `pause()` are the public mutators
- `isLive` means `simTime` tracks real wall time; `paused` freezes sim time while `_lastWall` continues advancing

### `animate()` loop (bottom of `<script>`)

Each frame in order:
1. `timeController.tick()` → `simDate`
2. `astronomyEngine.getMoonState(simDate)` → `moonState`
3. Conditionally rebuild orbit ring (rate-limited)
4. Update sun light direction and `nightMat.uniforms.sunDir` from `moonState.sunDir`
5. Update Earth GMST rotation (real-world terminator alignment)
6. `updateMoon(moonState)` — positions mesh, applies tidal lock quaternion, applies libration
7. `updateHUD(simDate, moonState)`
8. Camera fly-to animation (`cameraAnim`) or normal OrbitControls lerp
9. `renderer.render()`

### UI systems

**Focus system** — clicking Earth or Moon calls raycaster hit-test and sets `focusTarget`. Speed buttons are hidden when Moon is focused. `controls.target` lerps toward the focused body each frame unless a lesson fly-to is active.

**Orbit ring** — built by sampling 180 real Moon positions spaced evenly over one sidereal month from the current sim time. Rebuilt at most 10×/sec wall time when `|multiplier| ≥ 1000`, otherwise at most every 6 sim-hours. Uses vertex colors for fade-in/out at both ends.

**Night lights** — custom `ShaderMaterial` (`nightMat`) using additive blending. The `sunDir` uniform is updated each frame. The mesh is a child of `earthMesh` so it inherits GMST rotation automatically.

**Next Full Moon finder** — `findNextFullMoon()` bisects to ~1-minute precision using 6-hour scan steps. Result cached for 12 sim-hours (`_cachedFullMoon`).

**Lessons / Facts mode** — `btn-facts` (currently hidden, pending fixes) opens `#lesson-card`. `LESSONS` array contains 5 entries, each with `title`, `body`, and `getCamera()` returning `{pos, target}`. Opening lessons pauses the sim and saves state; closing restores it. `flyTo()` triggers `cameraAnim` and disables `OrbitControls` during the 1800ms eased flight. `_lastSunDir` is kept in sync each frame so lesson cameras that reference the terminator direction work correctly. To re-enable: remove `style="display:none"` from `#btn-facts`.

### Adding a new overlay / mode

Follow the pattern used by lessons:
1. Add CSS (position fixed, `display:none`, `.visible` class to show)
2. Add HTML div before `</body>`
3. Add a state object + constants block after `timeController`
4. Add scene objects (meshes/lines) before the button wiring section
5. Add button listener and toggle logic (save/restore sim state)
6. Add per-frame update inside `animate()` after `updateHUD()`
