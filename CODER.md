# CODER.md — Anime City 3D Web Game (Three.js, Single-File)

## Goal
Build a **single HTML page** Three.js game styled like a **Japanese anime cartoon** (cel-shaded / toon shading, pastel colors, cherry blossoms, kanji signs). The page must run standalone in a modern browser with no build step.

---

## Hard Requirements

### Tech
- **Single file**: `index.html` only. No bundler, no `package.json`.
- Use Three.js via **importmap** from a CDN (e.g. `https://unpkg.com/three@0.160.0/build/three.module.js` + `examples/jsm/`).
- No external assets, no model loaders, no textures fetched from the web. All visuals generated procedurally with Three.js primitives + `CanvasTexture`.
- Run by just opening the file (`file://`) in Chrome/Edge/Firefox.

### Visual style — "anime cartoon"
- Use **`MeshToonMaterial`** with a small custom **gradient map** (3–4 hard steps) for cel-shaded look. Do NOT use `MeshStandardMaterial` for the main objects.
- **Pastel color palette**: pink, baby blue, mint, lavender, cream. Avoid realism.
- **Gradient sky**: blue → pink → warm horizon (use a `CanvasTexture` set as `scene.background`).
- Add **atmospheric fog** in a pastel pink tone.

### Character — ONE anime boy
- Built from boxes/cylinders/spheres/cones. No external models.
- **Big anime head** (proportionally larger than realistic).
- **Spiky hair** with a signature **ahoge** (single antenna strand) and a highlight color (e.g. indigo hair with a violet highlight variant).
- **Large anime eyes**: white sclera + colored iris + black pupil + **two white highlights** (top-right big, bottom-left small). Eyes face forward.
- Eyebrows, small mouth, **pink blush circles** on cheeks.
- School uniform: navy blazer, white V-collar, **red tie** (with knot), dark pants, dark shoes.
- Skin tone material should be a warm pastel (e.g. `#ffe1c4`).

### Environment — 3D anime city
- Grid of city **blocks** separated by **roads** with lane stripes.
- **Buildings**: pastel-colored boxes with pitched/cone roofs, rows of small emissive window panels in varied colors (yellow/blue/pink/green), some buildings get a roof-mounted **Japanese-style sign** (rendered to a `CanvasTexture`) with kanji or katakana words like カフェ, 寿司, ラーメン, マンガ, アニメ, ゲーム, 神社, 学校, 花屋, 魔法.
- **Sakura (cherry blossom) trees** lining sidewalks — brown cylinder trunk + cluster of pink spheres for foliage.
- At least one **red torii gate**.
- Drifting **clouds** (white sphere clusters) and falling **pink petals** (lots of small planes with rotation).
- Sidewalks in a light pink/cream color.

### Camera & controls
- **Third-person follow camera** that smoothly lerps behind the character.
- **WASD movement, world-space (NOT camera-relative)**:
  - `W` → forward (the +Z direction in world space)
  - `S` → backward (−Z)
  - `A` → **rotate character RIGHT** (positive yaw, in place, no translation)
  - `D` → **rotate character LEFT** (negative yaw, in place, no translation)
- **A and D must rotate CONTINUOUSLY while held** — at a rate of **30° per second** (use `THREE.MathUtils.degToRad(30) * dt` per frame). Do NOT use keydown edge detection for turn; do NOT snap by 30° per press.
- W/S move the character along its current facing direction (use `sin(yaw)` / `cos(yaw)` for the forward vector).
- **Mouse drag** rotates the camera (yaw + pitch). Pitch is clamped (e.g. −0.3 to +1.0 radians).
  - **Vertical drag must be inverted**: drag mouse **up** → camera tilts **down**; drag **down** → tilts **up**. Apply this to BOTH mouse and touch handlers.
- **Mouse wheel** zooms in/out (clamp camera distance, e.g. 4–30 units).
- Camera is **decoupled from the character's facing direction** — the camera stays where the user dragged it, the character faces wherever the W/A/D keys set it.
- Include **touch support** for mobile (single-finger drag = rotate camera).

### Character animation
- **Walk cycle** when moving: arms swing opposite, legs swing opposite, torso bobs up/down. Use a phase variable incremented by `dt` (faster when running).
- **Run** is `Shift` held while moving — multiplies speed AND walk-cycle frequency.
- **Idle** smoothly returns limbs to neutral (decay coefficients).
- **Hair physics**: subtle sinusoidal sway on the bangs group.
- Character smoothly rotates to face its movement direction (use `quaternion.slerp`).

### Collision
- Each building stores its position and width/depth. A simple AABB check against a small character radius (~0.5) prevents walking through buildings.
- Clamp character position to a reasonable city bounding box (e.g. ±95 on X and Z).

### HUD / UI
- A small floating **HUD panel** with control instructions, styled with a pastel pink theme, rounded corners, soft drop shadow, no `pointer-events`.
- A title badge in the top-right ("🌸 ANIME CITY 🌸" style).
- A loading overlay that hides once `animate()` starts.
- Use a fun rounded font fallback stack: `'Comic Sans MS', 'Marker Felt', system-ui, sans-serif`.
- HUD must clearly state the actual controls: **W/S walk, A/D turn (30°/s), Mouse drag camera, Scroll zoom, Shift run**.

### Performance / quality
- Set `renderer.shadowMap.enabled = true` with PCFSoftShadowMap.
- A single `DirectionalLight` for sun + soft `HemisphereLight` + a fill `DirectionalLight`.
- `pixelRatio` capped at 2.
- `RoomEnvironment` for IBL (optional, helps if you mix in any standard material).
- All animations driven by `THREE.Clock` and `dt`, not raw frame counts.

---

## Lessons Learned (do not repeat these mistakes)

1. **Do not make WASD movement camera-relative.** The user wants absolute world-space control. The camera orbit is for viewing only; the character must face and move along its own yaw.
2. **W/S must map to world ±Z, not "away from camera."** In our first attempt, W was −Z; the user wanted it to be +Z. Treat W as forward along the character's current facing direction (driven by yaw), not a fixed axis.
3. **A and D do NOT strafe.** They **rotate the character in place**. Early attempts had A/D as lateral movement; the user explicitly corrected this twice. Final spec: A turns the character right, D turns the character left.
4. **The original A/D assignment (A=left, D=right) is swapped** to (A=right, D=left). Implement that final mapping from the start.
5. **A/D rotation must be continuous, not per-press.** First attempt used edge detection (one 30° snap per keydown). User wanted the rotation to repeat while the key is held. Use a per-second rate (`30 * dt` rad/s) so it scales with frame time.
6. **Vertical mouse drag must be inverted** (drag up = look down). The original sign produced the opposite feel; the user explicitly asked to swap it.
7. **Keep state variables minimal.** The `aPrev` / `dPrev` edge-detection flags are no longer needed after switching to continuous rotation — remove them rather than leaving dead code.
8. **HUD text must match the actual control scheme.** A stale "WASD — Move" label is misleading; describe what each key actually does.
9. **Use `THREE.MathUtils.degToRad`** for any degree value going into trig. Don't write `Math.PI / 6` and expect readers to know it's 30°.
10. **When the user repeatedly corrects the same control, read carefully across the whole session** — the *final* mapping is: W=+Z forward, S=−Z back, A=turn right (continuous), D=turn left (continuous), mouse drag = camera (with inverted vertical).

---

## Acceptance Checklist

Before declaring done, verify:
- [ ] Page opens directly in browser, no console errors.
- [ ] Character is clearly anime-styled (big eyes, spiky hair, ahoge, school uniform, blush).
- [ ] City has buildings, roads, sidewalks, sakura trees, torii gate, kanji signs, clouds, falling petals.
- [ ] All materials use `MeshToonMaterial` (cel-shaded look).
- [ ] W makes the boy walk forward in the direction he currently faces; S walks backward.
- [ ] A continuously rotates the boy to his right; D to his left — as long as the key is held.
- [ ] Camera does NOT follow the boy's facing; it stays where the user dragged it.
- [ ] Mouse drag rotates the camera; vertical drag is inverted (drag up = look down).
- [ ] Wheel zoom works within a clamped range.
- [ ] Walking/running animates arms, legs, and torso.
- [ ] Collision prevents passing through buildings.
- [ ] HUD lists the actual final controls.
- [ ] Single `index.html` file, no other deliverables required.
