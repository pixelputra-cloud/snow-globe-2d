# Product Requirements Document
## Interactive 2D Snow Globe
**Version:** 1.1  
**Prepared for:** Claude Code  
**Platform:** Web (Desktop & Mobile)  
**Stack:** Vanilla HTML / CSS / JS (Single file)

---

## 1. Project Overview

An interactive 2D snow globe widget built as a single self-contained HTML file. The user supplies three PNG assets (orb, scene, base). Snow particles are procedurally generated on a `<canvas>` element using a physics simulation with gravity, drift, and velocity-reactive behaviour. The globe responds to drag-and-release and click/tap interactions. The widget is designed to embed cleanly in a portfolio page.

---

## 2. Asset Specification

All assets are user-supplied PNGs with transparent backgrounds. The implementation must reference them by filename and make swapping trivial.

| Asset File | Description | Render Order (back → front) |
|---|---|---|
| `scene.png` | Nature illustration (interior of globe) | 1 — Bottommost |
| `base.png` | Decorative stand / base below the orb | 2 — Above scene |
| `orb.png` | Glass sphere shell with transparent interior | 4 — Topmost (renders over snow) |

> **Critical compositing rule:** Snow particles must render **between** `scene.png` and `orb.png` so they appear to exist inside the glass. The orb's glass overlay (reflections, edge shading) must sit on top of the snow layer.

### Asset Loading
- All three PNGs are loaded via `new Image()` before the animation loop starts.
- A simple loading state (spinner or opacity fade) is shown until all assets resolve.
- If an asset fails to load, a console warning is logged and a fallback placeholder is drawn.

---

## 3. Layout & Sizing

### Dimensions
- **Base canvas size:** 500px wide × 600px tall
- **Globe interior clip region:** Circular mask centered within the canvas, diameter approximately 380px (adjustable via a single `GLOBE_RADIUS` constant)
- **Responsive behaviour:** The entire widget scales down proportionally on viewports narrower than 520px using CSS `transform: scale()` so layout never breaks

### Centering
- The widget is centered horizontally and vertically on the page using CSS flexbox on the `<body>`
- Background of the page is transparent so it embeds cleanly into any portfolio layout

### Layer Stack (DOM + Canvas)
```
<div class="globe-wrapper">
  ├── <img class="asset-scene" />       ← scene.png (CSS background layer)
  ├── <canvas id="snow-canvas" />       ← procedural snow particles
  ├── <img class="asset-orb" />         ← orb.png (glass overlay, pointer-events: none)
  └── <img class="asset-base" />        ← base.png (outside/below the canvas clip)
</div>
```

> Alternatively, all layers including assets can be drawn onto a single `<canvas>` in the correct order. **Preferred approach:** Single `<canvas>` for full control over compositing and clipping. Assets are drawn each frame in z-order: scene → snow particles → orb glass overlay.

---

## 4. Snow Particle System

### Particle Properties
Each particle is an object with the following properties:

| Property | Type | Description |
|---|---|---|
| `x` | float | Current X position (canvas coords) |
| `y` | float | Current Y position (canvas coords) |
| `vx` | float | Horizontal velocity |
| `vy` | float | Vertical velocity |
| `radius` | float | Particle radius (range: 1.5–4px) |
| `opacity` | float | Alpha value (range: 0.4–1.0) |
| `mass` | float | Affects gravity and drag (derived from radius) |
| `wobble` | float | Per-particle horizontal sine oscillation offset |
| `wobbleSpeed` | float | Frequency of the horizontal wobble |
| `wobbleAmp` | float | Amplitude of the horizontal wobble |
| `settling` | boolean | True when particle has nearly stopped |
| `fadeTimer` | float | Countdown to fade-out once settled |

### Particle Counts
| State | Particle Count |
|---|---|
| Idle (ambient) | 12–18 slowly drifting particles |
| After light tap/click | +40 new particles spawned (total ~55) |
| After vigorous drag | +60–80 new particles spawned (total ~90) |
| Hard cap | 120 particles maximum at any time |

Particles beyond the cap are not spawned. Old settled particles are recycled first.

### Spawn Distribution
- Particles spawn at random X positions across the full interior width of the globe
- Y spawn position: top 20% of the globe interior
- Initial `vy` (downward): small positive value (0.2–0.8)
- Initial `vx`: near zero at idle; inherits shake impulse direction on interaction

### Physics Constants (all defined as named constants at top of file)
```js
const GRAVITY         = 0.012;   // Downward acceleration per frame
const DRAG            = 0.985;   // Velocity damping multiplier per frame (air resistance)
const WOBBLE_AMP_BASE = 0.6;     // Base horizontal sine oscillation amplitude
const SETTLE_SPEED    = 0.04;    // Velocity threshold below which particle is "settled"
const FADE_DURATION   = 180;     // Frames before a settled particle fades out (~3s at 60fps)
const IDLE_SPAWN_RATE = 0.08;    // Probability per frame of spawning a new idle flake
```

### Gravity & Drift Behaviour
1. Each frame: `vy += GRAVITY * particle.mass`
2. Each frame: `vx *= DRAG`, `vy *= DRAG`
3. Horizontal wobble: `x += Math.sin(frame * wobbleSpeed + wobble) * wobbleAmp`
4. Position update: `x += vx`, `y += vy`

### Globe Boundary Collision
- Particles are clipped to the circular interior using the globe's radius and center point
- If a particle's position exceeds the boundary circle, it is reflected inward with dampened velocity: `v *= -0.2`
- Bottom arc of the globe acts as the floor — particles slow and stop here

### Settling & Fade-out
1. When `Math.abs(vx) + Math.abs(vy) < SETTLE_SPEED` for 3 consecutive frames, particle enters `settling = true`
2. `fadeTimer` begins counting down from `FADE_DURATION`
3. Particle `opacity` linearly interpolates to 0 as `fadeTimer` reaches 0
4. At opacity 0, particle is removed from the array and may be respawned as an idle flake

---

## 5. Interaction Design

### 5A. Drag and Release

**Goal:** Dragging the globe and releasing creates a directional snow burst whose intensity and direction map to the drag vector.

**Implementation:**
1. On `pointerdown` on the globe wrapper: record `dragStart = { x, y, time }`
2. On `pointermove` while dragging: apply a subtle CSS `translate` to the globe wrapper (max ±8px) for tactile feedback
3. On `pointerup`: calculate `dragVector = { dx: end.x - start.x, dy: end.y - start.y }`
4. Calculate `speed = Math.hypot(dx, dy) / timeDelta` (clamped to a max)
5. Apply shake impulse: distribute `speed * impulseScale` across all existing particles as `vx += dx_normalised * impulse`, `vy += dy_normalised * impulse`
6. Spawn new particles proportional to speed (see counts above)
7. Reset globe CSS transform back to origin with a CSS transition of `200ms ease-out`

**Directional mapping:**
- Drag left → snow swirls left (negative `vx` bias)
- Drag right → snow swirls right (positive `vx` bias)
- Drag down → brief upward burst then gravity takes over (negative `vy` impulse)
- Drag up → particles scatter upward then arc back down under gravity

### 5B. Click / Tap

**Goal:** A single tap triggers a short sharp omnidirectional burst.

**Implementation:**
1. Differentiate tap from drag: if `pointerup` fires with `Math.hypot(dx, dy) < 8px` and `timeDelta < 300ms`, treat as a tap
2. On tap: spawn ~40 particles from the top of the globe with randomised `vx` (−2.5 to +2.5) and `vy` (0.5 to 2.5)
3. Apply a small radial impulse to all existing particles
4. Rapid tap stacking: if a second tap occurs within 500ms of the first, multiply impulse by 1.4×. Stack up to 3× cap.
5. Apply a brief CSS shake animation to the globe wrapper: `keyframes` translating ±5px over 120ms

### 5C. Idle Ambient State

**Goal:** When no interaction has occurred for > 3 seconds, maintain 12–18 slowly drifting flakes to suggest the globe was recently shaken.

**Implementation:**
- Each frame, if `particles.length < IDLE_MIN (12)` and `Math.random() < IDLE_SPAWN_RATE`, spawn one slow particle
- Idle particles have `vx` in range −0.3 to +0.3 and `vy` in range 0.2 to 0.5
- Idle particles have higher `wobbleAmp` for a dreamy floaty look
- Idle particles settle and fade normally, replaced by new ones to maintain the minimum count

### 5D. Device Motion — Mobile Shake (Progressive Enhancement)

**Goal:** On mobile devices, physically shaking the phone triggers a snow burst matching the shake direction and intensity. This is a progressive enhancement — desktop users are unaffected, and mobile users who decline permission retain full drag/tap functionality.

**Detection & Feature Gating**
- On page load, check `'DeviceMotionEvent' in window` to detect motion API availability
- If detected AND the device is mobile (check `navigator.maxTouchPoints > 1` or viewport width), show the **"Enable shake" button**
- If `DeviceMotionEvent` is absent (desktop browsers), the button is hidden entirely — no UI clutter

**"Enable Shake" Button — UI Spec**
- A small, unobtrusive pill button positioned below the globe base
- Label: `"Enable shake"` with a subtle shake icon (e.g. 📳 or a custom SVG waveform)
- Style: semi-transparent, matches the overall globe aesthetic — not a generic browser-style button
- On click: call `DeviceMotionEvent.requestPermission()` (iOS 13+)
  - If granted → hide button, activate motion listener, show a brief `"Shake me!"` hint that fades after 2s
  - If denied → hide button, log to console, motion remains disabled silently
- On Android: `DeviceMotionEvent.requestPermission` does not exist — skip the permission call and activate the listener directly on button click

**Permission Flow (iOS)**
```
User taps "Enable shake"
  → DeviceMotionEvent.requestPermission() called
    → Browser native permission dialog appears
      → Granted: listener attached, button hidden, hint shown
      → Denied:  button hidden, motion disabled, drag/tap unaffected
```

**Motion Processing**
1. Listen to `window.addEventListener('devicemotion', handler)`
2. Read `event.accelerationIncludingGravity` → `{ x, y, z }`
3. Calculate delta from previous frame: `deltaX = accel.x - prevAccel.x`, `deltaY = accel.y - prevAccel.y`
4. Compute magnitude: `magnitude = Math.hypot(deltaX, deltaY)`
5. Apply threshold: only trigger if `magnitude > MOTION_THRESHOLD (12)` — filters out walking vibration and ambient movement
6. Apply cooldown: enforce `MOTION_COOLDOWN_MS (350ms)` between triggered bursts to prevent continuous agitation
7. Normalise direction: map `deltaX / magnitude` and `deltaY / magnitude` to `vx` / `vy` impulse directions
8. Account for orientation: if `screen.orientation.angle === 90 || 270`, swap X/Y axes

**Impulse Mapping**
- `impulse = clamp(magnitude * MOTION_IMPULSE_SCALE, 1, MOTION_MAX_IMPULSE)`
- Apply to all existing particles: `vx += normX * impulse`, `vy += normY * impulse`
- Spawn new particles proportional to impulse strength (same burst counts as drag interaction)
- Direction follows physical shake: shake left → snow swirls left, sharp upward flick → particles scatter upward

**New Constants**
```js
const MOTION_THRESHOLD     = 12;    // Min acceleration delta to trigger a burst
const MOTION_COOLDOWN_MS   = 350;   // Ms between motion-triggered bursts
const MOTION_IMPULSE_SCALE = 0.18;  // Converts raw accel delta to particle impulse
const MOTION_MAX_IMPULSE   = 6;     // Caps the maximum impulse from a shake
```

**Graceful Degradation Summary**

| Platform | Motion Support | Button Shown | Behaviour |
|---|---|---|---|
| Desktop (all browsers) | ✗ | Hidden | Drag + tap only |
| Android Chrome | ✓ (no permission needed) | Shown | Tap button → instant activation |
| iOS Safari 13+ | ✓ (permission required) | Shown | Tap button → native dialog → activate |
| iOS Safari (denied) | ✗ | Hidden after denial | Drag + tap only, silent fallback |

---

## 6. Rendering Pipeline (Single Canvas, 60fps RAF Loop)

Each frame, the canvas is cleared and redrawn in this exact order:

```
1. ctx.clearRect(full canvas)
2. Draw scene.png (full canvas bounds)
3. ctx.save() → apply circular clip path (globe interior)
4.   Draw snow particles (circles with radial gradient for soft look)
5. ctx.restore() (remove clip)
6. Draw orb.png (full canvas bounds, on top of snow)
7. Draw base.png (positioned below orb, outside clip)
```

### Snow Particle Rendering
- Each particle drawn as `ctx.arc()` filled circle
- Soft edge via a short `ctx.shadowBlur = 4` with `shadowColor = 'rgba(255,255,255,0.8)'`
- Fill colour: `rgba(255, 255, 255, particle.opacity)`
- Radius ranges from 1.5px (small) to 4px (large) — randomly assigned at spawn

---

## 7. Technical Specification

### File Structure
```
snow-globe/
├── index.html          ← Single self-contained file (HTML + CSS + JS)
├── orb.png             ← User asset: glass orb
├── scene.png           ← User asset: interior nature scene
└── base.png            ← User asset: stand / base
```

### index.html Architecture
```
<head>
  <style> ... all CSS ... </style>
</head>
<body>
  <div class="globe-wrapper">
    <canvas id="snow-canvas" width="500" height="600"></canvas>
  </div>
  <script>
    // === CONSTANTS ===
    // === ASSET LOADING ===
    // === PARTICLE CLASS / FACTORY ===
    // === PHYSICS UPDATE ===
    // === RENDER LOOP ===
    // === INTERACTION HANDLERS (pointer + motion) ===
    // === DEVICE MOTION HANDLER ===
    // === INIT ===
  </script>
</body>
```

### No Dependencies
- Zero external libraries
- No build step
- Works by opening `index.html` directly in a browser or embedding via `<iframe>`

### Browser Support
- All modern browsers (Chrome, Firefox, Safari, Edge)
- Mobile: iOS Safari 14+, Android Chrome 90+
- Uses: `canvas 2D API`, `pointer events`, `requestAnimationFrame`, `CSS transform`, `DeviceMotionEvent`

---

## 8. Configuration Constants (Easy Tuning)

All tuneable values must be declared as named constants at the top of the `<script>` block so they are easy to adjust without hunting through physics code:

```js
// --- Asset paths ---
const ASSET_ORB   = 'orb.png';
const ASSET_SCENE = 'scene.png';
const ASSET_BASE  = 'base.png';

// --- Layout ---
const CANVAS_W      = 500;
const CANVAS_H      = 600;
const GLOBE_CENTER  = { x: 250, y: 270 };  // Center of the circular globe interior
const GLOBE_RADIUS  = 190;                  // Radius of the clipping circle in px

// --- Particle limits ---
const PARTICLE_MAX  = 120;
const IDLE_MIN      = 12;
const IDLE_MAX      = 18;

// --- Physics ---
const GRAVITY       = 0.012;
const DRAG          = 0.985;
const SETTLE_SPEED  = 0.04;
const FADE_DURATION = 180;

// --- Idle spawning ---
const IDLE_SPAWN_RATE = 0.08;

// --- Interaction ---
const TAP_MAX_DIST    = 8;     // px — max drag distance to count as a tap
const TAP_MAX_TIME    = 300;   // ms — max duration to count as a tap
const TAP_STACK_WIN   = 500;   // ms — window for rapid tap amplification
const TAP_STACK_MAX   = 3;     // max tap stack multiplier count
const DRAG_IMPULSE_SCALE = 3.5; // Multiplier converting drag speed to particle impulse

// --- Device motion ---
const MOTION_THRESHOLD     = 12;    // Min acceleration delta to trigger a burst
const MOTION_COOLDOWN_MS   = 350;   // Ms between motion-triggered bursts
const MOTION_IMPULSE_SCALE = 0.18;  // Converts raw accel delta to particle impulse
const MOTION_MAX_IMPULSE   = 6;     // Caps the maximum impulse from a shake
```

---

## 9. Responsive Behaviour

```css
.globe-wrapper {
  transform-origin: center top;
}

@media (max-width: 520px) {
  .globe-wrapper {
    transform: scale(0.75);
  }
}

@media (max-width: 380px) {
  .globe-wrapper {
    transform: scale(0.6);
  }
}
```

The canvas dimensions stay fixed (500×600). Only the visual scale changes. This avoids any canvas re-initialisation on resize.

---

## 10. Acceptance Criteria

| # | Criteria |
|---|---|
| 1 | All three PNG assets load and composite correctly: scene → snow → orb → base |
| 2 | Snow particles are clipped strictly to the circular globe interior |
| 3 | Idle state maintains 12–18 slowly drifting particles at all times |
| 4 | Click/tap triggers an omnidirectional burst of ~40 particles |
| 5 | Rapid taps (within 500ms) amplify intensity up to 3× cap |
| 6 | Drag-and-release produces a directional snow burst matching drag vector |
| 7 | Drag left → snow swirls left; drag right → snow swirls right |
| 8 | Snow particles slow under gravity and air resistance, settling at the globe bottom |
| 9 | Settled particles fade out over ~3 seconds and are recycled |
| 10 | Particle count never exceeds 120 |
| 11 | Globe wrapper applies a subtle CSS shake animation on tap/release |
| 12 | Drag provides tactile CSS translate feedback (max ±8px), returning with ease-out |
| 13 | Widget renders correctly on desktop Chrome, Safari, Firefox, and mobile iOS/Android |
| 14 | Widget scales down cleanly on viewports below 520px |
| 15 | All physics constants are declared at the top of the script for easy tuning |
| 16 | Single `index.html` file — zero external dependencies |
| 17 | "Enable shake" button is visible only on touch-capable mobile devices |
| 18 | On Android, tapping "Enable shake" immediately activates motion listener |
| 19 | On iOS, tapping "Enable shake" triggers native `DeviceMotionEvent.requestPermission()` dialog |
| 20 | If permission granted: button hides, "Shake me!" hint appears then fades, motion activates |
| 21 | If permission denied: button hides silently, drag/tap interaction remains fully functional |
| 22 | Physical shake triggers a directional snow burst matching shake vector |
| 23 | Walking vibration and ambient movement do not trigger bursts (threshold enforced) |
| 24 | Motion bursts are rate-limited to one per 350ms regardless of shake frequency |
| 25 | Landscape orientation correctly remaps X/Y axes for accurate directional response |

---

## 11. Out of Scope (v1.0)

- Audio / sound effects
- Snowflake shapes other than soft circles
- WebGL or particle library dependencies
- Accumulation / snow piling physics
- Export or screenshot functionality
- Colour themes or configurable UI controls

---

*End of PRD — v1.0*
