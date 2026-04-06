# Earth-Moon Orbital Visualizer — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` that renders a scientifically accurate, real-time 3D Earth-Moon system using Three.js, with tidal lock visualization and time controls.

**Architecture:** Three logical objects — `astronomyEngine` (stateless Meeus calculations), `timeController` (simulation time management), and a Three.js render loop that consumes them each frame. All inline in one HTML file, no build tools.

**Tech Stack:** Three.js r168 (ES module CDN), OrbitControls addon, vanilla JS, inline CSS.

---

## File

- Create: `earth-moon-viz/index.html`

## Scene Coordinate System

- 1 Three.js unit = 10,000 km
- Earth at origin
- Ecliptic plane = XZ plane, Y = ecliptic north
- Earth display radius: 2.0 units (visual scale-up from true 0.637)
- Moon display radius: 0.545 units (preserves Earth/Moon size ratio)
- Moon orbital distance: ~38.44 units (true to scale from Earth center)

---

## Task 1: HTML Shell + Three.js Bootstrap

**File:** `earth-moon-viz/index.html` (create)

- [ ] **Step 1: Create base HTML with full-screen canvas**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Earth–Moon Observatory</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { width: 100%; height: 100%; overflow: hidden; background: #000; }
    #canvas { display: block; width: 100%; height: 100%; }

    /* HUD */
    #hud {
      position: fixed; top: 16px; left: 16px;
      background: rgba(0,0,0,0.65);
      border: 1px solid rgba(255,255,255,0.15);
      border-radius: 8px;
      padding: 12px 16px;
      color: #e0e0e0;
      font-family: 'Courier New', monospace;
      font-size: 13px;
      line-height: 1.7;
      min-width: 220px;
      user-select: none;
      backdrop-filter: blur(4px);
    }
    #hud .hud-title {
      font-size: 11px; letter-spacing: 2px;
      color: #aaa; text-transform: uppercase; margin-bottom: 6px;
    }
    #hud .hud-row { display: flex; justify-content: space-between; gap: 12px; }
    #hud .hud-label { color: #888; }
    #hud .hud-value { color: #fff; font-weight: bold; }
    #live-badge {
      display: inline-block;
      width: 8px; height: 8px; border-radius: 50%;
      background: #00ff88; margin-right: 6px;
      animation: pulse 1.5s infinite;
    }
    @keyframes pulse {
      0%,100% { opacity: 1; } 50% { opacity: 0.3; }
    }
    #live-badge.hidden { display: none; }

    /* Time Controls */
    #controls {
      position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%);
      display: flex; flex-direction: column; align-items: center; gap: 8px;
      user-select: none;
    }
    #speed-label {
      color: #aaa; font-family: 'Courier New', monospace;
      font-size: 12px; letter-spacing: 1px;
    }
    #btn-row { display: flex; gap: 6px; }
    .ctrl-btn {
      background: rgba(0,0,0,0.7);
      border: 1px solid rgba(255,255,255,0.2);
      border-radius: 6px;
      color: #fff;
      font-family: 'Courier New', monospace;
      font-size: 12px;
      padding: 6px 10px;
      cursor: pointer;
      transition: background 0.15s, border-color 0.15s;
    }
    .ctrl-btn:hover { background: rgba(255,255,255,0.12); border-color: rgba(255,255,255,0.4); }
    .ctrl-btn.active { background: rgba(0,255,136,0.15); border-color: #00ff88; color: #00ff88; }

    /* Responsive */
    @media (max-width: 480px) {
      #hud { font-size: 11px; min-width: 170px; padding: 8px 12px; top: 8px; left: 8px; }
      .ctrl-btn { font-size: 11px; padding: 5px 8px; }
    }
  </style>
</head>
<body>
  <canvas id="canvas"></canvas>

  <!-- HUD -->
  <div id="hud">
    <div class="hud-title">Earth–Moon Observatory</div>
    <div class="hud-row">
      <span class="hud-label"><span id="live-badge"></span>Time (UTC)</span>
      <span class="hud-value" id="hud-time">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label">Distance</span>
      <span class="hud-value" id="hud-dist">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label">Phase</span>
      <span class="hud-value" id="hud-phase">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label">Speed</span>
      <span class="hud-value" id="hud-speed">—</span>
    </div>
    <div class="hud-row">
      <span class="hud-label">Libration</span>
      <span class="hud-value" id="hud-lib">—</span>
    </div>
  </div>

  <!-- Time Controls -->
  <div id="controls">
    <div id="speed-label">LIVE — 1×</div>
    <div id="btn-row">
      <button class="ctrl-btn" id="btn-rev3">◀◀◀ 100k×</button>
      <button class="ctrl-btn" id="btn-rev2">◀◀ 1000×</button>
      <button class="ctrl-btn" id="btn-rev1">◀ 100×</button>
      <button class="ctrl-btn" id="btn-pause">⏸ Pause</button>
      <button class="ctrl-btn" id="btn-fwd1">▶ 100×</button>
      <button class="ctrl-btn" id="btn-fwd2">▶▶ 1000×</button>
      <button class="ctrl-btn" id="btn-fwd3">▶▶▶ 100k×</button>
      <button class="ctrl-btn active" id="btn-live">● LIVE</button>
    </div>
  </div>

  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.168.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.168.0/examples/jsm/"
    }
  }
  </script>
  <script type="module">
    import * as THREE from 'three';
    import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

    // ── Scene bootstrap ──────────────────────────────────────────────────────
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.setSize(window.innerWidth, window.innerHeight);

    const scene = new THREE.Scene();

    const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 100000);
    camera.position.set(0, 20, 80);

    const controls = new OrbitControls(camera, canvas);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.minDistance = 5;
    controls.maxDistance = 500;

    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });

    // Ambient fill
    scene.add(new THREE.AmbientLight(0x111122, 0.4));

    // PLACEHOLDER: astronomy engine, time controller, scene objects, and render loop go here
    function animate() {
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
    }
    animate();
  </script>
</body>
</html>
```

- [ ] **Step 2: Serve and verify blank black canvas loads with no console errors**

```bash
cd /home/neb/projects/earth-moon-viz
python3 -m http.server 8080
# Open http://localhost:8080 in browser
# Expected: black screen, no JS errors in console
```

---

## Task 2: Astronomy Engine

Add the `astronomyEngine` object immediately after the `// PLACEHOLDER` comment (before `animate()`).

- [ ] **Step 1: Add the astronomy engine**

Replace `// PLACEHOLDER: astronomy engine, time controller, scene objects, and render loop go here` with the following block. This implements Meeus Ch. 47 truncated lunar theory.

```javascript
// ── Astronomy Engine (Meeus Ch. 47) ─────────────────────────────────────────
const astronomyEngine = {
  toRad: d => d * Math.PI / 180,
  toDeg: r => r * 180 / Math.PI,
  norm: d => ((d % 360) + 360) % 360,

  julianDay(date) {
    let y = date.getUTCFullYear();
    let m = date.getUTCMonth() + 1;
    const d = date.getUTCDate() +
      (date.getUTCHours() * 3600 + date.getUTCMinutes() * 60 + date.getUTCSeconds()) / 86400;
    if (m <= 2) { y--; m += 12; }
    const A = Math.floor(y / 100);
    const B = 2 - A + Math.floor(A / 4);
    return Math.floor(365.25 * (y + 4716)) + Math.floor(30.6001 * (m + 1)) + d + B - 1524.5;
  },

  getMoonState(date) {
    const JD  = this.julianDay(date);
    const T   = (JD - 2451545.0) / 36525;
    const T2  = T * T;

    // Fundamental arguments (degrees, normalized 0–360)
    const Lp = this.norm(218.3164477 + 481267.88123421*T - 0.0015786*T2); // Moon mean longitude
    const D  = this.norm(297.8501921 + 445267.1114034 *T - 0.0018819*T2); // Moon mean elongation
    const M  = this.norm(357.5291092 + 35999.0502909  *T - 0.0001536*T2); // Sun mean anomaly
    const Mp = this.norm(134.9633964 + 477198.8675055 *T + 0.0087414*T2); // Moon mean anomaly
    const F  = this.norm(93.2720950  + 483202.0175233 *T - 0.0036539*T2); // Moon arg of latitude
    const Om = this.norm(125.0445479 - 1934.1362608   *T + 0.0020754*T2); // Ascending node

    // Eccentricity correction for solar terms
    const E  = 1 - 0.002516*T - 0.0000074*T2;
    const E2 = E * E;

    const Dr = this.toRad(D), Mr = this.toRad(M), Mpr = this.toRad(Mp);
    const Fr = this.toRad(F), Omr = this.toRad(Om);

    // Longitude + distance terms (Meeus Table 45.A)
    // [D, M, M', F, sumL_coeff (×1e-6 deg), sumR_coeff (×0.001 km)]
    const lrTerms = [
      [0, 0, 1, 0,  6288774, -20905355],
      [2, 0,-1, 0,  1274027,  -3699111],
      [2, 0, 0, 0,   658314,  -2955968],
      [0, 0, 2, 0,   213618,   -569925],
      [0, 1, 0, 0,  -185116,     48888],
      [0, 0, 0, 2,  -114332,     -3149],
      [2, 0,-2, 0,    58793,    246158],
      [2,-1,-1, 0,    57066,   -152138],
      [2, 0, 1, 0,    53322,   -170733],
      [2,-1, 0, 0,    45758,   -204586],
      [0, 1,-1, 0,   -40923,   -129620],
      [1, 0, 0, 0,   -34720,    108743],
      [0, 1, 1, 0,   -30383,    104755],
      [2, 0, 0,-2,    15327,     10321],
      [0, 0, 1, 2,   -12528,         0],
      [0, 0, 1,-2,    10980,     79661],
      [4, 0,-1, 0,    10675,    -34782],
      [0, 0, 3, 0,    10034,    -23210],
      [4, 0,-2, 0,     8548,    -21636],
      [2, 1,-1, 0,    -7888,     24208],
      [2, 1, 0, 0,    -6766,     30824],
      [1, 0,-1, 0,    -5163,     -8379],
      [1, 1, 0, 0,     4987,    -16675],
      [2,-1, 1, 0,     4036,    -10034],
      [2, 0, 2, 0,     3994,    -12831],
      [4, 0, 0, 0,     3861,    -10445],
      [2, 0,-3, 0,     3665,    -11650],
      [0, 1,-2, 0,    -2689,     14403],
      [2, 0,-1, 2,    -2602,         0],
      [2,-1,-2, 0,     2390,     10056],
      [1, 0, 1, 0,    -2348,     -8379],
      [2,-2, 0, 0,     2236,     -7003],
      [0, 1, 2, 0,    -2120,         0],
      [0, 2, 0, 0,    -2069,         0],
      [2,-2,-1, 0,     2048,         0],
      [2, 0, 1,-2,    -1773,         0],
      [2, 0, 0, 2,    -1595,         0],
      [4,-1,-1, 0,     1215,         0],
      [0, 0, 2, 2,    -1110,         0],
    ];

    let sumL = 0, sumR = 0;
    for (const [d,m,mp,f, cl, cr] of lrTerms) {
      const arg = d*Dr + m*Mr + mp*Mpr + f*Fr;
      const eCorr = Math.abs(m) === 2 ? E2 : Math.abs(m) === 1 ? E : 1;
      sumL += eCorr * cl * Math.sin(arg);
      sumR += eCorr * cr * Math.cos(arg);
    }
    // Additive longitude corrections
    sumL += 3958 * Math.sin(this.toRad(119.75 + 131.849 * T));
    sumL += 1962 * Math.sin(this.toRad(Lp) - Fr);
    sumL +=  318 * Math.sin(this.toRad(53.09 + 479264.29 * T));

    // Latitude terms (Meeus Table 45.B)
    // [D, M, M', F, sumB_coeff (×1e-6 deg)]
    const bTerms = [
      [0, 0, 0, 1,  5128122],
      [0, 0, 1, 1,   280602],
      [0, 0, 1,-1,   277693],
      [2, 0, 0,-1,   173237],
      [2, 0,-1, 1,    55413],
      [2, 0,-1,-1,    46271],
      [2, 0, 0, 1,    32573],
      [0, 0, 2, 1,    17198],
      [2, 0, 1,-1,     9266],
      [0, 0, 2,-1,     8822],
      [2,-1, 0,-1,     8216],
      [2, 0,-2,-1,     4324],
      [2, 0, 1, 1,     4200],
      [2, 1, 0,-1,    -3359],
      [2,-1,-1, 1,     2463],
      [2,-1, 0, 1,     2211],
      [2,-1,-1,-1,     2065],
      [0, 1,-1,-1,    -1870],
      [4, 0,-1,-1,     1828],
      [0, 1, 0, 1,    -1794],
      [0, 0, 0, 3,    -1749],
      [0, 1,-1, 1,    -1565],
      [1, 0, 0, 1,    -1491],
      [0, 1, 1, 1,    -1475],
      [0, 1, 1,-1,    -1410],
      [0, 1, 0,-1,    -1344],
      [1, 0, 0,-1,    -1335],
      [0, 0, 3, 1,     1107],
      [4, 0, 0,-1,     1021],
      [4, 0,-1, 1,      833],
    ];

    let sumB = 0;
    for (const [d,m,mp,f, cb] of bTerms) {
      const arg = d*Dr + m*Mr + mp*Mpr + f*Fr;
      const eCorr = Math.abs(m) === 2 ? E2 : Math.abs(m) === 1 ? E : 1;
      sumB += eCorr * cb * Math.sin(arg);
    }
    sumB -= 2235 * Math.sin(this.toRad(Lp));
    sumB +=  382 * Math.sin(this.toRad(207.14 + 14712.317 * T));
    sumB +=  175 * Math.sin(this.toRad(119.75 + 131.849 * T) - Fr);
    sumB +=  175 * Math.sin(this.toRad(119.75 + 131.849 * T) + Fr);
    sumB +=  127 * Math.sin(this.toRad(Lp) - Mpr);
    sumB -=  115 * Math.sin(this.toRad(Lp) + Mpr);

    // Geocentric ecliptic coords
    const lon  = this.norm(Lp + sumL / 1e6);  // degrees
    const lat  = sumB / 1e6;                   // degrees
    const dist = 385000.56 + sumR / 1000;      // km

    // 3D position in scene units (1 unit = 10,000 km)
    // Ecliptic frame: XZ plane = ecliptic plane, Y = ecliptic north
    const lonR = this.toRad(lon);
    const latR = this.toRad(lat);
    const r    = dist / 10000;
    const scenePos = new THREE.Vector3(
      r * Math.cos(latR) * Math.cos(lonR),
      r * Math.sin(latR),
     -r * Math.cos(latR) * Math.sin(lonR)
    );

    // Orbital speed (km/s) — vis-viva approximation
    const a = 384400;
    const speed = 1.022 * Math.sqrt(a * (2 / dist - 1 / a) / a); // normalized to mean 1.022

    // Optical libration in longitude (eccentricity term, max ±7.9°)
    const e = 0.0549;
    const libration = this.toDeg(Math.atan2(e * Math.sin(Mpr), 1 - e * Math.cos(Mpr)));

    // Phase (0=new,180=full) based on elongation D
    const phaseAngle = this.norm(D);
    let phaseName, phaseIcon;
    if      (phaseAngle <  22.5) { phaseName = 'New Moon';        phaseIcon = '🌑'; }
    else if (phaseAngle <  67.5) { phaseName = 'Waxing Crescent'; phaseIcon = '🌒'; }
    else if (phaseAngle < 112.5) { phaseName = 'First Quarter';   phaseIcon = '🌓'; }
    else if (phaseAngle < 157.5) { phaseName = 'Waxing Gibbous';  phaseIcon = '🌔'; }
    else if (phaseAngle < 202.5) { phaseName = 'Full Moon';       phaseIcon = '🌕'; }
    else if (phaseAngle < 247.5) { phaseName = 'Waning Gibbous';  phaseIcon = '🌖'; }
    else if (phaseAngle < 292.5) { phaseName = 'Last Quarter';    phaseIcon = '🌗'; }
    else if (phaseAngle < 337.5) { phaseName = 'Waning Crescent'; phaseIcon = '🌘'; }
    else                          { phaseName = 'New Moon';        phaseIcon = '🌑'; }

    return { scenePos, distance: dist, speed, libration, phaseName, phaseIcon, lon, lat };
  },

  getSunDirection(date) {
    const JD = this.julianDay(date);
    const T  = (JD - 2451545.0) / 36525;
    const L0 = this.norm(280.46646 + 36000.76983 * T);
    const M  = this.norm(357.52911 + 35999.05029 * T);
    const Mr = this.toRad(M);
    const C  = (1.914602 - 0.004817*T) * Math.sin(Mr)
             + (0.019993 - 0.000101*T) * Math.sin(2*Mr)
             +  0.000289                * Math.sin(3*Mr);
    const sunLon = this.norm(L0 + C);
    const sl = this.toRad(sunLon);
    return new THREE.Vector3(Math.cos(sl), 0, -Math.sin(sl)).normalize();
  }
};
```

- [ ] **Step 2: Verify output looks reasonable**

Temporarily add after the engine definition, before `animate()`:
```javascript
const testDate = new Date('2026-04-06T00:00:00Z');
const s = astronomyEngine.getMoonState(testDate);
console.log('Moon dist km:', s.distance.toFixed(0));    // expect 370,000–400,000
console.log('Moon lon deg:', s.lon.toFixed(2));          // expect 0–360
console.log('Moon lat deg:', s.lat.toFixed(2));          // expect -5 to +5
console.log('Phase:', s.phaseName);
console.log('Libration°:', s.libration.toFixed(2));      // expect -7.9 to +7.9
console.log('Speed km/s:', s.speed.toFixed(3));          // expect ~0.9–1.1
```

Open http://localhost:8080, check browser console. Remove the test block after verifying.

---

## Task 3: Time Controller

Add after the astronomy engine block.

- [ ] **Step 1: Add time controller**

```javascript
// ── Time Controller ──────────────────────────────────────────────────────────
const timeController = {
  simTime:    Date.now(),         // ms — current simulation timestamp
  multiplier: 1,                  // positive = forward, negative = backward
  isLive:     true,
  paused:     false,
  _lastWall:  Date.now(),

  tick() {
    const now = Date.now();
    const wallDelta = now - this._lastWall;
    this._lastWall = now;
    if (this.isLive) {
      this.simTime = Date.now();   // stay pinned to wall clock
    } else if (!this.paused) {
      this.simTime += wallDelta * this.multiplier;
    }
    return new Date(this.simTime);
  },

  setSpeed(mult) {
    this.isLive    = false;
    this.paused    = false;
    this.multiplier = mult;
    this._lastWall  = Date.now();
  },

  goLive() {
    this.isLive    = true;
    this.paused    = false;
    this.multiplier = 1;
    this.simTime   = Date.now();
    this._lastWall  = Date.now();
  },

  pause() {
    this.isLive  = false;
    this.paused  = !this.paused;
    this._lastWall = Date.now();
  }
};
```

---

## Task 4: Starfield

Add after time controller.

- [ ] **Step 1: Add starfield**

```javascript
// ── Starfield ────────────────────────────────────────────────────────────────
(function buildStars() {
  const count   = 3000;
  const geo     = new THREE.BufferGeometry();
  const pos     = new Float32Array(count * 3);
  const sizes   = new Float32Array(count);
  const R       = 5000; // large sphere radius

  for (let i = 0; i < count; i++) {
    // Uniform random point on sphere surface
    const theta = Math.acos(2 * Math.random() - 1);
    const phi   = 2 * Math.PI * Math.random();
    pos[i*3  ]  = R * Math.sin(theta) * Math.cos(phi);
    pos[i*3+1]  = R * Math.sin(theta) * Math.sin(phi);
    pos[i*3+2]  = R * Math.cos(theta);
    sizes[i]    = 0.5 + Math.random() * 1.5;
  }

  geo.setAttribute('position', new THREE.BufferAttribute(pos,   3));
  geo.setAttribute('size',     new THREE.BufferAttribute(sizes, 1));

  const mat = new THREE.PointsMaterial({
    color: 0xffffff,
    size:  0.8,
    sizeAttenuation: false,
    transparent: true,
    opacity: 0.85
  });

  const stars = new THREE.Points(geo, mat);
  stars.matrixAutoUpdate = false; // fixed — never moves
  stars.updateMatrix();
  scene.add(stars);
})();
```

---

## Task 5: Earth

Add after the starfield block. Textures served from threejs.org CDN.

- [ ] **Step 1: Add Earth with texture, axial tilt, and atmosphere glow**

```javascript
// ── Earth ────────────────────────────────────────────────────────────────────
const EARTH_RADIUS   = 2.0;    // scene units
const EARTH_TILT_RAD = 23.44 * Math.PI / 180;

const texLoader = new THREE.TextureLoader();
const earthGroup = new THREE.Group(); // pivot for axial tilt
earthGroup.rotation.z = EARTH_TILT_RAD; // tilt north pole
scene.add(earthGroup);

const earthGeo = new THREE.SphereGeometry(EARTH_RADIUS, 64, 64);
const earthMat = new THREE.MeshPhongMaterial({
  map:      texLoader.load('https://threejs.org/examples/textures/planets/earth_atmos_2048.jpg'),
  specularMap: texLoader.load('https://threejs.org/examples/textures/planets/earth_specular_2048.jpg'),
  specular: new THREE.Color(0x333333),
  shininess: 15,
});
const earthMesh = new THREE.Mesh(earthGeo, earthMat);
earthGroup.add(earthMesh);

// Atmosphere halo (additive blending ring)
const atmGeo = new THREE.SphereGeometry(EARTH_RADIUS * 1.02, 64, 64);
const atmMat = new THREE.MeshPhongMaterial({
  color: 0x4488ff,
  transparent: true,
  opacity: 0.08,
  side: THREE.FrontSide,
  blending: THREE.AdditiveBlending,
  depthWrite: false,
});
earthGroup.add(new THREE.Mesh(atmGeo, atmMat));

// Sun directional light (positioned each frame)
const sunLight = new THREE.DirectionalLight(0xffffff, 1.2);
scene.add(sunLight);
```

---

## Task 6: Moon + Tidal Lock

Add after Earth block.

- [ ] **Step 1: Add Moon with texture and tidal-lock rotation logic**

```javascript
// ── Moon ─────────────────────────────────────────────────────────────────────
const MOON_RADIUS = 0.545; // scene units (proportional to Earth radius)

const moonGeo  = new THREE.SphereGeometry(MOON_RADIUS, 48, 48);
const moonMat  = new THREE.MeshPhongMaterial({
  map:      texLoader.load('https://threejs.org/examples/textures/planets/moon_1024.jpg'),
  shininess: 5,
  specular:  new THREE.Color(0x111111),
});
const moonMesh = new THREE.Mesh(moonGeo, moonMat);
scene.add(moonMesh);

// Helper: set Moon position and tidal-lock its face toward Earth (origin)
function updateMoon(state) {
  moonMesh.position.copy(state.scenePos);

  // Tidal lock: rotate Moon so its near-side (+Z local) faces Earth (origin)
  const toEarth = new THREE.Vector3(0,0,0).sub(moonMesh.position).normalize();
  const quat    = new THREE.Quaternion();
  quat.setFromUnitVectors(new THREE.Vector3(0, 0, 1), toEarth);
  moonMesh.quaternion.copy(quat);

  // Apply libration offset (rotation around local Y axis)
  moonMesh.rotateY(state.libration * Math.PI / 180);
}
```

---

## Task 7: Orbit Ring

Add after Moon block.

- [ ] **Step 1: Add elliptical orbit ring at correct inclination**

```javascript
// ── Orbit Ring ───────────────────────────────────────────────────────────────
(function buildOrbitRing() {
  const e    = 0.0549;               // Moon orbital eccentricity
  const a    = 384400 / 10000;       // semi-major axis in scene units (~38.44)
  const b    = a * Math.sqrt(1 - e*e); // semi-minor axis
  const inc  = 5.145 * Math.PI / 180; // orbital inclination to ecliptic

  // Build ellipse points
  const segments = 256;
  const pts      = [];
  for (let i = 0; i <= segments; i++) {
    const angle = (i / segments) * 2 * Math.PI;
    // Ellipse center is offset from focus by a*e (Earth is at focus)
    pts.push(new THREE.Vector3(
      a * Math.cos(angle) - a * e,  // shift so Earth is at focus
      0,
      b * Math.sin(angle)
    ));
  }

  const geo = new THREE.BufferGeometry().setFromPoints(pts);
  const mat = new THREE.LineBasicMaterial({
    color: 0xffffff,
    transparent: true,
    opacity: 0.18,
    depthWrite: false,
  });

  const ring = new THREE.Line(geo, mat);
  ring.rotation.x = inc; // tilt by orbital inclination
  scene.add(ring);
})();
```

---

## Task 8: Per-Frame Update Loop (Connecting Everything)

Replace the existing `animate()` function with the full version.

- [ ] **Step 1: Replace the stub animate() with the connected loop**

```javascript
// ── Per-frame update ─────────────────────────────────────────────────────────
const _tmpV3 = new THREE.Vector3();

function updateHUD(date, moonState) {
  // Time
  const iso      = date.toISOString().replace('T', ' ').slice(0, 19) + ' UTC';
  document.getElementById('hud-time').textContent  = iso;
  document.getElementById('live-badge').className  = timeController.isLive ? '' : 'hidden';

  // Moon data
  document.getElementById('hud-dist').textContent  = moonState.distance.toFixed(0) + ' km';
  document.getElementById('hud-phase').textContent = moonState.phaseIcon + ' ' + moonState.phaseName;
  document.getElementById('hud-speed').textContent = moonState.speed.toFixed(3) + ' km/s';
  document.getElementById('hud-lib').textContent   = (moonState.libration >= 0 ? '+' : '') +
    moonState.libration.toFixed(2) + '°';
}

function animate() {
  requestAnimationFrame(animate);

  const simDate  = timeController.tick();
  const moonState = astronomyEngine.getMoonState(simDate);
  const sunDir   = astronomyEngine.getSunDirection(simDate);

  // Sun light
  sunLight.position.copy(sunDir).multiplyScalar(1000);

  // Earth rotation: 1 full turn per 24 sim-hours
  const dayFrac = (simDate.getUTCHours() * 3600 + simDate.getUTCMinutes() * 60 +
    simDate.getUTCSeconds()) / 86400;
  earthMesh.rotation.y = dayFrac * 2 * Math.PI;

  // Moon position + tidal lock
  updateMoon(moonState);

  // HUD
  updateHUD(simDate, moonState);

  controls.update();
  renderer.render(scene, camera);
}
animate();
```

- [ ] **Step 2: Open http://localhost:8080, verify**
  - Earth visible with texture and day/night terminator
  - Moon orbiting, near-side face toward Earth
  - HUD populating all fields
  - No console errors

---

## Task 9: Time Control Button Wiring

- [ ] **Step 1: Add button event listeners before `animate()` call**

```javascript
// ── Button wiring ────────────────────────────────────────────────────────────
const speedLabel = document.getElementById('speed-label');
const liveBtn    = document.getElementById('btn-live');

function setActiveBtn(exceptLive = false) {
  document.querySelectorAll('.ctrl-btn').forEach(b => b.classList.remove('active'));
  if (exceptLive) liveBtn.classList.remove('active');
}

function applySpeed(mult, label) {
  timeController.setSpeed(mult);
  speedLabel.textContent = label;
  setActiveBtn(true);
}

document.getElementById('btn-rev3').addEventListener('click', () => applySpeed(-100000, '◀ 100,000× (reverse)'));
document.getElementById('btn-rev2').addEventListener('click', () => applySpeed(-1000,   '◀ 1,000× (reverse)'));
document.getElementById('btn-rev1').addEventListener('click', () => applySpeed(-100,    '◀ 100× (reverse)'));

document.getElementById('btn-fwd1').addEventListener('click', () => applySpeed(100,    '▶ 100×'));
document.getElementById('btn-fwd2').addEventListener('click', () => applySpeed(1000,   '▶▶ 1,000×'));
document.getElementById('btn-fwd3').addEventListener('click', () => applySpeed(100000, '▶▶▶ 100,000×'));

document.getElementById('btn-pause').addEventListener('click', () => {
  timeController.pause();
  setActiveBtn(true);
  document.getElementById('btn-pause').classList.add('active');
  speedLabel.textContent = timeController.paused ? '⏸ Paused' : '▶ Resuming';
});

document.getElementById('btn-live').addEventListener('click', () => {
  timeController.goLive();
  setActiveBtn();
  liveBtn.classList.add('active');
  speedLabel.textContent = 'LIVE — 1×';
});
```

- [ ] **Step 2: Verify controls work**
  - Click ▶▶▶: Moon visibly moves fast around Earth, HUD time advances rapidly
  - Click ◀◀◀: Time reverses, Moon moves backward
  - Click ● LIVE: snaps back to current real time, badge reappears
  - Click ⏸: Moon freezes

---

## Task 10: Final Polish + Responsive Check

- [ ] **Step 1: Add a subtle background gradient to the page (stars + depth)**

Add inside the `<style>` block:
```css
body { background: radial-gradient(ellipse at center, #050a15 0%, #000005 100%); }
```

- [ ] **Step 2: Verify mobile layout**
  - Open DevTools, toggle device emulation to iPhone 12
  - HUD should be readable at smaller size (font-size: 11px kicks in at 480px)
  - Pinch zoom and one-finger rotate should work (OrbitControls handles touch natively)
  - Controls row may overflow on very small screens — add `flex-wrap: wrap` and `max-width: 95vw` to `#btn-row`

```css
#btn-row { display: flex; gap: 6px; flex-wrap: wrap; justify-content: center; max-width: 95vw; }
```

- [ ] **Step 3: Final end-to-end check**

```
Open http://localhost:8080

Verify:
  ✓ Black space background with stars
  ✓ Earth at center, textured, spinning, day/night visible
  ✓ Moon orbiting with near-side facing Earth
  ✓ Orbit ring ellipse visible
  ✓ HUD shows date (today's date), distance ~370–400k km, correct phase name
  ✓ Speed buttons change simulation speed
  ✓ LIVE button returns to real time
  ✓ Click-drag rotates the scene
  ✓ Scroll zooms in/out
  ✓ No console errors
```

---

## Self-Review vs Spec

| Spec Requirement | Task |
|---|---|
| Three.js via CDN | Task 1 |
| Earth at center, Moon orbiting | Tasks 5, 6 |
| Moon's actual current orbital position (Meeus) | Task 2 |
| Real-time animation | Task 8 |
| Orbital path ring | Task 7 |
| Stars background | Task 4 |
| Click-drag rotate + scroll zoom | Task 1 (OrbitControls) |
| Realistic textures from CDN (threejs.org) | Tasks 5, 6 |
| HUD: date/time, speed, distance | Task 8 |
| Responsive / mobile | Task 1 CSS + Task 10 |
| Single self-contained index.html | All tasks |
| Tidal locking (near side faces Earth) | Task 6 |
| Libration | Tasks 2, 6 |
| Time controls with LIVE mode | Tasks 3, 9 |
| Earth axial tilt 23.44° | Task 5 |
| Day/night terminator | Tasks 5, 8 |
| Moon phase name + icon | Tasks 2, 8 |
