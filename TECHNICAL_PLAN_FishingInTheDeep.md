# Fishing In The Deep — Technical Implementation Plan
**Format:** Single HTML file · **Renderer:** Three.js (inline CDN) · **Target:** Playable Ad (PA)

---

## 1. Core Systems & Architecture

### Single-File Structure
```
index.html
 ├── <head>  — CDN imports: Three.js r160, GSAP 3, Howler.js
 ├── <style> — All CSS (UI layers, HUD, overlays, animations)
 └── <body>
      ├── #canvas-container   — Three.js WebGL canvas (3D world)
      ├── #ui-layer           — All HTML/CSS HUD overlays (pointer-events: none on canvas)
      └── <script>            — All JS: engine, systems, state machine
```

### State Machine (Central Controller)
Single `GameState` object drives everything. States are mutually exclusive; transitions call `onExit` / `onEnter` hooks.

```
IDLE → CASTING → DESCENDING → HOLDING → ASCENDING → SURFACING → SUMMARY → (ROUND_RESET | ENDSCREEN)
```

| State | Description |
|---|---|
| `IDLE` | Upgrade shop + gauge pendulum active. Play button visible. |
| `CASTING` | Player pressed Play. Gauge locked. Depth calculated. Hook begins descent. HUD swaps coin→fish counter. |
| `DESCENDING` | Camera follows hook down. Fish entities pre-spawned at random positions. |
| `HOLDING` | Hook paused at target depth. Round 1 only: tutorial tooltip shown for 2 s, then fades. |
| `ASCENDING` | Player drags hook L/R. Collision detection active. Fish caught → hooked + animated. |
| `SURFACING` | Max fish reached or hook crosses y=0. Hook + catch rises quickly. HUD swaps fish→coin counter. |
| `SUMMARY` | Fish jump animation → value pop-ups → coin burst per fish → total counter → coin fly to HUD. |
| `ROUND_RESET` | Reset hook/camera, re-show Play button. Round counter (hidden from player) incremented. |
| `ENDSCREEN` | After round 3 summary: full-screen CTA overlay. |

### Key Modules
- **GaugeSystem** — pendulum animation, depth calculation, UI rendering
- **CameraSystem** — follow-hook descent/ascent, lerp-based smooth tracking
- **FishSystem** — spawn, collision, catch, rarity, animation state
- **UpgradeSystem** — state, cost table, button enable/disable logic
- **HUDSystem** — counter swap logic, fish fill-bar, coin display
- **VFXSystem** — particles, DOM pop-ups, coin burst, fish jump at surface
- **AudioSystem** — Howler.js wrapper: play, stop, volume per event

---

## 2. Gameplay Flow

### Round Loop (×3)

```
[IDLE]
  Pendulum swings on gauge ←→
  Player optionally buys upgrades (coins permitting)
  Player clicks PLAY
      ↓
[CASTING]
  Gauge pointer freezes → depth = lerp(minDepth, maxDepth, gaugeNormalized)
  HUD: coin counter fades out → fish counter fades in (starts 0/N, fill bar empty)
  Hook starts descending
      ↓
[DESCENDING]
  Camera follows hook (smooth lerp, slight lag = juicy feel)
  Fish pre-spawned at randomized (x, depth) — rarer fish deeper
  Hook reaches targetDepth
      ↓
[HOLDING]
  Hook idles (bob animation)
  Round 1 only: tutorial text "Drag the hook to catch fish!" — 2 s, fade out
  0.5–1 s pause (2 s on round 1), then transition
      ↓
[ASCENDING]
  Hook moves up at constant speed
  Player drags mouse/touch left/right → hook follows on X axis (clamped)
  Collision check: hook hitbox vs fish hitbox each frame
  On catch:
    Fish snaps to hook (parent transform)
    Catch VFX plays at fish position
    "+$VALUE" coin pop-up bounces in
    Rarity title pops if applicable
    Fish counter increments, fill bar updates
  If fishCount == maxFish → hook.ascendFast = true (no more collision)
      ↓
[SURFACING]
  Hook crosses y = 0 (surface line)
  HUD: fish counter fades out → coin counter fades in
  Camera smoothly returns to surface framing
      ↓
[SUMMARY]
  Caught fish detach from hook, line up on surface, jump animation (staggered, GSAP)
  Per fish (staggered 0.3 s delay each):
    Value text pops in with bounce
    Rarity label if applicable
    Coin burst particle effect
  After all fish resolved:
    Total earnings shown ("+ $XXX") with count-up tween
    Coins fly from total → HUD coin counter (arc trajectory, CSS/GSAP)
    Coin counter ticks up
      ↓
[ROUND_RESET or ENDSCREEN]
  If round < 3 → reset scene, show Play button (IDLE)
  If round == 3 → ENDSCREEN
```

### Gauge Mechanics (IDLE state)
- **Shape:** Semi-circle arc rendered in HTML Canvas 2D (overlaid on the Play button area).
- **Color zones:** Left/right ends = red ("shallow"), center = green ("deep"). Gradient sweep: `red → yellow → green → yellow → red`.
- **Pointer:** A triangular SVG indicator mounted on a rotating arm from the arc center. Rotates from `−70°` to `+70°` (arc sweep).
- **Pendulum motion:** GSAP `gsap.to()` with `ease: "sine.inOut"`, alternating between `−70°` and `+70°`. Base duration `1.4 s` per side. Speed increases very slightly each round (subtle pressure).
- **Depth mapping:** `normalizedPos = (angle + 70) / 140`. `depth = minDepth + normalizedPos × (maxDepth − minDepth)`. Center (green) = maxDepth. Edges (red) = minDepth.
- **On click:** GSAP `.kill()` on tween → pointer frozen → depth locked → state transitions to CASTING.

---

## 3. Entities & Data

### Game Config (all tunable constants)
```js
const CONFIG = {
  rounds: 3,
  startCoins: 100,
  gauge: { minAngle: -70, maxAngle: 70, swingDuration: 1.4 },
  depth: { startMax: 5, minOffset: 2 },  // minDepth = maxDepth - 2
  fish: { startMax: 6 },
  upgrades: {
    maxFish: [
      { value: 7,  cost: 50  },
      { value: 8,  cost: 100 },
      { value: 9,  cost: 200 },
      { value: 10, cost: 400 },
    ],
    maxDepth: [
      { value: 10, cost: 50  },
      { value: 15, cost: 75  },
      { value: 20, cost: 150 },
      { value: 30, cost: 300 },
    ],
  },
};
```

### Fish Rarity Table
| Rarity | Depth Range | Spawn Weight | Coin Value | Label | VFX Tier |
|---|---|---|---|---|---|
| Common | Any | 50% | 5–15 | — | Tier 1 |
| Uncommon | ≥ 40% depth | 28% | 20–40 | — | Tier 1 |
| Rare | ≥ 60% depth | 14% | 50–80 | "Rare!" | Tier 2 |
| Epic | ≥ 80% depth | 6% | 100–150 | "Amazing!" | Tier 2 |
| Legendary | ≥ 90% depth | 2% | 200–350 | "Legendary!" | Tier 3 |

Rarity weights re-rolled per fish. Fish spawned with `depth × rarityBias` so going deeper materially improves the catch pool.

### Fish Entity
```js
{
  mesh: THREE.Mesh,        // Basic 3D shape (see §5)
  rarity: 'common'|...,
  coinValue: Number,
  depth: Number,           // Y position in world units
  xPos: Number,            // Random X within swim lane
  swimPhase: Number,       // For idle bobbing animation
  caught: Boolean,
  hooked: Boolean,
}
```

### Hook Entity
```js
{
  mesh: THREE.Group,       // Line + hook shape
  targetX: Number,         // Updated by player drag
  depth: Number,           // Current Y world position
  speed: Number,           // Descent/ascent speed
  ascendFast: Boolean,
  caughtFish: Fish[],
}
```

### Player State
```js
{
  coins: Number,
  round: Number,           // 1–3, hidden from player
  maxFish: Number,
  maxDepth: Number,
  fishUpgradeLevel: Number,  // 0–4
  depthUpgradeLevel: Number, // 0–4
}
```

---

## 4. VFX, SFX & Particles

### VFX Tiers
| Tier | Used For | Elements |
|---|---|---|
| **Tier 1** | Common/Uncommon catch | Small CSS particle burst (8 dots, 0.4 s), small "+$N" pop-up, quick scale punch on fish |
| **Tier 2** | Rare/Epic catch | Larger particle burst (16 dots), ripple ring SVG expand, larger bouncy "+$N" with glow, label badge drops in from top |
| **Tier 3** | Legendary catch | Full screen edge flash (brief white vignette), massive particle burst (32 dots), screen shake (GSAP `x` jitter on canvas container), huge glowing "+$N", "LEGENDARY!" badge with shimmer animation |

### Particle System (CSS-based, no canvas)
- DOM `<div>` particles created on demand, appended to `#vfx-layer`, auto-removed after animation.
- Each particle: random angle, random distance, random size variant.
- CSS `@keyframes`: `transform: translate + scale → 0`, `opacity: 1 → 0`.
- Color palette per rarity: Common=teal, Uncommon=blue, Rare=purple, Epic=orange, Legendary=gold+white.

### Coin Burst (Summary Phase)
- Per fish: spawn 8–16 coin divs (`🪙` emoji or gold circle div) at fish position.
- GSAP arc tween: each coin moves to HUD coin counter position over 0.8–1.2 s.
- Stagger across fish catches by 0.3 s.
- HUD coin counter does a "punch" scale animation each time coins arrive.

### Surface Jump Animation
- GSAP timeline per caught fish: `y: -80px → 0 → -40px → 0` with `bounce` ease.
- Fish meshes converted to CSS-positioned overlays for this phase (simpler to animate than Three.js).

### Screen Shake (Legendary only)
```js
gsap.to('#canvas-container', { x: '+=6', yoyo: true, repeat: 7, duration: 0.05, ease: 'none' });
```

### Audio (Howler.js)
All files provided externally. Keys and trigger points:

| Audio Key | File | Trigger |
|---|---|---|
| `splash_cast` | splash_cast.mp3 | Hook enters water (CASTING start) |
| `hook_descend` | hook_descend.mp3 | Loop during DESCENDING |
| `hook_ascend` | hook_ascend.mp3 | Loop during ASCENDING |
| `fish_catch_common` | catch_common.mp3 | Common/Uncommon fish caught |
| `fish_catch_rare` | catch_rare.mp3 | Rare/Epic fish caught |
| `fish_catch_legendary` | catch_legendary.mp3 | Legendary fish caught |
| `surface_splash` | surface.mp3 | Hook breaks surface |
| `coin_pop` | coin_pop.mp3 | Each fish value pops in summary |
| `coin_fly` | coin_fly.mp3 | Coins fly to HUD counter |
| `upgrade_buy` | upgrade_buy.mp3 | Upgrade purchased |
| `gauge_tick` | gauge_tick.mp3 | Play button clicked / gauge locked |
| `bg_music` | bg_music.mp3 | Looping ambient, plays throughout |

---

## 5. 3D Visual Approach (Three.js, No Real Assets)

### Scene Setup
- **Renderer:** `THREE.WebGLRenderer({ antialias: true, alpha: true })`, fills container.
- **Camera:** `THREE.PerspectiveCamera(60°)`, positioned at `(0, 2, 10)`, looking toward `(0, -2, 0)`.
- **Lighting:** 1× `AmbientLight` (soft warm, 0.6 intensity) + 1× `DirectionalLight` (top-right, cast shadows off for perf).
- **World orientation:** Y-up. Surface at `y = 0`. Ocean floor at `y = -maxDepth`.

### Environment
- **Sky/surface:** A wide flat `PlaneGeometry` at `y = 0.1`, semi-transparent `MeshBasicMaterial` with blue tint — represents the water surface seen from above.
- **Underwater background:** Gradient `CanvasTexture` applied to a back-plane (`PlaneGeometry` behind everything): dark navy at top → near-black at bottom. Parallax-scrolls slightly as camera descends.
- **Depth fog:** `scene.fog = new THREE.FogExp2(0x001133, 0.04)` — naturally hides far depth, adds atmosphere.
- **Floating particles:** 40× tiny `SphereGeometry(0.05)` white dots drifting upward slowly (bubbles). `userData.speed` randomized.

### Fisherman (Surface, Static)
- Body: `BoxGeometry(0.6, 1.2, 0.4)` — muted brown.
- Head: `SphereGeometry(0.3)` — skin tone.
- Hat: `CylinderGeometry(0.15, 0.35, 0.3)` stacked on head.
- Arm: `BoxGeometry(0.15, 0.7, 0.15)` angled forward holding rod.
- Rod: `CylinderGeometry(0.03, 0.03, 2.5)` thin, angled up.
- Fishing line: `THREE.Line` with `LineBasicMaterial`, stretches dynamically from rod tip to hook position each frame.

### Hook
- Pole end: `SphereGeometry(0.08)` (junction point).
- Hook body: `TorusGeometry(0.15, 0.03, 8, 16, Math.PI)` — half-torus, rotated to hang downward in classic hook shape.
- All grouped: `THREE.Group`. X position updated by player input; Y updated by descent/ascent.

### Fish Shapes (vary by rarity for quick visual read)
| Rarity | Shape | Color |
|---|---|---|
| Common | `SphereGeometry(0.3)` + small tail `ConeGeometry` | Grey-blue |
| Uncommon | `CapsuleGeometry(0.25, 0.4)` + tail | Teal |
| Rare | Elongated `SphereGeometry` scaled `(1.6, 1, 1)` + fin box | Vivid purple |
| Epic | Same as Rare + `emissive` orange glow | Orange-gold |
| Legendary | Same + large `PointLight` child (glow aura), animated scale pulse | Bright gold, shimmering |

Fish idle animation: `mesh.position.x += Math.sin(time + swimPhase) * 0.01` each frame — gentle swimming sway.

---

## 6. Step-by-Step Implementation Plan

### Step 1 — HTML Skeleton & Layers
- Create `index.html` with `<canvas>` fill + `#ui-layer` div (absolute, pointer-events passthrough on canvas).
- Import Three.js, GSAP, Howler via CDN `<script>` tags in `<head>`.
- Define all CSS variables: colors, fonts (use a display font like `Bangers` for labels, `Nunito` for UI via Google Fonts).
- Build static DOM structure: `#hud`, `#gauge-container`, `#play-btn`, `#upgrade-panel`, `#fish-counter`, `#coin-counter`, `#vfx-layer`, `#endscreen`.

### Step 2 — Three.js Scene Foundation
- Init renderer, scene, camera, lights, fog.
- Build and position fisherman group at surface.
- Build underwater background plane + bubble particles.
- Add `window.addEventListener('resize', onResize)` — renderer and camera aspect update.
- Implement `animate()` loop: `requestAnimationFrame` + bubble drift + fish idle sway.

### Step 3 — Gauge System
- Draw semi-circle arc on `<canvas id="gauge-canvas">` using `ctx.arc()`.
- Fill arc with `createLinearGradient`: red → yellow → green → yellow → red.
- Overlay SVG pointer (`<div id="gauge-pointer">`) absolutely positioned, `transform-origin: bottom center`.
- GSAP pendulum: `gsap.to('#gauge-pointer', { rotation: 70, duration: 1.4, ease:'sine.inOut', yoyo:true, repeat:-1 })`.
- `#play-btn` click handler: kill tween, read rotation, calculate `normalizedPos`, store `targetDepth`, call `startCasting()`.

### Step 4 — State Machine & Game Loop
- Implement `setState(newState)`: sets `currentState`, calls `states[newState].enter()`.
- Implement each state's `enter()`, `update(delta)`, `exit()` as plain objects.
- Wire `animate()` loop to call `states[currentState].update(delta)` each frame.

### Step 5 — Descending & Camera System
- In `DESCENDING.update`: `hook.position.y -= speed * delta`. Camera `y` lerps toward `hook.position.y + offset`. Line geometry updated (`setFromPoints`).
- Spawn fish entities: for each of `maxFish` slots, roll rarity (depth-weighted), set random `x ∈ [-3, 3]` and `y ∈ [-minDepth, -targetDepth]` distributed evenly with jitter.

### Step 6 — Ascending & Input
- Mouse/touch `mousemove` / `touchmove` → map X to world X (via `raycaster` or simpler: `(clientX / window.innerWidth - 0.5) * 6`) → `hook.targetX`.
- In `ASCENDING.update`: `hook.position.x = lerp(hook.position.x, hook.targetX, 0.15)`. `hook.position.y += ascentSpeed * delta`.
- Collision: per un-caught fish, `distanceSq(hook, fish) < threshold²` → `catchFish(fish)`.
- `catchFish`: mark caught, parent fish to hook group, play catch SFX + VFX tier, update HUD.
- If `caughtFish.length >= maxFish`: set `ascendFast = true`, boost speed, skip collision checks.

### Step 7 — HUD Systems
- **Coin counter:** CSS `position:fixed top-left`, animated number using `gsap.to({val:current}, {val:target, onUpdate})`.
- **Fish counter:** Replace coin counter div (crossfade via GSAP `opacity` tween). Fill bar: `width: (caught/maxFish * 100)%`, CSS transition.
- **Counter swap:** Fade-out coin → fade-in fish on CASTING enter. Reverse on SURFACING enter.

### Step 8 — Upgrade Panel
- Two buttons per upgrade (Fish, Depth), each showing current level, next cost.
- On click: check `coins >= cost`, deduct, level up, update CONFIG live values, play `upgrade_buy` SFX, disable button if max level reached.
- Disable all upgrade buttons during non-IDLE states.

### Step 9 — Summary & Coin Burst
- SURFACING enter: camera lerps back to surface. Hook group stops.
- SUMMARY enter: detach fish from hook, position them in a row above surface.
- GSAP staggered timeline per fish: jump animation → value badge scales in → coin burst divs created → arc-fly to HUD counter via GSAP `motionPath` or manual bezier.
- After all resolved: show total with count-up → grand coin fly → call `endRound()`.

### Step 10 — Round Reset & End Screen
- `endRound()`: if `round < 3` → reset hook/fish/scene → `setState('IDLE')`. If `round == 3` → `setState('ENDSCREEN')`.
- `ENDSCREEN`: full-screen overlay fades in. Shows only a large CTA button `"PLAY NOW"`. `onclick`: `window.open('', '_blank')`.
- Hide `#gauge-container`, `#upgrade-panel`, `#play-btn` during end screen.

### Step 11 — Polish Pass
- Tutorial tooltip (round 1 HOLDING): absolutely-positioned div, fade in → 2 s hold → fade out, CSS styled as underwater bubble caption.
- Rarity labels on catch: DOM badge drops in from above fish, CSS `@keyframes dropIn`, auto-remove after 1.5 s.
- Legendary screen shake: GSAP jitter on canvas container.
- Ensure all audio loops (`hook_descend`, `hook_ascend`, `bg_music`) stop correctly on state exit.
- Test full 3-round loop end-to-end. Tune GSAP durations for feel.
