# BAR Maps — Triplanar Terrain Prototype

A single-file three.js prototype for prototyping materials, shaders, triplanar/biplanar projection (TAPS / PX) and layered terrain compositing for [Beyond All Reason](https://www.beyondallreason.info/) map authoring.

Everything lives in [triplanar-terrain.html](triplanar-terrain.html) — open it in a modern browser and the editor runs locally, no build step.

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

- Renders a heightmap-driven terrain with a custom WebGL shader that does triplanar (3 taps) or biplanar (2 taps) texture projection.
- Composites up to N material layers (sand, grass, cliff, snow, …) using per-layer height and slope band masks with soft falloffs and organic edge noise.
- **Fixed layers** — toggle FIX on a layer card and that layer reads the *pre-crater* height and slope, so explosions can only erase its coverage, never grow new coverage in a fresh crater floor.
- Per-layer drop shadows and sun-facing highlight lips for a fake "thickness" / 3D ledge look.
- Click-to-explode craters with a real BAR/Recoil crater profile (steep bowl + raised rim), a damage mask (R = scorch, G = signed crater depth), and a tiered debris burst: large boulders, medium chunks and small gravel with bounce physics and opacity fade-out.
- BAR-style flash visual on every blast: a hot-white → orange → red fireball billboard, an expanding ground shockwave ring, and a dynamic terrain flash light that briefly illuminates the surrounding ground.
- THREE.Water plane with reflections, configurable color tint, distortion, ripple size and flow speed, clipped to the terrain footprint. Off by default and lazy-built on first enable.
- Topographic contour line overlay with height/slope filtering.
- ACES Filmic tone mapping + sRGB output.
- Save / load full configurations as JSON, including all explosion settings.

## Controls

- **Left panel** — global terrain, water, topo, lighting and explosion controls.
- **Right panel** — per-layer materials, height/slope bands, softness, brightness, drop shadow distance/blur, highlight width/softness, edge noise, and the **FIX** toggle.
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

### Damage mask performance notes

The damage mask is a 1024²×RGBA×Float32 texture (16 MB). Naïvely re-uploading it every frame during a blast animation (~30 frames) would eat ~500 MB/s of GPU bandwidth per explosion. Instead, `paintExplosionContribution` tracks a dirty rectangle per frame and `uploadDamageMaskDirtyRect` pushes only that sub-region via a raw `gl.texSubImage2D` on a parked texture unit, saving/restoring WebGL state so Three.js' texture-bind cache stays consistent. This drops bandwidth by ~70–600× depending on crater size, and keeps the per-frame cost effectively flat across overlapping blasts.

## Save / load

The **Save Config** button writes a JSON snapshot of everything: world size, layer stack (including each layer's `fixed` flag), shader tunables, lighting, topo overlay, water, per-material settings, all explosion sliders, and the actual heightmap/skybox image bytes embedded as data URLs. **Load Config** restores the full scene. The same JSON is auto-persisted to `localStorage` so refreshing the page brings back your last setup.

## Status

Experimental prototype — APIs, shader code and JSON config schema can change at any time.
