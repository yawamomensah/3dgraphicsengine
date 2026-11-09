# Real Time Software Based 3D Graphics Engine

A fixed-function 3D graphics engine built from scratch in C, driven by a specific question: **how does classical 3D graphics mathematics actually execute on a CPU?**

<img src="assets/renderer-annotated.png" alt="CPU software renderer — multi-mesh scene with perspective-correct texturing and wireframe overlay" width="850">
Every stage of the rendering pipeline is implemented by hand — no graphics API, no hardware acceleration. The CPU does all of it: transformation, clipping, rasterization, depth testing, and texturing.

---

## Demo

<div style="display: flex; flex-wrap: wrap; justify-content: center; gap: 10px;">
  <img src="assets/wireframe-plane.gif" alt="Wireframe of plane model">
  <img src="assets/wireframe-crab.gif" alt="Wireframe of crab model">
  <img src="assets/flat-plane.gif" alt="Plane model in flat shading mode">
  <img src="assets/flat-crab.gif" alt="Crab model in flat shading mode">
  <img src="assets/textured-plane.gif" alt="Plane model in textured mode">
  <img src="assets/textured-crab.gif" alt="Crab model in textured mode">
</div>

---

## What Was Built

A complete CPU-based rendering pipeline capable of loading and rendering multiple textured OBJ meshes in a live interactive scene, with switchable rendering modes and real-time camera movement.

| Module | Responsibility |
|---|---|
| `matrix.c` | 4×4 matrix construction and multiplication; perspective projection; look-at view matrix |
| `triangle.c` | Scanline rasterization; barycentric coordinate interpolation; perspective-correct texture sampling; Z-buffer depth testing |
| `clipping.c` | Sutherland-Hodgman polygon clipping against six view frustum planes in camera space |
| `camera.c` | First-person camera with pitch/yaw rotation and delta-time movement |
| `mesh.c` | OBJ file parsing; multi-mesh scene management |
| `light.c` | Directional lighting via surface normal and light direction dot product |
| `display.c` | SDL2 framebuffer management; pixel and primitive drawing |

---

## The Intellectual Arc

The central question this project answers is not "how do I use a graphics API" but "what is a graphics API actually doing." Every matrix multiply, every barycentric weight, every perspective divide, every depth comparison is executed by hand on the CPU.

What became clear building each stage is that 3D graphics is fundamentally a problem of coordinate system transitions. A vertex begins in model space, moves into world space via the world matrix, then into camera space through the view transform, and finally into clip space via the projection matrix.

This is not something you understand by reading about it. It becomes visible when you implement it.

---

## Rendering Pipeline

```
OBJ file → vertex data
    |
    ↓
World transform       ← scale, rotation, translation matrices composed
    |
    ↓
Camera transform      ← look-at view matrix; world → camera space
    |
    ↓
Backface culling      ← dot product of face normal and camera ray;
    |                    discard faces pointing away from the camera
    ↓
Frustum clipping      ← Sutherland-Hodgman against 6 planes in camera space;
    |                    polygon → retriangulated output triangles
    ↓
Perspective projection ← perspective matrix; perspective divide by w;
    |                     NDC → screen space
    ↓
Rasterization         ← scanline traversal; barycentric weights per pixel
    |
    ↓
Depth test            ← Z-buffer comparison per pixel
    |
    ↓
Texture sampling      ← perspective-correct UV interpolation
    |
    ↓
Framebuffer           ← SDL2 render to screen
```

---

## Key Technical Decisions

### Depth Sorting: From Painter's Algorithm to Z-Buffering

The first working version of the renderer used the painter's algorithm for depth ordering — sorting triangles back-to-front by their average Z depth before drawing them. It works, up to a point.

The pivot to Z-buffering resolved both problems. Rather than sorting triangles, a depth buffer — a flat array of floats matching the framebuffer dimensions — stores the depth value of the closest drawn pixel per screen location.

The depth value stored is `1/w` rather than raw Z — a consequence of the perspective projection. After the perspective divide, linear interpolation of Z across a triangle in screen space is not enough to maintain correct visibility across perspective distortion.

### Memory Layout and Cache Behaviour

Both the framebuffer and the Z-buffer are flat 1D arrays indexed as `y * width + x`. This is not incidental — it is the layout that matches the rasterizer's access pattern and the way CPUs load memory.

CPUs do not read from RAM one byte at a time. They load memory in 64-byte chunks called cache lines. When a pixel is written, the CPU loads that pixel and the 15 adjacent pixels into cache simultaneously.

The hardware prefetcher compounds this further. Modern CPUs watch memory access patterns and speculatively load data into cache before it is requested. The inner rasterization loop — `for (int ...` — follows row-major memory access, which the prefetcher recognizes.

This is also why the rasterizer decomposes triangles into horizontal scanlines rather than traversing vertically. Screen pixels are stored row-major — rows are contiguous in memory. A vertical traversal would repeatedly hop across rows, reducing cache locality and blowing out the CPU cache.

The Z-buffer is indexed identically to the framebuffer, so the depth read and write that accompany every pixel write are sequential and prefetchable by the same mechanism. The two hottest arrays stay cache-friendly across the render loop.

### Perspective-Correct Texture Mapping

Naive texture mapping interpolates UV coordinates linearly across a triangle in screen space. This produces the characteristic warping visible on textured surfaces in motion — a well-known artifact of incorrect interpolation.

The correct approach interpolates `u/w` and `v/w` — the UV coordinates divided by the clip-space w value at each vertex — and then recovers the true UV at each pixel by dividing by the interpolated `1/w` term.

Implemented in `draw_texel`: barycentric weights are computed for the current pixel, `u/w`, `v/w`, and `1/w` are interpolated using those weights, and the recovered UV coordinates are used to index the texture memory.

### Backface Culling

For any closed mesh, approximately half of all triangles face away from the camera at any given moment and will be completely occluded by front-facing geometry. Computing, clipping, projecting, and rasterizing them all is wasted work.

Backface culling eliminates them early: the face normal is computed in camera space via the cross product of two triangle edges, and the dot product of that normal with the camera ray determines whether the face points toward or away from the viewer.

### View Frustum Clipping

Triangles that straddle the boundary of the view frustum — partially inside, partially outside — cannot simply be discarded or projected as-is. Discarding them leaves visible gaps at screen edges, and projecting them without clipping creates geometry outside the visible region.

Sutherland-Hodgman clipping runs in camera space, before projection, against all six frustum planes. A triangle entering the clipper may exit as a polygon with up to nine vertices; that polygon is then retriangulated before rasterization.

### The Look-At View Matrix

The view matrix transforms world-space coordinates into camera-relative coordinates, making the camera the origin of the scene for all subsequent calculations. It is constructed by computing three orthonormal axes — forward, right, and up — from the camera's target and position.

---

## Mathematics at the Core

Every rendering operation reduces to a small set of linear algebra primitives:

- **Matrix-vector multiplication** — applies every spatial transformation: scale, rotation, translation, projection
- **Cross product** — computes surface normals for lighting and culling; constructs the orthonormal camera basis in the look-at matrix
- **Dot product** — measures alignment between vectors; drives both backface culling (face normal vs camera ray) and directional lighting (face normal vs light direction)
- **Barycentric coordinates** — express any point inside a triangle as a weighted sum of its vertices; the weights drive correct per-pixel interpolation of any vertex attribute (UV coordinates, depth, and lighting values)
- **Perspective divide** — the w-division that maps clip space to NDC; the source of all perspective-correctness requirements downstream

---

## Installation

**Dependencies**: GCC, Make, SDL2

On Ubuntu/Debian:
```bash
sudo apt install gcc make libsdl2-dev
```

On macOS (Homebrew):
```bash
brew install gcc make sdl2
```

**Build and run**:
```bash
git clone https://github.com/yawamomensah/3d-software-renderer.git
cd 3d-software-renderer
make
make run
```

---

## Camera Controls

| Input | Action |
|---|---|
| `W` / `S` | Pitch camera up / down |
| `←` / `→` | Yaw camera left / right |
| `↑` / `↓` | Move forward / backward |
| `1` | Wireframe with vertex points |
| `2` | Wireframe only |
| `3` | Solid fill |
| `4` | Solid fill with wireframe overlay |
| `5` | Textured |
| `6` | Textured with wireframe overlay |
| `C` | Enable backface culling |
| `X` | Disable culling |
| `ESC` | Exit |

---

## References

- Scratchapixel — *The Perspective Projection Matrix* and *Rasterization: a Practical Implementation*
- Mathematical Structures for Computer Graphics - Janke
