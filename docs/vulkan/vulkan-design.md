# Vulkan backend design research for heavyjam-code/melonDS

Related: [Implementation plan](implementation-plan.md), [architecture](architecture.md), [rendering research](rendering-research.md).

Research date: 28 October 2026. Target: Windows first, accurate rendering with high resolution. Fork inspected: `bottom-right-layout` at `dd8043aa10795ba36d3ef8c3c4fa24782013896b`. This memo proposes architecture and implementation gates; no renderer was implemented, built, run, or benchmarked.

## Recommendation

Build a full `VulkanRenderer` using the existing integer compute 3D algorithm as the first accuracy reference, plus Vulkan implementations of 2D composition, capture coherence, and presentation. Keep a native Vulkan output path. A Vulkan 3D → CPU readback → existing presentation bridge is a useful short-lived correctness milestone, but it does not constitute the finished high-resolution renderer.

Use one `VulkanContext` for each `EmuInstance`; every window belonging to that instance shares its logical device, renderer resources, and submission owner. Give each window its own surface, swapchain, presentation state, and semaphore resources. Preserve the current software/OpenGL paths as explicit choices and as initialization fallbacks. Windows-first Vulkan does not require Linux/macOS Vulkan parity for its initial release.

The most consequential early risks are the frontend ownership model, synchronous CPU capture callbacks, compute scratch memory, and shader resource requirements. These need spikes and measurable gates before a large renderer port.

## What the inspected fork implies

The [architecture report](architecture.md) establishes the source integration points; these additional findings were directly checked against the pinned fork:

| Evidence | Design implication |
| --- | --- |
| `GPU.h:818–875`: `Renderer` owns both 2D engines and 3D, scanline/sprite callbacks, VBlank, capture allocation/synchronization, and savestate hooks. | Scope the finished backend as a complete renderer. Replacing only `Renderer3D` leaves GL parent ownership and 2D/capture responsibilities unresolved. |
| `GPU.h:858–861`: `GetFramebuffers(void**, void**)` returns RAM-vs-other as a boolean. Existing presenter treats “other” as GL texture handles. | Add a typed output contract; never place Vulkan handles in the existing GL texture arm. |
| `EmuThread.cpp:228–323`: renderer update, `RunFrame()`, and drawing occur on the emulation thread. Core renderer callbacks happen during that work. | Keep core GPU command recording/submission on that thread initially. Preserve synchronous callback semantics. |
| `CMakeLists.txt:1,35–38,72`: CMake 3.16, C++17, and optional OpenGL. The Qt frontend retains the Qt5/Qt6 choice. | Avoid assuming current Qt6-only or new CMake helper APIs are already available. Vulkan is another optional backend, not a mandatory startup dependency. |
| `GPU3D_Compute.cpp:42–68,72–175`: source assembly uses GLSL 4.30 plus defines and 33 compile steps. | Extract and compile auditable Vulkan shader modules; translate descriptor/uniform declarations explicitly. |
| `GPU3D_Compute_shaders.h:486–637`: active extended arithmetic uses `umulExtended`, `uaddCarry`, and 32-bit limbs. `int64_t` matches in this section are commented alternatives. | Do not require `shaderInt64` merely because a text search finds that token. Native 64-bit arithmetic can remain an optional, separately verified optimization. |
| `GPU3D_Compute.cpp:338–397`: tile size changes with scale; scratch, work, span, and output allocations depend on scale. | Query workgroup/descriptor/range/memory limits before compiling or allocating a requested scale. A scale supported in GL is not automatically supported by the chosen Vulkan device. |

Pinned source links: [renderer interface](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU.h#L818), [compute host](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L325), [compute shaders](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute_shaders.h#L486), [emulation thread](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/frontend/qt_sdl/EmuThread.cpp#L228), [build floor](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/CMakeLists.txt#L1).

## API and capability profile

Recommended initial product profile: Vulkan 1.2 or newer, `timelineSemaphore` supported and enabled, and synchronization2 supplied either by Vulkan 1.3 core or `VK_KHR_synchronization2` on a 1.2 device. This is a proposed support policy, not a claim that all Vulkan 1.2 devices provide synchronization2. An incompatible device receives a clear diagnostic and the configured software/OpenGL fallback. The actual target GPU list must approve this policy in the initial spike.

This profile gives one modern barrier/submission path and a monotonic completion serial for readback and resource retirement. There is no renderer-specific need to require Vulkan 1.3 as an unconditional version floor. A broader Vulkan 1.1 profile with legacy barriers and fence retirement remains possible, but maintaining two paths should follow demonstrated hardware demand. Timeline semaphores were promoted to 1.2; synchronization2 was promoted to 1.3, and its extension dependency is properties2 or Vulkan 1.1. `VK_KHR_create_renderpass2` is an interaction, not a dependency. [Timeline semaphore specification](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_timeline_semaphore.html), [synchronization2 specification](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_synchronization2.html), [Khronos dependency clarification](https://docs.vulkan.org/guide/latest/extensions/VK_KHR_synchronization2.html).

Keep WSI acquire/present semaphores binary; timeline semaphores do not replace them. Enable feature bits explicitly after querying them; API version promotion is not permission to assume every feature bit is enabled. [Timeline semaphore specification](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_timeline_semaphore.html).

Initial device admission checks should include:

- A graphics+compute queue family and presentation support for every active surface, or an explicitly implemented separate present family. Start with the common shared-family path; report unsupported combinations instead of pretending they work.
- `VK_KHR_swapchain` for windowed presentation; it is unnecessary for a headless offscreen replay test.
- Reflected shader resource requirements per entry point: active storage/uniform buffer counts, storage images/texel buffers/sampled image counts, descriptor set totals, push constant size, buffer/image dimensions and ranges, and workgroup invocations/dimensions/shared memory.
- `vkGetPhysicalDeviceFormatProperties`/image-format checks for each intended usage combination, including storage, sampled, transfer source/destination, and presentation formats. Prefer ordinary packed 32-bit representations for exact DS state; do not acquire optional storage-format or 16-bit capabilities without a measured reason.
- Bounds on dispatch dimensions, indirect-dispatch command contents, upload-ring alignment, `maxStorageBufferRange`, `maxTexelBufferElements`, memory allocation sizes, and per-scale memory budget.
- If a feature, format, or limit fails, lower the offered maximum scale or select a validated alternative kernel. Never silently truncate work or substitute lower arithmetic precision.

The Vulkan features specification describes which shader capabilities and accesses are conditional; queried format support can replace broad assumptions. [Physical-device features](https://docs.vulkan.org/refpages/latest/refpages/source/VkPhysicalDeviceFeatures.html), [storage image and texel-buffer guide](https://docs.vulkan.org/guide/latest/storage_image_and_texel_buffers.html), [compute limits and shared-memory rules](https://docs.vulkan.org/guide/latest/compute_shaders.html).

The first port should make this source-specific capability table executable, with final values derived from the actual emitted modules:

| Current compute source | Admission/check rule |
| --- | --- |
| `Rasterise` concatenates Polygon, WorkDesc, XSpan, Binning, and three tile SSBOs (`GPU3D_Compute_shaders.h:1055–1061`). | Source-level raster kernel uses seven storage-buffer resources. Reflect the final entry point and compare active counts to `maxPerStageDescriptorStorageBuffers` and descriptor-set totals; combine resources or reject incompatible devices. Sparse binding indices 0–7 alone are not a count. |
| Rasterise/DepthBlend use `local_size_x = TileSize`, `local_size_y = TileSize`; tile size is 8, 16, or 32 depending on scale. | Support 64, 256, or 1024 local invocations as selected; verify both dimensions and total against device limits. Use a smaller validated kernel if needed, rather than assuming 1024 everywhere. |
| Setup indices use `rgba16ui` `uimageBuffer`; final output uses an `rgba8` storage image. | Query storage texel-buffer/image format support and range limits, or deliberately port packed setup indices to an SSBO. Preserve exact packing and resulting arithmetic. |
| Coarse binning uses shared `uint`, `atomicOr`, and workgroup barriers; subgroup/ballot alternative is under `#if 0`. | No subgroup operation or subgroup-size requirement for the active baseline. Retain workgroup synchronization. |
| Extended arithmetic is 32-bit limbs; native int64 examples are comments. | No `shaderInt64` production requirement unless a new optional module actually declares it. |
| Scratch storage grows as scale squared. | Check each descriptor range plus full allocation budget; 8× produces a 192 MiB range per existing tile buffer before batching. |

These source facts are visible in the [pinned shader declarations and kernels](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute_shaders.h#L1055) and [host scale/allocation logic](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L338).

## Frontend and ownership

### Choose a shared-device native presenter

Use a small Vulkan-capable `QWindow`, embedded in the existing widget layout if necessary, with `QVulkanInstance` providing Qt platform surface integration. Let the emulator own logical-device creation and feature selection. `QVulkanInstance::surfaceForWindow()` and `supportsPresent()` wrap platform-specific surface/presentation operations. Qt documents native-container limitations, so the initial spike must test focus, input/touch transforms, stacking/overlays, fullscreen, DPI, and the fork's bottom-right layout. [QVulkanInstance](https://doc.qt.io/qt-6/qvulkaninstance.html), [QVulkanWindow and container caveats](https://doc.qt.io/qt-6/qvulkanwindow.html).

`QVulkanWindow` is a reasonable standalone demo but is a less natural default for this fork: it manages a device, graphics queue, command pool/buffer, FIFO swapchain, and resource recreation, while the renderer lives in the GUI thread. Its window ownership can conflict with the desired one-device/multiple-window emulator context. Features2 modification hooks are only available since Qt 6.7, while this fork retains Qt5 and unspecified Qt6 versions. Avoid an unplanned Qt floor increase or reliance on private Qt device adoption. [QVulkanWindow](https://doc.qt.io/qt-6/qvulkanwindow.html), [renderer lifecycle](https://doc.qt.io/qt-6/qvulkanwindowrenderer.html).

### Assign each object one owner

| Owner | Responsibility |
| --- | --- |
| GUI thread | Create and retain native Qt windows; process input, exposure, DPI, resize, and platform-surface events; publish immutable size/layout/surface-generation updates; keep its event/message pump running. |
| Emulation thread / `VulkanContext` | Create/use renderer resources and command pools; perform renderer command recording, submissions, retirement, capture readback, and per-window acquire/present; serialize all operations on a queue. |
| Per-window presentation state | Surface-generation token, extent/present-mode request, swapchain image/view resources, per-image present semaphores, frame/acquire resources, and retired-swapchain state. |
| Per-emulation frame state | Immutable geometry/register/VRAM snapshot references plus upload allocation and completion serial. Reused only after all consumers finish. |

Vulkan queue host access needs external synchronization. One submit owner is the simplest initial enforcement; if a separate present queue/thread is later introduced, document its ownership and synchronization explicitly. Submission order alone does not provide all memory dependencies. [Queue submission specification](https://docs.vulkan.org/refpages/latest/refpages/source/vkQueueSubmit.html), [presentation specification](https://docs.vulkan.org/refpages/latest/refpages/source/vkQueuePresentKHR.html).

On Windows, swapchain create/destroy/acquire/present can send a synchronous Win32 message to the HWND owner thread. If that thread is waiting for the emulation thread, a deadlock can result. Therefore close/recreate is an asynchronous request → worker retirement → queued acknowledgment → GUI destruction handshake; audit existing blocking render-lock/context-borrow patterns. Never make the GUI wait for worker WSI while its message pump is stopped. This is an explicit Windows acceptance gate. [Win32 WSI threading requirement](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateWin32SurfaceKHR.html).

Native-window recreation increments a surface generation. The worker retires swapchain users before the GUI releases the old surface/window. Pending work must reject obsolete generations. On zero extent/minimize, suspend presentation and avoid creating a swapchain; continue core rendering/capture and required readbacks. The Win32 specification expressly permits zero extent when minimized and prohibits creating a zero-sized swapchain. [WSI specification](https://docs.vulkan.org/spec/latest/chapters/VK_KHR_surface/wsi.html).

A secondary window must not independently throttle core emulation. Use the primary window's chosen pacing policy; acquire a secondary image only when available and skip that presentation if needed. Skipping presentation does not mean skipping emulated rendering or capture. Multiple `EmuInstance` objects initially keep separate logical contexts, simplifying local-multiplayer isolation. Process-wide shared-device pooling can be evaluated later if measurements justify its complexity.

### Type the output boundary

Replace the boolean/opaque pointer convention with a discriminated output kind, or a presenter-specific accessor contract. A conceptual output describes `CpuBgra`, `OpenGLTexture`, or `VulkanImage`, dimensions, color interpretation, frame/generation identity, and a lifetime/completion token. The generic core header should not require Vulkan headers just to compile software-only builds; Vulkan-specific details can live behind an opaque backend interface. The presenter must agree with the producing backend and retain the image until sampling completes.

## Shader port and data model

Port the existing integer interpolation/coverage/order algorithm before redesigning it. Native graphics-triangle rendering is a different accuracy project; generic Vulkan depth, rasterization, and blending cannot be assumed to reproduce DS behavior. Existing Vulkan prior art should be evaluated for helpers and proven fixes as described in the [rendering research report](rendering-research.md), with API-generation and algorithm differences kept explicit.

The OpenGL shaders are not compile-and-replace Vulkan sources:

- OpenGL has independent binding namespaces for uniform buffers, storage buffers, images, and samplers. Vulkan has one binding namespace per descriptor set. The fork uses binding 0 for different resource classes; define an explicit set/binding map, then reflect and verify it.
- Replace standalone uniforms with push constants or a uniform block; preserve exact host/GPU layout and add size/offset assertions. Check `std140`/`std430`, array stride, and packing in the emitted SPIR-V.
- Replace the generated OpenGL GLSL preamble with Vulkan-compatible source and target flags; specify the selected Vulkan/SPIR-V environment. Keep shader variants enumerated and test every variant.
- Keep 32-bit limb math and original rounding/shift rules first. Do not introduce float interpolation, subgroup-width assumptions, or unconditional native int64 arithmetic as a porting convenience.
- Preserve polygon order through binning and final blending. Atomic allocation can produce nondeterministic work-list positions; correctness depends on reconstructing the required semantic order, not on invocation scheduling.
- Separate accuracy state (integer color/depth/attributes, native capture contents) from display-only conversion/filtering. Verify image origin and BGRA/RGBA swizzle explicitly. Do not accidentally apply swapchain sRGB conversion inside the emulated arithmetic.

Vulkan consumes SPIR-V; official toolchain options include glslang/glslangValidator and Shaderc/glslc. SPIRV-Tools and SPIRV-Cross provide validation and inspection/reflection facilities. Prefer offline compilation and embedded/shipped modules for production so a shader compiler is not required on the player's PC. A developer rebuild option can retain readable source iteration. [SPIR-V and toolchain guide](https://docs.vulkan.org/guide/latest/what_is_spirv.html).

## Memory, synchronization, and readback

### Bound the memory before offering high scales

For the existing algorithm, three tile scratch buffers together use `9 MiB × scale²`: 9 MiB at 1×, 144 MiB at 4×, 576 MiB at 8×, and 2.25 GiB at 16×. This follows directly from `MaxWorkTiles = (W/tile) × (H/tile) × 16`, each buffer's `4 × tile² × MaxWorkTiles`, and three layers. This excludes final results, span buffers, work lists, textures/captures, staging, and duplicated in-flight resources. [Allocation source](https://github.com/heavyjam-code/melonDS/blob/dd8043aa10795ba36d3ef8c3c4fa24782013896b/src/GPU3D_Compute.cpp#L354).

At 8×, one such scratch buffer is already 192 MiB, so the individual descriptor range is a separate gate from total VRAM budget. Do not triple-buffer the entire scratch heap merely because presentation has multiple frames in flight. A single sequential core queue can reuse scratch after the appropriate dependency while frame output and uploads are separately retained. If the straightforward port exceeds limits, implement bounded tile/work batches with explicit overflow detection and a retry/split policy that preserves ordering. An overflow may trigger a safe renderer fallback before state becomes inconsistent; it must never drop polygons or frames.

Use an upload ring with persistently mapped host-visible memory, frame serials, and aligned ranges; device-local storage for repeatedly used render state; separate readback buffers for CPU-visible capture/screenshot needs. VMA is an optional implementation helper, not a correctness requirement. Its flush/invalidate helpers handle atom alignment, but mapping does not itself synchronize CPU/GPU access. [VMA memory mapping and coherence](https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/memory_mapping.html).

### Write a pass dependency table, not ad hoc barriers

| Producer → consumer | Required relationship |
| --- | --- |
| CPU upload → transfer/compute | Completed CPU writes, non-coherent flush where required, then submission with correct ownership/range; never overwrite a still-used ring range. |
| Transfer upload → compute | Transfer-write availability and compute shader-read visibility. |
| Compute setup/binning → later compute | Shader-storage write → required shader-storage read/write memory dependency for the actual ranges. |
| Compute-generated dispatch arguments → indirect dispatch | Shader-write → draw-indirect/indirect-command-read dependency in addition to data-buffer dependencies. |
| 3D output → 2D compositor / presenter | Producer compute writes → consumer sampling/read, with the proper image layout and retained lifetime. |
| GPU capture result → CPU VRAM read | Finish all producing commands for the requested range/version; transfer or storage-write → host-read dependency; wait exact completion; invalidate non-coherent readback memory; then publish bytes. |
| Previous presentation → next image use | Acquire/wait and image-layout dependency; not an assumption based only on CPU frame index. |

Khronos gives concrete compute-to-compute, indirect-work, image-layout, and host-readback examples. A fence wait establishes completion; non-coherent host memory may still need invalidation. [Synchronization examples](https://docs.vulkan.org/guide/latest/synchronization_examples.html).

`GetLine`, `SyncVRAMCapture`, reset, backend switch, and `PreSavestate` are correctness boundaries. When core execution requests data, flush the relevant pending work and block for that data even if rendering is otherwise asynchronous. Native-resolution capture/readback semantics must stay correct at all display scales. A savestate must contain coherent emulated memory/state; do not serialize transient Vulkan handles. Restore rebuilds device resources from the restored emulated state and invalidates stale cache generations.

Do not use a full-device wait every frame. Target the completion serial of the resource actually needed. Device/queue drain can remain a slow-path operation for backend teardown or exceptional recovery, with a separate presentation-lifetime policy.

### Presentation resources have different lifetimes

Allocate render-finished/present-wait binary semaphores per swapchain image and index them with the acquired image index. Frame fences and per-frame-slot resources do not prove that a presentation engine has finished using a present-wait semaphore. For swapchain recreation, use present fences when an appropriate swapchain-maintenance extension is available, otherwise follow the Khronos deferred-retirement approach. Plain `WaitIdle` is a practical legacy shutdown convention but is not a formal proof of presentation completion in unextended Vulkan; keep that limitation documented instead of overstating it. [Semaphore reuse guide](https://docs.vulkan.org/guide/latest/swapchain_semaphore_reuse.html), [swapchain recreation sample](https://docs.vulkan.org/samples/latest/samples/api/swapchain_recreation/README.html).

Handle `VK_ERROR_OUT_OF_DATE_KHR`/suboptimal surface changes without rebuilding the emulated renderer; retain offscreen images while rebuilding presentation resources. On device loss, stop using all old device handles, invalidate pending output tokens, and present a recoverable error/fallback path. Exact in-flight capture contents may be unavailable after device loss; avoid silently resuming with guessed emulated memory. Qt's renderer documentation illustrates resource release/reinitialization separation, but custom context recovery is the application's responsibility. [Qt lifecycle/device loss](https://doc.qt.io/qt-6/qvulkanwindowrenderer.html).

## Dependency, cache, and packaging policy

Add an optional `ENABLE_VULKANRENDERER` path, gated independently from `ENABLE_OGLRENDERER`. Software-only and GL-only builds should configure without Vulkan tools. A Vulkan-enabled portable Windows binary should still start when the loader/driver is absent: use runtime loading through a small loader or an audited upstream loader such as volk, and fail gracefully into another backend. Directly linking the Vulkan loader may make it a startup dependency. [Khronos loader guide](https://docs.vulkan.org/guide/latest/loader.html).

The fork's CMake 3.16 floor is below several modern FindVulkan conveniences: `Vulkan::glslc` appeared in 3.19, `Vulkan::Headers` and `Vulkan::glslangValidator` in 3.21, and component selection in 3.24. Either add a compatible helper using `find_program`/existing variables and pinned shader tools, or explicitly raise/document the build minimum in the foundation change. Do not copy current FindVulkan examples and assume the old floor supports them. [Official CMake version annotations](https://cmake.org/cmake/help/latest/module/FindVulkan.html).

Pin dependency/compiler revisions for release builds, compile every supported shader variant in CI, validate SPIR-V for the selected target, and embed a source/variant hash. Pipeline creation still involves driver work even with offline SPIR-V. Cache pipelines on disk with a bounded, validated header and backend/shader version. The standard cache header carries vendor ID, device ID, and pipeline-cache UUID; reject stale/corrupt/incompatible data and retry an empty cache. Do not treat a cache file as required for correctness. [Pipeline-cache guide](https://docs.vulkan.org/guide/latest/pipeline_cache.html), [cache-header specification](https://docs.vulkan.org/refpages/latest/refpages/source/VkPipelineCacheHeaderVersionOne.html).

Explicit GL interoperability is a later optional optimization if a remaining UI integration needs it. A `VkImage` is not a GL texture. Shared GPU memory requires matching external memory and synchronization capabilities on both APIs, same-device compatibility, platform handles, and correct import/export ownership. This introduces Windows handle/GL extension requirements and more recovery paths. It is avoidable in the initial native Vulkan presenter. [Khronos interop sample](https://docs.vulkan.org/samples/latest/samples/extensions/open_gl_interop/README.html), [external memory and synchronization](https://docs.vulkan.org/guide/latest/extensions/external.html).

For future macOS work, query actual portability requirements rather than encoding an old MoltenVK API ceiling. Enable `VK_KHR_portability_enumeration` plus its instance flag when using portability devices, and enable `VK_KHR_portability_subset` if the selected device advertises it. Inspect subset features/limits. The initial Windows release keeps other platforms on validated existing backends; a current Apple runtime/toolchain packaging decision belongs to the later portability milestone. [Portability enumeration](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_portability_enumeration.html), [portability subset](https://docs.vulkan.org/refpages/latest/refpages/source/VK_KHR_portability_subset.html).

## Proposed stage gates

These are integration milestones for the main implementation plan, not a claim that each is an independent renderer feature.

| Stage | Concrete result | Go/no-go evidence |
| --- | --- | --- |
| 0. Baseline and requirements | Pinned fork/replay inputs, selected device profile, current SW/GL captures and timings, build dependency policy | Target GPU/driver capability report, bounded scale memory model, exact reproduction commands; no speed claim before this exists |
| 1. Shared context and presenter spike | Optional backend discovery, typed outputs, one emulation instance with two native Vulkan windows | Windows resize/minimize/restore/fullscreen/DPI/window-close loop under validation; no GUI/emu deadlock; missing loader falls back; core rendering still works without a presentable surface |
| 2. Offscreen compute port | Extracted setup/bin/raster/depth/final kernels, replay → readback at 1×, explicit descriptors/barriers | All shader variants compile/validate; layout assertions; deterministic intermediate/final comparisons to the pinned references; overflow and device-limit failures are handled safely |
| 3. Full renderer integration | 2D engines/composition plus accurate capture/VRAM ownership, renderer factory/config/settings | Exact native framebuffer/capture bytes on reference cases; CPU read-after-capture, capture reused as texture, savestate/reset/backend-switch tests pass |
| 4. High-resolution and robustness | 2×/4×/8× where admitted, complete presentation/layout/OSD/input integration, cache and resource retirement | Memory-budget and overflow tests; image/capture checks at each scale; multiple windows/instances; clean cold/warm startup; recoverable initialization/device/surface failures |
| 5. Performance and Windows release | Targeted profiling and capability-aware user options, packaged build | Paired CPU/GPU/frame-time results and compatibility matrix; all release acceptance gates below; Linux/macOS Vulkan may follow separately |

A bridge that reads the Vulkan result back to CPU can finish stage 2 quickly and help isolate correctness. It must be labeled as a bridge; measure transfer bytes, readback waits, and lost overlap before deciding whether it has any useful permanent niche.

For planning only, an experienced renderer engineer with working build tools, reproducible test cases, and hardware access should reserve roughly 8–16 engineer-weeks plus contingency for full Windows correctness, high-resolution behavior, lifecycle testing, and tuning. This is an uncertain planning range, not a benchmark- or prototype-derived estimate. A positive prior-art reuse audit may reduce helper work; full capture/2D integration and scratch-budget redesign can expand it substantially. Re-estimate after stages 1 and 2 using actual evidence.

## Verification and performance protocol

### Accuracy evidence

Build a deterministic replay harness outside normal window presentation. Record input state/geometry/register/texture/capture snapshots at defined emulated boundaries, render the same fixture through SW, current GL compute, and Vulkan, and identify frame/scanline/tile/polygon on differences. Do not use ordinary asynchronous screenshots as the only oracle. Hardware-derived expectations are needed where both existing renderers share a bug.

At native resolution compare integer color, depth/attribute buffers where available, and capture/VRAM bytes exactly. Separate a genuine known reference defect from the comparison result rather than widening tolerance to hide it. For display scaling/filtering, define a per-effect allowed-difference mask and numerical rule justified by the chosen sampling/conversion operation; keep the emulated native capture comparison exact. Count mismatched pixels, maximum component delta, affected regions, and first divergent stage. Preserve mismatch artifacts for review.

Include Z and W buffering, clipped/degenerate polygons, shared edges/coverage, native AA, shadows and polygon IDs, translucent ordering/depth updates, fog/toon/edge marking, texture/palette changes, capture→texture loops, display switching, and active capture during pause/minimize/reset/savestate. The detailed DS cases are covered in the [rendering research report](rendering-research.md).

### CI split

| Environment | What it proves | What it cannot prove |
| --- | --- | --- |
| Every change, CPU build jobs | Software/GL-only build preservation; Vulkan configure/build; shader variants and SPIR-V validation; host/GPU layout checks; fixture parser/replay behavior | GPU driver correctness or presentation |
| Pinned Linux software Vulkan ICD (for example Lavapipe), offscreen tests | Vulkan command/resource correctness, deterministic fixture comparison on that pinned implementation, synchronization-validation checks | Windows WSI, vendor shader behavior, hardware performance, cross-driver determinism |
| Windows hardware smoke on available NVIDIA/AMD/Intel targets | Selected capability profile, shader execution, correctness fixtures, Win32 surface/multi-window behavior | Other GPUs/drivers not actually tested |
| Scheduled/manual Windows hardware regression | Long sessions, repeated resize/minimize/fullscreen/close, DPI/monitor moves, capture-heavy scenes, backend switches, multi-instance, cache corruption, memory pressure | Untested machines and unreproducible driver-reset behavior |
| Later Linux/macOS Vulkan jobs | Their own WSI, driver/runtime and portability compatibility | Automatic equivalence to the Windows results |

Lavapipe's developer describes VM/CI use without hardware; use it as another implementation under test, not an official reference rasterizer or performance proxy. [Lavapipe developer presentation](https://www.vulkan.org/user/pages/09.events/vulkanised-2025/T5-Lucas-Fryzek-Igalia.pdf).

Run normal core validation and synchronization validation in routine correctness tests. Run GPU-assisted validation in separate slow diagnostic jobs where the device supports its requirements; it consumes descriptor resources and has substantial overhead. Its requirements are diagnostics requirements, not automatically production-backend requirements. Validation can miss host-memory races, aliasing, and some indirect access; a clean validation log is necessary evidence but not proof of emulation correctness. [Synchronization validation and limitations](https://vulkan.lunarg.com/doc/view/latest/windows/synchronization_usage.html), [GPU-assisted validation](https://github.com/KhronosGroup/Vulkan-ValidationLayers/blob/main/docs/gpu_validation.md).

### Performance evidence

Use the same machine, build settings, title/replay, emulated interval, scale, coordinates option, vsync/limiter setting, and driver for paired measurements against SW and existing GL compute. Separate uncapped throughput from paced presentation. Report 1×, 2×, 4×, and 8× when the device/scale gate admits them. On each case record:

- Median/p95/p99 CPU frame time and frame-time spikes, GPU pass and total time where timestamp support allows, with timed-interval and warm-up policy recorded.
- Shader/pipeline cold-start time and warm-cache behavior; pipeline creation spikes during play.
- Upload/readback bytes and wait time, capture-specific stalls, queue/present/acquire stalls, allocations, high-water device memory and staging usage.
- Correctness comparison outcome and actual active device/driver/features alongside the timing table.

Choose release targets after baselines: a stable no-regression budget for native resolution on supported hardware, useful headroom at high resolution, bounded memory within the admitted scale budget, and no new correctness/lifecycle failures. A convenient FPS number alone does not satisfy these gates. No benchmark or hardware renderer test was run for this research memo.

## Research coverage and source quality

Exa search accounting: `sources_reviewed = 64` under the skill's convention (sum of requested results), across 12 distinct queries; this is **search-result exposure**, not 64 independently verified pages. Fetched 28 unique primary-document URLs, including one Mesa LLVMpipe page excluded from the Vulkan analysis because it was the wrong driver documentation. Major decisions above use the cited specification, framework, toolchain, allocator, or project-source pages. Search-result duplicates and secondary tutorials were not treated as independent support.

Sources are authored by the relevant API/project maintainers or, for Lavapipe, a directly contributing developer. Official framework/tutorial examples can still simplify edge cases: the Vulkan specification and current synchronization guide govern the Windows message-pump and presentation-lifetime decisions. The current fork was inspected directly for the source facts above. Portability runtime recommendations change; the memo intentionally leaves future Apple packaging to its own refreshed implementation milestone.
