# Fishing In The Deep — Technical Implementation Plan
**Target:** Single `.html` file · No build tools · No external dependencies  
**Renderer:** Three.js (CDN) for 3D · Vanilla JS for game logic · CSS for UI overlays  
**Author note:** This doc is written for a senior developer doing a single-pass implementation.

---

## 1. Core Systems & Architecture

### 1.1 Module Structure (all in one file, clearly sectioned)

```
<html>
  <head>  CSS variables, font, keyframes  </head>
  <body>
    <!-- DOM layers (z-indexed) -->
    <canvas id="three-canvas"/>      <!-- Three.js scene, always present -->
    <canvas id="vfx-canvas"/>        <!-- 2D particle overlay, pointer-events:none -->
    <div id="ui-root">               <!-- All HUD + screens as DOM overlays -->
      #screen-shop                   <!-- Idle/upgrade phase -->
      #screen-hud                    <!-- In-round HUD (counter, gauge) -->
      #screen-surface                <!-- Per-fish summary animation -->
      #screen-end                    <!-- Final CTA -->
    </div>
  </body>
  <script>
    // ── SECTION 1: Constants & Config
    // ── SECTION 2: GameState + FSM
    // ── SECTION 3: AudioManager
    // ── SECTION 4: SceneManager (Three.js)
    // ── SECTION 5: GaugeController
    // ── SECTION 6: HookController
    // ── SECTION 7: FishSpawner
    // ── SECTION 8: VFX (particles, popups, coins, screen shake)
    // ── SECTION 9: UIManager (screen transitions, counters)
    // ── SECTION 10: GameLoop (rAF master loop)
    // ── SECTION 11: Boot
  </script>
</html>
```

### 1.2 State Machine

The game runs through a strict linear FSM. Each state owns its input handlers; all others are blocked via a guard at the top of every event listener.

```
SHOP → CASTING → DESCENDING → PAUSED_AT_DEPTH → CATCHING → ASCENDING_FAST
     → SURFACE_SUMMARY → (round < 3: back to SHOP | round == 3: END)
```

`GameState` is a single plain object (no classes, no reactivity framework):

```js
const GameState = {
  fsm: 'SHOP',
  round: 0,               // 0-indexed, never displayed
  coins: 100,
  maxFish: 6,
  maxDepth: 5,
  maxFishLevel: 0,        // index into CFG.MAX_FISH_UPGRADES
  maxDepthLevel: 0,       // index into CFG.MAX_DEPTH_UPGRADES
  gaugeAngle: 0,          // live radians, updated by GaugeController
  selectedDepth: 0,       // set on throw; derived from gaugeAngle
  caughtFish: [],         // FishData[] for current throw
  totalRoundCoins: 0,
};
```

`setState(name)` sets `GameState.fsm`, calls `onExit[prev]?.()` then `onEnter[name]()`. All transitions go through this single function — never set `fsm` directly elsewhere.

### 1.3 Constants & Config

```js
const CFG = {
  ROUNDS: 3,
  BASE_COINS: 100,
  BASE_MAX_FISH: 6,
  BASE_MAX_DEPTH: 5,
  MIN_DEPTH_OFFSET: 2,          // minDepth = maxDepth − 2

  HOOK_DESCEND_SPEED: 3,        // m/s
  HOOK_ASCEND_SPEED: 2,         // m/s  
  HOOK_FAST_ASCEND_SPEED: 6,    // m/s (after max fish reached)
  HOOK_DRAG_LERP: 0.18,         // horizontal follow smoothness (per frame, 60fps)
  WORLD_UNITS_PER_METER: 0.5,   // scale: 1 meter = 0.5 Three.js units

  PAUSE_AT_DEPTH_DEFAULT: 0.75, // seconds
  PAUSE_AT_DEPTH_ROUND1: 2.0,   // seconds (includes tutorial)
  TUTORIAL_DISPLAY: 1.5,        // seconds text is visible during round 1 pause

  GAUGE_PERIOD: 2.2,            // seconds for one full left→right→left sweep
  GAUGE_EASE_K: 3.0,            // power curve exponent for pendulum ease

  FISH_SPAWN_CHANCE: 0.4,       // probability per 0.5m depth step
  FISH_H_RANGE: 3.5,            // ±world units for horizontal spawn range
  FISH_POOL_MULTIPLIER: 2,      // total fish spawned = maxFish × this (ensures catchable supply)

  MAX_FISH_UPGRADES: [
    { value: 6,  cost: 0   },
    { value: 7,  cost: 50  },
    { value: 8,  cost: 100 },
    { value: 9,  cost: 200 },
    { value: 10, cost: 400 },
  ],
  MAX_DEPTH_UPGRADES: [
    { value: 5,  cost: 0   },
    { value: 10, cost: 50  },
    { value: 15, cost: 75  },
    { value: 20, cost: 150 },
    { value: 30, cost: 300 },
  ],

  // Rarity tiers — weight is relative (not percent); filtered by minDepth at spawn time
  FISH_TIERS: [
    { id:'common',    minDepth:0,  weight:60, valueRange:[5,15],   color:'#5bc8f5', meshScale:1.0, label:null,         shake:0,  flashAlpha:0   },
    { id:'uncommon',  minDepth:3,  weight:25, valueRange:[20,40],  color:'#78e08f', meshScale:1.1, label:null,         shake:0,  flashAlpha:0   },
    { id:'rare',      minDepth:8,  weight:10, valueRange:[50,90],  color:'#f9ca24', meshScale:1.25,label:'Rare!',      shake:2,  flashAlpha:0   },
    { id:'epic',      minDepth:15, weight:4,  valueRange:[100,180],color:'#e056fd', meshScale:1.5, label:'Amazing!',   shake:5,  flashAlpha:0   },
    { id:'legendary', minDepth:22, weight:1,  valueRange:[200,400],color:'#ff4757', meshScale:1.9, label:'Legendary!', shake:10, flashAlpha:0.6 },
  ],
};
```

---

## 2. Gameplay Flow

### Phase A — SHOP (Idle)

The screen is divided into three vertical zones using CSS Grid (`grid-template-rows: auto 1fr auto`):
- **Top:** Coin counter (always visible, top-center).
- **Middle:** Play button with gauge SVG mounted directly above it, centered.
- **Bottom:** Two upgrade button groups, `padding-bottom: max(16px, env(safe-area-inset-bottom))`.

**Layout insurance:** Upgrade buttons use `display:flex; flex-wrap:wrap; justify-content:center; gap:clamp(12px,3vw,28px)`. Each button: `width:clamp(130px,42vw,180px); min-height:64px`. This guarantees no overlap from 320px width upward on any device. Play button has `margin-bottom: 16px` above the upgrade row.

Upgrade buttons show: current value, arrow, next value, cost. If unaffordable: `opacity:0.45; pointer-events:none`. If at max tier: display "MAX", same disabled style.

Gauge pendulum is **always running** while in SHOP state. The player can press Play at any moment — there is no separate "aim" state.

### Phase B — CASTING (press Play)

On button press: `gaugeAngle` snapshot → `selectedDepth` computed → immediately enter DESCENDING. No intermediate state needed. Play `rob_use.mp3` (whoosh variant, pitch 1.2). Fish counter fades in replacing coin counter.

### Phase C — DESCENDING

Hook moves from y=0 downward at `HOOK_DESCEND_SPEED`. Camera Y follows hook with a lag of 0.08 lerp factor per frame — prevents jarring snap. Fish are pre-spawned at scene load per round; they become visible (`mesh.visible = true`) only when `camera.position.y` passes their world Y by at least 1 unit (lazy reveal prevents pop-in at surface).

### Phase D — PAUSED_AT_DEPTH

Hook stops. Timer counts down.  
**Round 1 only:** DOM tutorial label `"Move the hook to catch fish"` fades in at t=0, auto-fades out at t=`TUTORIAL_DISPLAY`. Remaining pause time completes. Label is a centered, semi-transparent pill — not a full modal.  
On timer end → CATCHING.

### Phase E — CATCHING

Hook ascends at `HOOK_ASCEND_SPEED`. Full drag input active.

**Input handling (critical):**
- `pointerdown` / `touchstart` anywhere on `#three-canvas` or `#ui-root` (both listeners): immediately compute `worldX` from `clientX` and **teleport** `hookGroup.position.x = worldX` (no lerp on first contact — instant).
- `pointermove` / `touchmove`: update `input.targetX = worldX`. Each frame: `hookGroup.position.x = lerp(hook.x, input.targetX, HOOK_DRAG_LERP)`.
- `pointerup` / `touchend` / `touchcancel`: `input.targetX = 0` (hook drifts back to center naturally).
- Screen-to-world X: `worldX = ((clientX / window.innerWidth) − 0.5) * CFG.FISH_H_RANGE * 2`.
- Guard: input only processed when `fsm === 'CATCHING'`.

**Collision:** Per frame, for each uncaught fish: `Math.abs(hook.x − fish.worldX) < 0.6 + bonus && Math.abs(hook.y − fish.worldY) < 0.5`. Touch bonus: radius ×1.5 when `isTouchDevice`. On hit → `catchFish(fish)`.

When `caughtFish.length >= maxFish` → ASCENDING_FAST (input disabled immediately).

### Phase F — ASCENDING_FAST

Hook speed → `HOOK_FAST_ASCEND_SPEED`. When hook reaches y ≥ −0.1 (surface) → SURFACE_SUMMARY. Fish counter crossfades back to coin counter during final 0.5m of ascent.

### Phase G — SURFACE_SUMMARY

Camera lerps back to y=0. `playSummarySequence()` runs as a setTimeout chain:

1. **Per caught fish (staggered 350ms apart):** DOM "fish card" element (not 3D — positioned above hook via screen projection) bounces with CSS keyframe (`scale: 1→1.6→0.9→1.2→1.0`). Coin value pops above it. Rarity label pops above that (if any). Coin burst spawns. Sound plays.
2. **Total roll:** After last fish + 200ms delay, total coin value counter rolls from 0 to sum over 600ms using `setInterval`.
3. **Coin fly:** Spawn animated `🪙` DOM elements at total-label `getBoundingClientRect()` center; CSS `@keyframes` arc them to coin counter `getBoundingClientRect()` center. Snapshot both rects *before* sequence starts to avoid reflow mid-animation.
4. **Coin counter update:** Flash + scale pop on coin counter when coins land.
5. **Reset:** 800ms pause → if `round < 2`: reset for next throw (state → SHOP, fish counter hidden, Play button re-appears). If `round === 2`: → END.

### Phase H — END

Dark overlay (`rgba(0,0,0,0.88)`) fades in over 600ms. Contains only:
- A large CTA button, centered, bold, bright color.
- `onclick: window.open('about:blank', '_blank')`.

Background music fades out over 1000ms via `GainNode.gain.linearRampToValueAtTime`.

---

## 3. Entities & Data

### FishData

```js
{
  tier: CFG.FISH_TIERS[i],  // tier config reference
  value: 37,                 // randomized in [tier.valueRange[0], tier.valueRange[1]]
  spawnDepth: 8.4,           // meters below surface
  worldX: 1.2,               // spawn X (fixed; fish don't move on X toward player)
  worldY: -4.2,              // Three.js Y (= −spawnDepth * WORLD_UNITS_PER_METER)
  mesh: THREE.Group,         // fish body + tail group
  caught: false,
}
```

### AudioManager

```js
AudioManager = {
  ctx: AudioContext,
  masterGain: GainNode,
  buffers: { bgMusic, catch, robUse },  // loaded at boot
  bgSource: BufferSourceNode,           // kept for fade-out reference
  play(bufferKey, { volume=1, pitch=1, loop=false }),
  fadeOut(node, duration),
}
```

### SceneObjects (Three.js scene graph)

```
scene
├── ambientLight
├── directionalLight
├── waterSurface (PlaneGeometry, animated verts)
├── seabed (PlaneGeometry, static)
├── fisherman (Group: box+sphere+cylinders)
├── fishingLine (Line, BufferGeometry, 2 points updated per frame)
├── hookGroup (Group)
│   ├── hookMesh (TorusGeometry)
│   └── [caught fish Groups, added dynamically]
├── fishPool[] (Group[], pre-spawned per round, positioned at world Y)
└── bubbles (Points, 200 particles, ambient drift)
```

---

## 4. VFX, SFX & Particles

### Audio

All audio via Web Audio API. Load all 3 files at boot with `Promise.all([fetch+decode, ...])`. AudioContext must be resumed inside the first user gesture handler (iOS requirement — add resume call to every button's click handler as a safeguard).

| File | When | Volume | Pitch override |
|---|---|---|---|
| `background_music.mp3` | Loop from boot | 0.25 | — |
| `catch.mp3` | Per fish catch | 0.8 | Common 1.0 / Uncommon 1.1 / Rare 1.3 / Epic 1.5 / Legendary 1.8 |
| `rob_use.mp3` | Upgrade purchase | 0.5, pitch 1.0 | Also: hook launch, volume 0.7, pitch 1.2 |

Each play creates a fresh `BufferSourceNode` — never reuse nodes. Connect: `source → gainNode → masterGain → destination`.

### Rarity Juice Escalation

| Tier | Screen Shake | Popup Font | Popup Color | Extra FX |
|---|---|---|---|---|
| Common | none | 14px | #ffffff | 6 coin particles |
| Uncommon | none | 16px, bold | #78e08f | 10 coin particles, scale pop |
| Rare | 2px, 80ms | 20px, bold | #f9ca24 | 18 coins, glow ring on label |
| Epic | 5px, 150ms | 26px, bold | #e056fd | 30 coins + 8 star sprites on vfx-canvas |
| Legendary | 10px, 250ms | 36px, bold + outline | #ff4757 | 50 coins + 16 stars + full-screen white flash |

**Screen Shake:** On catch, apply `transform: translate(dx, dy)` to `document.body` (or `#ui-root`). `dx/dy` = random unit vector × `tier.shake`px. Duration: linear decay over `tier.shake * 25` ms, updated via rAF.

**Coin Burst Particles:** Pool of pre-created `<div>` elements styled as `🪙`, parked off-screen. On activation: set `position:fixed`, compute screen coords from fish world pos via `vector.project(camera)` → CSS `left/top`. Animate with CSS custom properties for velocity and arc. Return to pool on `animationend`.

**Star Burst (Epic/Legendary):** Draw on `#vfx-canvas` (2D context). Each star: line-based 5-point star, random angle/speed outward from catch point, fades over 400ms. Cleared per frame during active bursts.

**Legendary Flash:** `<div id="flash-overlay">` with `position:fixed; inset:0; background:white; pointer-events:none`. Set `opacity:0.6`, then CSS transition `opacity 80ms → 0` immediately after.

### Underwater Ambience

- 200 `THREE.Points` sprites (white, size 0.05), randomly placed in a box covering the full depth column and `±FISH_H_RANGE` on X.
- Each frame: `bubble.y += 0.02 * dt`; if `y > 0` wrap to `y = -maxDepth * WORLD_UNITS_PER_METER`.
- `scene.fog = new THREE.FogExp2(0x0a3d62, 0.035)`. Fog density lerps from 0.02 (surface) to 0.07 (max depth) as hook descends, giving a darkening underwater feel.

---

## 5. 3D Visual Approach

**Philosophy:** All geometry is Three.js primitives. `MeshLambertMaterial` throughout (flat-ish shading, no PBR cost). Color and scale carry all visual weight. Target aesthetic: charming toy-like lo-fi 3D.

**Scene setup:** `PerspectiveCamera(60, aspect, 0.1, 200)`. Renderer `alpha:true` (transparent background). Body CSS: `background: linear-gradient(180deg, #0a3d62 0%, #0c2461 100%)`. Camera X fixed at 0, Z fixed at 6. Camera Y: `lerp(camera.y, hook.y + 1.5, 0.08)` per frame.

**Fisherman:** `BoxGeometry(0.4,0.6,0.3)` torso, `SphereGeometry(0.2,8,8)` head, two `CylinderGeometry(0.05,0.05,0.5)` arms. Warm tan color (`#c8a882`). Positioned at `x=−0.8, y=0.4, z=0` leaning over the edge. Static — no animation.

**Hook:** `TorusGeometry(0.15, 0.04, 8, 16, Math.PI)` (open half-circle), gray metallic color (`#aab4bd`). Fishing line: `THREE.Line` with a `BufferGeometry` of 2 vertices — updated each frame from fisherman's right-hand position to hook position.

**Fish (per tier):**
- Body: `SphereGeometry(0.3, 8, 6)` with `scale.z = 0.45` (flattened into fish shape).
- Tail: `ConeGeometry(0.18, 0.3, 6)` attached behind body, slightly angled.
- Material: `MeshLambertMaterial({ color: tier.color })`.
- Group scale: `tier.meshScale` (Legendary fish are 1.9× the size of Common — visually unmissable).
- While swimming (pre-catch): `mesh.position.x += Math.sin(time * 2 + fish.worldX) * 0.02` per frame (gentle sway).
- When caught: attach to `hookGroup`, zero out sway, bob gently (`position.y += Math.sin(time * 3) * 0.01`). Fish stack below hook, spaced 0.35 units apart vertically.

**Water Surface:** `PlaneGeometry(20, 4, 16, 1)` rotated flat (`rotation.x = -Math.PI/2`) at y=0. `MeshLambertMaterial({ color: 0x1e90ff, transparent: true, opacity: 0.55 })`. Each frame: loop through vertex Y positions in geometry attributes, apply `Math.sin(time * 1.5 + x * 0.8) * 0.04` displacement. Mark `needsUpdate = true`.

**Seabed:** Dark brown `PlaneGeometry(20, 4)` at `y = -CFG.MAX_DEPTH_UPGRADES[4].value * WORLD_UNITS_PER_METER`. Static.

---

## 6. Gauge — Detailed Spec

### Shape

Inline SVG, `width="240" height="130" viewBox="0 0 240 130"`. Mounted directly above the Play button in the DOM flow.

- **Arc:** SVG `<path>` describing a 180° arc from `(10,120)` to `(230,120)`, sweeping upward, radius 110. Stroke width 12px, `stroke-linecap="round"`.
- **Gradient fill on arc stroke:** `<linearGradient id="gaugeGrad" x1="0%" y1="0%" x2="100%" y2="0%">` with stops: `#e74c3c` (0%) → `#f39c12` (25%) → `#2ecc71` (50%) → `#f39c12` (75%) → `#e74c3c` (100%)`. The green center = max depth; red edges = min depth.
- **Tick marks (optional, non-PRD but trivial):** 5 small lines at 0%, 25%, 50%, 75%, 100% positions along the arc. Adds visual polish at zero cost.
- **Needle:** SVG `<polygon points="120,115 115,30 125,30">` (a thin triangle pointing up). `transform-origin: 120px 115px`. Color: white with drop shadow (`filter: drop-shadow(0 2px 4px rgba(0,0,0,0.5))`).
- **Center dot:** Small circle at `(120,115)` in white to anchor needle visually.

### Pendulum Motion

```js
// In GaugeController.update(dt), only runs when fsm === 'SHOP'
GaugeController.time += dt;
const t = GaugeController.time / (CFG.GAUGE_PERIOD / 2);
// raw: 0→1→0 sine wave for half-period
const raw = (Math.sin(t * Math.PI - Math.PI / 2) + 1) / 2;
// Apply power ease — makes it linger at extremes, fast through center
const eased = Math.pow(raw, 1 / CFG.GAUGE_EASE_K);
// Map to angle: -90° (left/red) → 0° (center/green) → +90° (right/red)
GameState.gaugeAngle = (eased * 2 - 1) * (Math.PI / 2);
// Apply to SVG needle:
needle.setAttribute('transform', `rotate(${GameState.gaugeAngle * 180 / Math.PI}, 120, 115)`);
```

### Depth Mapping (on throw)

```js
// gaugeAngle: 0 = center = maxDepth; ±π/2 = edges = minDepth
const minDepth = GameState.maxDepth - CFG.MIN_DEPTH_OFFSET;
const normalizedToCenter = 1 - Math.abs(GameState.gaugeAngle) / (Math.PI / 2); // 0..1
GameState.selectedDepth = minDepth + normalizedToCenter * CFG.MIN_DEPTH_OFFSET;
// Example: maxDepth=10 → range [8m, 10m]. Center=10m, edges=8m.
```

On throw: needle freezes (stop calling `GaugeController.update`). Arc stroke flashes white for 150ms via CSS class toggle, then reverts.

---

## 7. Step-by-Step Implementation Plan

**Step 1 — HTML Shell & CSS**  
Single `<html>` file. `<meta viewport>` with `maximum-scale=1`. Import one Google Font (e.g., "Fredoka One") via CDN. Define CSS variables for all colors, font sizes, z-indices. Write all `@keyframes` (bounce, coin-fly, popup-fade, screen-flash, counter-pop). Set `touch-action: none` on both canvases. Wire up all screen divs with `display:none` by default.

**Step 2 — Three.js Bootstrap**  
Load Three.js from cdnjs r128 CDN. Init `WebGLRenderer({ alpha:true, antialias:true })`, `PerspectiveCamera`, `Scene`. Handle `resize` event: update `renderer.setSize`, `camera.aspect`, `camera.updateProjectionMatrix`. Build all static meshes (fisherman, water, seabed). Start `requestAnimationFrame` master loop.

**Step 3 — AudioManager**  
Implement `loadAudio(url)` → `fetch` → `arrayBuffer()` → `ctx.decodeAudioData`. `Promise.all` all 3 files at boot. Implement `play(key, opts)` that creates fresh `BufferSourceNode` each call. Add `ctx.resume()` call to every button click handler (belt-and-suspenders iOS compat). Start background music loop after decode.

**Step 4 — GameState & FSM**  
Implement `setState`, `onEnter` map, `onExit` map. Wire up all 8 state handlers with stubs. Add `update(dt)` dispatch that calls active state's `onUpdate` if defined.

**Step 5 — SHOP Screen UI**  
Build DOM for coin counter, upgrade buttons (×2 groups), Play button. Implement `renderUpgradeButtons()` that reads `GameState` and re-renders both buttons. Wire click handlers with coin validation. Implement gauge SVG (inline in Play button container). Wire Play button → snapshot gauge → `setState('DESCENDING')`.

**Step 6 — Gauge Controller**  
Implement pendulum math (see §6). Hook into master rAF loop — only active in SHOP state. On throw: freeze angle, flash arc, compute `selectedDepth`.

**Step 7 — Fish Spawner**  
`spawnFishForRound()`: iterate depth steps, roll spawn chance, pick weighted tier (filtered by `minDepth`), create `FishData` and Three.js mesh group. Place all fish in scene. Cap at `maxFish * FISH_POOL_MULTIPLIER`. Store in `FishSpawner.pool`. Call from `onEnter['DESCENDING']`.

**Step 8 — Hook Controller & Descent**  
Implement `HookController.update(dt)`: move hook Y, update camera Y with lerp, update fishing line geometry. Detect arrival at `selectedDepth` → `setState('PAUSED_AT_DEPTH')`. Detect arrival at surface (y ≥ -0.1) in ASCENDING states → `setState('SURFACE_SUMMARY')`.

**Step 9 — Catch Phase Input**  
Add `pointerdown`/`pointermove`/`pointerup` listeners to `document` (covers both canvas and UI). On down: teleport hook X immediately. On move: set `input.targetX`. On up: `targetX = 0`. All guarded by `fsm === 'CATCHING'`. In `onUpdate['CATCHING']`: lerp hook X, run collision check, call `catchFish` on hit.

**Step 10 — catchFish & Per-Catch VFX**  
`catchFish(fish)`: mark caught, attach mesh to hookGroup, play catch audio with pitch, trigger screen shake, spawn coin popup DOM element (positioned via `camera.project` → CSS left/top), show rarity label if applicable, spawn coin particles, trigger flash overlay for Legendary, update fish counter + fill bar. If max fish reached → `setState('ASCENDING_FAST')`.

**Step 11 — Surface Summary Sequence**  
Implement `playSummarySequence()` as a sequential setTimeout chain. For each fish: create DOM card at hook screen position, run bounce keyframe, show value popup, show rarity label popup, spawn burst. After all fish: roll total counter. Fly coins. Update coin counter. Pause. Advance round or go to END.

**Step 12 — End Screen**  
Fade in overlay. Show CTA button. Wire `window.open`. Fade out BGM. Freeze Three.js updates (optional — can let scene keep rendering for atmosphere).

**Step 13 — Polish & Device Testing**  
- Verify no round counter is ever exposed in DOM.  
- Test on 320×568 (SE), 375×667 (iPhone 8), 414×896 (XR), tablet.  
- Verify button layout has no overlap at any breakpoint.  
- Verify audio starts after first gesture on iOS Safari.  
- Verify `touch-action:none` prevents scroll-jank during catch drag.  
- Verify Legendary catch feels dramatically different from Common (size, sound, shake, flash, particle count).  
- Verify coin counter is always correct after each summary (cumulative across rounds).

---

## Appendix: Key Implementation Gotchas

| Issue | Solution |
|---|---|
| iOS audio silence | `ctx.resume()` in every button `touchend`; `AudioContext` created lazily on first gesture |
| Hook X teleport → jitter | Teleport only on `pointerdown`; lerp only on `pointermove`; never lerp on first contact |
| Fish screen position for particle spawn | `vector.set(fish.worldX, fish.worldY, 0).project(camera)` → `left = (v.x+1)/2 * window.innerWidth` |
| Coin-fly destination changes after DOM reflow | Snapshot both `getBoundingClientRect()` positions before summary sequence begins |
| Three.js line disappears on resize | Update both renderer size and camera projection matrix on `resize` |
| Legendary popup too large on small screens | All popup font sizes use `clamp(14px, 4vw, 36px)` |
| Fish visible before camera reaches them | Set `mesh.visible = false` at spawn; reveal when `hook.worldY > fish.worldY + 1.0` |
| Round counter accidentally shown | Never write `round` to any DOM textContent; audit all `UIManager` render calls |
