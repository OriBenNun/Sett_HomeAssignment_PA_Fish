# Fishing In The Deep — Technical Implementation Plan
> Single-file HTML Playable Ad · Senior Developer Reference

---

## 1. Core Systems & Architecture

### Single-File Structure
Everything lives in one `.html` file with three canonical blocks: `<style>` (CSS + CSS variables), one `<canvas>` element for the 3D scene, a layered HTML overlay for all 2D UI, and one `<script>` block that owns all logic.

```
index.html
├── <style>           CSS variables, UI layout, animations, keyframes
├── <canvas id="c">  Three.js WebGL scene (3D fisherman, hook, fish, water)
├── <div id="ui">    All 2D HUD overlays (coin counter, fish counter, gauge, buttons)
└── <script>
    ├── ThreeJS       Loaded from CDN (r128, cdnjs)
    ├── AudioManager  Wraps Web Audio API
    ├── GameState     Central FSM
    ├── SceneManager  Three.js world, camera rig, animation loop
    ├── UIManager     DOM manipulation, counters, transitions
    ├── GaugeSystem   Pendulum logic & depth calculation
    ├── FishingRound  Per-round logic (hook descent, sweep, ascent)
    ├── FishSystem    Fish pool, rarity, spawn, catch logic
    └── VFXSystem     Particles, coin bursts, text pop-ups (canvas 2D overlay + CSS)
```

### State Machine (FSM)
```
IDLE → SHOP → GAUGE_ACTIVE → HOOK_DESCENDING → HOOK_PAUSED →
HOOK_ASCENDING → SURFACE_SUMMARY → ROUND_RESET → (SHOP again x3)
→ END_SCREEN
```
Round index tracked internally but **never displayed.** Transitions driven by `GameState.transition(newState)` which fires `onExit` / `onEnter` callbacks on each manager.

### Rendering Architecture
- **Three.js r128** (CDN) renders the underwater world on the main `<canvas>`.
- A **second transparent `<canvas id="vfx">`** (CSS `position:absolute, pointer-events:none`) sits above for 2D particle bursts drawn with the Canvas 2D API.
- All UI text, counters, and buttons are HTML/CSS divs positioned absolutely over both canvases. This keeps text crisp and animatable via CSS without Three.js text rendering overhead.

### Coordinate Conventions
- World Y = 0 is water surface. Y goes negative downward (fish live at negative Y).
- Camera starts at surface looking down; follows hook via a lerped `cameraTarget.y`.

---

## 2. Gameplay Flow (State-by-State)

### IDLE / SHOP
- Coin counter visible, fish counter hidden.
- Upgrade buttons rendered. Each button checks `coins >= cost` and shows enabled/disabled state.
- Background music loops from first user interaction (Web Audio requires gesture).
- Fisherman 3D model bobs gently on a boat at Y=0. Idle hook hangs at surface.
- "PLAY" button visible with the gauge below it (pointer already oscillating).

### GAUGE_ACTIVE
- Gauge pointer oscillates continuously with a **sine-wave driven pendulum** (see §5 Gauge System).
- Player taps/clicks PLAY → pointer freezes → `selectedDepth` computed → transition to HOOK_DESCENDING.
- `rob_use.mp3` fires on button press.

### HOOK_DESCENDING
- Coin counter crossfades out; fish counter crossfades in (count = 0, fill bar = 0).
- Hook mesh animates from Y=0 downward to `−selectedDepth` over `duration = selectedDepth * 0.35` seconds (constant speed feel).
- Camera Y lerps behind hook with a slight lag (`lerpFactor = 0.08` per frame).
- Fish instances are pre-spawned at random X/Z and depth-weighted Y positions *before* descent begins (off-screen). Player cannot see them yet.

### HOOK_PAUSED
- Hook holds at target depth for **2 seconds on round 1**, **0.75 seconds on rounds 2–3**.
- On round 1 only: tutorial toast "MOVE THE HOOK TO CATCH FISH" fades in for 1.4 s then fades out; game resumes.

### HOOK_ASCENDING
- Hook moves upward at a base speed. Player can drag/touch horizontally to move hook in X (clamped to ±viewport_half_world_units).
- Fish collision: each frame, check distance from hook's XZ to each live fish's XZ. If within catch radius (`0.6` world units) and fish Y is within hook's current Y ± `0.5`: catch event fires.
- **Catch event:** `catch.mp3` plays, fish mesh parents to hook, fish counter +1, VFX burst fires (see §4), rarity label pops if applicable.
- When `caughtCount >= maxFish`: hook speed multiplies by **3×**, no further catches registered.
- As hook crosses Y = −1.5 (just before surface): fish counter crossfades back to coin counter.

### SURFACE_SUMMARY
- Hook arrives at Y = 0. Camera pans back to original surface framing (lerped, ~1 s).
- Caught fish animate with a **bouncy spring** (scale 1→1.4→0.9→1.1→1) one by one, 0.3 s apart.
- For each fish: coin value label appears with a +N animation floating upward; coin burst VFX fires from fish position.
- Rarity fish additionally trigger: screen-edge flash overlay, larger value label, attaboy title ("RARE!", "AMAZING!", "LEGENDARY!") with juicier scale animation and a starburst particle behind.
- After all fish animated: total summary label slides in, coins fly as particles from summary position to coin counter, counter ticks up.

### ROUND_RESET
- Hook resets to surface; fish counter hides; PLAY button re-appears; upgrade buttons re-render.
- Fish pool is cleared and re-randomized for next round.

### END_SCREEN (after 3rd round summary)
- Full-screen overlay fades in over ~0.5 s with a dark vignette.
- Single centered CTA button: **"PLAY NOW"** — `onclick="window.open('about:blank','_blank')"`.
- No round counter shown anywhere. No score breakdown.

---

## 3. Entities & Data

### GameState (singleton object)
```js
{
  state: String,          // FSM state name
  coins: Number,          // starts 100
  round: Number,          // 1–3, never displayed
  maxFish: Number,        // starts 6
  maxDepth: Number,       // starts 5 (meters)
  selectedDepth: Number,  // set by gauge on each throw
  caughtFish: Fish[],     // cleared each round
  upgrades: {
    maxFish:  [6,7,8,9,10],  costs: [0,50,100,200,400]
    maxDepth: [5,10,15,20,30], costs: [0,50,75,150,300]
  }
}
```

### Fish (data object, one per instance)
```js
{
  id: Number,
  rarity: 'common'|'uncommon'|'rare'|'amazing'|'legendary',
  value: Number,        // coin reward
  label: String|null,  // "RARE!" etc, null for common/uncommon
  mesh: THREE.Mesh,     // basic 3D shape (see §5)
  worldPos: Vector3,    // spawn position
  caught: Boolean,
  catchEffectPlayed: Boolean
}
```

### Fish Rarity Table
| Rarity | Spawn Weight | Depth Bias | Value | Label | VFX Intensity |
|---|---|---|---|---|---|
| common | 55% | any | 5–15 | — | small burst |
| uncommon | 25% | ≥ 5m | 20–40 | — | medium burst |
| rare | 12% | ≥ 10m | 60–100 | "RARE!" | large burst + flash |
| amazing | 6% | ≥ 15m | 150–250 | "AMAZING!" | starburst + shake |
| legendary | 2% | ≥ 20m | 400–600 | "LEGENDARY!" | full-screen flash + long starburst |

Depth bias is enforced at spawn time: fish of a rarity requiring depth ≥ Xm are only placed at Y ≤ −X. If `selectedDepth` is below minimum depth for a rarity tier, that tier is excluded from the weighted roll.

### Upgrade Buttons (UI data)
Each button renders: label, current level indicator (e.g. "6 fish"), next level value, and coin cost. Disabled styling when `coins < cost` or at max level. Purchase deducts coins immediately and re-renders button.

---

## 4. VFX, SFX & Particles

### Audio (Web Audio API)
Three files are loaded via `fetch` + `decodeAudioData` on first user gesture:
- **`background_music.mp3`** — loop from start, low gain (~0.4), resume/suspend with page visibility.
- **`catch.mp3`** — played on every fish catch. For rarer fish: pitch-shift via `AudioBufferSourceNode.playbackRate` (rare = 1.1×, amazing = 1.2×, legendary = 1.4×) for a satisfying escalation without extra files.
- **`rob_use.mp3`** — played on PLAY button press (gauge lock).

All three preloaded into `AudioBuffer` objects. Playback via `createBufferSource().start(0)` (cheap, fire-and-forget).

### Particle System (Canvas 2D overlay)
Lightweight custom particle pool on `<canvas id="vfx">`. Each particle: `{x, y, vx, vy, life, maxLife, radius, color, alpha}`. Updated and drawn in the same `requestAnimationFrame` loop as Three.js.

**Burst types:**
- **Coin burst** — 12–30 gold particles, radial velocity, gravity pull, 0.8 s life.
- **Catch burst** (common) — 8 white sparkle particles, fast fade.
- **Catch burst** (rare) — 20 colored particles + screen-edge color flash (CSS `box-shadow inset` pulse, 2 frames).
- **Catch burst** (amazing) — 35 particles, starburst shape (alternating long/short rays as lines), screen shake (CSS `transform: translate` jitter on `#ui` div, 3 frames).
- **Catch burst** (legendary) — 60 particles, full-screen white flash overlay (`opacity 0→0.8→0` in 0.3 s), sustained starburst, `catch.mp3` at 1.4× pitch, screen shake 6 frames.
- **Coin fly** (surface summary) — coins animate from fish world position (projected to screen) toward coin counter DOM position via CSS `transform` on `<span>` clones, ~0.6 s cubic-bezier arc.

### CSS Animations (keyframed)
- Fish catch label: `@keyframes popFloat` — scale 0.5→1.3→1 + translateY upward.
- Attaboy label: `@keyframes attaboy` — scale 0→1.5→1.1, color glow.
- Coin counter update: `@keyframes coinPulse` — brief scale 1→1.15→1 on the counter element.
- Tutorial toast: `@keyframes toastFade` — opacity 0→1→1→0 over 1.8 s.

---

## 5. 3D Visual Approach (Three.js, No Real Assets)

### Scene Composition
```
Scene
├── AmbientLight (soft blue, 0.6)
├── DirectionalLight (warm, from upper-right, cast shadows off)
├── WaterPlane (at Y=0)
├── Boat + Fisherman (at Y=0, always visible near top)
├── Hook + Line (animated, camera follows)
├── Fish[] (scattered below surface)
└── UnderwaterFog (THREE.FogExp2, deep blue)
```

**Water plane:** Large `PlaneGeometry` with a custom `ShaderMaterial` — simple animated sine wave vertex displacement + blue/teal gradient fragment. Rendered with `side: THREE.DoubleSide`. Transparency: `transparent:true, opacity:0.7`.

**Fisherman:** `BoxGeometry` body + `SphereGeometry` head. Warm tan color. Sits atop a boat (`BoxGeometry`, dark wood brown). Fishing rod = a thin `CylinderGeometry`. The fishing line is a `Line` object (`BufferGeometry` with 2 points: rod tip → hook position), updated each frame.

**Hook:** Small `TorusGeometry` (partial arc via `thetaLength: Math.PI`) in metallic silver (`MeshStandardMaterial`, `metalness:0.9, roughness:0.1`).

**Fish (per rarity):**
- common: flat `BoxGeometry` (elongated), grey-blue.
- uncommon: slightly larger, green-teal.
- rare: `ConeGeometry` + `SphereGeometry` combined (group), orange-gold.
- amazing: same as rare + `RingGeometry` halo spinning around it, bright yellow.
- legendary: larger compound mesh + animated point light child (colored glow), rainbow shimmer via hue-shifting `emissive` color in animation loop.

All fish have a gentle idle float animation: `mesh.position.y += Math.sin(time + id) * 0.003` per frame.

**Camera Rig:** Single `PerspectiveCamera` (FOV 60). In SHOP/surface states: position `(0, 2, 8)`, looking at `(0, 0, 0)`. During descent/ascent: camera Y tracks hook Y with lerp, offset `(0, hookY + 3, 8)`. Smooth via `camera.position.lerp(target, 0.06)` each frame.

**Underwater Atmosphere:** `THREE.FogExp2(0x001133, 0.04)`. Below Y=−2, a gradient `PlaneGeometry` quad (dark blue, additive blend) moves with camera to simulate depth darkening.

---

## 6. Step-by-Step Implementation Plan

### Step 1 — HTML Skeleton & CSS Variables
Set up the single file. Define CSS variables: `--deep-blue, --gold, --surface-teal, --ui-font`. Create `<canvas id="c">` (full-screen, z-index 0) and `<canvas id="vfx">` (same size, z-index 1, pointer-events:none) and `<div id="ui">` (z-index 2). Add Google Fonts link for a display font (e.g. `Fredoka One` for the game feel). Load Three.js from `cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` via `<script src>`.

### Step 2 — Audio Manager
Write `AudioManager` with `init()` (called once on first user gesture), `load(name, url)` → returns `AudioBuffer`, `play(name, rate=1.0)`, `playLoop(name)`. Preload all 3 files. Gate music start on `GameState.state !== 'END_SCREEN'`.

### Step 3 — Three.js Scene Setup
Initialize renderer (`antialias:true, alpha:false`), scene, camera, fog. Build all static meshes (boat, fisherman, rod, water plane with shader). Build hook mesh and line. Add lights. Start `animate()` loop calling `renderer.render(scene, camera)` and `VFXSystem.tick(dt)`.

### Step 4 — Fish System
Write `FishSystem.spawnPool(selectedDepth, maxFish, maxDepth)`: rolls N fish using weighted rarity table (depth-filtered), places them at random `(x: ±3, z: ±1, y: random in [−selectedDepth, −1])`. Each fish mesh added to scene, visibility `false` until hook descends past their Y. `FishSystem.checkCatch(hookPos)`: iterates live fish, checks 2D XZ distance + Y proximity.

### Step 5 — Gauge System
In the UI layer (HTML/CSS), render a **semi-circle gauge**:
- SVG arc from 180° to 0° (left to right), color gradient: red (both ends) → yellow (mid-outer) → green (center peak).
- A triangular pointer SVG element rotates around the arc center.
- Pendulum: `angle = centerAngle + amplitude * Math.sin(time * speed)` where `speed` starts slow and gradually increases each round (adds urgency). `ease = Math.pow(Math.abs(Math.cos(time * speed)), 0.5)` applied to velocity for ease-in/out feel at extremes.
- On PLAY press: freeze `angle`, map to depth: `depth = minDepth + (angle_normalized) * (maxDepth - minDepth)`.
- Visual zones on arc: red zones (edges) = shallow/min depth, green zone (center) = max depth.

### Step 6 — FSM & Game Loop
Implement `GameState` with `transition(newState)` calling each manager's `onStateEnter(state)` / `onStateExit(state)`. Implement all 8 states as described in §2. Hook tick logic inside `animate()` based on current state: move hook, update camera, tick fish system, update line geometry endpoints.

### Step 7 — UI Manager
DOM elements for: coin counter (`#coins`), fish counter (`#fish-count`), fish fill bar (`#fish-bar`), upgrade buttons (`#upgrade-fish`, `#upgrade-depth`), play button (`#play-btn`), gauge wrapper (`#gauge`), toast (`#tutorial-toast`), summary overlay (`#summary`), end screen (`#end-screen`). All transitions via CSS class toggling (`hidden`, `visible`, `fading`). Coin counter and fish counter share one slot — toggled with `opacity` + `pointer-events` crossfade (CSS transition ~0.3 s).

### Step 8 — VFX System
Implement particle pool (array of 200 pre-allocated objects, reused). Write `emit(type, screenX, screenY)` with per-type configs. Write `tick(dt)`: update all alive particles, clear vfx canvas, draw each particle. Implement screen shake: set a `shakeFrames` counter; in `animate()`, apply `document.getElementById('ui').style.transform = shakeOffset` for `shakeFrames` frames then clear.

### Step 9 — Surface Summary Sequence
Orchestrated via a `Promise`-chain or async/await sequence:
1. Camera lerp back to surface view.
2. For each caught fish (sequential, 0.3 s apart): spring-scale animation on mesh, spawn coin burst VFX, show value label, if rarity ≥ rare fire attaboy label.
3. After all fish done: show total summary div, animate coin particles to counter, tick counter up with `setInterval` tween.
4. On complete: if `round < 3` → ROUND_RESET, else → END_SCREEN.

### Step 10 — End Screen
Full-screen `<div id="end-screen">` with `position:fixed, z-index:100`. Fades in via CSS class. Contains only: game logo text (styled), and a large CTA button `<button onclick="window.open('about:blank','_blank')">PLAY NOW</button>`. No score, no round counter.

### Step 11 — Polish Pass
- Responsive: canvas resizes on `window.resize`, Three.js renderer `setSize`, UI uses `vw/vh` units.
- Mobile touch: `touchstart`/`touchmove`/`touchend` listeners mirror `mousedown`/`mousemove`/`mouseup` for hook drag and button taps.
- Ensure music does not auto-start (Web Audio policy) — defer `AudioManager.init()` to first `pointerdown` on document.
- Cap delta-time in `animate()` to 100ms to prevent physics jumps on tab-switch resume.
- Test at 360×640 (mobile portrait) and 1920×1080 (desktop).

---

## Key Implementation Notes

**Gauge feel (critical):** The pointer must *feel* like a pendulum, not a linear ping-pong. Use `position = Math.sin(elapsed * angularFreq)` — the sine curve naturally produces ease-in/out at extremes with no extra easing code. Increase `angularFreq` slightly each round (e.g. 0.8 → 1.0 → 1.2 rad/s) to add urgency without making it feel unfair. The SVG arc should clearly show the green "good zone" (deep) in the center and red "bad zone" (shallow) at edges so the player intuitively understands they're aiming for center = depth = more/rarer fish.

**Rarity escalation (critical):** Legendary catches must feel dramatically different from common catches. Budget: common = 1 particle burst + sound; legendary = full-screen white flash + 60 particles + screen shake + pitched-up sound + 2-second camera hold before resuming ascent. This contrast is what makes the game feel rewarding.

**Fish counter fill bar:** `width = (caughtCount / maxFish) * 100%`, CSS transition on width for smooth fill. Color shifts green → gold as it approaches full.

**Audio fallback:** If audio load fails (e.g. files not yet provided), catch the error silently and continue — game must function without audio during development.
