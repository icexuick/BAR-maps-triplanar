# BAR Maps — Triplanar Terrain Prototype

A single-file three.js prototype for prototyping materials, shaders, triplanar/biplanar projection (TAPS / PX) and layered terrain compositing for [Beyond All Reason](https://www.beyondallreason.info/) map authoring.

Everything lives in [triplanar-terrain.html](triplanar-terrain.html) — open it in a modern browser and the editor runs locally, no build step.

## What it does

- Renders a heightmap-driven terrain with a custom WebGL shader that does triplanar (3 taps) or biplanar (2 taps) texture projection.
- Composites up to N material layers (sand, grass, cliff, snow, …) using per-layer height and slope band masks with soft falloffs and organic edge noise.
- Per-layer drop shadows and sun-facing highlight lips for a fake "thickness" / 3D ledge look.
- Click-to-explode craters with animated damage mask (R = scorch, G = crater depth) and a tiered debris burst: large boulders, medium chunks and small gravel with bounce physics and opacity fade-out.
- THREE.Water plane with reflections, configurable color tint, distortion, ripple size and flow speed, clipped to the terrain footprint. Off by default and lazy-built on first enable.
- Topographic contour line overlay with height/slope filtering.
- ACES Filmic tone mapping + sRGB output.
- Save / load full configurations as JSON.

## Controls

- **Left panel** — global terrain, water, topo and lighting controls.
- **Right panel** — per-layer materials, height/slope bands, softness, brightness, drop shadow distance/blur, highlight width/softness, edge noise.
- **Drag** to orbit, **scroll** to zoom, **right-drag** to pan.

## Heightmaps

Load any 8/16-bit grayscale PNG via the **Load Heightmap** button, or just **drag and drop** the PNG anywhere on the page. Non-square heightmaps are supported; the world plane stretches to match the image aspect ratio.

## Explosions & debris

Left-click on the terrain (while **Explosions** is enabled in the left panel) to punch a crater. The damage mask animates a scorch ring and crater depth over ~0.55 s, and each blast spawns a debris burst sampled from the current base material's colour texture — 5 large chunks, 15 medium, 30 small tetrahedra with non-uniform per-instance scale so no two look alike.

Debris chunks bounce with restitution + friction, tumble with angular drag, and fade out via per-instance alpha once settled. Launch velocity scales sub-linearly (`sqrt`) with crater radius so bigger blasts don't hurl chunks off-screen.

### Damage mask performance notes

The damage mask is a 1024²×RGBA×Float32 texture (16 MB). Naïvely re-uploading it every frame during a blast animation (~30 frames) would eat ~500 MB/s of GPU bandwidth per explosion. Instead, `paintExplosionContribution` tracks a dirty rectangle per frame and `uploadDamageMaskDirtyRect` pushes only that sub-region via a raw `gl.texSubImage2D` on a parked texture unit, saving/restoring WebGL state so Three.js' texture-bind cache stays consistent. This drops bandwidth by ~70–600× depending on crater size, and keeps the per-frame cost effectively flat across overlapping blasts.

## Status

Experimental prototype — APIs, shader code and JSON config schema can change at any time.
