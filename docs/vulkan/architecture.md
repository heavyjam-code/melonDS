# melonDS fork architecture and Vulkan integration map

Related: [Implementation plan](implementation-plan.md), [rendering research](rendering-research.md), [Vulkan design](vulkan-design.md).

Inspected 2026-10-28. This is a source inspection and implementation proposal; no renderer implementation, build, gameplay test, GPU profiling, or fork mutation was performed.

## Verified fork and baseline

- Fork: [heavyjam-code/melonDS](https://github.com/heavyjam-code/melonDS).
- Intended preservation baseline: [`bottom-right-layout` at `dd8043aa10795ba36d3ef8c3c4fa24782013896b`](https://github.com/heavyjam-code/melonDS/commit/dd8043aa10795ba36d3ef8c3c4fa24782013896b).
- Fork `master`: `1e654443e4c0a54b40709c0208aa0eeba41e6142`.
- At research time, GitHub branch enumeration returned exactly those two branches, and a clean source checkout matched both commit IDs.
- The fork is owned by `heavyjam-code`; the baseline retains native upstream fast-forward muting and the bottom-aligned horizontal layout. The layout commit changes only `ScreenLayout.cpp`, `ScreenLayout.h`, Qt `Screen.cpp`, and Qt `Window.cpp` (21 insertions, four deletions). It does not modify emulation/rendering.
- Implementation priority: Windows first, accurate rendering and faster high-resolution play. 8x is an important validation target, not a promised Vulkan speedup. Linux/macOS keep working through their current fallback backends; Vulkan parity there is a later stage.

Recommended working base is a single future feature branch from `bottom-right-layout` (or an upstream update with that one layout commit retained). Preserve the custom layout rather than replacing it with a Vulkan-specific approximation.

## What this version actually renders

The current architecture is a full renderer backend, including accelerated 2D. A proposal based on the older melonDS design of software 2D plus hardware 3D misses a large part of this fork.

| Area | Direct source evidence | Consequence for Vulkan |
|---|---|---|
| Backend interface | [`GPU.h:818-875`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.h#L818) defines `RendererSettings` and `Renderer` | Backend owns reset/stop, scanlines, sprites, 3D start/finish/restart, VBlank, capture, outputs, shader progress. Add Vulkan at this seam. |
| 3D interface | [`GPU3D.h:321-346`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D.h#L321) defines `Renderer3D` | Compute 3D can remain a separate implementation owned by Vulkan parent, retaining `RenderFrame` timing and geometry engine. |
| Software baseline | [`GPU_Soft.cpp:28-45`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU_Soft.cpp#L28) constructs software A/B 2D plus software 3D | Useful differential reference. It is not a modular high-resolution Vulkan bridge without refactoring: `SoftRenderer2D` takes `SoftRenderer&`, and the software parent dynamically casts to `SoftRenderer3D` in settings/savestate code. |
| GL parent | [`GPU_OpenGL.cpp:36-52`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU_OpenGL.cpp#L36) constructs GL A/B 2D and either GL or compute 3D | A compute 3D port alone does not replace GL dependencies or preserve full high-resolution rendering. |
| Compute kernel pipeline | [`GPU3D_Compute.cpp:1059-1216`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L1059) | Port known integer rasterization behavior and explicit pass dependencies before redesigning the algorithm. |
| Texture cache | [`GPU3D_Texcache.h`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Texcache.h) is templated on loader and handle; [`GPU3D_TexcacheOpenGL.h:15-29`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_TexcacheOpenGL.h#L15) supplies GL loader | Reuse decoding/invalidation with a Vulkan loader and stable logical texture IDs; preserve array allocation and defer reuse/deletion until submissions finish. |

`RendererSettings` has scale factor, software threading, high-resolution coordinates, and GL improved polygon splitting. They should not all be interpreted as equivalent Vulkan features. High-resolution coordinates are an optional accuracy/appearance choice; native-resolution comparisons must use a known matching setting.

## Data and timing boundaries to preserve

The CPU geometry engine produces transformed/clipped `Vertex`/`Polygon` records. Its public state includes `VertexRAM[6144*2]`, `PolygonRAM[2048*2]`, `RenderPolygonRAM[2048]`, latched render registers, and polygon counts ([`GPU3D.h:33-82,248-315`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D.h#L248)). `GPU3D::VBlank` handles ordering and register latching ([`GPU3D.cpp:2425-2498`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D.cpp#L2425)). Preserve geometry, FIFO, clipping, sorting and emulated event timing initially.

`GPU::StartHBlank` draws visible scanlines/sprites and starts 3D at VCount 215; `FinishFrame` swaps backend buffers. VBlank finishes 3D, latches the next render state and calls backend VBlank ([`GPU.cpp:1138-1195`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.cpp#L1138), [`GPU.cpp:1272-1308`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.cpp#L1272)). GPU execution can be asynchronous, but game-visible reads/capture must still see the correct emulated state.

The current code does not provide an immutable Vulkan frame packet. Pack polygon spans, uniforms, texture uploads and line state into backend-owned staging/ring allocations before the CPU mutates them; keep each allocation alive through GPU completion. Do not queue pointers to live `PolygonRAM`, palettes, or reusable CPU decoding buffers. A small frame ring must not silently add emulated capture latency.

The existing compute stages, in order, are:

1. CPU span/variant setup and texture cache updates.
2. Clear bin masks and indirect counts.
3. X-span interpolation.
4. Polygon binning.
5. Work-list offsets and sorting.
6. Per-variant rasterization via indirect dispatches.
7. Depth/blend composition.
8. Final anti-aliasing, edge marking and fog output.

Vulkan resource accesses must encode storage writes to subsequent compute reads/writes, generated-indirect writes to indirect-command reads, and final image writes to 2D/compositor sampling. GLSL currently reuses binding numbers across OpenGL resource namespaces (e.g. uniform block binding 0, storage block binding 0, texture binding 0). Vulkan descriptor set assignments need a deliberate layout rather than a textual API rename. Verify CPU/SPIR-V struct offsets (`MetaUniform`, spans, work headers) with `static_assert` and reflection.

## 2D, display capture and coherency are launch work

[`GLRenderer::DrawScanline` at 399](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU_OpenGL.cpp#L399) detects display/capture/brightness changes and renders previous scanline ranges before updating state. It records per-line screen swap, auxiliary VRAM/FIFO input and 3D X scroll for capture. [`GLRenderer2D::UpdateAndRender` at 517](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU2D_OpenGL.cpp#L517) tracks dirty VRAM/palettes/OAM, prerenders BG/sprites, updates per-line affine/mosaic/window state, and flushes screen sections when critical state changes.

A Vulkan implementation can preserve the same host-side batching logic and translate its resource/graphics passes. Snapshot each section's state and referenced data or flush ordered work before overwriting resources. A single end-of-frame uniform snapshot would break raster effects. Do not multiply command submissions per scanline by default; record sections in a command stream and flush when CPU-visible dependencies require it.

The GL renderer composes A/B 2D plus 3D and auxiliary inputs into a double-buffered two-layer output texture. Relevant shaders are listed in [`OpenGL_shaders/CMakeLists.txt:1-28`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/OpenGL_shaders/CMakeLists.txt#L1): 2D layer prerender, sprite prerender/render, 2D compositor, final pass, capture and downscale. Port these behaviors alongside compute 3D rather than declaring completion when a standalone 3D test triangle appears.

Display capture stays on the GPU when possible. Core capture flags track A-D banks and 32 KiB blocks; CPU reads can synchronize a capture and writes invalidate it ([`GPU.cpp:1556-1665`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.cpp#L1556)). Textures/BG/sprites can consume captures without ordinary CPU VRAM decoding. Current GL readback downscales and writes native 1555 data into emulated VRAM ([`GPU_OpenGL.cpp:635-914`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU_OpenGL.cpp#L635)). Vulkan needs equivalent native bit encoding, offsets/wraparound, capture reuse, CPU read/write visibility, and partial capture handling; scheduling a readback for a later unrelated frame is incorrect.

`GPU::DoSavestate` calls `PreSavestate`, synchronizes captures before saving, serializes VRAM/state, invalidates caches after load and calls `PostSavestate`. `GPU::SetRenderer` synchronizes captures before replacing a backend, falls back to software on init failure and dirties VRAM/OAM/palettes ([`GPU.cpp:205-345`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.cpp#L205)). Preserve that order, ensure outstanding work is retired safely, and report the actual fallback backend in the UI. Device loss needs a defined failure path because a lost device may make pending capture data irretrievable; do not claim all device-loss recovery is lossless.

## Frontend: why another enum value is insufficient

| Current behavior | Evidence | Required change |
|---|---|---|
| Renderer IDs are `0` software, `1` GL raster, `2` GL compute | [`EmuInstance.h:71-79`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/EmuInstance.h#L71) | Append Vulkan as `3`; preserve stored numeric meanings. Validate unsupported/malformed selection and use a real fallback. |
| Every nonsoftware renderer is assumed GL | `EmuInstance.cpp:375-379`, `Window.cpp:818-819`, `VideoSettingsDialog.cpp:32-36` | Replace these boolean tests with explicit renderer/presentation backend capabilities. |
| Opaque output means pointer to a GL texture ID | [`GPU_OpenGL.cpp:915-922`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU_OpenGL.cpp#L915), [`Screen.cpp:1135-1160`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/Screen.cpp#L1135) | Introduce tagged RAM/OpenGL/Vulkan output plus size, layers, lifetime and completion contract, or a separately typed Vulkan accessor. Never pass `VkImage` through the old false/GL path. |
| Qt frontend supports Qt6 default and Qt5 alternative | [`qt_sdl/CMakeLists.txt:65-72`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/CMakeLists.txt#L65) | No explicit minimum Qt version declared here; do not assume recent Qt6 Vulkan APIs. Make Vulkan opt-in/Qt6-only if chosen, keep Qt5 legacy builds valid. |
| GL display is a native `QWidget` with custom `GL::Context` | [`Screen.cpp:869-911`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/Screen.cpp#L869) | Choose and prototype Vulkan surface integration; embedding a `QWindow` can preserve shared layout/input abstractions, but introduces focus/DPI/container work. |
| Secondary windows share the primary GL context | `Screen.cpp:885-910`, `EmuInstance.h:32` (`kMaxWindows=4`) | Share renderer output on a device with per-window surfaces/swapchains. Avoid a device per `QVulkanWindow` without a sharing plan. |
| Emu thread owns rendering and sequential per-window drawing | [`EmuThread.cpp:229-252,304-323`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/EmuThread.cpp#L229), `EmuInstance.cpp:427-434` | Prefer one Vulkan submission owner per instance initially. GUI creates/destroys native surfaces and sends lifecycle requests; commands/resources cannot race. |
| Backend changes use GL/non-GL boolean | [`Window.cpp:2264-2324`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/Window.cpp#L2264), `EmuThread.cpp:858-888` | Use old/new backend identity, pause/drain old backend and output consumers before panel/device teardown; then initialize and resume or rollback. |
| Settings modify config live; Cancel restores old values | [`VideoSettingsDialog.cpp:55-164`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/VideoSettingsDialog.cpp#L55) | Add Vulkan state to accepted/rejected paths; changing GL to Vulkan must trigger recreation even though both are accelerated. |

Presentation should consume the existing `ScreenLayout` matrices and touch mapping, including the existing bottom-right option. `ScreenPanel` stores the shared layout/matrices; input calls `layout.GetTouchCoords`; the GL draw loops `screenMatrix` ([`Screen.cpp:181,263,296,320,365,1168`](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/Screen.cpp#L181)). Preserve top-left source coordinates `(0,0)..(256,192)`, matrix conventions, filtering, OSD and splash. Handle Vulkan viewport/image-origin decisions in the presenter instead of changing input math.

Keep device/queue ownership per emulation instance initially. Multiple emulation instances may run different threads; any future global shared device requires external queue synchronization and independent resource ownership. Current secondary windows can belong to another emulation instance, as `Window.cpp:2298-2304` explicitly notes; distinguish a screen window sharing an instance from a local multiplayer instance.

Fast-forward currently disables GL swap interval; Vulkan must map fast-forward to presentation pacing without perturbing the native mute logic (`EmuThread.cpp:344-355`). Resize/minimize/fullscreen/DPI/window close and swapchain recreation must not free outputs still used by other windows.

## High-resolution resource risk in the existing compute design

[`ComputeRenderer3D::SetRenderSettings` at 323-398](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L323) allocates three tile buffers using `4 * TileSize² * MaxWorkTiles`, with `MaxWorkTiles = pixel_count / TileSize² * 16`, plus `24 * pixel_count` bytes for final tile memory. Algebraically that is `216 * pixel_count` bytes before other buffers, 2D targets, textures, capture and presentation.

At 8x (`2048*1536 = 3,145,728` pixels), those four allocations alone total **679,477,248 bytes = 648 MiB**: 192 MiB per color/depth/attribute tile buffer plus 72 MiB final memory. Each 192 MiB storage buffer exceeds Vulkan's universally guaranteed minimum `maxStorageBufferRange` of 128 MiB; exact Vulkan capability requirements are covered by the [Vulkan design report](vulkan-design.md). Query actual limits, memory budget and supported format operations; provide a deliberate scale limit, chunking scheme or compact allocation. A direct port cannot assume any Vulkan 1.x device accepts the 8x allocation layout.

This is a major design constraint for the high-resolution objective. Establish correctness with the current algorithm first, but include bounded work allocation/tiling and overflow fallback as an explicit milestone. Do not multiply the entire scratch set by every frame in flight by default. Memory lifetime can be managed with a single serialized scratch graph and multiple small upload/output slots.

## Proposed implementation ownership and sequence

| Stage | Proposed touched/new files | Exit evidence |
|---|---|---|
| 0. Reproducible baseline and test seam | New `tests/render/` fixture/replay harness; `CMakeLists.txt`, `src/CMakeLists.txt` for opt-in tests | One fixed native scene packet can be replayed in software and GL compute, outputs/capture hashes stored with settings and source SHA. No commercial ROMs added. Current tree has build workflows, not an emulator renderer regression suite. |
| 1. Typed backend and lifecycle contracts | `GPU.h/.cpp`, `EmuInstance.h/.cpp`, `EmuThread.h/.cpp`, `Window.h/.cpp`, `VideoSettingsDialog.h/.cpp/.ui`, `Config.cpp`, new `RenderOutput.h` if appropriate | Existing three renderers and layout work unchanged; config IDs 0-2 retain meaning; GL↔Vulkan selection will recreate backend/presentation correctly; failed init restores a usable backend. |
| 2. Vulkan context and Windows presenter | New backend runtime/context/resource files, `ScreenVulkan.h/.cpp`, Vulkan presentation shaders, Qt CMake | Software-rendered frames uploaded to Vulkan and displayed with bottom-right layout, touch, OSD/filtering; resize/minimize/fullscreen/multi-window work with no validation errors. This proves presentation only and is not an accelerated DS renderer. |
| 3. Offscreen compute 3D parity | New `GPU3D_Vulkan.h/.cpp`, Vulkan compute shaders, `GPU3D_TexcacheVulkan.h/.cpp`, shared compute setup extracted only where useful | Native-resolution corpus matches the chosen software reference or every difference has a documented hardware/known-existing explanation; depth/W-buffer, shadows, alpha, fog, edge, AA and texture formats exercised. |
| 4. Full 2D/3D composition and capture | New `GPU_Vulkan.h/.cpp`, `GPU2D_Vulkan.h/.cpp`, port 2D/final/capture/downscale shader behavior | Raster register changes, sprites/windows/mosaic/blending, VRAM/FIFO display, screen swaps and capture feedback correct; byte-identical native capture data and save/load round trips. This is minimum full backend functionality. |
| 5. High-resolution and pacing | Vulkan resource allocator, compute scheduling, presenter, settings/diagnostics | 2x/4x/8x correctness and memory limits validated; warm/cold performance, CPU/GPU timings and percentile frametimes measured against GL compute. Report gains per workload; no speed guarantee. |
| 6. Release hardening | Windows/Ubuntu/macOS build workflows, packaging/dependency manifests, docs | Optional Vulkan builds and runtime fallback work; Windows vendor coverage and long sessions pass; Linux/macOS legacy paths remain green. Vulkan on other OSes is subsequent scope. |

Only split workers after the typed interfaces, device ownership and output completion contract are agreed. Compute shader/host port, 2D/capture translation and Windows presenter can then have separate file owners, with integration kept serialized.

Suggested initial build flag is independent `ENABLE_VULKANRENDERER`; do not make Vulkan require `ENABLE_OGLRENDERER`. Top-level `CMakeLists.txt:68`, core `src/CMakeLists.txt:97-110`, frontend `CMakeLists.txt`, `vcpkg.json`, `BUILD.md`, and the four existing build workflows are the concrete dependency/build integration points. Build-time SPIR-V compilation/embedding should declare source dependencies and generator target, following the existing generated-shader target pattern without requiring end-user shader compilers.

For configuration, append renderer ID 3 and add Vulkan-specific keys while reading shared initial scale/hi-res preferences from the old GL keys only when Vulkan settings are absent. Preserve the current 8x preference if supported; do not relabel existing GL values or rewrite old config semantics. Feature gating must distinguish unsupported device, Vulkan compiled out, and initialization failure. Keep truthful requested-versus-active backend diagnostics.

## Verification checklist and limits of this memo

- Native-resolution differential fixtures should cover empty/clear bitmap, clipped/degenerate polygons, Z and W depth/equality rules, translucent polygon IDs, shadow mask/shadow, compressed and palette textures, mirror/repeat, AA/edge/fog, 3D horizontal scroll and repeated identical frames.
- 2D/capture fixtures should include mid-frame BG/palette/OAM/register changes, affine/mosaic/window/blend rules, VRAM remapping, display FIFO, capture source A/B blending, all capture sizes/offsets/wraparound, capture reused as texture/BG, CPU read then write, and save/load/backend switch with pending capture.
- Lifecycle fixtures should include changing scale repeatedly, same-settings no-op, Cancel, Vulkan unavailable/disabled/failing, software↔GL↔Vulkan switching while paused/running, screen window close/reopen, more than one emulation instance, device loss, minimize/restore, fullscreen, monitor DPI changes and fast-forward transitions.
- Use Vulkan validation and synchronization validation during development; require no unresolved errors. Add explicit bounds/overflow checks for 2048 polygons, 6144 vertices, span arrays and work lists. Validation cannot prove DS pixel correctness.
- Compare 8x performance at identical emulation speed, audio/pacing, high-res-coordinate setting, resolution, warm cache and scene. Include capture-heavy and 2D-heavy scenes, not only polygon-heavy scenes. Separate frame-cap limited measurements from uncapped throughput and include memory usage.
- This memo inspects source and remote branch identity only. Existing build/workflow presence does not establish successful Vulkan builds. Current hardware and driver behavior were not examined. No claim is made that software output is a perfect hardware oracle or that Vulkan will outperform current GL compute.
