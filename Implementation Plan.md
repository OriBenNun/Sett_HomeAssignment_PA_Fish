# Fishing In The Deep — Technical Plan

## 1. Core Systems & Architecture

The playable is a single HTML file using **Three.js** (loaded from CDN) for 3D rendering, with an HTML/CSS overlay for the HUD. All game entities are real 3D meshes built from primitive geometries — no imported models or textures.

### 1.1 Game Loop

A standard `requestAnimationFrame` loop with delta-time accumulation:

- **update(dt)** — advances game state (physics, hook movement, fish AI, animations, timers).
- **renderer.render(scene, camera)** — Three.js draws the scene each frame.
- A simple **state machine** drives the top-level flow (see §2).

### 1.2 Rendering Strategy (Three.js)

All entities use Three.js primitive geometries with `MeshPhongMaterial` or `MeshToonMaterial` for simple shading. No textures, no imported assets.

- **Scene setup**: A single `THREE.Scene` with a directional light + ambient light. A fog component (`THREE.FogExp2`) tinted deep blue to sell underwater depth naturally.
- **Camera**: `THREE.PerspectiveCamera` looking slightly downward. The camera's Y position tracks the hook during descent/ascent, providing the vertical scroll.
- **Ocean background**: A large `PlaneGeometry` behind the action with a gradient shader (vertex-colored from light blue at top to dark navy at bottom). Optionally a second plane further back with subtle animated caustic patterns via a simple vertex displacement.
- **Depth markers**: `THREE.Sprite` or `CSS2DRenderer` labels at 5 m intervals, positioned in world space so they scroll naturally with the camera.
- **Particles**: Use `THREE.Points` with `BufferGeometry` for coin bursts and sparkles — GPU-friendly and trivial to emit/recycle.
- **Device Pixel Ratio (DPR)**: Set `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` — cap at 2× to prevent mobile GPUs (especially 3× iPhone screens) from rendering at unnecessarily high resolutions. This is a major mobile performance lever.
- **Geometry complexity**: Keep segment counts low on all primitives (e.g. `SphereGeometry(r, 12, 8)` not 32×32). On mobile GPUs, vertex count matters more than desktop.

### 1.3 Entity Geometry Reference

| Entity | Geometry | Notes |
|---|---|---|
| Fish (common) | `SphereGeometry` (squashed X) + `ConeGeometry` tail | Colored by rarity tier |
| Fish (rare+) | Same but slightly larger, emissive glow | `emissive` property on material |
| Hook | `TorusGeometry` (half arc) + `CylinderGeometry` barb | Small scale, grey metal material |
| Fishing line | `THREE.Line` from rod tip to hook | Update positions each frame |
| Fisherman | `BoxGeometry` body + `SphereGeometry` head + `CylinderGeometry` rod | Positioned at surface, static |
| Coin particle | `CircleGeometry` or `Points` | Gold colored, short lifespan |

### 1.4 Coordinate System

- Three.js default: Y-up. We invert the convention for depth: the surface is at world Y = 0, and deeper = negative Y. Alternatively, keep Y-up and treat the hook going down as decreasing Y — this is more natural for Three.js.
- World units = meters. 1 unit = 1 m.
- Horizontal play area: roughly ±2.5 m (X axis) for hook dragging.
- Z axis unused for gameplay (fixed camera angle), but fish can have slight Z variation for visual depth.

### 1.5 Input

- Use the **Pointer Events API** (`pointerdown`, `pointermove`, `pointerup`) as the single input path — it unifies mouse and touch without separate code branches.
- **Gauge stop & UI buttons**: `pointerdown` on the relevant element.
- **Hook dragging**: on `pointerdown` during `FISHING` state, track the pointer ID. On `pointermove` for that ID, map the pointer's screen-X to world-X for the hook. On `pointerup`, release. This avoids multi-touch confusion.
- **Prevent browser defaults**: apply `touch-action: none` on the canvas and HUD to block pull-to-refresh, scroll bounce, pinch-zoom, and the 300ms tap delay. Also call `e.preventDefault()` on touch events on the canvas as a safety net.
- **Hit testing**: all interactive HUD elements (buttons, gauge) must be standard DOM elements — not drawn on the WebGL canvas — so native pointer events just work without manual hit-rect math.

---

## 2. Gameplay Flow (State Machine)

```
READY → DESCENDING → TUTORIAL_PAUSE (round 1 only) → FISHING → REEL_SPEEDUP → SURFACE_SUMMARY → (repeat or END_SCREEN)
```

| State | Description |
|---|---|
| `READY` | Gauge pointer is already oscillating. Upgrade buttons are visible and interactive. Coin counter visible. Tapping the play button stops the gauge, computes target depth, and transitions out. Upgrades can be purchased at any time while the gauge runs. |
| `DESCENDING` | Hook drops to target depth. Camera follows. Fish are spawned/streamed into the world column. Coin counter swaps to fish counter. |
| `TUTORIAL_PAUSE` | (Round 1 only) Hook pauses 2 s, overlay text "Move the hook to catch fish" appears then fades. |
| `FISHING` | Hook ascends. Player drags horizontally. Collision detection catches fish. Effects fire per catch. Fish counter fills. |
| `REEL_SPEEDUP` | Triggered when `caughtCount == maxFish`. Hook speed doubles+, no more catches. |
| `SURFACE_SUMMARY` | Fish-by-fish reveal animation, coin burst, total tally with counter roll-up, coins fly to HUD. |
| `END_SCREEN` | After round 3's summary. Shows only the CTA button (e.g. "Download Now" / "Play the Full Game"). No gameplay stats or recap — clean and focused. |

A `roundIndex` (0–2) tracks the current round. After `SURFACE_SUMMARY`, if `roundIndex < 2`, increment and go back to `READY`; otherwise go to `END_SCREEN`.

---

## 3. Entities & Data

### 3.1 Player State

```
playerState = {
  coins: 100,
  round: 0,            // 0–2
  maxFish: 6,           // current upgrade level
  maxDepth: 5,          // meters, current upgrade level
  maxFishLevel: 0,      // index into upgrade table (0 = base)
  maxDepthLevel: 0,
}
```

### 3.2 Upgrade Tables

**Max Fish upgrades** (level → value, cost):

| Level | Max Fish | Cost |
|---|---|---|
| 0 (base) | 6 | — |
| 1 | 7 | 50 |
| 2 | 8 | 100 |
| 3 | 9 | 200 |
| 4 | 10 | 400 |

**Max Depth upgrades**:

| Level | Max Depth (m) | Cost |
|---|---|---|
| 0 (base) | 5 | — |
| 1 | 10 | 50 |
| 2 | 15 | 75 |
| 3 | 20 | 150 |
| 4 | 30 | 300 |

### 3.3 Fish Definition

Each fish type is a data record:

```
fishTypes = [
  { id: "common",    value: 5,   color: "#6CB4EE", minDepth: 0,  weight: 50, rarity: null },
  { id: "uncommon",  value: 10,  color: "#3CB371", minDepth: 3,  weight: 30, rarity: null },
  { id: "rare",      value: 25,  color: "#DA70D6", minDepth: 8,  weight: 12, rarity: "Rare!" },
  { id: "amazing",   value: 50,  color: "#FFD700", minDepth: 14, weight: 6,  rarity: "Amazing!" },
  { id: "legendary", value: 100, color: "#FF4500", minDepth: 22, weight: 2,  rarity: "Legendary!" },
]
```

- `minDepth`: fish only spawns at this depth or deeper.
- `weight`: relative spawn probability (filtered by depth, then weighted random).
- `rarity`: label shown on catch (null = no label).

### 3.4 Fish Instance (runtime)

```
fish = {
  type: <fishTypes ref>,
  mesh: <THREE.Group>,  // the 3D mesh in the scene
  x: Number,            // world X (horizontal)
  y: Number,            // world Y (depth)
  vx: Number,           // horizontal drift speed
  dir: 1 | -1,          // facing direction (flip mesh scale.x)
  caught: false,
  catchAnim: null,       // active tween when caught
}
```

Fish swim horizontally back and forth within a range. They reverse direction at world edges or randomly.

### 3.5 Hook

```
hook = {
  x: 0,
  y: 0,
  targetDepth: 0,       // set by gauge
  speed: 2,             // m/s base descent/ascent speed
  caughtFish: [],        // references to caught fish instances
}
```

### 3.6 Gauge

```
gauge = {
  pointerT: 0,          // 0–1 normalized position
  direction: 1,
  speed: 0.6,           // oscillations per second
  stopped: false,
}
```

The gauge runs continuously during the `READY` state while upgrades are available. It maps a sinusoidal eased value:

```
displayT = (sin(pointerT * PI - PI/2) + 1) / 2   // ease-in-out
depth = minDepth + displayT * (maxDepth - minDepth)
minDepth = max(0, maxDepth - 2)
```

This gives the pendulum feel with ease at extremes.

### 3.7 Particle System

Uses `THREE.Points` with a `BufferGeometry` for GPU-efficient coin bursts and catch sparkles:

```
// Per-particle attributes in the buffer:
positions: Float32Array,  // x, y, z per particle
velocities: Float32Array, // vx, vy, vz
lifetimes: Float32Array,  // remaining life
colors: Float32Array,     // r, g, b per particle
```

A fixed pool of ~100 particles. On emission, assign position/velocity/color and set lifetime. Each frame, update positions and fade opacity. Dead particles sit at off-screen positions until recycled.

---

## 4. Step-by-Step Implementation Plan

### Step 1 — Scaffold & Game Loop

- Create the HTML file. Include the viewport meta tag: `<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">` to lock mobile scaling.
- Load Three.js r128 from CDN (`cdnjs.cloudflare.com`). Add an overlaid HUD `<div>` on top of the WebGL canvas.
- Apply `touch-action: none` on `html`, `body`, and canvas to suppress all browser touch gestures (scroll, zoom, pull-to-refresh).
- Initialize `THREE.WebGLRenderer` with `antialias: false` (cheaper on mobile — primitives don't need AA), `THREE.Scene`, `THREE.PerspectiveCamera`, and lighting (one `DirectionalLight` + one `AmbientLight`).
- Cap pixel ratio: `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`.
- Implement `requestAnimationFrame` loop with delta time, clamped to a max dt of ~50ms (0.05s) to prevent physics jumps when the tab loses focus or the mobile browser throttles.
- Handle window resize and orientation change: update renderer size, camera aspect ratio. Design for portrait-first (9:16ish) but adapt gracefully to landscape by adjusting camera FOV or frustum rather than breaking the layout.
- Create the state machine skeleton with stub states.

### Step 2 — Ocean Environment & Camera

- Create a large `PlaneGeometry` behind the play area with vertex colors (light blue at top → dark navy at bottom) to serve as the ocean backdrop.
- Add `THREE.FogExp2` to the scene (deep blue tint) so distant/deep geometry fades naturally.
- Implement camera Y tracking: camera.position.y follows the hook during descent/ascent.
- Add depth marker sprites/labels at 5 m intervals in world space.
- Position a "surface" zone at Y = 0 — a lighter-colored plane or band representing the waterline.

### Step 3 — HUD (HTML/CSS overlay)

- All HUD elements use CSS relative units (`vh`, `vw`, `em`, `%`) so they scale to any screen size. No hardcoded pixel dimensions.
- Respect mobile safe areas: apply `padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left)` on the HUD container to avoid notches and system bars.
- Coin counter (top area): icon + animated number.
- Fish counter (same slot): icon + count + a horizontal fill bar (`current / maxFish`).
- Toggle visibility between them based on state.
- Upgrade buttons (bottom area): two styled buttons showing next level cost. Grey-out if unaffordable or maxed. Minimum touch target size of 44×44 CSS pixels (Apple HIG guideline) with adequate spacing between buttons to prevent mis-taps.
- Play button (bottom center): large and prominent — this is the primary action, so make it the biggest tap target on screen.
- All text uses `clamp()` for font sizing (e.g. `font-size: clamp(14px, 3.5vw, 22px)`) to remain readable on small phones and proportional on tablets.

### Step 4 — Ready State (Gauge + Upgrades)

- Render a horizontal gauge bar with gradient zones (green center = max, red edges = min).
- Animate a pointer using sinusoidal easing (pendulum). The gauge starts running immediately on entering `READY`.
- Upgrade buttons are visible and interactive alongside the gauge. On click: check `coins >= cost`. If yes, deduct coins, increment level, update `maxFish` or `maxDepth`, refresh button labels. If at max level, button shows "MAX" and is disabled.
- Note: upgrading max depth mid-gauge changes the depth range the pointer maps to in real time — the gauge stays running, just the output scale shifts.
- On tap of play button: freeze pointer, compute `targetDepth`, hide upgrade buttons, transition to `DESCENDING`.

### Step 5 — Hook, Line & Fisherman (3D)

- Build the fisherman from primitives: `BoxGeometry` torso, `SphereGeometry` head, `CylinderGeometry` rod. Position at the surface, static.
- Build the hook: half `TorusGeometry` arc + `CylinderGeometry` barb, grouped into a `THREE.Group`.
- Fishing line: `THREE.Line` with `BufferGeometry` — update the two endpoint positions each frame (rod tip → hook).
- Descent: hook group's Y position moves toward targetDepth at constant speed; camera follows.
- Arrival pause (0.5–1 s, or 2 s for round 1 tutorial).

### Step 6 — Fish Spawning & AI (3D)

- Build a reusable fish mesh factory: squashed `SphereGeometry` body + `ConeGeometry` tail fin, wrapped in a `THREE.Group`. Material color and scale vary by type. Rare+ fish get an `emissive` glow on their material.
- When entering `DESCENDING`, instantiate fish meshes and add to scene, distributed across the water column.
- Distribution: split the column (0 to targetDepth) into depth bands. For each band, pick eligible fish types by `minDepth`, then weighted-random select.
- Total fish count: `maxFish * 3` (enough that the player can't catch them all but has plenty of targets). Spread evenly across the depth range.
- Fish AI: simple horizontal oscillation with random speed and range. Update mesh position + flip scale.x for direction. Reverse at world edges.

### Step 7 — Fishing (Ascent + Catching)

- Hook ascends at base speed. Player drags hook left/right via pointer events.
- Map pointer screen-X to world-X using the camera's projection. On mobile, use the pointer's *delta* from drag-start rather than absolute position — this avoids the hook snapping to the finger and lets the player's thumb stay near the bottom of the screen while controlling the hook above. A sensitivity multiplier (e.g. 1.5×) ensures the full horizontal range is reachable without large swipes.
- Clamp hook.x to world horizontal bounds.
- **Collision**: distance check between hook position and fish position in XY plane (depth + horizontal). On distance < catch radius and `!fish.caught` and `caughtCount < maxFish`:
  - Set `fish.caught = true`, attach to hook (offset below hook, stacking).
  - Increment fish counter, animate fill bar.
  - Spawn catch effect (sparkle burst + bouncing coin value text).
  - Show rarity label if applicable (floating text that scales up and fades).
- When `caughtCount == maxFish` → `REEL_SPEEDUP`: multiply ascent speed ×2.5, disable further catches.

### Step 8 — Surface Summary Animation

- When hook.y ≤ 0, transition to `SURFACE_SUMMARY`.
- Swap fish counter back to coin counter (but don't update value yet).
- Sequentially animate each caught fish:
  - Fish jumps out of water (bounce tween).
  - Value text pops above it.
  - Rarity attaboy label if present.
  - Coin burst particles from fish position.
  - Short delay, then next fish.
- After all fish: show total in a large centered counter that rolls up from 0.
- Coin-fly animation: small coin sprites travel from the total to the HUD coin counter.
- HUD coin counter updates to new total.
- Brief pause, then transition to `READY` (next round) or `END_SCREEN`.

### Step 9 — Tutorial Overlay (Round 1)

- After hook arrives at depth in round 1, display a semi-transparent overlay with "Move the hook to catch fish" text.
- Auto-dismiss after 2 seconds (fade out over 0.3 s).
- Only triggers when `round == 0`.

### Step 10 — End Screen

- After round 3 summary, fade to end screen.
- Show only the CTA button (e.g. "Download Now"), centered. No scores, no recaps — keep it clean and conversion-focused.
- The 3D scene can remain visible behind a dimmed overlay for visual continuity.

### Step 11 — Polish & Juice

- **Easing library** (inline): a few easing functions (easeOutBack, easeInOutSine, easeOutElastic) for tweens.
- **Camera shake** on legendary catch (offset camera position for a few frames).
- **Hook sway**: slight sinusoidal X offset on the hook mesh while descending.
- **Fish idle animation**: gentle Y-axis bobbing via `sin(time + offset)` per fish.
- **Coin particles**: `THREE.Points` with animated positions + fading opacity.
- **Sound hooks**: add empty `playSound(id)` calls at key moments for future audio integration.
- **Performance budget (mobile-first)**:
  - Target: stable 60 fps on mid-range phones (e.g. iPhone 12, Galaxy A54).
  - Total scene draw calls: aim for < 50. Reuse `Geometry` instances across fish of the same type — only the `Material` or `mesh.material.color` should differ.
  - Remove fish meshes from scene when they scroll off-camera (above or below the visible frustum).
  - Particle pool capped at 100. If a burst would exceed the pool, recycle the oldest particles.
  - Avoid runtime allocations in the game loop (no `new Vector3()` per frame) — pre-allocate reusable scratch vectors.
  - `renderer.info.render` is useful during development to monitor draw calls and triangles per frame.

---

## Appendix: File Structure (Single HTML)

```
<!DOCTYPE html>
<html>
<head>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
  <style>
    /* Reset & mobile lockdown */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body, canvas { 
      width: 100%; height: 100%; overflow: hidden; 
      touch-action: none; -webkit-touch-callout: none; user-select: none;
    }
    /* HUD overlay, buttons (min 44px tap targets), counters,
       safe-area padding, responsive font sizes via clamp(),
       animations (coin-fly, bounce, fade) — all HTML/CSS on top of WebGL */
  </style>
</head>
<body>
  <div id="hud">
    <!-- coin counter, fish counter, upgrade buttons, play button, gauge -->
  </div>
  <script>
    // §1  Constants & Config (fish types, upgrade tables, tuning)
    // §2  Three.js setup (renderer, scene, camera, lights, fog)
    // §3  State machine & player state
    // §4  Input manager (pointer/touch)
    // §5  Mesh factories (createFish, createHook, createFisherman)
    // §6  Gauge logic
    // §7  Fish spawner (instancing + scene management)
    // §8  Collision detection
    // §9  Animation/tween system
    // §10 Particle system (THREE.Points)
    // §11 HUD controller (show/hide/update DOM elements)
    // §12 Game loop (update + renderer.render + RAF)
    // §13 Init & resize
  </script>
</body>
</html>
```

All code lives in a single `<script>` block, organized into clearly commented sections. Three.js is the only external dependency (loaded from CDN). The HUD is pure HTML/CSS overlaid on the WebGL canvas — no CSS2DRenderer needed for UI, only for optional in-world depth labels.
