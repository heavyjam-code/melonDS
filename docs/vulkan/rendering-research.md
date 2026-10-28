# melonDS Vulkan renderer: rendering research and prior-art assessment

Related: [Implementation plan](implementation-plan.md), [architecture](architecture.md), [Vulkan design](vulkan-design.md).

Research date: 2026-10-28. Target: Windows first, accurate rendering and faster play at high internal resolutions. This memo separates inspected evidence from proposed engineering decisions. It supports a plan, and does not claim that a renderer was implemented or benchmarked.

## Recommendation

Build Vulkan around the fork's existing programmable compute rasterization semantics, then port the current GPU 2D/composition/capture path. Treat native-resolution accuracy, coherent emulated VRAM, and high-resolution presentation as distinct contracts. Make bounded scratch memory and lossless overflow handling early architectural requirements. A translation of API calls alone will retain the current compute renderer's capacity assumptions and will not demonstrate a speedup.

This is an engineering recommendation based on the inspected code and maintainer explanations, not a measured comparison of implementations. A graphics rasterizer can remain a future optional speed experiment; it should not become the initial accuracy baseline. [RSDuck's rationale](https://github.com/melonDS-emu/melonDS/issues/984#issuecomment-772483634), [current compute code](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp).

## Exact code bases inspected

| Code base | Pinned evidence | Assessment |
| --- | --- | --- |
| Target fork | `heavyjam-code/melonDS`, `bottom-right-layout`, `dd8043aa10795ba36d3ef8c3c4fa24782013896b` | Existing software, GL graphics and GL compute paths; local source inspection at the pinned fork revision. |
| Current upstream comparison base | `melonDS-emu/melonDS`, `1e654443e4c0a54b40709c0208aa0eeba41e6142` | The fork master verified in the architecture report; also the independently resolved API comparison base. |
| WatermelonDS core fork | `SapphireRhodonite/melonDS-android-lib`, `DevelopSub`, `65f4b25ef463b7ac4d9c8741f1a35e92af50e504` | Real, substantial Vulkan implementation, inspected selectively. Head commit committed 2026-10-16. Android-oriented, substantially diverged. |
| Wunkolo experiment | `Wunkolo/melonDS`, `vulkan-base`, `006ab35b5fa4960de594a323bf3b658a97ff7bcf` | Historical initialization/clear/readback scaffold; inspected `RenderFrame` is not a completed polygon renderer. |

The live GitHub comparison reports the Watermelon core branch **233 commits ahead and 77 behind** upstream, with merge base `b86390e4428bf38ce4c1ce0e9ca446d6d25955e8` dated 2025-11-18, and 128 changed files. Those counts describe this comparison at research time, not a compatibility guarantee. [Comparison](https://github.com/SapphireRhodonite/melonDS-android-lib/compare/1e654443e4c0a54b40709c0208aa0eeba41e6142...65f4b25ef463b7ac4d9c8741f1a35e92af50e504).

## Why the rasterizer choice matters

The DS geometry engine already supplies processed polygons. The difficult boundary is rasterization and downstream composition: preserving integer precision, polygon ordering and attributes while allowing increased presentation resolution. Existing source explicitly implements different Z/W equality tolerances, a facing-dependent equality exception, and integer blending. Replacing these operations with the host GPU's usual depth comparison and blending cannot be assumed equivalent. [Software reference, especially lines 381–435](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Soft.cpp#L381).

The compute shaders are a useful starting specification: span interpolation, binning, variant rasterization, ordered depth/attribute blending, then edge/fog/AA final passes. Texture mode variants already distinguish modulation, decal, toon, highlight and shadow mask behavior. Preserve their data and arithmetic first; establish where the existing compute path differs from the software reference before changing algorithms. [Shader source](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute_shaders.h#L31).

The older maintainer discussion remains relevant evidence for the architectural tradeoff: fixed-function host rasterizers differ from the DS, and 3D is a layer consumed by the 2D engines. It also explains why returning only a 3D texture does not solve blending or capture. Its descriptions of the then-current 2D implementation are historical, so use the present fork's source for integration details. [Issue #984](https://github.com/melonDS-emu/melonDS/issues/984).

| Approach | Accuracy and implementation tradeoff | Recommendation for this fork |
| --- | --- | --- |
| Port existing compute rasterization | Keeps DS span coverage, arithmetic and attribute model visible in shaders; still inherits existing defects unless tested and corrected. Requires explicit Vulkan layouts, barriers and memory management. | Preferred first renderer. |
| Rebuild GL graphics path with Vulkan graphics pipelines | Familiar draw calls and potential fixed-function throughput, but polygon splitting/interpolation, fill rules and depth/blending approximations remain. API replacement does not itself improve emulation accuracy. | Optional later performance experiment after an accurate reference exists. |
| Import WatermelonDS wholesale | Provides implemented texture upload, graphics effects, capture and Vulkan resource machinery, but large Android and core-interface divergence must be reconciled. | Do not use as the initial integration strategy. |
| Compute 3D plus CPU 2D/readback | Useful offscreen/diagnostic checkpoint. Continuous full-frame readback and upload can erase speed benefits, and native CPU composition cannot automatically preserve high-resolution 3D output. | Explicitly limited bring-up stage only; do not label it finished high-resolution Vulkan support. |

These judgments are proposed engineering choices, informed by the sources above and the code assessment below. No relative performance figures have been measured here.

## High-resolution memory and overflow: resolve before optimizing

The fork sets `MaxWorkTiles = TilesPerLine * TileLines * 16`, then allocates three scratch arrays, each `4 * TileSize * TileSize * MaxWorkTiles` bytes. This yields **192 × width × height bytes**, or **9 MiB × scale²**, for those three allocations alone. The final buffers, spans, textures, 2D resources and multiple in-flight frames are additional. This is a derived calculation from the allocation expressions. [Allocation code](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L359).

| Internal scale | Three tile scratch allocations only |
| --- | --- |
| 1× | 9 MiB |
| 4× | 144 MiB |
| 8× | 576 MiB |
| 16× | 2.25 GiB |

A recent upstream issue reports two independent high-resolution failure mechanisms: indirect work counts exceeding a device dispatch dimension, and an overdraw budget overflowing the work/tile arrays. Inspection confirms the relevant unbounded allocation/count structure remains in the fork. The issue's proposed clipping of allocations prevents corruption by dropping work; that policy sacrifices layers and does not meet accurate rendering goals. Its testing and vendor conclusions are the issue author's reports, not independently reproduced results here. [Issue #2683](https://github.com/melonDS-emu/melonDS/issues/2683).

Proposed implementation requirements:

- Use device limits to split indirect work into legal dispatch dimensions, with a uniform tail guard and checked integer arithmetic.
- Bound scratch memory by processing subsets of independent tiles, or by another proven partition scheme. Preserve polygon order, color/depth/attributes, shadow state and AA's second layer wherever work is continued.
- Retain neighbors or a halo for final edge/AA processing when processing spatial batches.
- Detect every capacity overflow. Retry with larger storage, subdivide work, or run a correct fallback for that frame. Do not silently drop polygons or continue with partially written arrays.
- Size memory for all retained/in-flight products, then expose a supported maximum scale determined by actual device limits and allocation results.
- Make output equality tests cover multiple batch sizes, including deliberately tiny capacities, so the bounded algorithm proves equivalent to its unbounded reference.

PR #2065 already introduced tile-size scaling and was merged 2025-11-17. The present fork contains that strategy; implementing the old PR again would not address every remaining overflow issue. [Merged PR #2065](https://github.com/melonDS-emu/melonDS/pull/2065).

## WatermelonDS: detailed reuse assessment

The project advertises Vulkan and a 2026 release discussion describes both Android frontend and core integration. The backend exists in the separate `melonDS-android-lib` repository, not merely in frontend presentation code. The current `.gitmodules` still mentions `GBARumble_PR`, but that branch did not resolve during research; the backend inspected here is the independently resolved `DevelopSub` head. Do not use the stale branch name as an implementation pin. [Project](https://github.com/SapphireRhodonite/WatermelonDS), [release discussion #77](https://github.com/SapphireRhodonite/melonDS-android/discussions/77), [submodule configuration](https://github.com/SapphireRhodonite/WatermelonDS/blob/Develop/.gitmodules).

### What it actually implements

`GPU3D_Vulkan.cpp` is approximately 15,380 lines and 673 KB. Its inspected primary rendering path creates graphics pipelines and issues triangle draws; the observed compute dispatches export capture lines. It therefore is **not a ready-made Vulkan translation of melonDS's compute rasterizer**. It includes texture descriptors, render-context retention, GPU output, capture identity/readback and lifecycle machinery. This was a selective architectural review, not an exhaustive correctness audit. [Backend source, graphics draw example](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU3D_Vulkan.cpp#L9733), [capture compute example](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU3D_Vulkan.cpp#L12623).

The associated accelerated frontend builds triangle/line scenes and coverage-expansion metadata. The fragment shader uses floating varyings and optional explicit fragment-depth writes. Those are architectural facts; they are not proof of visible errors, and neither can they establish pixel equality with the software renderer. [Scene model](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU3D_AcceleratedFrontend.h), [fragment shader](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU3D_Vulkan_GraphicsRasterShader.frag).

### Why it is not a desktop drop-in

Its `Renderer3D` overrides accept `GPU&` in calls such as `RenderFrame` and `Reset`, and it coordinates structured 2D metadata and capture products. The current fork instead has a newer aggregate `Renderer` owning 2D, 3D and capture operations. Porting this code would also require reconstructing the downstream composition/capture contract; replacing only a 3D class is insufficient. [Vulkan class interface](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU3D_Vulkan.h), [CPU 2D/capture types](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/GPU2D_Soft.h).

Its context requires `VK_KHR_android_surface` and `VK_KHR_surface`, requests Vulkan 1.1 and requires the swapchain extension. Timeline semaphores and dynamic sampled-image indexing have fallback paths. The dispatch header directly includes `vulkan_android.h`. Consequently, Windows needs a new platform/context boundary and presenter; existing Android readiness does not demonstrate Windows support. [Context requirements](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/VulkanContext.cpp#L20), [dispatch header](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/src/VulkanDispatch.h#L5).

### Transfer strategy

Use it as a catalog of implementation ideas: texture staging/descriptors, retaining render products until presentation/capture is done, device profiling and capture diagnostics. Reimplement those concepts against the current fork's interface. Port individual source components only after checking their dependencies, generated shader process, correctness and per-file notices. Do not cherry-pick its broad core commits into the fork as the main plan.

Repository metadata identifies GPL-3.0, and inspected renderer source headers specify GPL version 3 or later. Preserve provenance and notices for reused material and inspect every added dependency; this memo makes no legal compatibility guarantee. [License](https://github.com/SapphireRhodonite/melonDS-android-lib/blob/65f4b25ef463b7ac4d9c8741f1a35e92af50e504/LICENSE).

No Windows build, cross-vendor runtime test or pixel-exact conformance run was performed for WatermelonDS. No published conformance result sufficient to establish parity was verified in the inspected material. Its quantity of code and advertised feature list are not acceptance evidence.

## Other useful precedents

Wunkolo's 2021 experiment motivated modular renderer work; PR #990 was merged. At the pinned `vulkan-base` head, `RenderFrame` clears an image, copies it to a staging buffer and waits for completion, with bitmap clear unfinished. It is useful historical scaffolding and design context, not a renderer to import. [Historical implementation](https://github.com/Wunkolo/melonDS/blob/006ab35b5fa4960de594a323bf3b658a97ff7bcf/src/GPU3D_Vulkan.cpp#L423), [merged modularity PR](https://github.com/melonDS-emu/melonDS/pull/990).

paraLLEl-RDP supplies a better methodological precedent: a programmable Vulkan rasterizer, deterministic reference comparisons, recorded workload replay, headless validation, specialization with an available fallback shader, and stage-level timing. Its chip semantics and memory architecture differ from DS, so use its engineering methods rather than copying the renderer. Its README's vendor/version statements include historical testing and are not a current melonDS device-support matrix. The repository declares MIT. [Project and test architecture](https://github.com/Themaister/parallel-rdp/blob/master/README.md), [license](https://github.com/Themaister/parallel-rdp/blob/master/LICENSE).

## Required correctness and performance experiments

The following are proposed acceptance tests, not tests executed during this research.

| Area | Required scenarios and observable output |
| --- | --- |
| Coverage/interpolation | Triangles, quads and clipped polygons; degenerate/horizontal edges; shared edges; native and high-resolution coordinates. Compare color plus coverage/attribute buffers where available. |
| Depth | Z and W modes, clear bitmap depth, equality tolerance boundaries and front-over-back equal-depth case. Use values immediately below/at/above thresholds. |
| Transparency | Same/different translucent polygon IDs, mixed opaque/translucent texels, alpha-test boundaries, optional depth writes, consecutive shadow masks and shadow polygons. Verify submission/variant batching never changes original order. |
| Effects | Modulation/decal/toon/highlight, fog boundaries/density interpolation, edge marking, DS AA and second-layer behavior. DS AA is its own algorithm; host MSAA is not an automatic substitute. |
| Textures/VRAM | All existing formats; repeat/clamp/mirror; palette changes; overlapping banks; writes that invalidate cached textures; reading captured output as a texture. |
| 2D and capture | Both engines, priorities, windows, mosaic, blending/master brightness, FIFO, screen swap, capture source A/B/blends, partial bank reads/writes and alternating 3D screens. Capture VRAM must have the correct frame and native packing, regardless of displayed scale. |
| Lifecycle | Reset, savestate load/rewind if supported, renderer/scale change, pause/stop, rapid window resize, device-loss/init failure and repeated reopen. Invalidate resource generations and stale frame identities. |
| Capacity | Dense fullscreen translucent stacks and per-variant counts across dispatch limits; tiny artificial scratch capacities; allocation failures. Every frame must be complete or use a correct fallback. |
| Performance | Same recorded scenes and settings on software, GL graphics, GL compute and Vulkan at 1×/2×/4×/8×; warm/cold pipelines separately. Measure CPU submission, GPU stages, capture wait, present wait and peak committed memory. Report p50/p95/p99 frame times, not FPS alone. |

Use software output as the deterministic regression reference at native resolution, while documenting known reference uncertainty and disagreements. Exact hardware accuracy requires independent hardware evidence for disputed behavior. At higher resolutions there is no equivalent physical DS framebuffer: test consistency against the chosen scaling policy and native capture contract. Do not require arbitrary downsampling of enhanced output to match every native pixel.

Create a frame/state replay artifact containing renderer-visible polygon/state/VRAM snapshots or ordered mutations as required by the chosen boundary. Store expected intermediate and final outputs, classify known differences explicitly, and make the same replays runnable without the Qt frontend. Add legal homebrew/minimal cases first; use locally owned game copies for manual regression scenes. The recent issue specifically supplies a Metroid Prime Hunters dense-effect stress scenario; treat that as a reproducer suggestion pending verification, not a benchmark already run. [Reported reproducer](https://github.com/melonDS-emu/melonDS/issues/2683).

## Research scope and limitations

Exa successful searches requested 50 result slots across initial and follow-up query angles; 32 additional requested slots belonged to four schema-rejected initial calls and produced no search results. Under the skill's literal sum-of-requested-results accounting, `sources_reviewed = 82`; this is not 82 read or unique sources. Duplicate searches/results were not treated as independent evidence. GitHub connector reads and targeted local source inspection supplied the verified code evidence.

This memo cites 22 distinct source URLs/documents, counting separate source files separately and treating line anchors in the same file as one document. The maintainer website returned HTTP 403 for full fetches of posts 56 and 143, so the final conclusions rely on accessible code and GitHub discussion rather than assuming those full posts were reviewed. No renderer was built, run, compared, fuzzed or benchmarked in this research. The research does not establish that no other Vulkan fork exists.
