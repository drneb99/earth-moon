# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file 3D Earth–Moon observatory. Everything lives in `index.html` — HTML, CSS, and JS are all inline. There is no build system, no bundler, no npm. Three.js r162 is vendored locally under `lib/`.

Local assets:
- `images/earth_atmos_2048.jpg` — Earth day/color map
- `images/earth_normal_4k.jpg` — 4K normal map generated from NASA GEBCO elevation data
- `images/earth_specular_2048.jpg` — specular map
- `images/earth_night.jpg` — NASA Black Marble city-lights texture
- `images/moon_4k.jpg` — 4K Moon surface
- `lib/three/three.module.js` — Three.js r162 (vendored, no CDN)
- `lib/three/addons/controls/OrbitControls.js` — vendored

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

## Security

A `Content-Security-Policy` meta tag is set in `<head>`. It allows only `'self'` for scripts and images — no external domains. If you add a new external resource, update the CSP accordingly.

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
1. Adaptive `controls.rotateSpeed` — scales with `camera.distanceTo(controls.target)`, divisor differs by `focusTarget` (`10` for Moon, `30` for Earth)
2. `timeController.tick()` → `simDate`
3. `astronomyEngine.getMoonState(simDate)` → `moonState`
4. Conditionally rebuild orbit ring (rate-limited)
5. Update sun light direction; copy to `nightMat.uniforms.sunDir` and `atmMat.uniforms.sunDir`
6. Update `atmMat.uniforms.camDist` (camera distance from Earth origin)
7. Update Earth GMST rotation (real-world terminator alignment)
8. `updateMoon(moonState)` — positions mesh, applies tidal lock quaternion, applies libration
9. `updateHUD(simDate, moonState)`
10. Camera fly-to animation (`cameraAnim`) or normal OrbitControls lerp
11. `renderer.render()`

### UI systems

**Focus system** — clicking Earth or Moon calls raycaster hit-test, sets `focusTarget`, and triggers `flyTo()`. Clicking Moon flies camera to ~2 Moon-radii out; clicking Earth flies to a default Earth view. Speed buttons are hidden when Moon is focused. `controls.target` lerps toward the focused body each frame via `_lerpTarget`. When `cameraAnim` ends, `_lerpTarget` is synced to `cameraAnim.endTarget` to prevent a snap/bump.

**Orbit ring** — built by sampling 180 real Moon positions spaced evenly over one sidereal month from the current sim time. Rebuilt at most 10×/sec wall time when `|multiplier| ≥ 1000`, otherwise at most every 6 sim-hours. Uses vertex colors for fade-in/out at both ends.

**Night lights** — custom `ShaderMaterial` (`nightMat`) using additive blending. The `sunDir` uniform is updated each frame. The mesh is a child of `earthMesh` so it inherits GMST rotation automatically.

**Atmosphere glow** — custom `ShaderMaterial` (`atmMat`) on a sphere at `EARTH_RADIUS * 1.015` using `THREE.BackSide` and additive blending. Fresnel rim effect in view-space; power and fade both adapt with `camDist` uniform (soft/wide when close, tight/faded when far). Includes a subtle orange terminator tint from Rayleigh scattering. `sunDir` and `camDist` uniforms updated each frame.

**Next Full Moon finder** — `findNextFullMoon()` bisects to ~1-minute precision using 6-hour scan steps. Result cached for 12 sim-hours (`_cachedFullMoon`).

**Lessons / Facts mode** — `btn-facts` (currently hidden, pending fixes) opens `#lesson-card`. `LESSONS` array contains 8 entries, each with `title`, `body`, and `getCamera()` returning `{pos, target}`. Opening lessons pauses the sim and saves state; closing restores it. `flyTo()` triggers `cameraAnim` and disables `OrbitControls` during the 1800ms eased flight. `_lastSunDir` is kept in sync each frame so lesson cameras that reference the terminator direction work correctly. To re-enable: remove `style="display:none"` from `#btn-facts`.

### Adding a new overlay / mode

Follow the pattern used by lessons:
1. Add CSS (position fixed, `display:none`, `.visible` class to show)
2. Add HTML div before `</body>`
3. Add a state object + constants block after `timeController`
4. Add scene objects (meshes/lines) before the button wiring section
5. Add button listener and toggle logic (save/restore sim state)
6. Add per-frame update inside `animate()` after `updateHUD()`
