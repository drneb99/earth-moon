# Artemis II Trajectory Overlay Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a toggleable Artemis II mission overlay that jumps the sim to launch time, shows the real free-return trajectory as an orange line, and animates a pulsing spacecraft dot along the path based on sim time.

**Architecture:** All code lives inline in `index.html`, following the existing single-file pattern. A `buildArtemisTrajectory()` function computes a CatmullRom spline through five real waypoints (derived from the astronomy engine at known mission timestamps). A per-frame update moves the spacecraft dot and refreshes the mission card. Toggle on/off saves and restores sim state.

**Tech Stack:** Three.js r168 (CatmullRomCurve3, Mesh, Line), existing `astronomyEngine` object, existing `timeController` object.

---

## File Structure

Only one file changes: `index.html`

Sections modified in order:
- `<style>` — add `#artemis-card` styles
- `<body>` — add `#artemis-card` HTML div, add `#btn-artemis` button
- `<script>` — add ARTEMIS2 constants, artemisState, buildArtemisTrajectory(), helper functions, scene objects, button wiring, per-frame update, DOM cache additions

---

## Mission Reference Data

```
Launch:    2026-04-01T22:24:00Z
Flyby:     2026-04-06T23:02:00Z  (T+120h 38min = T+434,280,000ms)
Splashdown:2026-04-11T00:07:00Z  (T+217h 43min = T+783,720,000ms)
Flyby alt: 6,550 km above surface → 8,287 km from Moon center → 0.8287 scene units
```

---

### Task 1: Add HTML for mission card and Artemis toggle button

**Files:**
- Modify: `index.html` (HTML section only)

- [ ] **Step 1: Add the mission card div to `<body>` before the closing `</body>` tag**

Add after `</div>` (the closing of `#focus-indicator`):

```html
  <div id="artemis-card">
    <div class="hud-title">🚀 Artemis II</div>
    <div class="hud-row">
      <span class="hud-label">MET</span>
      <span class="hud-value" id="artemis-met">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label">Phase</span>
      <span class="hud-value" id="artemis-phase">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label" style="align-self:flex-start">Crew</span>
      <span class="hud-value" style="font-size:10px;line-height:1.6;text-align:right">
        Wiseman · Glover<br>Koch · Hansen
      </span>
    </div>
  </div>
```

- [ ] **Step 2: Add the Artemis II toggle button to `#btn-row`**

In `#btn-row`, add after `#btn-orbit`:

```html
      <button class="ctrl-btn" id="btn-artemis">🚀 Artemis II</button>
```

- [ ] **Step 3: Verify the HTML parses cleanly**

Open `http://localhost:8181` in the browser. The button should appear in the control row. The mission card won't be visible yet (we add CSS next). No JS errors in console.

---

### Task 2: Add CSS for mission card

**Files:**
- Modify: `index.html` (`<style>` block)

- [ ] **Step 1: Add `#artemis-card` styles to the `<style>` block**

Add after the `#focus-indicator` responsive media query block (`@media (max-width: 480px) { #focus-indicator ... }`):

```css
    #artemis-card {
      position: fixed; top: 110px; right: 16px;
      background: rgba(0,0,0,0.65);
      border: 1px solid rgba(255,140,0,0.3);
      border-radius: 8px;
      padding: 10px 14px;
      color: #e0e0e0;
      font-family: 'Courier New', monospace;
      font-size: 12px;
      line-height: 1.7;
      min-width: 190px;
      user-select: none;
      backdrop-filter: blur(4px);
      display: none;
    }
    #artemis-card.visible { display: block; }
    @media (max-width: 480px) {
      #artemis-card { font-size: 11px; top: 90px; right: 8px; padding: 6px 10px; }
    }
```

- [ ] **Step 2: Verify card appearance**

Temporarily add `class="visible"` to `#artemis-card` in the HTML. Open `http://localhost:8181`. The orange-bordered card should appear at top-right below the focus indicator, showing "🚀 Artemis II" with dashes for MET/Phase. Remove the temporary `visible` class after checking.

---

### Task 3: Add ARTEMIS2 constants and artemisState object

**Files:**
- Modify: `index.html` (JS `<script>` block)

- [ ] **Step 1: Add the ARTEMIS2 constants block**

Add after the `// ── Time Controller` block closes (after the `};` on the last line of `timeController`), before `// ── Starfield`:

```javascript
    // ── Artemis II Mission Data ───────────────────────────────────────────────
    const ARTEMIS2 = {
      launch:     new Date('2026-04-01T22:24:00Z'),
      flyby:      new Date('2026-04-06T23:02:00Z'),
      splashdown: new Date('2026-04-11T00:07:00Z'),
      // Distances in km: Moon radius 1,737 + flyby altitude 6,550 = 8,287 km from center
      flybyDistKm: 8287,
      crew: ['Wiseman (CDR)', 'Glover (PLT)', 'Koch', 'Hansen (CSA)'],
    };
    // totalMs and launchMs derived once for use throughout
    const ARTEMIS2_TOTAL_MS = ARTEMIS2.splashdown.getTime() - ARTEMIS2.launch.getTime();
    const ARTEMIS2_FLYBY_MS = ARTEMIS2.flyby.getTime()      - ARTEMIS2.launch.getTime();

    const artemisState = {
      active:          false,
      savedSimTime:    0,
      savedMultiplier: 1,
      savedIsLive:     false,
      trajectory:      null, // populated by buildArtemisTrajectory() on toggle-on
    };
```

- [ ] **Step 2: Verify constants in console**

Open browser console and run:
```javascript
ARTEMIS2_TOTAL_MS / 3600000  // should be ~217.72
ARTEMIS2_FLYBY_MS / 3600000  // should be ~120.63
```
Expected output: approximately `217.72` and `120.63`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Artemis II HTML structure, CSS card, and mission constants"
```

---

### Task 4: Add trajectory computation and helper functions

**Files:**
- Modify: `index.html` (JS `<script>` block)

- [ ] **Step 1: Add `buildArtemisTrajectory()` after the ARTEMIS2 constants block**

```javascript
    function buildArtemisTrajectory() {
      const launchMs    = ARTEMIS2.launch.getTime();
      const flybyMs     = ARTEMIS2.flyby.getTime();
      const splashMs    = ARTEMIS2.splashdown.getTime();

      // Sample Moon positions at 5 key times to anchor the spline
      const moonAtT60   = astronomyEngine.getMoonState(new Date(launchMs + 60   * 3600000));
      const moonAtFlyby = astronomyEngine.getMoonState(new Date(flybyMs));
      const moonAtT169  = astronomyEngine.getMoonState(new Date(launchMs + 169  * 3600000));

      // WP 0 — Launch: just outside Earth in the direction of early transit
      const launchDir = moonAtT60.scenePos.clone().normalize().multiplyScalar(0.4);

      // WP 1 — Translunar midpoint: ~50% of the way to the Moon at T+60h
      const mid1 = moonAtT60.scenePos.clone().multiplyScalar(0.52);

      // WP 2 — Closest approach: Moon surface + flybyDistKm toward Earth
      const moonPos  = moonAtFlyby.scenePos.clone();
      const toEarth  = new THREE.Vector3(0, 0, 0).sub(moonPos).normalize();
      const flybyPos = moonPos.clone().addScaledVector(toEarth, ARTEMIS2.flybyDistKm / 10000);

      // WP 3 — Return midpoint: ~50% of the way back from Moon at T+169h
      const mid2 = moonAtT169.scenePos.clone().multiplyScalar(0.48);

      // WP 4 — Splashdown: just outside Earth, approach from return direction
      const splashDir = moonAtT169.scenePos.clone().normalize().multiplyScalar(0.35);

      const waypoints = [launchDir, mid1, flybyPos, mid2, splashDir];
      const curve  = new THREE.CatmullRomCurve3(waypoints, false, 'catmullrom', 0.5);
      const points = curve.getPoints(500);

      return { curve, points };
    }
```

- [ ] **Step 2: Add `getSpacecraftPosition()` after `buildArtemisTrajectory()`**

```javascript
    // Returns a Vector3 along the trajectory for the given simDate,
    // or null if outside the mission window.
    function getSpacecraftPosition(simDate) {
      if (!artemisState.trajectory) return null;
      const elapsed = simDate.getTime() - ARTEMIS2.launch.getTime();
      if (elapsed < 0 || elapsed > ARTEMIS2_TOTAL_MS) return null;
      const t = elapsed / ARTEMIS2_TOTAL_MS; // 0..1
      return artemisState.trajectory.curve.getPoint(t);
    }
```

- [ ] **Step 3: Add `getArtemisPhase()` and `formatMET()` after `getSpacecraftPosition()`**

```javascript
    function getArtemisPhase(elapsedMs) {
      const h = elapsedMs / 3600000;
      if (h <   0) return null;
      if (h <   6) return 'Trans-Lunar Injection';
      if (h < 114) return 'Translunar Coast';
      if (h < 128) return 'Lunar Flyby';
      if (h < 218) return 'Return Coast';
      return null; // post-splashdown
    }

    function formatMET(elapsedMs) {
      if (elapsedMs < 0) return '—';
      const totalSec = Math.floor(elapsedMs / 1000);
      const days = Math.floor(totalSec / 86400);
      const hrs  = Math.floor((totalSec % 86400) / 3600);
      const mins = Math.floor((totalSec % 3600) / 60);
      const secs = totalSec % 60;
      return `FD ${days + 1} / ${String(hrs).padStart(2,'0')}:${String(mins).padStart(2,'0')}:${String(secs).padStart(2,'0')}`;
    }
```

- [ ] **Step 4: Verify functions exist in console**

Open browser console:
```javascript
// Check phase labels at known times
getArtemisPhase(0)               // "Trans-Lunar Injection"
getArtemisPhase(6 * 3600000)     // "Translunar Coast"
getArtemisPhase(121 * 3600000)   // "Lunar Flyby"
getArtemisPhase(150 * 3600000)   // "Return Coast"
getArtemisPhase(220 * 3600000)   // null

// Check MET formatting
formatMET(0)                     // "FD 1 / 00:00:00"
formatMET(86400000)              // "FD 2 / 00:00:00"
formatMET(5 * 3600000 + 1800000) // "FD 1 / 05:30:00"
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add Artemis II trajectory computation and helper functions"
```

---

### Task 5: Add spacecraft mesh and trajectory line to scene

**Files:**
- Modify: `index.html` (JS `<script>` block — scene setup section)

- [ ] **Step 1: Add spacecraft mesh after the Moon mesh setup**

Add after the `// ── Orbit Ring` block (after `let _lastOrbitWallMs = 0;`):

```javascript
    // ── Artemis II Scene Objects ──────────────────────────────────────────────
    const spacecraftGeo = new THREE.SphereGeometry(0.09, 8, 8);
    const spacecraftMat = new THREE.MeshBasicMaterial({
      color:     0xff8c00,
      depthTest: false,       // always render on top
    });
    const spacecraftMesh = new THREE.Mesh(spacecraftGeo, spacecraftMat);
    spacecraftMesh.visible = false;
    scene.add(spacecraftMesh);

    // Glow ring around spacecraft (slightly larger, translucent)
    const spacecraftGlowGeo = new THREE.SphereGeometry(0.14, 8, 8);
    const spacecraftGlowMat = new THREE.MeshBasicMaterial({
      color:       0xff8c00,
      transparent: true,
      opacity:     0.25,
      depthTest:   false,
    });
    const spacecraftGlow = new THREE.Mesh(spacecraftGlowGeo, spacecraftGlowMat);
    spacecraftMesh.add(spacecraftGlow); // child of spacecraft so it moves together

    const trajectoryGeo = new THREE.BufferGeometry();
    const trajectoryMat = new THREE.LineBasicMaterial({
      color:       0xff8c00,
      transparent: true,
      opacity:     0.45,
      depthWrite:  false,
    });
    const trajectoryLine = new THREE.Line(trajectoryGeo, trajectoryMat);
    trajectoryLine.visible = false;
    scene.add(trajectoryLine);
```

- [ ] **Step 2: Verify scene objects exist in console**

Open browser console:
```javascript
spacecraftMesh  // should log a THREE.Mesh object
trajectoryLine  // should log a THREE.Line object
```

No errors. Neither is visible yet.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Artemis II spacecraft mesh and trajectory line to scene"
```

---

### Task 6: Wire the toggle button and implement on/off logic

**Files:**
- Modify: `index.html` (JS `<script>` block — button wiring section)

- [ ] **Step 1: Cache the artemis card DOM elements**

In the `// ── Cached DOM references` block, add to the `hudEls` object or just after it:

```javascript
    const artemisCardEl  = document.getElementById('artemis-card');
    const artemisCardEls = {
      met:   document.getElementById('artemis-met'),
      phase: document.getElementById('artemis-phase'),
    };
```

- [ ] **Step 2: Exclude `#btn-artemis` from `setActiveBtn` clearing**

Find:
```javascript
      document.querySelectorAll('.ctrl-btn:not(#btn-orbit)').forEach(b => b.classList.remove('active'));
```
Replace with:
```javascript
      document.querySelectorAll('.ctrl-btn:not(#btn-orbit):not(#btn-artemis)').forEach(b => b.classList.remove('active'));
```

- [ ] **Step 3: Add the toggle handler after the orbit button handler**

Add after `document.getElementById('btn-orbit').addEventListener(...)`:

```javascript
    document.getElementById('btn-artemis').addEventListener('click', function () {
      if (!artemisState.active) {
        // ── Toggle ON ──
        artemisState.active           = true;
        artemisState.savedSimTime     = timeController.simTime;
        artemisState.savedMultiplier  = timeController.multiplier;
        artemisState.savedIsLive      = timeController.isLive;

        // Build trajectory (uses current astronomy engine state)
        artemisState.trajectory = buildArtemisTrajectory();

        // Populate trajectory line geometry
        trajectoryGeo.setFromPoints(artemisState.trajectory.points);
        trajectoryLine.visible    = true;
        spacecraftMesh.visible    = true;
        artemisCardEl.classList.add('visible');

        // Jump to launch time and play at 1000x
        timeController.setSpeed(1000);
        timeController.simTime   = ARTEMIS2.launch.getTime();
        timeController._lastWall = Date.now();

        // Update speed label and button states
        speedLabel.textContent = '▶▶ 1,000×';
        setActiveBtn(false);
        document.getElementById('btn-fwd2').classList.add('active');
        this.classList.add('active');

      } else {
        // ── Toggle OFF ──
        artemisState.active = false;

        trajectoryLine.visible    = false;
        spacecraftMesh.visible    = false;
        artemisCardEl.classList.remove('visible');

        // Restore sim state
        if (artemisState.savedIsLive) {
          timeController.goLive();
          speedLabel.textContent = 'LIVE \u2014 1\u00d7';
          setActiveBtn(true);
        } else {
          timeController.simTime    = artemisState.savedSimTime;
          timeController.multiplier = artemisState.savedMultiplier;
          timeController.isLive     = false;
          timeController.paused     = false;
          timeController._lastWall  = Date.now();
        }
        this.classList.remove('active');
      }
    });
```

- [ ] **Step 4: Verify toggle on/off works visually**

Open `http://localhost:8181`. Click "🚀 Artemis II":
- Sim should jump to April 1, 2026 near 22:24 UTC
- Speed label should show "▶▶ 1,000×"
- Orange trajectory line should appear
- Orange dot should appear near Earth
- Artemis II card should appear top-right

Click again:
- All overlay elements should disappear
- Sim time should restore to before the toggle

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: wire Artemis II toggle button with sim state save/restore"
```

---

### Task 7: Per-frame spacecraft animation and mission card updates

**Files:**
- Modify: `index.html` (JS `<script>` block — `animate()` function)

- [ ] **Step 1: Add the Artemis per-frame update block in `animate()`**

In `animate()`, add after `updateHUD(simDate, moonState);`:

```javascript
      // Artemis II per-frame update
      if (artemisState.active) {
        const pos = getSpacecraftPosition(simDate);
        if (pos) {
          spacecraftMesh.visible = true;
          spacecraftMesh.position.copy(pos);
          // Pulse: scale oscillates using wall clock (independent of sim speed)
          const pulse = 1 + 0.35 * Math.sin(Date.now() * 0.004);
          spacecraftMesh.scale.setScalar(pulse);
          // Update mission card
          const elapsedMs = simDate.getTime() - ARTEMIS2.launch.getTime();
          artemisCardEls.met.textContent   = formatMET(elapsedMs);
          artemisCardEls.phase.textContent = getArtemisPhase(elapsedMs) || 'Complete';
        } else {
          // Outside mission window — hide dot but keep trajectory line
          spacecraftMesh.visible = false;
          const elapsedMs = simDate.getTime() - ARTEMIS2.launch.getTime();
          artemisCardEls.met.textContent   = elapsedMs < 0 ? 'Pre-launch' : 'Mission Complete';
          artemisCardEls.phase.textContent = '—';
        }
      }
```

- [ ] **Step 2: Verify spacecraft moves along the trajectory**

Open `http://localhost:8181`. Toggle Artemis II on. With 1000× speed:
- The orange dot should start near Earth and move outward toward the Moon over ~14 real seconds
- Around sim-time April 6, the dot should pass close to the Moon
- The MET counter in the card should increment rapidly
- The phase should change from "Trans-Lunar Injection" → "Translunar Coast" → "Lunar Flyby" → "Return Coast"
- After sim-time April 11, the dot should disappear and card should show "Mission Complete"

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: animate Artemis II spacecraft dot along trajectory with mission card updates"
```

---

### Task 8: Final polish — speed label restore on toggle-off and sync backup

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Fix speed label on toggle-off for non-live restores**

The toggle-off block already handles the live case via `timeController.goLive()` which resets the label. For the non-live case, add a label update after the `else` block. Find in the toggle-off `else` block, after `timeController._lastWall = Date.now();`:

```javascript
          const mult = artemisState.savedMultiplier;
          const absM = Math.abs(mult);
          const dir  = mult < 0 ? '\u25c4 ' : '\u25ba ';
          speedLabel.textContent = dir + absM.toLocaleString() + '\u00d7' + (mult < 0 ? ' (reverse)' : '');
          setActiveBtn(false);
          // re-highlight the speed button that matches the restored multiplier
          const btnMap = { 100: 'btn-fwd1', 1000: 'btn-fwd2', 100000: 'btn-fwd3',
                          '-100': 'btn-rev1', '-1000': 'btn-rev2', '-100000': 'btn-rev3' };
          const matchBtn = document.getElementById(btnMap[String(mult)]);
          if (matchBtn) matchBtn.classList.add('active');
```

- [ ] **Step 2: Verify speed label restores correctly**

Test: set speed to 100× forward, then toggle Artemis II on, then toggle off. Speed label should return to "▶ 100×". Test again from LIVE mode — label should return to "LIVE — 1×".

- [ ] **Step 3: Sync the backup copy**

```bash
cp -r /home/neb/projects/earth-moon-viz/. "/media/neb/Samsung USB/obsidian/05_Andromeda/earth-moon/"
echo "Backup updated"
```

- [ ] **Step 4: Final commit**

```bash
git add index.html
git commit -m "feat: restore speed label correctly on Artemis II toggle-off"
```

---

## Acceptance Checklist

- [ ] Clicking "🚀 Artemis II" jumps sim to 2026-04-01T22:24:00Z at 1000×
- [ ] Orange trajectory line shows the free-return path Earth → Moon → Earth
- [ ] Orange pulsing dot tracks the spacecraft along the path in sim time
- [ ] Dot is hidden before launch and after splashdown
- [ ] Mission card shows MET, correct phase, and crew names
- [ ] Phase transitions at correct MET thresholds
- [ ] Toggling off restores the previous sim time and speed
- [ ] Speed buttons still work normally when Artemis overlay is inactive
- [ ] Orbit toggle button state not affected by Artemis toggle
