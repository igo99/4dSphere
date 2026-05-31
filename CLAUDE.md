# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the project

No build step. Open `index.html` directly in a browser:

```
open index.html
```

Three.js is loaded via CDN importmap, so an internet connection is required on first load (the browser caches it after that). There is no package.json, no bundler, and no server needed.

## What this is

A mathematical visualisation of **S³**, the 4-dimensional unit sphere `{x²+y²+z²+w²=1} ⊂ ℝ⁴`. Because we can't see 4D directly, the code projects S³ into 3D using stereographic projection and renders the result with Three.js. Two modes exist:

- **Hopf fibration** — decomposes S³ into a family of circles (great circles) over S². Every pair of circles is topologically linked exactly once.
- **Cross-section sweep** — slices S³ with the hyperplane `w = const`. Each slice is a 2-sphere of radius `√(1−w²)`. Sweeping w from 0 to ±1 stacks these shrinking spheres into a solid 3-ball; S³ is two such balls glued along their shared boundary sphere (at w=0).

## Architecture

Everything lives in a single `index.html` file: HTML structure, CSS, and one ES module `<script>`. The file has six logical sections marked with comments.

### 1. Pure math

Four self-contained functions with no Three.js dependency:

| Function | What it does |
|---|---|
| `rot4(v, plane, a)` | Rotates a 4D vector `v` by angle `a` in one of the three planes `xw`, `yw`, `zw`. Each plane mixes two coordinates and leaves the other two fixed. |
| `stereo([x,y,z,w])` | Stereographic projection S³→ℝ³. Draws a ray from the north pole `(0,0,0,1)` through the point and finds where it crosses `w=0`. Formula: `(X,Y,Z) = (x,y,z)/(1−w)`. Points near `w=1` project to large radii; the code caps this at radius 7. |
| `hopfFiber(theta, phi, nSeg)` | Returns `nSeg+1` points forming one Hopf fiber — the preimage of the point `(sinθ cosφ, sinθ sinφ, cosθ)` on S² under the Hopf map. The fiber is parametrised by `ψ ∈ [0,2π]`. |
| `fibSphere(n)` | Returns `n` evenly-spaced `{theta, phi}` points on S² using the Fibonacci spiral. Omits the near-south-pole point whose fiber maps to infinity under stereographic projection. |

### 2. Three.js setup

Standard boilerplate: `WebGLRenderer`, `PerspectiveCamera`, `OrbitControls`. The canvas is positioned with `left: 210px` to leave room for the panel. `resize()` keeps the camera aspect ratio correct.

### 3. Scene objects

Two `THREE.Group`s sit in the scene; exactly one is visible at a time depending on mode:

- **`fiberGroup`** — contains one `THREE.Line` per Hopf fiber. Each line's geometry has a pre-allocated `Float32Array` position buffer.
- **`sliceGroup`** — contains two `THREE.Mesh` objects: `ghostMesh` (full unit sphere wireframe, always dim) and `sliceMesh` (scaled each frame to the current slice radius).

### 4. State

Two plain objects hold all mutable state:

- **`S`** — Hopf mode state: fiber count, segment count, the three 4D rotation angles, animation angle accumulator, speed, active rotation plane, and current mode string.
- **`CS`** — Slice mode state: current `w` value, the angle accumulator (`w = sin(angle)`), speed, and animating flag.

### 5. Geometry pipeline

**Hopf mode** has two levels of update:

- `rebuildFibers()` — called when fiber count or segment count changes. Clears `fiberGroup`, allocates fresh `BufferGeometry` and `LineBasicMaterial` for each fiber (color computed once from hue/lightness), then calls `redrawFibers()`.
- `redrawFibers()` — called every animated frame. Loops over existing line objects, recomputes each point: `hopfFiber → rot4 (×3) → stereo → pos.setXYZ`. Marks `needsUpdate = true`. Never allocates.

The animation angle (`S.animAngle`) is added on top of the manual rotation sliders, so sliders and animation are independent.

**Slice mode** uses `updateSlice()`:
- Computes `r = √(1−w²)` and calls `sliceMesh.scale.setScalar(r)` — avoids rebuilding geometry.
- Sets hue: orange at `w=−1`, blue at `w=+1`.
- Syncs the w slider position and all text labels.

The animation drives `CS.angle` forward each frame; `CS.w = sin(CS.angle)` gives a smooth oscillation between −1 and +1 automatically.

### 6. Mode switching

`setMode('hopf' | 'slice')` toggles group visibility, panel section visibility, overlay text, and repositions the camera (Hopf fibers extend further so the camera starts further back).

### UI binding pattern

`bindRange(id, valId, key, obj, deg, rebuild)` wires a `<input type=range>` to a state key in `obj`. The `deg` flag converts degrees to radians. The `rebuild` flag chooses between `rebuildFibers()` (expensive, reallocates) and `redrawFibers()` (cheap, updates buffers only). It also calls `trackRange()` to update the CSS custom property `--p` that drives the filled-track gradient.

## Adding new visualisation modes

The pattern for a new mode:
1. Add a `THREE.Group` to the scene, `visible: false`.
2. Add a state object.
3. Write an `update*()` function that only mutates buffer positions or mesh scales — no allocation per frame.
4. Add a panel section `<div id="foo-ctrl" style="display:none">`.
5. Extend `setMode()` to show/hide the group and panel section.
6. Add a branch in the render loop.
