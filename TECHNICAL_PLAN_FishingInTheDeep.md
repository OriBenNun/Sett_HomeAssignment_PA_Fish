# Fishing In The Deep — Technical Implementation Plan
### Single-File HTML Playable Ad

---

## 1. Core Systems & Architecture

The entire game runs in **one self-contained HTML file** with no external dependencies. All rendering is done via **Three.js** (loaded from CDN), all UI is DOM/CSS overlaid on top of the canvas.

### Module Breakdown

| Module | Responsibility |
|---|---|
| `GameState` | Single source of truth (coins, round, upgrades, phase) |
| `SceneManager` | Three.js scene, camera, renderer, lighting |
| `HookController` | Hook descent/ascent, lateral drag input |
| `FishSpawner` | Procedural fish placement across depth layers |
| `CollisionSystem` | AABB overlap between hook and fish hitboxes |
| `UIController` | All DOM overlays: counters, gauge, buttons, panels |
| `VFXManager` | Particles, coin bursts, popups, fish jump animation |
| `AudioManager` | Web Audio API: synth sounds, no audio assets needed |
| `UpgradeSystem` | Validates and applies upgrades; persists state |
| `GameLoop` | `requestAnimationFrame` tick dispatching to all modules |

### Data Flow

```
Player Input → UIController → GameState mutation
GameState → SceneManager / HookController / VFXManager (reactive update each tick)
CollisionSystem → FishSpawner (mark caught) → VFXManager (trigger effect) → GameState (update counter)
```

---

## 2. Gameplay Flow

### Phase Machine

```
IDLE → CASTING → DESCENDING → PAUSED → ASCENDING → SURFACING → SUMMARY → IDLE (next round)
```

**IDLE** — Lobby screen. Upgrade panel + Play button visible. Gauge needle animates (pendulum, ease-in-out). Coin counter shown.

**CASTING** — Player clicks Play. Needle freezes. Depth = lerp(minDepth, maxDepth, needleNorm). Coin counter swaps to fish counter (0 / maxFish). Camera begins following hook.

**DESCENDING** — Hook moves down at constant speed (~3–4 m/s). Camera tracks it. Fish are already pre-spawned but invisible until camera reveals them.

**PAUSED** — Hook reaches target depth. Holds 0.5–1 s (Round 1: 2 s). Round 1 only: tutorial tooltip "Move the hook to catch fish!" fades in then out.

**ASCENDING** — Hook moves upward at base speed. Player drags left/right (mouse/touch). Collision detection active. Fish attach to hook visually. At maxFish caught: speed multiplies ×2.5, collision disabled.

**SURFACING** — Hook passes y=0 (surface). Fish counter swaps back to coin counter. Hook+fish enter jump/bounce animation above water.

**SUMMARY** — Per-fish value popups, coin burst VFX, rarity labels. Total coins tallied with counter spin. Coins fly to coin counter UI. Counter updates. After 3 rounds → END SCREEN; otherwise → IDLE.

**END SCREEN** — Full-screen overlay showing total earnings, CTA button.

---

## 3. Entities & Data

### GameState Object
```js
{
  coins: 100,
  round: 1,             // 1–3
  phase: 'IDLE',
  maxFish: 6,           // upgrade: 6→7→8→9→10
  maxDepth: 5,          // upgrade: 5→10→15→20→30 (meters)
  currentFishCount: 0,
  caughtFish: [],       // array of FishData
  selectedDepth: 0,     // determined at cast
  needleNorm: 0,        // 0..1 pendulum position
}
```

### Upgrade Tables
```js
MAX_FISH_UPGRADES = [
  { fish: 7,  cost: 50  },
  { fish: 8,  cost: 100 },
  { fish: 9,  cost: 200 },
  { fish: 10, cost: 400 },
]

MAX_DEPTH_UPGRADES = [
  { depth: 10, cost: 50  },
  { depth: 15, cost: 75  },
  { depth: 20, cost: 150 },
  { depth: 30, cost: 300 },
]
```

### Fish Types (by depth tier)
```js
FISH_TYPES = [
  { id:'sardine',   minDepth:0,  maxDepth:30, value:5,   color:0x88ccee, size:0.3, rarity:null       },
  { id:'bass',      minDepth:0,  maxDepth:30, value:12,  color:0x4488aa, size:0.45, rarity:null      },
  { id:'tuna',      minDepth:5,  maxDepth:30, value:30,  color:0x226688, size:0.6, rarity:'Rare!'    },
  { id:'swordfish', minDepth:10, maxDepth:30, value:75,  color:0x113355, size:0.8, rarity:'Amazing!' },
  { id:'anglerfish',minDepth:15, maxDepth:30, value:150, color:0x221133, size:0.7, rarity:'Legendary!'},
  { id:'oarfish',   minDepth:20, maxDepth:30, value:300, color:0xaa3322, size:1.1, rarity:'JACKPOT!' },
]
```

Each fish is spawned with `{ type, worldPos:{x,y,z}, caught:false, mesh, hitbox }`.

### Hook Entity
```js
{ mesh, lineGeometry, worldPos, velocity, lateralPos, caughtFish:[] }
```

---

## 4. VFX, SFX & Particles

### VFX (CSS + Three.js)

| Effect | Implementation |
|---|---|
| Fish catch popup | DOM `div` absolutely positioned at fish screen coords, CSS keyframe: scale 0→1.3→1, fade out upward |
| Rarity label | Same popup with gold/purple glow shadow; stacked below value |
| Coin burst | Three.js `Points` system — 12–20 gold particles ejected radially from fish, arc downward with gravity |
| Hook speed trail | Three.js `Line` trailing the hook, fading alpha over 8 frames |
| Water surface | Animated sine-wave displacement on a flat `PlaneGeometry` shader; ripple on hook entry/exit |
| Fish jump anim | CSS `@keyframes` translateY bounce (3 bounces, ease-out) on summary phase |
| Coins fly to UI | DOM coin `div`s animated via JS `animate()` API from fish position to coin counter DOM rect |
| Counter spin | CSS digit-slot scroll animation on coin counter number |

### SFX (Web Audio API — zero assets)

| Event | Sound Synthesis |
|---|---|
| Needle tick | Short `OscillatorNode` sine 880Hz, 50ms |
| Cast / plunge | White noise burst + low-pass filter sweep down |
| Fish caught | Sine chord (root + 5th), 200ms, pitch varies by fish value |
| Rarity catch | Arpeggiated chord + reverb (convolver with impulse response) |
| Coin collect | High-pitched sine 1200Hz with fast decay |
| Hook surface splash | Noise burst + bandpass 400Hz |
| Summary fanfare | Short 3-note ascending synth melody |

---

## 5. 3D Visual Approach

**Renderer:** Three.js WebGL, `antialias: true`, tone mapping `ACESFilmic`.

**Camera:** `PerspectiveCamera` (FOV 60), positioned at `(0, 2, 10)` in lobby. During descent/ascent it smoothly tracks the hook's Y with a small lag (`lerp` factor 0.08) and a fixed X offset to keep the hook slightly left-of-center.

**Scene Geometry (primitive-only, no external assets):**

- **Hook:** `TorusGeometry` (small arc) + `CylinderGeometry` for the shank, merged. Silver `MeshStandardMaterial`.
- **Line:** Dynamic `BufferGeometry` `Line` from hook to top of frame, updated every tick.
- **Fish:** `SphereGeometry` flattened (scaleY 0.5) + `ConeGeometry` tail, grouped. Each species has a distinct `color` and `emissive` glow at depth.
- **Fisherman:** `BoxGeometry` body, `SphereGeometry` head, `CylinderGeometry` rod — static at top, partially clipped by frame top edge.
- **Water surface:** `PlaneGeometry` (20×20, 40 segments) with animated vertex Y displacement via `ShaderMaterial`. Semi-transparent blue.
- **Underwater background:** Gradient fog (`scene.fog = new THREE.FogExp2`) in deep blue. Scattered `SphereGeometry` bubble particles drifting up slowly.
- **Depth layers:** Subtle color temperature shift of `scene.background` as hook descends (surface: light cyan → deep: near-black indigo).

**Lighting:**
- `DirectionalLight` (sun, above-water warm) fades out as hook descends.
- `PointLight` on the hook — cool blue, intensity increases with depth.
- `AmbientLight` constant low-level.

---

## 6. Step-by-Step Implementation Plan

### Step 1 — HTML Shell & Boilerplate
- Single `index.html` with inline `<style>` and `<script>`.
- Import Three.js from `unpkg.com` CDN via `<script type="module">`.
- `<canvas id="game">` fills 100vw/100vh.
- `<div id="ui">` absolutely positioned over canvas — holds all DOM UI layers.

### Step 2 — Three.js Scene Setup
- Init `WebGLRenderer`, `Scene`, `PerspectiveCamera`, `AnimationLoop`.
- Add water plane, fog, ambient + directional lights.
- Build fisherman geometry and fix it at top of scene.

### Step 3 — GameState & Phase Machine
- Implement `GameState` object and `setPhase(phase)` function.
- `setPhase` drives all module transitions (show/hide UI, start/stop animations).

### Step 4 — Gauge & Cast System
- DOM overlay: semicircular SVG gauge + animated needle (CSS `@keyframes` pendulum, `animation-timing-function: cubic-bezier(0.45, 0, 0.55, 1)`).
- On Play click: freeze needle, compute `selectedDepth`, call `setPhase('CASTING')`.

### Step 5 — Upgrade UI
- Two upgrade panels as DOM elements.
- Each button reads `GameState.coins`; disabled + greyed if insufficient.
- On purchase: deduct coins, update `maxFish`/`maxDepth`, re-render button states.

### Step 6 — Fish Spawner
- On `CASTING`, call `spawnFish(selectedDepth, maxFish * 2.5)` — overspawn so player has targets.
- Place fish at random `(x: -3..3, y: -1..-selectedDepth, z: -1..1)` filtered by depth tier.
- Create Three.js mesh per fish, add to scene.

### Step 7 — Hook Descent
- In `DESCENDING` phase: each tick move hook `y -= speed * delta`.
- Update line geometry endpoints.
- Camera Y lerps to hook Y.
- On reaching `selectedDepth`: store pause timer → `PAUSED`.

### Step 8 — Tutorial Tooltip (Round 1)
- In `PAUSED`, if `round === 1`: show DOM tooltip div with fade-in CSS, auto-dismiss after 1.5 s, then resume.

### Step 9 — Hook Ascent & Drag Input
- `ASCENDING`: hook Y increases each tick.
- Mouse/touch `mousemove`/`touchmove`: map screen X to world X (`-3..3`), lerp hook X.
- Each tick: run collision check — for each uncaught fish, test hook AABB overlap.
- On collision: `catchFish(fish)` → mark caught, attach mesh to hook group, trigger VFX popup, update counter.
- If `caughtFish.length >= maxFish`: boost speed, disable collisions.

### Step 10 — Surface & Summary
- When hook Y ≥ 0: `setPhase('SURFACING')`.
- Swap fish counter DOM back to coin counter.
- Trigger fish bounce CSS animation.
- After bounce: iterate `caughtFish` — show value popup per fish, coin burst particles, rarity label.
- After all popups: show total, animate coins flying to counter, update `GameState.coins`.
- After 1.5 s: if `round < 3` → reset + `setPhase('IDLE')`; else `setPhase('END')`.

### Step 11 — Round Reset
- Remove all fish meshes from scene.
- Reset hook position to surface.
- Camera back to lobby position.
- Clear `caughtFish`, reset `currentFishCount`.
- Increment `round`.

### Step 12 — VFX & Audio Polish
- Implement `VFXManager.coinBurst(worldPos)` using Three.js `Points`.
- Implement `VFXManager.floatLabel(text, screenPos, style)` for value/rarity popups.
- Implement `AudioManager` with all synth sounds wired to game events.
- Animate water surface vertices each tick via sine wave.

### Step 13 — End Screen
- Full-screen DOM overlay, semi-transparent dark background.
- Display total coins earned across 3 rounds.
- "Play Again" button → reset full `GameState`, `setPhase('IDLE')`.

### Step 14 — Polish Pass
- Ensure needle ease-in-out feels satisfying (tune cubic-bezier).
- Confirm coin counter uses slot-machine scroll animation.
- Verify all upgrade buttons disable/enable reactively.
- Test on mobile: touch events for drag and tap.
- Clamp hook X within world bounds during drag.
- Add subtle screen shake (camera) on rarity catch.
