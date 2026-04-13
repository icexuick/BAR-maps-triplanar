# BAR Maps — Triplanar Terrain Prototype

A single-file three.js prototype for prototyping materials, shaders, triplanar/biplanar projection (TAPS / PX) and layered terrain compositing for [Beyond All Reason](https://www.beyondallreason.info/) map authoring.

Everything lives in [triplanar-terrain.html](triplanar-terrain.html) — open it in a modern browser and the editor runs locally, no build step.

## What it does

- Renders a heightmap-driven terrain with a custom WebGL shader that does triplanar (3 taps) or biplanar (2 taps) texture projection.
- Composites up to N material layers (sand, grass, cliff, snow, …) using per-layer height and slope band masks with soft falloffs and organic edge noise.
- Per-layer drop shadows and sun-facing highlight lips for a fake "thickness" / 3D ledge look.
- THREE.Water plane with reflections, configurable color tint, distortion, ripple size and flow speed, clipped to the terrain footprint.
- Topographic contour line overlay with height/slope filtering.
- ACES Filmic tone mapping + sRGB output.
- Save / load full configurations as JSON.

## Controls

- **Left panel** — global terrain, water, topo and lighting controls.
- **Right panel** — per-layer materials, height/slope bands, softness, brightness, drop shadow distance/blur, highlight width/softness, edge noise.
- **Drag** to orbit, **scroll** to zoom, **right-drag** to pan.

## Heightmaps

Load any 8/16-bit grayscale PNG via the **Load Heightmap** button. Non-square heightmaps are supported; the world plane stretches to match the image aspect ratio.

## Status

Experimental prototype — APIs, shader code and JSON config schema can change at any time.
