P# Triplanar / Biplanar Terrain Rendering for Recoil Engine

**A feature proposal based on the [BAR-maps-triplanar](triplanar-terrain.html) prototype**

Author: ixstudios (Martijn Hoppenbrouwer)
Status: Proposal / discussion document
Target: Recoil Engine map renderer & shader pipeline

---

## TL;DR

A feature proposal to evolve Recoil's terrain rendering from single-projection planar splatting to a biplanar, layer-driven material system with hand-painted overrides, color tinting, BAR-shaped crater deformation, and surface-aligned impact decals. Below is the full feature list, with cost vs. Recoil's current ~10 taps/px baseline.

| § | Feature | What it gives | Cost |
|---|---|---|---|
| **2** | **Biplanar projection** (recommended core) | No more cliff-stretching. Top-2 axis sampling per pixel; triplanar (3 axes) is an opt-in quality mode. | Foundation; cost shows up in §3 |
| **3** | **Layered material compositing** | Up to N materials per map (sand/grass/cliff/snow/...) defined by slope+height bands, soft edges, organic FBM edge-noise. Replaces hand-painted splatmaps. | +4 taps per layer (biplanar) |
| **3.1** | Per-layer drop shadows + sun-facing highlight lips | Fake 3D ledge depth without geometry. | Free (math only) |
| **3.1** | Per-layer height + slope gradient tints | Up to 8 color stops per axis per layer. | Free |
| **3.2** | **Global paint mask** (RGBA, one texture) | One channel per layer; mappers can hand-override the procedural mask. Combines procedural ("looks natural everywhere") with painted ("but here exactly differently"). | +1 tap, *constant* — not per layer |
| **3.3** | **Color layers** | Re-tint composited albedo without committing a new material. Stack freely with material layers. | +0 taps (reuses §3.2 sample) |
| **3.4** | **DNTS-style diffuse-in-alpha** (carried from Recoil) | Free per-texel diffuse modulator from the normal map's alpha channel. Cracks/crevices darken naturally without a separate AO map. | +0 taps (alpha is in the same fetch) |
| **4** | **Damage mask + Recoil crater profile** | GPU-side signed depth mask (same Sombrero formula as `BasicMapDamage.cpp`). Vertex shader subtracts. Eased animation, dirty-rect uploads (~70-600× bandwidth saving). | +1 tap |
| **5** | **"Fixed" layers** | Layers flagged `fixed` read pre-deformation height/slope, so e.g. grass can be *eroded* by a crater but never *grow back* in a fresh crater floor. | Negligible (2 extra varyings) |
| **6** | **Surface-aligned crater decals** | Oriented projection (impact normal, not world-up) wraps decals onto cliffs and overhangs. **Additive to** Recoil's existing GL4 decal pass — no re-authoring of existing art. | +0..2 taps per visible decal |
| **7** | **Topographic contour overlay** | World-Y isolines with normal-tilted groove walls, slope/height-gated. Sun direction picks up groove shading. | Negligible |
| **8** | Save/load + slider-driven prototyping workflow | Mappers tune sliders in a hot-reload editor instead of painting splatmaps. | N/A (tooling) |
| **9** | Staged rollout in 7 phases | Stages 1+2+3 deliver the core visual win; everything after is incremental. | — |

**Performance summary** (§10, audited from `SMFFragProg.glsl`):
- Recoil today, typical BAR map: **~10 taps/px**
- Proposal, biplanar + 1 base + 4 layers, procedural only: **~25 taps/px (2.5×)**
- Proposal, same + paint mask + color layers: **~26 taps/px (2.6×)**
- Triplanar opt-in: ~35 taps/px (3.5×)
- Lean preset (1 base + 2 layers): ~17 taps/px — within Recoil's existing all-features-on ceiling

The proposal is more expensive than what Recoil renders today; the justification is the visual win (no cliff stretching, post-deformation re-binding, real per-material parameters) and the authoring win (slider-driven instead of splatmap-painted), not perf-neutrality.

---

## 1. Why this proposal

The current Recoil terrain pipeline composites map textures via a single planar projection (the heightmap's UV space). On steep cliff faces and overhangs this stretches the texels along the vertical axis and creates the well-known "smeared rock face" look that BAR maps work hard to hide with painted-in cliff layers and clever heightmap authoring.

The prototype in this repository ([triplanar-terrain.html](triplanar-terrain.html)) explores a different approach: project material textures from three (or two) world-space axes and blend by surface-normal weighting, then composite multiple BAR-style material layers on top using slope/height masks. The result is:

- No texture stretching on cliffs, regardless of slope angle.
- Material layers that automatically rebind themselves after terrain deformation (craters, lifts), without any per-blast retexturing logic.
- A shader-driven "material recipe" that lives entirely in the map definition rather than in baked diffuse textures, which keeps map files tiny and authoring tractable.

This document describes the prototype's feature set in concrete terms and proposes a path for adopting equivalent functionality in Recoil. The explicit recommendation is **biplanar projection (2 taps) as the default**, with optional triplanar (3 taps) as an opt-in quality mode — so the per-pixel texture-fetch budget rises by ~33% over the current single-projection path rather than the 200% a naive triplanar implementation would cost.

---

## 2. Recommended core: biplanar projection

### 2.1 Triplanar vs. biplanar — the cost/quality trade

A standard triplanar shader samples each material texture (color, normal, ORM/roughness…) from all three world-space planes (XY, XZ, YZ) and blends by the squared/powered components of the world-space normal. For an N-sampler material this costs **3 × N** texture reads per pixel.

Biplanar projection ([Inigo Quilez, 2020](https://iquilezles.org/articles/biplanar/)) observes that for any given normal the third axis contributes near-zero weight after the blend-sharpness power is applied, so it can be skipped entirely. Costs drop to **2 × N** per pixel. The visual difference is essentially imperceptible at any reasonable blend sharpness (≥ 4) and disappears completely at high sharpness.

| Mode | Taps per material (color + normal) | Notes |
|---|---|---|
| Single planar (current Recoil) | 2 | Stretches on cliffs |
| **Biplanar (proposed default)** | **4** | Pick top-2 axes per pixel |
| Triplanar (opt-in) | 6 | All 3 axes, slightly cleaner near 45° corners |

The prototype implements both ([triplanar-terrain.html:1623-1742](triplanar-terrain.html#L1623-L1742)) and toggles between them at shader-recompile time. The TAPS/PX HUD at the top of the prototype lets the user verify the cost in real time.

### 2.2 Biplanar implementation sketch (GLSL, lifted from the prototype)

The hot path is an unrolled major/medium-axis selection. WebGL1 forced the unrolled `if/else` chain in the prototype; in Recoil's GLSL 4.5+ context this can be compressed using `mix()` and component swaps, but the unrolled version is the easiest to read.

```glsl
vec3 biplanarSample(sampler2D map, vec3 wp, vec3 normal, float scale) {
    vec3 an = abs(normal);
    // Pick major (ma) and medium (me) axis — discard minor.
    // Sample those two planes, weight by |normal.axis|^sharpness, normalize.
    // (See triplanar-terrain.html:1631-1662 for the full unrolled form.)
}
```

For normal maps the prototype uses the [Bgolus "whiteout" blend](https://bgolus.medium.com/normal-mapping-for-a-triplanar-shader-10bf39dca05a) on the two selected axes ([triplanar-terrain.html:1689-1742](triplanar-terrain.html#L1689-L1742)). This preserves detail strength better than the older "RNM" or "swizzle" approaches, and matches well between bi- and triplanar modes.

### 2.3 Suggested Recoil integration

- A new map-shader path `mapshaders/terrain/biplanar.glsl` that replaces (or sits next to) the current planar terrain shader.
- A per-map opt-in flag in `mapinfo.lua` (e.g. `customparams.terrain_projection = "biplanar"`) so existing maps render unchanged.
- A global `/terrain projection 0|1|2` springsetting fallback for users who want to force-enable on legacy maps.

---

## 3. Layered material compositing

The prototype's biggest authoring win is not the projection itself — it's that material layers are **defined by slope-and-height bands instead of painted RGB texturesets**. A BAR-style "Colorado" map definition reduces to:

```
base:   sand
layer0: cliff   (slope ≥ 0.45)
layer1: grass   (height 0.10..0.70, slope ≤ 0.40)   FIXED
layer2: snow    (height ≥ 0.75, slope ≤ 0.35)
```

…and the shader paints each layer with soft band edges, organic edge noise, drop shadows and sun-facing highlight lips. Map authors stop hand-painting splatmaps; they tune sliders.

### 3.1 Per-layer parameter set (from the prototype)

Each non-base layer carries the following uniforms ([triplanar-terrain.html:1472-1505](triplanar-terrain.html#L1472-L1505)):

| Param | Purpose |
|---|---|
| `material` | Material library id (sand/grass/cliff/snow/rock…) |
| `hMin / hMax` | Normalized world-Y band where the layer is visible |
| `sMin / sMax` | Slope band (0 = flat, 1 = vertical) |
| `soft` | Smoothstep softness on band edges |
| `strength` | Final mask multiplier (0 = layer off, 1 = full coverage) |
| `brightness` | Per-layer brightness, applied after lighting tint |
| `thickness` + `highlightWidth` + `highlightSoft` | Sun-facing highlight lip (fake 3D ledge) |
| `shadowStrength` + `shadowDist` + `shadowBlur` | Drop-shadow on the underlying layer outside this layer |
| `edgeNoiseEnabled / Scale / Amount / Width` + 4 per-edge toggles | Organic FBM noise on band boundaries |
| `gradEnabled` + 8 `gradStops` | Optional per-layer height-gradient tint |
| `slopeGradEnabled` + 8 `slopeGradStops` | Optional per-layer slope-gradient tint |
| `fixed` | Layer reads pre-deformation height/slope — see §5 |
| `paintMaskChannel` (proposed) | Which channel of the global paint-mask texture this layer reads (R/G/B/A), or `none` — see §3.2 |
| `useDiffuseAlpha` (proposed, DNTS-style) | Reinterpret the layer normal map's alpha channel as a diffuse modulator — see §3.4 |

### 3.2 Global paint mask (proposed addition)

The slope/height/edge-noise mask system covers the *procedural* case very well, but there are always cases where map authors want explicit hand control: "no grass in this exact area", "concrete platform here", "the path through the canyon should be sand even though the slope band would normally pick up cliff". The prototype does not implement this yet, but the natural extension is:

- **One** RGBA paint-mask texture per map, sampled **once** per pixel by the terrain shader. Each channel maps to one auto-layer, addressed by the layer's `paintMaskChannel = "r" | "g" | "b" | "a"` field. So a single texture covers up to 4 layers; an 8-layer map needs a second texture.
- Sampled in the heightmap's UV space (top-down planar — same projection as Recoil's existing splat distribution texture, so authoring is identical to today's BAR splatmap workflow).
- Multiplied into each layer's procedural mask after edge-noise, before final `strength` scaling. So `finalMask = procMask * paintMaskChannel * strength`.
- A per-layer `paintMaskMode` flag controls whether the channel **multiplies** (default — the painted value can only *subtract* coverage from the procedural mask) or **adds** (the painted value can also *add* coverage outside the slope/height band).
- Layers with `paintMaskChannel = none` ignore the paint mask and run procedural-only.

Cost: **+1 tap total**, regardless of how many layers reference the mask. The single texture fetch happens once at the top of the fragment shader and the four channels are reused across all layer evaluations. Code-gen omits the sampler entirely when no layer declares a `paintMaskChannel`.

This combines the best of both worlds: procedural for "make it look natural everywhere" and painted for "but exactly this area must be different" — at constant cost.

### 3.3 Color layers (proposed addition)

Beyond material swaps, mappers often want to *tint* areas without committing a whole new material — "this region of the swamp should look greener", "the burn scars from a historic battle area read warmer than the rest of the grass". A natural extension:

- A new layer **type** alongside the material auto-layers: `colorLayer = { tint = {0.6, 0.8, 0.5}, strength = 0.6, paintMaskChannel = "g", … }`.
- Applied as a single pass over the composited albedo, **after** all material layers are evaluated — so a color layer re-tones whatever material happens to be underneath rather than competing for coverage. This matches how concept artists already think about terrain ("materials change the surface, color layers re-tone it").
- Carries the same slope/height/edge-noise band system as material layers, plus a `paintMaskChannel` reading from the same global paint-mask texture as §3.2 (so the mask texture's channels are shared between material layers and color layers — author whichever uses each channel).
- Instead of sampling new material textures, it just `mix()`es a color (or `multiply`s, switchable) into the composited albedo. Roughness/normal pass through untouched.
- **Zero extra texture taps** — the global paint mask is already sampled once for §3.2.

Multiple color layers can be stacked (e.g. one for swamp tint, one for burn scars), each one paying only the cost of its procedural mask math. The single global mask sample covers all of them.

### 3.4 DNTS-style diffuse-in-alpha (carried over from Recoil)

Recoil's existing `SMF_DETAIL_NORMAL_DIFFUSE_ALPHA` path (sometimes referred to as DNTS — "Detail Normal Texture Splatting" with the alpha repurposed) gets a free per-texel diffuse modulator out of the normal map's alpha channel without spending an extra sampler. Implementation lives in [`SMFFragProg.glsl:115-139`](https://github.com/beyond-all-reason/RecoilEngine/blob/master/cont/base/springcontent/shaders/GLSL/SMFFragProg.glsl#L115-L139), `GetSplatDetailTextureNormal`, with the gate set via `splatDetailNormalTex = { ..., alpha = true }` in `mapinfo.lua`.

This trick is genuinely valuable and **should carry forward into the per-layer system** rather than being abandoned with the splat-detail-normal path it currently lives in:

- Each material layer optionally opts in via a `useDiffuseAlpha = true` flag.
- The biplanar/triplanar normal-map sampler reads RGBA instead of RGB; the alpha channel becomes a per-texel diffuse modulator (typically a darken, e.g. `albedo *= mix(1.0, alpha, intensity)`), applied after albedo sampling but before lighting.
- **Zero extra samplers** — the alpha is already in the same texture fetch as the normal.
- Particularly useful for cliffs and rocks: cracks and crevices darken naturally without needing a separate AO map.

The only constraint is that the layer's normal map must be authored as RGBA (with intentional alpha content) instead of RGB. For BAR's existing material library this means an art-pass on the cliff/rock textures — but that work is already partly done because Recoil maps that use DNTS today already ship those textures.

### 3.5 Composite order

The prototype evaluates layers top-to-bottom over the base material ([triplanar-terrain.html:1812](triplanar-terrain.html#L1812)). Each layer computes a 2D mask (height-band × slope-band, both smoothstepped, both optionally edge-noise displaced) and `mix()`es its sampled albedo / normal / roughness on top.

For Recoil a sensible mapping:

- **Base material** = the map's existing diffuse fallback. Always rendered.
- **Up to N auto-layers** = additional materials defined per-map in `mapinfo.lua`, drawn in declaration order. Eight layers covers every prototype config in this repo; sixteen would be plenty for any plausible map.

### 3.6 Shader code-gen, not data-driven loops

A practical observation from prototyping: layer-evaluation as a `for` loop over uniforms is significantly slower than emitting one explicit code block per active layer at shader-compile time. The prototype generates GLSL with a JS template ([triplanar-terrain.html:1328-1466](triplanar-terrain.html#L1328-L1466)); Recoil should do the same on the C++ side, recompiling the terrain shader when the map's layer stack changes (i.e. once per map load).

The cost: one shader compile per map. The benefit: every per-pixel branch and uniform indirection collapses. Layers contributing zero weight at a given pixel still pay for the smoothstep math, but there are no sampler reads behind a dynamic branch — important on tile-based mobile and AMD GPUs that struggle with non-uniform texture access.

### 3.7 Suggested Recoil integration

- Extend `mapinfo.lua` with a `terrain.layers = { {…}, {…} }` table.
- Map loader reads the layer stack, generates the terrain fragment shader, compiles, caches.
- Existing single-texture maps fall through to a "1 layer = the diffuse" path with no behaviour change.

---

## 4. Per-blast crater + damage mask (already Recoil-shaped)

The prototype's crater system was specifically modelled on Recoil's `BasicMapDamage.cpp`, so the porting story here is the shortest one.

### 4.1 Crater profile

Identical to Recoil's "faked Sombrero hat, cut in half" formula ([BasicMapDamage.cpp](https://github.com/beyond-all-reason/RecoilEngine/blob/master/rts/Map/BasicMapDamage.cpp), prototype impl at [triplanar-terrain.html:3441-3461](triplanar-terrain.html#L3441-L3461)):

```
r in [0..1]   normalized radial distance
c1 = cos((r - 0.1) * (π + 0.3))
c2 = cos(max(0, 3r - 2) * π)
shape = c1 * (1 - r) * (0.5 + 0.5 * c2)
```

`shape` is positive in the inner bowl (sink), crosses zero around `r ≈ 0.57`, and goes negative on the rim (lift) before fading to zero at `r = 1`. This already matches what Recoil applies to heightmap samples in CPU code, so the engine gets to keep the exact same authoring "feel" for crater radius/depth tuning that BAR mappers are used to.

What the prototype adds on top:

- **`depthFalloff` exponent** — reshapes `r` before feeding the formula. `>1` keeps the centre full-depth longer (wider flat floor + steeper rim), `<1` softens it into a cone.
- **`rimHeight` slider** — scales only the *negative* (lift) band, so users can dial upthrust without affecting bowl depth.
- **Eased animation** — out-cubic over ~0.35 s for depth, ~0.55 s for scorch. Each frame writes only the *delta* against the prior frame's progress, so overlapping blasts compose correctly without re-stamping the full crater.

### 4.2 Damage mask layout

A single texture (in the prototype: 1024² RGBA Float32, ~16 MB) covers the full map ([triplanar-terrain.html:1065-1070](triplanar-terrain.html#L1065-L1070)):

| Channel | Purpose |
|---|---|
| `R` | Scorch intensity (0..1, max-blended across overlapping blasts) |
| `G` | Signed crater depth in `[-1..+1]`, scaled by `uDamageMaxDepth`. Vertex shader does `height -= maskG * uDamageMaxDepth`. |
| `B` | reserved (could carry per-pixel "moisture" or layer-erase factor) |
| `A` | reserved |

Float32 is required because the rim *lifts* terrain — an unsigned format would clamp the negative band to zero. `RG16F` would also work and halves the memory footprint to ~8 MB at 1024² (4 MB at 512²) without losing the signed range.

### 4.3 Dirty-rect uploads

Naively re-uploading the full mask each frame during a blast (~30 frames × 16 MB = 500 MB/s GPU bandwidth per explosion) is a non-starter. The prototype's `paintExplosionContribution` tracks an axis-aligned dirty rect per frame and `uploadDamageMaskDirtyRect` pushes only that sub-region via `glTexSubImage2D` ([triplanar-terrain.html:3500-3550](triplanar-terrain.html#L3500-L3550)). This reduces bandwidth by 70-600× depending on crater size.

For Recoil the equivalent is `glTextureSubImage2D` (DSA) or `glBindTexture` + `glTexSubImage2D` per dirty region. The prototype dirty-rects per blast and merges per frame; Recoil could go finer (per-blast individual sub-uploads) since GL4 has no significant bind cost.

### 4.4 Suggested Recoil integration

- `BasicMapDamage` stays as the CPU-side authority for *gameplay-relevant* heightmap state (pathing, LOS, weapon hit detection).
- A new `MapDamageVisualizer` (or extension of `BasicMapDamage`) writes the same per-blast contribution into the GPU damage-mask texture using the dirty-rect strategy above.
- The terrain shader subtracts `maskG * uDamageMaxDepth` from sampled heightmap values (and recomputes the analytic normal from the damaged height field — the prototype does this cross-derivative-style at [triplanar-terrain.html:1197-1212](triplanar-terrain.html#L1197-L1212)).

This decoupling means visuals and gameplay can run at different update rates if ever needed (e.g. visual smooths the deformation across multiple frames while gameplay sees the snapped result immediately).

---

## 5. "Fixed" layers — material erosion without regrowth

This is the feature that solves a long-standing BAR visual bug: when a crater is punched through a grassy hillside, the crater floor should be *exposed dirt/rock*, not a fresh grass disk because the crater bottom happens to fall in the grass layer's height/slope band.

The prototype's solution ([triplanar-terrain.html:1077-1079, 1781-1788](triplanar-terrain.html#L1077-L1079)):

1. Vertex shader computes **two** height values: `vHeight` (post-damage) and `vHeightOriginal` (pre-damage, i.e. the heightmap value before the crater subtract).
2. Same trick for the surface normal — `vWorldNormal` and `vWorldNormalOriginal`.
3. Layers with `fixed = true` evaluate their height/slope masks against the *original* values; non-fixed layers use the post-damage values.

Result: a "FIX"-flagged grass layer can only *lose* coverage where craters intersect its mask. It can never *gain* coverage in a freshly-dug crater floor, because as far as the grass mask is concerned the original ground there was always under-the-cliff or above-the-band.

For map authors the typical recipe ends up being:

- **base + cliff** = unfixed (so they always show where appropriate, including new crater walls)
- **grass + snow** = fixed (so they can be eroded but never appear in new craters)
- **damage / scorched-rock** = unfixed special-case overlay applied in the post-damage region

### 5.1 Suggested Recoil integration

- Extend each layer descriptor in `mapinfo.lua` with a `fixed = true|false` flag.
- Vertex shader emits both pre- and post-damage interpolants. Cost: 2 extra `varying float`s and 2 extra `varying vec3`s, plus a second cross-derivative normal calculation. Negligible.

---

## 6. Crater decals (additive to BAR's GL4 decal system)

**This feature is intended as an addition to Recoil's existing GL4 decal pass, not a replacement.** The current ground-decal system stays as the authoritative path for static and projected decals; the surface-aligned projection described here adds a new in-shader crater/impact decal type that solves the cliff-stretching and overhang-wrap problems specifically for blast impacts. Existing decal art works unchanged in the existing pass.

The prototype implements per-blast ground-crack decals composited inside the terrain shader ([triplanar-terrain.html:1550-1925](triplanar-terrain.html#L1550-L1925)). Channel layout deliberately matches BAR's existing GL4 decal system so the same source textures can feed both paths:

- `uDecalAlbedo`: RGB albedo (×2× tint), A = decal opacity mask
- `uDecalNormal`: RGB tangent-space normal, A = glow/emissive mask

### 6.1 Surface-aligned projection (not screen-space, not heightmap-UV)

Each decal carries an oriented tangent frame (`T`, `B`, derived `N = T × B`) anchored to the impact normal. World-space vertex positions are projected into the decal's local frame to derive UVs, with a `±0.7` slab fade along `N` so the decal wraps cleanly onto the impacted face but doesn't bleed onto unrelated terrain on the other side of a ridge.

A surface-normal-mismatch fade (`smoothstep(0.0, 0.5, max(dot(geoNormal, N), 0.0))`) handles the case where curved terrain wraps the projection volume — perpendicular pixels drop out gracefully, matching ones stay at full strength.

This solves two BAR-specific problems for blast impacts on non-flat terrain:

- Crater decals on cliffs no longer stretch (heightmap-UV projection smears them along the vertical axis).
- Crater decals on overhangs wrap correctly because the projection axis is the *impact* normal, not world-up.

### 6.2 In-shader composite, additive to the GL4 decal pass

The prototype puts the decal loop inside the terrain shader. For each pixel:

1. Loop over `MAX_DECALS` (compile-time constant; typically 64).
2. Per decal: 6 multiplies + 3 dots + a fast oriented-AABB reject. If the pixel is outside the decal's box, `continue` — no texture reads.
3. If inside: 2 texture reads (albedo + normal), composite.

This is roughly equivalent to a forward decal pass but eats no extra G-buffer bandwidth and doesn't require a deferred renderer to be in place. For Recoil this lives alongside the GL4 decal pass; the channel layout is identical, so the same source decal textures can be authored once and used by both systems.

The "older decals get recycled when the buffer fills" policy is a simple LRU on the decal array ([triplanar-terrain.html:1043-1049](triplanar-terrain.html#L1043-L1049)).

---

## 7. Topographic contour overlay

Optional, gameplay-aid visualization ([triplanar-terrain.html:1931-2000](triplanar-terrain.html#L1931-L2000)): renders subtle horizontal grooves at constant world-Y intervals, gated by configurable slope and height ranges.

Implementation is interesting because it doesn't just *darken* a band — it also tilts the per-pixel normal across the groove cross-section, so the existing NdotL lighting picks up the groove walls. This means contours on sun-facing slopes get a subtle highlight on the upper lip and shadow on the lower, while anti-sun slopes flip automatically. Much more readable than flat-color contour lines, and zero extra lighting passes.

For Recoil this is a natural fit for the existing `/contourLines` toggle, with the addition of slope/height filtering so high-altitude contours can be drawn at finer spacing than valley-floor ones.

---

## 8. Save/load and prototyping workflow

The prototype embeds the entire material setup, layer stack, lighting, water, topo, explosion sliders, and even the heightmap and skybox bytes into a single JSON file ([triplanar-terrain.html:5040](triplanar-terrain.html#L5040), example: [colorado-test46.json](colorado-test46.json)). Forty-eight saved configs in this repo were all produced via the in-browser editor.

For Recoil, the natural mapping is `mapinfo.lua` for the static definition plus a runtime debug overlay (toggle with e.g. `/terrain_editor`) that can hot-tweak slider values and dump the resulting Lua snippet to clipboard. The prototype's JSON schema can serve as the authoritative reference for what parameters need to be exposed.

---

## 9. Suggested rollout

A staged rollout that delivers user-visible value at every step and avoids a big-bang shader rewrite:

| Stage | Scope | Risk |
|---|---|---|
| **1. Biplanar projection** | New shader path, opt-in per map. Single base material, no layer stack. | Low — shader-only change, zero gameplay impact |
| **2. Damage mask + crater profile** | GPU mask synced from `BasicMapDamage`, dirty-rect uploads, vertex shader applies signed offset. Visual only — gameplay still uses CPU heightmap. | Low — additive, can be force-disabled |
| **3. Layer stack** | `mapinfo.lua` schema extension, code-gen terrain shader, soft band edges + edge noise. | Medium — touches map format, needs author docs |
| **4. Drop shadows + highlight lips + per-layer gradients** | Pure shader additions on top of stage 3. | Low |
| **5. Fixed layers** | Extra `varying`s in vertex shader, `fixed = true` flag on layer descriptors. | Low |
| **6. Surface-aligned crater decals** | Either replace the GL4 decal projection or add a new "ground impact" decal type. Same channel layout as today. | Medium — interacts with existing decal authoring |
| **7. Triplanar opt-in** | Recompile shader with the third-axis tap pair. For maps with extreme overhangs / arches. | Low |

Stages 1, 2, and 3 deliver the core visual win. Everything after is incremental polish.

---

## 10. Performance budget — and the elephant in the room

**This is the section where this proposal asks the most of the engine team.** The prototype's per-pixel texture-fetch budget is meaningfully higher than what Recoil currently spends on terrain, and any honest evaluation of this approach has to start there.

### 11.1 What Recoil's terrain shader spends today

Audited from `cont/base/springcontent/shaders/GLSL/SMFFragProg.glsl` (the only modern terrain fragment shader in the tree — no GL4 SMF variant exists; ground decals are a separate pass in `GroundDecalsFragProg.glsl`):

| Feature | Taps | Compile gate (`#ifdef`) | Source line |
|---|---|---|---|
| Diffuse / base color | 1 | always | 337 |
| World-space normal map | 1 | `SMF_ADV_SHADING` | 148, 295 |
| Splat detail-normals (4-channel + distribution) | 4 + 1 = **5** | `SMF_DETAIL_NORMAL_TEXTURE_SPLATTING` | 177, 183-186 |
| Shadow depth (hardware PCF, single `shadow2DProj`) | 1 | `HAVE_SHADOWS` | 371 |
| Shadow color | 1 | `HAVE_SHADOWS` | 370 |
| Specular | 1 | `SMF_SPECULAR_LIGHTING` | 405 |
| *Optional:* sky reflection cubemap + modulator | +2 | `SMF_SKY_REFLECTIONS` | 345-346 |
| *Optional:* blend-normals | +1 | `SMF_BLEND_NORMALS` | 301 |
| *Optional:* light emission | +1 | `SMF_LIGHT_EMISSION` | 395 |
| *Optional:* parallax (height + normal re-fetch) | +2 | `SMF_PARALLAX_MAPPING` | 125 |
| *Optional:* info-overlay (LOS / metal map / etc.) | +1 | `HAVE_INFOTEX` | 355 |

| Configuration | Total taps/px |
|---|---|
| Floor (no advanced shading, no shadows, no splatting) | **3** |
| **Typical BAR map** (`SMF_ADV_SHADING` + `HAVE_SHADOWS` + `SMF_SPECULAR_LIGHTING` + `SMF_DETAIL_NORMAL_TEXTURE_SPLATTING`) | **10** |
| Ceiling (everything on, including parallax + sky-reflect + emission + infotex) | **~18** |

Notable observations from the audit:

- **Recoil's terrain is strictly single-projection.** All UVs derive from `vertexWorldPos.xz` (top-down heightmap projection) — `diffuseTexCoords`, `normTexCoords = vertexWorldPos.xz * normalTexGen`, splat coords `vertexWorldPos.xzxz * splatTexScales`. No triplanar/biplanar anywhere. Cliffs and overhangs stretch the planar UV by design.
- **Shadow PCF is a single hardware tap.** `shadow2DProj` lets the driver do an internal 2×2 bilinear comparison; Recoil does no multi-tap software PCF on the terrain.
- **Splatting is the dominant cost** when enabled (5 of the 10 typical taps). It's also what the proposed layer system would replace.

Caveat: these counts are static analysis of `texture*()` call sites in the shader source, not RenderDoc-measured. The driver may collapse adjacent fetches in some cases.

### 11.2 Measured TAPS/PX in the prototype

Measured directly from the in-browser HUD on a representative scene (Colorado, 1 base + 4 auto-layers, biplanar):

| Source | Taps / pixel |
|---|---|
| Base material (color + normal, biplanar) | 4 |
| Auto-layer 1 (color + normal, biplanar) | 4 |
| Auto-layer 2 (color + normal, biplanar) | 4 |
| Auto-layer 3 (color + normal, biplanar) | 4 |
| Auto-layer 4 (color + normal, biplanar) | 4 |
| **Material subtotal** | **20** |
| Shadow PCF (prototype: 16-tap soft shadow) | +16 |
| Damage mask | +1 |
| Decals (visible decals only, 2 taps each) | +0..N |
| **Prototype total (typical view)** | **~37** |

The shadow PCF count is **wildly over-budget** for an engine context — it's a prototype artefact. Recoil's existing terrain shadow is a single hardware-PCF tap; that is the right target. The realistic terrain-shader cost the engine team should evaluate is therefore:

| Configuration | Material | +Global paint mask (§3.2) | +Color layers (§3.3) | Shadow | Damage | Decals | **Total** | vs. Recoil today |
|---|---|---|---|---|---|---|---|---|
| **Recoil today (typical BAR map, audited)** | **7** | — | — | **2** | 0 | (separate pass) | **~10** | baseline |
| Recoil today, ceiling (everything on) | up to 13 | — | — | 2 | 0 | (separate pass) | **~18** | 1.8× |
| **Proposal: biplanar, 1 base + 4 layers (procedural only)** | **20** | 0 | 0 | **2** | **1** | **0..2** | **~25** | **2.5×** |
| Proposal: same + global paint mask (any number of layers) | 20 | +1 | 0 | 2 | 1 | 0..2 | ~26 | 2.6× |
| Proposal: same + paint mask + N color layers | 20 | +1 | 0 | 2 | 1 | 0..2 | ~26 | 2.6× |
| Proposal: biplanar, 1 base + 2 layers (lean preset, procedural only) | 12 | 0 | 0 | 2 | 1 | 0..2 | ~17 | 1.7× |
| Proposal: triplanar opt-in, 1 base + 4 layers (procedural only) | 30 | 0 | 0 | 2 | 1 | 0..2 | ~35 | 3.5× |

So against the audited typical-BAR-map baseline of **10 taps/px**, the recommended biplanar + 4-layer config lands at **~25 taps/px (~2.5×)** procedural-only, or **~26 taps/px (~2.6×)** with the global paint-mask authoring path on — regardless of how many layers (and how many color layers) reference the mask. A leaner 2-layer preset stays close to Recoil's *ceiling* config today (~17 vs ~18). Triplanar is ~3.5× and should be reserved for high-end maps that explicitly opt in.

**DNTS-style diffuse-in-alpha (§3.4) costs zero extra taps** — it reuses the alpha channel of the layer's existing normal-map sample. Carrying it forward into the per-layer system is a free quality win.

The global paint mask and color layers are pay-once-when-used: a map that ships no `paintMaskChannel` references on any layer pays nothing for either feature (code-gen omits the sampler). When in use, the cost stays at +1 tap whether a single layer references the mask or all eight do.

These numbers do *not* include the separate ground-decal pass (`GroundDecalsFragProg.glsl`), nor any GBuffer/lighting/post-process taps Recoil performs around terrain — those stay constant, but they push the absolute frame cost higher than the relative ratio suggests.

### 11.3 Where the cost actually lands

A few honest qualifications on those numbers:

- **Layer masks don't gate texture reads at runtime.** The compile-time code-gen described in §3.6 still emits a sampler call for every layer; only layers that resolve to literal-zero contribution (e.g. a snow layer on a map below 0.4 normalized height) can be DCE'd by the GLSL compiler. In practice, in any visible terrain view at least 2-3 layers actively contribute, so **20 taps is the typical-case number for a 4-layer map, not just the worst case.**
- **Bandwidth, not just count.** 20 BC7 (or DXT5) texture reads per pixel at 4K covers ~33 GB/s of GPU bandwidth before any other shader work. That's well within the budget of mid-range desktop GPUs but eats into the headroom the rest of the frame relies on.
- **Layer count is the dominant lever.** Each additional auto-layer adds 4 taps under biplanar (or 6 under triplanar). A "premium" map running 6 layers under triplanar is **40 material taps/pixel** — at that point we are firmly in "art directors must justify each layer" territory.
- **Recoil's 5-tap splat path partially overlaps with this proposal.** A BAR map today already pays 5 taps for the splat detail-normals (4 channels + 1 distribution). The proposed system replaces that with explicit layer materials — so the *net* increase is closer to **+15 taps over a splat-enabled BAR map** (10 → 25), not +20 over the absolute floor.

### 11.4 Mitigations worth considering

If this cost increase is unacceptable as-is, options to bring it down include (in rough order of effort vs. payoff):

1. **Cap layer count per map.** Soft cap at 4 auto-layers in `mapinfo.lua` (matching the prototype baseline). Hard cap at 8. A 2-layer "lean" preset lands at ~17 taps/px — within the same envelope as Recoil's all-features-enabled ceiling today.
2. **Single normal map, not per-layer.** Materials share a normal map (or composite normal in a separate pass); per-layer texture reads drop from 4 to 2 under biplanar. Cost of 1 base + 4 layers becomes ~10 material taps instead of 20 — i.e. parity with Recoil's existing splat-normal cost. Quality loss is real but not catastrophic; most BAR materials don't have wildly different normal-map character anyway.
3. **Channel-packed material atlases.** Pack color + roughness + AO into a single RGBA8 sample, normal into a second. Today's prototype uses separate textures for clarity; an engine-quality build can collapse this. Recoil already does the equivalent for its splat distribution texture.
4. **Compute-pre-pass material ID + deferred composite.** A first pass writes material weights to a thin GBuffer; a second pass samples only the active materials per pixel. More engine work, but caps per-pixel taps to ~the number of *active* layers per pixel (typically 2-3) instead of *declared* layers per map.

LOD-based layer culling is **explicitly not on this list**: all declared layers must remain visible at all distances. Distant terrain that drops to a "cliff only" simplification breaks the visual consistency the layer system exists to provide, and would re-introduce the splatmap-vs-base-texture seams that BAR maps already work around today.

The prototype implements none of the mitigations above — they are concrete paths the engine team can take if (1)+(2) alone aren't enough headroom.

### 11.5 Bottom line

**The proposed system is more expensive than what Recoil renders today, and there is no honest way to spin that.** Against the audited typical-BAR-map baseline of 10 taps/px, the recommended biplanar + 4-layer config lands at ~25 taps/px (2.5×). With mitigation (1) (lean preset) plus (2) (shared normal map), that drops to ~12 taps/px — within ~20% of Recoil's current cost — at the price of some authoring flexibility.

The justification has to come from the visual win (no cliff stretching, real per-material parameters, post-deformation re-binding) and the authoring win (slider-driven instead of splatmap-painted), not from a perf-neutral story. The engine team should (a) measure on actual hardware before committing, and (b) treat the layer-count cap as a non-negotiable part of the design rather than a configuration knob.

---

## 11. Open questions for the engine team

1. **Map format.** Extending `mapinfo.lua` is the obvious path. Is there appetite for the layer-stack schema described in §3, or should this go through a separate `mapinfo_terrain.lua` or `customparams.terrain.*`?
2. **Damage-mask resolution.** 1024² is the prototype default. For BAR's largest maps (24×24, 32×32) this is too coarse — probably wants 2048² or a per-map scaling. Memory at 2048² × RG16F = 16 MB, very manageable.
3. **Existing GL4 decal system.** The surface-aligned projection from §6 is intended to **sit alongside** Recoil's current GL4 decal pass, not replace it. The existing system stays as the authoritative path for ground decals; the in-shader oriented projection is purely an additional crater/impact decal type for cases where wrap-onto-cliff behaviour matters. No re-authoring of existing decal art required.
4. **Shader compilation cost.** Stage 3 (layer code-gen) means a shader recompile on map load. Acceptable, but if the engine supports shader caches (SPIR-V or driver binary), this could be cached per-map.

---

## 12. References

- Prototype source: [triplanar-terrain.html](triplanar-terrain.html) (single file, no build step)
- Inigo Quilez, Biplanar mapping: <https://iquilezles.org/articles/biplanar/>
- Ben Golus, Normal mapping for a triplanar shader: <https://bgolus.medium.com/normal-mapping-for-a-triplanar-shader-10bf39dca05a>
- Recoil `BasicMapDamage.cpp`: <https://github.com/beyond-all-reason/RecoilEngine/blob/master/rts/Map/BasicMapDamage.cpp>
- Example saved configs (heightmap embedded): [colorado-test46.json](colorado-test46.json), [colorado-test48.json](colorado-test48.json)

---

*This document is a discussion starter, not a finished spec. Numbers and recommendations are based on a single prototype on one developer's machine; before committing to any implementation path, the engine team should validate the per-pixel costs on a representative range of target hardware.*
