# BAR Maps — Triplanar Terrain Prototype

A single-file three.js prototype for prototyping materials, shaders, triplanar/biplanar projection (TAPS / PX) and layered terrain compositing for [Beyond All Reason](https://www.beyondallreason.info/) map authoring.

Everything lives in [triplanar-terrain.html](triplanar-terrain.html) — open it in a modern browser and the editor runs locally, no build step.

A companion design document, [RECOIL-ENGINE-PROPOSAL.md](RECOIL-ENGINE-PROPOSAL.md), distils the prototype into a concrete feature proposal for the [Recoil Engine](https://github.com/beyond-all-reason/RecoilEngine) terrain pipeline — biplanar projection, layered compositing, painted masks, color layers, BAR-shaped craters, fixed layers, surface-aligned crater decals, plus a survey of modern AAA terrain techniques (height-based blending, stochastic texturing, runtime virtual texturing, VRS) evaluated for top-down RTS use. Audited against Recoil's current `SMFFragProg.glsl` so every cost claim links back to a specific shader line.

## Running it

There is no build pipeline, no `npm install`, no bundler. Two options:

1. **Quickest** — clone the repo and double-click [triplanar-terrain.html](triplanar-terrain.html). Most browsers will run it straight off the filesystem (`file://`).
2. **Recommended** — serve the folder over a local HTTP server so heightmap / skybox / config drag-drops behave consistently across browsers:

   ```sh
   # any of these works, pick what you have
   python -m http.server 8000
   npx serve .
   php -S localhost:8000
   ```

   Then open `http://localhost:8000/triplanar-terrain.html`.

Three.js r128 is loaded from a CDN inside the HTML; no other dependencies. Tested on recent Chrome, Edge and Firefox.

## What it does

- Renders a heightmap-driven terrain with a custom WebGL shader. **Biplanar** (2-axis, 4 taps/material) is the default; toggle **TRIPLANAR ON** to fall back to the 3-axis variant for comparison. See [§2 of the proposal](RECOIL-ENGINE-PROPOSAL.md#2-recommended-core-biplanar-projection) for the rationale.
- Composites up to N material layers (sand, grass, cliff, snow, …) using per-layer height and slope band masks with soft falloffs and organic edge noise.
- **Fixed layers** — toggle FIX on a layer card and that layer reads the *pre-crater* height and slope, so explosions can only erase its coverage, never grow new coverage in a fresh crater floor.
- Per-layer drop shadows and sun-facing highlight lips for a fake "thickness" / 3D ledge look.
- **Height-based blending** (Frostbite-style) — a global slider that lets the sampled albedo's luma push layer mask boundaries crisply outward on bright texels and inward on dark ones. Zero extra texture taps.
- **Sharpen Blend** toggle (Primozic squared-weights tweak) — squares the projection weights post-pow for visibly sharper triplanar/biplanar transitions on cliffs at zero tap cost.
- **Hex Tile** per-material toggle (Mikkelsen stochastic texturing) — replaces visible texture repetition by sampling 3 randomly-offset copies on a triangular grid. 3× the per-axis tap count for materials that opt in.
- **Sediment / talus mask** — CPU-baked from the heightmap by sampling 8 ring neighbours and detecting "high neighbours above me", multiplied by `(1 - my own slope)`. Lives in the damage mask's B channel so it shares a sampler with scorch + crater depth (no MAX_TEXTURE_IMAGE_UNITS pressure). Visualised by a sand tint overlay; in production drives a sediment material layer or a paint-mask channel.
- Click-to-explode craters with a real BAR/Recoil crater profile (steep bowl + raised rim), a damage mask (R = scorch, G = signed crater depth, B = sediment), and a tiered debris burst: large boulders, medium chunks and small gravel with bounce physics and opacity fade-out.
- BAR-style flash visual on every blast: a hot-white → orange → red fireball billboard, an expanding ground shockwave ring, and a dynamic terrain flash light that briefly illuminates the surrounding ground.
- Surface-aligned crater decals (oriented projection from the impact normal) that wrap onto cliffs without stretching.
- THREE.Water plane with reflections, configurable color tint, distortion, ripple size and flow speed, clipped to the terrain footprint. Off by default and lazy-built on first enable.
- Topographic contour line overlay with height/slope filtering.
- ACES Filmic tone mapping + sRGB output.
- Save / load full configurations as JSON, including all explosion and sediment settings, plus the embedded heightmap and skybox bytes.

## Controls

- **Left panel** — global shader (texture scale, Blend Sharp, Height Blend), heightmap, lighting, topo lines, sediment, water, and explosions.
- **Right panel — Materials** — per-material tint, brightness, roughness, normal strength, specular, UV scale/rotation, and the **Hex Tile** stochastic-texturing toggle.
- **Right panel — Layers** — per-layer height/slope bands, softness, brightness, drop shadow distance/blur, highlight width/softness, edge noise, the **FIX** toggle, and per-layer height/slope gradient stops.
- **Top-left HUD** — TAPS/PX, ALU ops, varyings/uniforms, projection mode, and the **TRIPLANAR** + **SHARPEN BLEND** toggles.
- **Drag** to orbit, **scroll** to zoom, **right-drag** to pan.
- **Left-click** on the terrain (with Explosions enabled) to fire a blast at that point.

## Heightmaps

Load any 8/16-bit grayscale PNG via the **Load Heightmap** button, or just **drag and drop** the PNG anywhere on the page. Non-square heightmaps are supported; the world plane stretches to match the image aspect ratio.

## Explosions & debris

Left-click on the terrain to punch a crater. The crater profile is the same "faked Sombrero hat, cut in half" formula Recoil uses in [`BasicMapDamage.cpp`](https://github.com/beyond-all-reason/RecoilEngine/blob/master/rts/Map/BasicMapDamage.cpp) — a steep inner bowl that crosses zero around `r ≈ 0.57` and a raised rim peaking at `r ≈ 0.8` before fading flush with the surrounding terrain at `r = 1`. The damage mask G channel stores **signed** Float32 values so the rim band can lift the terrain via the same `height - maskG` subtract that the bowl uses to sink it.

Each blast also spawns:

- **5 large + 15 medium + 30 small** tetrahedral debris chunks, sampled from the current base material's colour texture, with non-uniform per-instance scale, restitution + friction bouncing, angular drag, and per-instance alpha fade once settled.
- A **fireball billboard** that snap-grows to ~2.4× the crater radius and fades over 0.32 s, colour-ramping from hot-white through orange to deep red.
- A **ground shockwave ring** that expands with an ease-out curve to ~1.35× the crater radius over 0.7 s.
- A **dynamic terrain flash light** that lights the surrounding ground out to ~2.66× crater radius for the duration of the blast. The terrain shader has its own `uFlashPos / uFlashColor / uFlashIntensity / uFlashRadius` uniforms because the custom shader doesn't pick up Three's built-in `PointLight` (only debris does).

Launch velocity scales sub-linearly (`sqrt`) with crater radius so bigger blasts don't hurl chunks off-screen.

### Explosion sliders (left panel)

- **Radius** — crater footprint (also drives debris speed, fireball size and shockwave size)
- **Depth** — peak bowl depth in world units
- **Depth falloff** — `>1` keeps the centre full-depth longer (wider flat floor + steeper rim); `<1` softens it into a cone
- **Rim height** — scales the lifted rim ring without touching bowl depth
- **Scorch size** — scorch radius as a fraction of crater radius
- **Scorch falloff** — exponent on the scorch falloff curve
- **Scorch spikes** — 0 = round scorch, 1 = strong radial rays/spikes (the crater itself stays round either way; only the R-channel scorch gets the blast-shape treatment, with each blast getting its own random ray seed)

### Damage mask layout and performance notes

The damage mask is a 1024²×RGBA×Float32 texture (16 MB). Channel layout:

- **R** — scorch / explosion overlay strength (max-blended across overlapping blasts)
- **G** — signed crater depth in `[-1, +1]`, scaled by `uDamageMaxDepth`. Vertex shader subtracts.
- **B** — sediment / talus mask (CPU-baked from heightmap, single tap shared with R + G)
- **A** — unused

Naïvely re-uploading the full mask every frame during a blast animation (~30 frames) would eat ~500 MB/s of GPU bandwidth per explosion. Instead, `paintExplosionContribution` tracks a dirty rectangle per frame and `uploadDamageMaskDirtyRect` pushes only that sub-region via a raw `gl.texSubImage2D` on a parked texture unit, saving/restoring WebGL state so Three.js' texture-bind cache stays consistent. This drops bandwidth by ~70–600× depending on crater size, and keeps the per-frame cost effectively flat across overlapping blasts.

Sediment writes go through a full-texture upload via `damageMaskTex.needsUpdate = true` because they're authoring-time edits (Radius/Threshold/Slope Fade slider tweaks), not per-frame deltas. Crater painting only writes R + G, so the baked sediment B channel survives all explosions; only the **Reset Explosions** button or a heightmap reload triggers a re-bake.

## Save / load

The **Save Config** button writes a JSON snapshot of everything: world size, layer stack (including each layer's `fixed` flag), shader tunables (texScale, blendSharp, heightBlend, biplanar/sharpenBlend toggles), lighting, topo overlay, water, sediment config, per-material settings (including the `hexTile` flag), all explosion sliders, and the actual heightmap/skybox image bytes embedded as data URLs. **Load Config** restores the full scene. The same JSON is auto-persisted to `localStorage` so refreshing the page brings back your last setup.

## See also

- [RECOIL-ENGINE-PROPOSAL.md](RECOIL-ENGINE-PROPOSAL.md) — the engine-side design document this prototype was built to back.
- Recoil Engine source: <https://github.com/beyond-all-reason/RecoilEngine>
- Beyond All Reason: <https://www.beyondallreason.info/>

## Status

Experimental prototype — APIs, shader code and JSON config schema can change at any time.
