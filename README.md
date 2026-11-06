1| # Real Time Software Based 3D Graphics Engine
2| 
3| A fixed-function 3D graphics engine built from scratch in C, driven by a specific question: **how does classical 3D graphics mathematics actually execute on a CPU?**
4| 
5| <img src="assets/renderer-annotated.png" alt="CPU software renderer — multi-mesh scene with perspective-correct texturing and wireframe overlay" width="850">
6| Every stage of the rendering pipeline is implemented by hand — no graphics API, no hardware acceleration. The CPU does all of it: transformation, clipping, rasterization, depth testing, and texturing.
7| 
8| ---
9| 
10| ## Demo
11| 
12| <div style="display: flex; flex-wrap: wrap; justify-content: center; gap: 10px;">
13|   <img src="assets/renderer-demo-wireframe.gif" alt="Wireframe renderer demo" width="320">
14|   <img src="assets/renderer-demo-wireframe-2.gif" alt="Wireframe rotation demo" width="320">
15|   <img src="assets/renderer-demo-flat-shaded.gif" alt="Flat shaded demo" width="320">
16|   <img src="assets/renderer-demo-flat-shaded-2.gif" alt="Flat shaded lighting demo" width="320">
17|   <img src="assets/renderer-demo-textured.gif" alt="Textured engine demo" width="320">
18|   <img src="assets/renderer-demo-textured-2.gif" alt="Textured overlay demo" width="320">
19| </div>
20| 
21| ---
22| 
23| ## What Was Built
24| 
25| A complete CPU-based rendering pipeline capable of loading and rendering multiple textured OBJ meshes in a live interactive scene, with switchable rendering modes and real-time camera movement.
26| 
27| | Module | Responsibility |
28| |---|---|
29| | `matrix.c` | 4×4 matrix construction and multiplication; perspective projection; look-at view matrix |
30| | `triangle.c` | Scanline rasterization; barycentric coordinate interpolation; perspective-correct texture sampling; Z-buffer depth testing |
31| | `clipping.c` | Sutherland-Hodgman polygon clipping against six view frustum planes in camera space |
32| | `camera.c` | First-person camera with pitch/yaw rotation and delta-time movement |
33| | `mesh.c` | OBJ file parsing; multi-mesh scene management |
34| | `light.c` | Directional lighting via surface normal and light direction dot product |
35| | `display.c` | SDL2 framebuffer management; pixel and primitive drawing |
36| 
37| ---
38| 
39| ## The Intellectual Arc
40| 
41| The central question this project answers is not "how do I use a graphics API" but "what is a graphics API actually doing." Every matrix multiply, every barycentric weight, every perspective divide, and every depth comparison is executed by hand on the CPU.
42| 
43| What became clear building each stage is that 3D graphics is fundamentally a problem of coordinate system transitions. A vertex begins in model space, moves into world space via the world matrix, then into camera space through the view transform, and finally into clip space via the projection matrix.
44| 
45| This is not something you understand by reading about it. It becomes visible when you implement it.
46| 
47| ---
48| 
49| ## Rendering Pipeline
50| 
51| ```
52| OBJ file → vertex data
53|     |
54|     ↓
55| World transform       ← scale, rotation, translation matrices composed
56|     |
57|     ↓
58| Camera transform      ← look-at view matrix; world → camera space
59|     |
60|     ↓
61| Backface culling      ← dot product of face normal and camera ray;
62|     |                    discard faces pointing away from the camera
63|     ↓
64| Frustum clipping      ← Sutherland-Hodgman against 6 planes in camera space;
65|     |                    polygon → retriangulated output triangles
66|     ↓
67| Perspective projection ← perspective matrix; perspective divide by w;
68|     |                     NDC → screen space
69|     ↓
70| Rasterization         ← scanline traversal; barycentric weights per pixel
71|     |
72|     ↓
73| Depth test            ← Z-buffer comparison per pixel
74|     |
75|     ↓
76| Texture sampling      ← perspective-correct UV interpolation
77|     |
78|     ↓
79| Framebuffer           ← SDL2 render to screen
80| ```
81| 
82| ---
83| 
84| ## Key Technical Decisions
85| 
86| ### Depth Sorting: From Painter's Algorithm to Z-Buffering
87| 
88| The first working version of the renderer used the painter's algorithm for depth ordering — sorting triangles back-to-front by their average Z depth before drawing them. It works, up to a point.
89| 
90| The pivot to Z-buffering resolved both problems. Rather than sorting triangles, a depth buffer — a flat array of floats matching the framebuffer dimensions — stores the depth value of the closest drawn pixel per screen location.
91| 
92| The depth value stored is `1/w` rather than raw Z — a consequence of the perspective projection. After the perspective divide, linear interpolation of Z across a triangle in screen space is not enough to maintain correct visibility across perspective distortion.
93| 
94| ### Memory Layout and Cache Behaviour
95| 
96| Both the framebuffer and the Z-buffer are flat 1D arrays indexed as `y * width + x`. This is not incidental — it is the layout that matches the rasterizer's access pattern and the way CPUs load memory.
97| 
98| CPUs do not read from RAM one byte at a time. They load memory in 64-byte chunks called cache lines. When a pixel is written, the CPU loads that pixel and the 15 adjacent pixels into cache simultaneously.
99| 
100| The hardware prefetcher compounds this further. Modern CPUs watch memory access patterns and speculatively load data into cache before it is requested. The inner rasterization loop — `for (int y = ... )` — follows row-major memory access, which the prefetcher recognizes.
101| 
102| This is also why the rasterizer decomposes triangles into horizontal scanlines rather than traversing vertically. Screen pixels are stored row-major — rows are contiguous in memory. A vertical traversal would repeatedly hop across rows, reducing cache locality and blowing out the CPU cache.
103| 
104| The Z-buffer is indexed identically to the framebuffer, so the depth read and write that accompany every pixel write are sequential and prefetchable by the same mechanism. The two hottest arrays stay cache-friendly across the render loop.
105| 
106| ### Perspective-Correct Texture Mapping
107| 
108| Naive texture mapping interpolates UV coordinates linearly across a triangle in screen space. This produces the characteristic warping visible on textured surfaces in motion — a well-known artifact of incorrect interpolation.
109| 
110| The correct approach interpolates `u/w` and `v/w` — the UV coordinates divided by the clip-space w value at each vertex — and then recovers the true UV at each pixel by dividing by the interpolated `1/w` term.
111| 
112| Implemented in `draw_texel`: barycentric weights are computed for the current pixel, `u/w`, `v/w`, and `1/w` are interpolated using those weights, and the recovered UV coordinates are used to index the texture memory.
113| 
114| ### Backface Culling
115| 
116| For any closed mesh, approximately half of all triangles face away from the camera at any given moment and will be completely occluded by front-facing geometry. Computing, clipping, projecting, and rasterizing them all is wasted work.
117| 
118| Backface culling eliminates them early: the face normal is computed in camera space via the cross product of two triangle edges, and the dot product of that normal with the camera ray determines whether the face points toward or away from the viewer.
119| 
120| ### View Frustum Clipping
121| 
122| Triangles that straddle the boundary of the view frustum — partially inside, partially outside — cannot simply be discarded or projected as-is. Discarding them leaves visible gaps at screen edges, and projecting them without clipping creates geometry outside the visible region.
123| 
124| Sutherland-Hodgman clipping runs in camera space, before projection, against all six frustum planes. A triangle entering the clipper may exit as a polygon with up to nine vertices; that polygon is then retriangulated before rasterization.
125| 
126| ### The Look-At View Matrix
127| 
128| The view matrix transforms world-space coordinates into camera-relative coordinates, making the camera the origin of the scene for all subsequent calculations. It is constructed by computing three orthonormal axes — forward, right, and up — from the camera's target and position.
129| 
130| ---
131| 
132| ## Mathematics at the Core
133| 
134| Every rendering operation reduces to a small set of linear algebra primitives:
135| 
136| - **Matrix-vector multiplication** — applies every spatial transformation: scale, rotation, translation, projection
137| - **Cross product** — computes surface normals for lighting and culling; constructs the orthonormal camera basis in the look-at matrix
138| - **Dot product** — measures alignment between vectors; drives both backface culling (face normal vs camera ray) and directional lighting (face normal vs light direction)
| - **Barycentric coordinates** — express any point inside a triangle as a weighted sum of its vertices; the weights drive correct per-pixel interpolation of any vertex attribute (UV coordinates, depth, and lighting values)
139| - **Perspective divide** — the w-division that maps clip space to NDC; the source of all perspective-correctness requirements downstream
140| 
141| ---
142| 
143| ## Installation
144| 
145| **Dependencies**: GCC, Make, SDL2
146| 
147| On Ubuntu/Debian:
148| ```bash
149| sudo apt install gcc make libsdl2-dev
150| ```
151| 
152| On macOS (Homebrew):
153| ```bash
154| brew install gcc make sdl2
155| ```
156| 
157| **Build and run**:
158| ```bash
159| git clone https://github.com/yawamomensah/3d-software-renderer.git
160| cd 3d-software-renderer
161| make
162| make run
163| ```
164| 
165| ---
166| 
167| ## Camera Controls
168| 
169| | Input | Action |
170| |---|---|
171| | `W` / `S` | Pitch camera up / down |
172| | `←` / `→` | Yaw camera left / right |
173| | `↑` / `↓` | Move forward / backward |
| | `1` | Wireframe with vertex points |
173| | `2` | Wireframe only |
174| | `3` | Solid fill |
175| | `4` | Solid fill with wireframe overlay |
176| | `5` | Textured |
177| | `6` | Textured with wireframe overlay |
178| | `C` | Enable backface culling |
179| | `X` | Disable culling |
180| | `ESC` | Exit |
181| 
182| ---
183| 
184| ## References
185| 
186| - Scratchapixel — *The Perspective Projection Matrix* and *Rasterization: a Practical Implementation*
187| - Mathematical Structures for Computer Graphics - Janke
188| 
