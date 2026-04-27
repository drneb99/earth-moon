# Earth–Moon Observatory

A scientifically accurate, real-time 3D visualization of the Earth-Moon system running entirely in the browser. No install, no build tools — just open and explore.

![Three.js](https://img.shields.io/badge/Three.js-r162-black) ![Single File](https://img.shields.io/badge/single%20file-index.html-blue)

## What it shows

- The Moon's exact position at any point in time, computed using Jean Meeus' _Astronomical Algorithms_ (Ch. 47 lunar theory, ~0.1° accuracy)
- Tidal locking — the Moon always faces the same side toward Earth
- Optical libration — the slight wobble that lets us see just over 50% of the Moon's surface over time
- The day/night terminator on Earth, tracking the real Sun direction
- City lights on Earth's dark side (NASA Black Marble 2016 data)
- Moon phase, distance, and orbital speed updated live
- Next full moon timestamp and countdown

## Controls

| Control | Action |
|---|---|
| Click + drag | Rotate the scene |
| Scroll | Zoom in/out |
| Click Earth or Moon | Switch camera focus |
| ◀◀◀ / ▶▶▶ buttons | Reverse/forward time at 100× / 1,000× / 100,000× |
| ● LIVE | Snap back to real time |
| ⏸ Pause | Freeze simulation |
| ⊙ Orbit | Toggle Moon's orbital path |

## Running locally

Requires a static HTTP server (ES modules don't work over `file://`):

```bash
git clone https://github.com/drneb99/earth-moon.git
cd earth-moon
python3 -m http.server 8181
```

Then open [http://localhost:8181](http://localhost:8181) in your browser.

## Astronomy model

The Moon's position is computed from the Meeus Ch. 47 truncated series using 39 longitude/distance terms and 30 latitude terms, with eccentricity correction applied to solar terms. Earth rotation uses Greenwich Mean Sidereal Time (GMST) so the day/night terminator matches real-world geography.

**Coordinate system:** 1 Three.js unit = 10,000 km. The ecliptic plane is the XZ plane; Y is ecliptic north. Earth sits at the origin.
