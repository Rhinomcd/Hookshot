> [!NOTE]
> gpt-6.1-sol responding on behalf of Ryan.

# Rust interop with Hookshot's vendored renderer libraries

Research date: 2026-11-10. Source baseline: `f9f8cf53758e982ef573003394981dc2cd121ac9`.

Keep the native implementations and put the boundary around a renderer operation, rather than around each C++ method. Use `ash` for Vulkan types and calls, fed by the existing loader. FidelityFX needs no runtime binding at all. These recommendations follow the map's plain-C strangler and unchanged-vendor requirements, and the uses described below. They do not propose a vendor upgrade. See `.scratch/rust-port/map.md:15–16` and `:46`.

This was source and documentation research only. No Rust installation, compilation, tests, or game runs occurred; ABI and parity recommendations remain uncompiled. Neither forbidden directory was opened or enumerated. Web reading used Firecrawl CLI; retrieved pages stayed outside the checkout. Release dates below come from package version histories or release notes, not documentation build dates. “Blocker” means a documented limitation or unresolved Hookshot integration requirement, not a claim that every upstream issue was audited.

## Summary

Effort is my estimate for the binding and ownership changes alone. S means roughly 1 to 2 engineer days, M means 3 to 5, and L means 1 to 2 weeks. Estimates exclude porting the component's remaining logic, toolchain setup, and runtime parity evaluation. Each row links to its evidence and tradeoffs.

| Library | Recommended binding | Runner-up | Parity risk | Effort |
| --- | --- | --- | --- | --- |
| [Eden](#eden-shader-recompiler) | Handwritten C facade over the existing translation glue; opaque C++ ownership and flat metadata | Private `cxx` bridge over the same facade | Low for SPIR-V with unchanged inputs and pass order; high for a new metadata/cache representation | M–L |
| [Sirit](#sirit) | Keep it with Eden; C operation for cooperative-matrix probe generation | `cxx` over an opaque probe builder | Low if instruction emission order stays fixed; shader bytes change with another emitter | S |
| [Vulkan](#vulkan) | `ash`, using the existing `vkGetInstanceProcAddr` and borrowed device | Generated C bindings and explicit optional function table | Low for calls; loader search, missing extensions, and work accounting need care | M |
| [VMA](#vulkan-memory-allocator) | C wrappers over today's single allocator and policies | Pinned `vk-mem` 0.5.0 at a later, complete allocator ownership handover | Low with the same VMA; high with `gpu-allocator` or changed pools | S; M for handover |
| [Dear ImGui](#dear-imgui) | Keep menu, draw helpers, fonts, texture uploads, and backend in C++; expose frame/menu operations | `cimgui` generated for exactly the vendored revision, plus small custom wrappers | Low while retained; high for wrapper version, rasterizer, backend, or drawing changes | S retained; L to expose drawing |
| [FidelityFX](#fidelityfx) | Keep checked-in SPIR-V; retain C++ pass recorder behind C until it is ported | Port the first-party recorder to Rust/`ash`, consuming the same shader words | Shader assets can stay identical; constants, resource order, and float math can change pixels | S retained; M–L recorder port |
| [NVIDIA NGX](#nvidia-ngx) | C facade over existing optional C++ integration | Bindgen over a new narrow C header, with SDK helpers still in C++ | Low only with the same SDK/runtime, parameters, inputs, and history | S facade; M if orchestration moves |
| [zstd](#zstd) | Bind the eight existing C decoder functions, or retain the font reader behind C | `zstd`/`zstd-safe` over a pinned native decoder | Valid decoded bytes can match; defaults change allocation and accepted inputs | S |

“Byte-identical” here distinguishes shader words, decoded bytes, and UI drawing data from process addresses, GPU handles, or driver pipeline blobs. Keeping a library implementation makes exact output parity possible; it does not prove it. Eden's cache additionally stores native object representations, so file-byte parity is a separate problem. See `gpu/vk/vk_shader.cpp:1020` and `:1260`.

## Call-site inventory

These are textual source counts, not runtime call frequencies. `rg -c` counts matching lines, while a line can contain several references. The reproduction code at the end limits itself to first-party `gpu/vk/*.cpp` and `*.h`.

| Dependency | Current use | Source evidence |
| --- | --- | --- |
| Eden | `Shader::` occurs on 159 lines in `vk_shader.cpp`, 40 in `vk_draw.cpp`, and 11 in `vk_coopmat.cpp`; respectively 169, 41, and 11 occurrences | Translation/environment at `gpu/vk/vk_shader.cpp:382`, emission at `:1490`, draw metadata at `gpu/vk/vk_draw.cpp:2395`, tuning globals at `gpu/vk/vk_coopmat.cpp:19` |
| Sirit | 19 lines with `Sirit::`, 26 qualified occurrences, all in `vk_coopmat.cpp`; another 56 `m.method(...)` matches use its module directly | `gpu/vk/vk_coopmat.cpp:50–106` and the later probe builders in the same file |
| Vulkan | 659 call-shaped matches in 18 `.cpp` files, 150 distinct names; another 54 matches in `vk_work.h` are macro declarations and their forwarded calls. `vk_loader.h` names 164 distinct commands, including the Windows-only surface entry | `gpu/vk/vk_loader.h:17`, `gpu/vk/vk_main.cpp:27–63`, `gpu/vk/vk_work.h:20` |
| VMA | 23 calls to 13 distinct functions, all in `vk_vma.cpp` | `gpu/vk/vk_vma.cpp:17–255` |
| ImGui | 11 `ImGui::` references in `vk_imgui.cpp`, 3 in `vk_menu.cpp`; 7 backend calls. Those small counts omit direct draw-list/font calls: the method filter below finds 38 in `vk_ui.h`, 57 in `vk_menu.cpp`, and 7 in `vk_imgui.cpp` | `gpu/vk/vk_ui.h:44–97`, `gpu/vk/vk_menu.cpp:1188`, `:1753`, `gpu/vk/vk_imgui.cpp:290–429` |
| FidelityFX | Zero `ffxX(...)` runtime calls and zero `ffx` header includes in those first-party files. `vk_fsr3.cpp` includes 9 FSR headers and `vk_fsr3fi.cpp` includes 17. The generator defines 11 upscaler variants and 17 interpolation/optical-flow variants | `gpu/vk/vk_fsr3.cpp:28–36`, `gpu/vk/vk_fsr3fi.cpp:26–42`, `scripts/fsr3_shaders.py:43–99` |
| NGX | 19 `NVSDK_NGX_*(` matches on 17 lines and 4 `NGX_*(` helper calls, all in `vk_dlss.cpp`. Of the 23, 7 are success/failure predicates; the other 16 are API/helper expressions | `gpu/vk/vk_dlss.cpp:276`, `:398–446`, `:750–810`, `:1166` |
| zstd | Eight calls, one each to eight distinct decoder functions, in `vk_game_font.h`; counting only `.cpp` would miss them | `gpu/vk/vk_game_font.h:50–59`, `:110–119` |

The ticket's menu-test count is stale under this simple definition. Twenty `tests/test_*.py` files contain `vk_menu.cpp` at this baseline. That is a count of source consumers, not twenty pixel tests. For example, `tests/test_menu_utf8.py:1–6` compiles extracted functions without ImGui or a GPU.

## Eden shader recompiler

### The boundary includes metadata and callbacks

`Env` derives from `Shader::Environment` and answers instruction, constant-bank, texture, memory-size, workgroup, and pointer questions. Some answers depend on the current state and recorded memory. `Translation` owns three `ObjectPool` specializations and an `IR::Program`; the compiler lowers Ampere, constructs Maxwell control flow, translates IR, applies local fixes, and emits SPIR-V. This is a stateful C++ service, not a single pure function over shader bytes. Sources: `gpu/vk/vk_shader.cpp:186–292`, `:382–499`, `:551–565`, `:880–955`, `:1425–1506`.

The types that cross today's first-party/Eden boundary are:

| Role | Types and data | Source |
| --- | --- | --- |
| Input interpretation | `Ampere::Instruction`, `ProgramCaptureState`, lowered program/address mapping, `ProgramHeader`, `Stage` | `gpu/vk/vk_shader.cpp:124`, `:186–292`, `:798` |
| Translation lifetime | `Environment`, `HostTranslateInfo`, `Profile`, `ObjectPool<IR::Inst>`, `ObjectPool<IR::Block>`, `ObjectPool<Maxwell::Flow::Block>`, CFG, `IR::Program` | `gpu/vk/vk_shader.cpp:382`, `:503–565` |
| Emission | `RuntimeInfo`, `Backend::Bindings`, `MeshInfo`, `std::vector<uint32_t>`; per-GPU and thread-local compiler globals | `gpu/vk/vk_shader.cpp:59–71`, `:1425–1506`, `:1646–1717`; `gpu/vk/eden/shader_recompiler/backend/spirv/emit_spirv.h:51` |
| Draw-facing reflection | `Info`; constant/storage/texture/image-buffer/texture/image descriptor arrays; used constant-buffer sizes, attribute load/store masks, global tables, feature flags, handle programs | `gpu/vk/vk_shader.h:20–25`; `gpu/vk/eden/shader_recompiler/shader_info.h:166–394`; `gpu/vk/vk_draw.cpp:2062`, `:2376–2430` |
| Shared values and helpers | `AttributeType`, `TextureType`, `ImageFormat`, `IR::Attribute`, push-constant offsets, mesh fallback capacity, `EvaluateHandleProgram` | `gpu/vk/vk_shader.h:129–130`; `gpu/vk/vk_draw.cpp:1862–1881`, `:2964–2983`; `gpu/vk/vk_shader.cpp:605–643` |

`Info` contains vectors, a map, bitsets, and compatibility containers, including nested vectors of handle instructions. It is not a C struct that Rust can safely mirror by applying `repr(C)`. Even the existing cache writer treats its dynamic members specially. Sources: `gpu/vk/eden/shader_recompiler/shader_info.h:275–394`, `gpu/vk/vk_shader.cpp:1258–1300`.

### Binding choice

Recommend a handwritten C facade around the existing shader glue. Keep `Env`, IR, passes, per-thread globals, and serialization in C++. Return an opaque result owner, borrowed SPIR-V words, and explicitly sized arrays of flat reflection records. Give the owner one C++ destroy function. Export the draw-required constants and enums as fixed-width values. Keep handle-program evaluation in C++ initially, with explicit callbacks for the same submitted-memory readers. This follows the actual lifetime and uses above, and avoids a foreign call for every IR instruction or descriptor field.

When draw moves to Rust, flatten reflection once per translated stage. Include all six descriptor lists, handle-program references, used sizes, attribute masks, global-table metadata, and flags that draw reads. Preserve descriptor order, secondary-handle shifts, the distinction between a missing size and zero, and the memory snapshot used by callbacks. The current descriptor walk and handle evaluator establish those requirements. Sources: `gpu/vk/vk_draw.cpp:2395–2430`, `gpu/vk/vk_shader.cpp:605–643`, `:1425–1506`.

| Option | What it buys | Hookshot cost and blocker |
| --- | --- | --- |
| Handwritten C | A small, stable header; no C++ container layout in Rust; current native toolchain still owns construction and destruction | Manual ownership, array lengths, status codes, and callback contracts. Catch C++ exceptions inside the facade and return errors before returning to Rust. This is the best fit for the existing plain-C plan and throwing `Env` implementation. `gpu/vk/vk_shader.cpp:398`; [bindgen exception limitation][bindgen-cpp] |
| `cxx` | Checked bridge signatures, opaque types, `UniquePtr`, `CxxVector`, and explicit exception-to-`Result` handling | Runner-up for a private bridge. It still needs adapters for `Info`, callbacks, maps, and nontrivial types. `CxxVector` cannot pass by value; maps and `Option` remain pending bindings. A throwing function omitted from `Result` terminates; a Rust callback panic aborts. Generated bridge code also has to join the existing C++20 build. [CXX bindings][cxx-bindings], [error handling][cxx-errors], `nx2/build.py:835–845` |
| `autocxx` | Automatically discovers more pre-existing C++ declarations and uses CXX underneath | Reject for new work. The book says it is “no longer actively maintained.” Non-CXX template instantiations are opaque; exceptions are unsupported. Eden would still need substantial adapters. [Autocxx book][autocxx], [template types][autocxx-types], [exceptions][autocxx-features] |
| Direct C++ `bindgen` | Good declaration extraction and useful layout descriptions | Reject as the Eden interface. Its book documents unsupported template methods, automatic destruction, cross-language inheritance, and exceptions, plus invalid cases for nontrivial by-value C++ calls. Use it on the new C facade instead, generating and reviewing bindings in development/CI. [Bindgen C++ support][bindgen-cpp] |

### Maintenance and parity

The local modified Eden copy has no independently pinned library release identified in its call-site/build contract. Treat this checkout as the version, preserving its local passes and compatibility include order. The build explicitly warns that another `Shader::Info` layout previously crashed descriptor traversal. Sources: `nx2/build.py:835–854`, `gpu/vk/vk_shader.cpp:33–50`, `:880–955`.

The relevant binding-tool release history is current as follows. These are published packages, not proposed upgrades to Hookshot.

| Tool | Last published release observed | Maintenance and unresolved limitation |
| --- | --- | --- |
| `cxx` | 1.0.205, 2026-10-11 | Established 1.0 line with recent releases in its [repository][cxx-repo]; explicit subset of C++, not arbitrary-header support. Maps and `Option` remain pending. [Version history][cxx-versions], [supported types][cxx-bindings] |
| `autocxx` | 0.30.0, 2025-03-03 | Book explicitly disclaims active maintenance; template and exception limitations remain documented. [Version history][autocxx-versions], [book][autocxx] |
| `bindgen` | 0.73.2, 2026-09-08 | Active published tool; its [repository][bindgen-repo] documents libclang generation requirements. Documented C++ restrictions remain relevant regardless of release recency. The book links exception tracking issue #1208. [Version history][bindgen-versions], [C++ limitations][bindgen-cpp] |

SPIR-V can stay byte-identical if the same code, profile, runtime settings, callback answers, pass order, and descriptor-binding accumulator reach the unchanged emitter. Moving environment decisions into Rust, changing enum conversions, or resetting the thread-local globals differently risks shader output. Keeping those decisions in C++ is the reason for the coarse facade. Sources: `gpu/vk/vk_shader.cpp:503–548`, `:1425–1506`, `:1515–1567`, `:1646–1717`.

Do not promise byte-identical cache files. The cache stamp hashes `sizeof(Program)`, `sizeof(Info)`, and the C++ library version, while `put_info` writes the native `Info` representation before its lists. A Rust replacement for `Program` can invalidate the stamp; native container representations also prevent a portable deterministic file format. Keep this codec and its native cache-bearing types in C++ for the first stage. An explicit portable cache format would be a separate change. Sources: `gpu/vk/vk_shader.cpp:1020–1031`, `:1260–1300`.

## Sirit

Sirit is both Eden's emitter dependency and a direct first-party dependency. `vk_coopmat.cpp` builds SPIR-V probe shaders with `Sirit::Module`, `Id`, type creation, constants, decorations, and cooperative-matrix operations. `build_shader` returns `m.Assemble()`. The probe results select matrix element order and whether native matrix multiplication is usable. Sources: `gpu/vk/vk_coopmat.cpp:1–10`, `:50–106`; build includes the local mesh-shader patch at `nx2/build.py:212–217`.

Keep Sirit inside the existing native compilation unit. If the probe orchestration moves to Rust, export a C operation that takes a probe kind and parameters and returns an owned word buffer. A `cxx` wrapper over that operation is the runner-up. Exposing all 56 module-method matches would recreate a C++ builder API in Rust, and direct `bindgen` encounters Sirit's inline and templated methods. Sources: `gpu/vk/sirit/include/sirit/sirit.h:44–65`, `:1517`; [CXX bindings][cxx-bindings], [bindgen limitations][bindgen-cpp].

There is no standalone Sirit release pin in Hookshot's build declaration; the local patched snapshot is the reproducible version. This research does not establish a maintained drop-in Rust emitter with identical Sirit instruction ordering. Keep its current snapshot rather than assigning it the maintenance status of a different SPIR-V project. Source: `nx2/build.py:212–217`, `:841–854`.

Probe SPIR-V can remain byte-identical by executing the same builder sequence. A replacement assembler may emit semantically equivalent instructions with different IDs or order, which fails the map's byte gate. It could also alter the native-MMA decision and therefore Eden output. The narrow C operation avoids that change. Sources: `gpu/vk/vk_coopmat.cpp:1–10`, `:63–106`, `gpu/vk/vk_shader.cpp:237–238`.

## Vulkan

The current loader owns process-wide function pointers. It loads `vulkan-1.dll` from System32 on Windows, obtains the instance resolver, and resolves commands at global, instance, and device scopes. The Linux compatibility path maps the DLL name to `libvulkan.so.1`. Extension commands may be null. Sources: `gpu/vk/vk_main.cpp:27–63`, `gpu/vk/vk_loader.h:1`, `:146`.

Recommend `ash` for Rust types and low-level Vulkan calls. Construct its entry from the already-loaded resolver using `Entry::from_static_fn`; load instance/device wrappers for the existing handles. Keep the loader module alive for their full lifetime. Do not switch to `Entry::load()` during the pilot because that would replace Hookshot's loader-search policy. Do not use `linked`, which adds a loader link dependency. `ash` documents custom loaders, raw function tables, optional dynamic loading, and this constructor. Sources: [Ash entry API][ash-entry], [Ash loader documentation][ash-repo], `gpu/vk/vk_main.cpp:38–63`.

The same instance, device, queue, and command buffers can be borrowed by both languages. Device function pointers must be used only with that device or its children, as Vulkan explicitly requires. Name one destruction owner, and make Rust wrappers borrow that owner's lifetime. A clone of a dispatch table must not become a second owner of the actual device. This is a proposed ownership contract based on [Vulkan's resolver rules][vk-proc], the existing `Device` at `gpu/vk/vk_internal.h:37–43`, and the resolver export at `gpu/vk/vk_main.cpp:73`.

Runner-up is checked-in `bindgen` output from a narrow C header that includes the pinned Vulkan headers, plus a hand-managed table matching `VKX_*`. It most closely preserves null checks but creates local generation and API-maintenance work. Do not bind `vk_loader.h` directly as C: it includes Windows compatibility headers, a C++ namespace, and `using namespace`. Sources: `gpu/vk/vk_loader.h:5–15`, `:161–179`; [bindgen][bindgen-cpp].

The latest published `ash` observed is 0.38.0+1.3.281, 2024-04-01. It is an established generated binding with a long gap between published versions. Its README documents generation, extension modules, and unsafe caller obligations. However, Hookshot's Linux header pin is v1.4.345, so an inventory of every used enum, structure, and command against the published crate is an implementation gate. Do not infer support for all 1.4 headers from support for Vulkan 1.3. Sources: [Ash version history][ash-versions], [repository][ash-repo], `scripts/linux/deps.sh:35–37`.

One open upstream issue matters before adopting the published wrapper methods. [Ash #920][ash-920] remains open and discusses uninitialized tails in fixed-size output arrays. The published 0.38.0 `get_physical_device_memory_properties` source uses `MaybeUninit::uninit()` and `assume_init()`. For the pilot, call the raw function pointer with a zero-initialized output struct, as Hookshot currently does through its zero-initialized `Device`, and traverse only the returned counts. Review similar fixed-array output methods. This workaround avoids the affected convenience wrapper; it is not evidence that a newer crate release contains a fix. Sources: [published method source][ash-source], `gpu/vk/vk_internal.h:37–43`, `gpu/vk/vk_main.cpp:72`, `:210`.

Two concrete parity hazards remain. Ash represents missing loaded functions with panic stubs rather than Hookshot's null pointers, so keep explicit enabled-extension/capability guards. Hookshot's `vk_work.h` also increments `work_epoch` before GPU work; bypassing that through Rust can incorrectly remove a later barrier. A Rust command wrapper must preserve those increments. Allocation callbacks, `pNext` contents, feature enables, synchronization, and command order must stay the same for pixel parity. Sources: [Ash loading behavior][ash-repo], `gpu/vk/vk_loader.h:146`, `gpu/vk/vk_work.h:1–20`.

## Vulkan Memory Allocator

The declared VMA version is 3.3.0, from a CPM cache. This research did not inspect any cache or verify the Windows cache's actual header hash. Linux explicitly pins 3.3.0. There is one `VmaAllocator`, with device-address and optional memory-budget flags, Vulkan 1.3, and 64 MiB preferred blocks. Sources: `gpu/vk/vk_vma.cpp:1–30`, `scripts/linux/deps.sh:37`.

The 23 calls implement more than allocation and free. Images use 128 MiB pools by default, separate texture and render-target pools, dedicated-first allocation when requested, within-budget/device-local preferences, and a final system-memory fallback. Buffers distinguish sequential write from cached/random host access, dedicated allocations, mapping, flush, and invalidate. Heap-budget queries advance VMA's frame index. Sources: `gpu/vk/vk_vma.cpp:33–176`, `:180–255`.

Recommend C wrappers over the existing `create_image`, `create_buffer`, mapping/cache operations, and budget queries. Make `VmaAllocation` opaque in Rust. Keep one native allocator and the same destruction ordering throughout the mixed-language phase. The present C++ reference outputs need pointer/out-parameter adapters, not an allocator rewrite. Sources: `gpu/vk/vk_internal.h:184–187`, `:228–249`; `gpu/vk/vk_vma.cpp:100–189`.

`vk-mem` is a credible runner-up at a later whole-allocator handover. Its latest published 0.5.0 is dated 2025-08-12 and explicitly embeds VMA 3.3.0; it depends on `ash` 0.38. Its repository has later VMA 3.4.0 work, so `master` is already a different allocator version. Pin the release rather than the moving branch. Sources: [release notes][vk-mem-release], [package dependencies][vk-mem-versions], [repository history][vk-mem-repo].

There are three integration blockers to treating `vk-mem` as a drop-in wrapper:

1. Its `Allocator` contains a private native pointer; `new()` creates a new allocator and `Drop` destroys it. The inspected 0.5.0 API provides no public borrowed `from_raw` allocator constructor. Creating one beside Hookshot's allocator splits pools and budgets. [Allocator source][vk-mem-source].
2. Its build compiles another VMA implementation and sets both static and dynamic Vulkan loading to zero, supplying Ash functions instead. Hookshot sets dynamic loading to one. Replacing the implementation is possible, but linking both and treating their allocators as interchangeable is not the proposed strangler seam. [Build script][vk-mem-build], [allocator source][vk-mem-source], `gpu/vk/vk_vma.cpp:4–6`. VMA documents both loading mechanisms in [its configuration guide][vma-config].
3. The 0.5.0 build script selects `stdc++` for Windows GNU targets, while Hookshot's llvm-mingw link uses `-lc++ -lunwind`. The compiler, standard library, flags, and final link need explicit integration; this was not built here. [Build script][vk-mem-build], `nx2/build.py:1049`.

`gpu-allocator` 0.28.0, released 2025-09-26, is an actively developed, independent Rust allocator for Vulkan and other APIs. Its repository shows subsequent activity and exposes allocation descriptors and block-size settings. It is not a VMA binding. Reproducing the caller's broad memory preferences would not preserve VMA's placement, pools, dedicated-allocation decisions, fallback sequence, or budget queries. Reject it for the byte-parity port; reconsider only as a separately evaluated optimization. Sources: [package history][gpu-allocator-versions], [repository and allocation API][gpu-allocator].

Upstream VMA is maintained: the release page lists 3.5.0 dated 2026-10-26, with 3.3.0 dated 2025-05-12. A vendor upgrade would therefore be a real algorithm/configuration change, not binding housekeeping. Keep 3.3.0 and the exact call ordering for the pilot. Allocation placement is only expected to match under the same driver and allocation history; addresses are not a portable byte contract. Sources: [VMA releases][vma-releases], `gpu/vk/vk_vma.cpp:17–189`.

## Dear ImGui

Hookshot vendors unmodified Dear ImGui 1.92.9b and the upstream Vulkan backend. Hookshot draws directly onto `ImDrawList`, supplies its own input/menu logic, and performs texture copies without the backend's queue waits. Its font baking and logo use 1.92-era texture APIs and internal calls. Sources: `third_party/imgui/PROVENANCE.md:1–10`, `third_party/imgui/imgui.h:30–35`, `gpu/vk/vk_ui.h:44–97`, `gpu/vk/vk_imgui.cpp:18–21`, `:290–325`, `:335–343`. The native project is actively releasing: 1.92.9b dates to 2026-07-31 and latest 1.93.0 to 2026-10-14. [Upstream release history][imgui-native].

Recommend keeping `vk_menu.cpp`, `vk_ui.h`, and `vk_imgui.cpp` together in C++ for the early stages. Export operations to update menu state and record overlays into the shared command buffer, with fixed-width POD state. Rust does not need to own `ImGuiContext`, `ImFont`, `ImTextureData`, or the draw lists to port replay, textures, or memory. This keeps today's drawing and font path intact. Sources: `gpu/vk/vk_imgui.h:19–36`, `gpu/vk/vk_menu.cpp:1188`, `:1744–1765`, `:2552`.

If menu logic later moves, the runner-up is `cimgui` regenerated against exactly the vendored 1.92.9b sources and configuration. It emits a C API for ImGui and optionally its internals, permits choosing the underlying revision, and documents MinGW and clang generation. It is not a pure-Rust UI replacement. Its latest formal GitHub release is the old v1.53.1, 2018-01-02, but its repository has 2026 commits and generated 1.93 work; judging maintenance by the release badge alone is misleading. Pin generated output and the underlying source revision together. Source: [cimgui repository and generator instructions][cimgui].

Even `cimgui` is not a complete off-the-shelf solution for these call sites. The menu constructs a temporary draw list, calls `_ResetForNewFrame`, uses font texture references, and draws a warmup frame. The backend manages dynamic rendering and Hookshot rewrites colors for sRGB/HDR and handles texture updates. Keep those operations in custom C++ wrappers unless exact matching generated APIs have been inspected. Sources: `gpu/vk/vk_menu.cpp:1753–1765`, `gpu/vk/vk_imgui.cpp:301–325`, `:394–429`.

The Rust wrapper alternatives have different tradeoffs:

| Option | Release and maintenance | Blocker for unchanged Hookshot |
| --- | --- | --- |
| `dear-imgui-rs` | Latest published 0.22.0, 2026-10-31. Active releases; safe API plus `dear-imgui-sys`/cimgui and an Ash backend. Current docs explicitly support llvm-mingw source builds on `x86_64-pc-windows-gnullvm` | Published stack is Dear ImGui 1.93.1 WIP docking, not Hookshot's 1.92.9b snapshot. Safe wrappers do not remove native version, configuration, internal API, or backend differences. Current core requires Rust 1.95. Useful future candidate, but adopting it as-is changes the vendor. [Published documentation][dear-imgui-versions], [repository][dear-imgui] |
| `imgui-rs` / crate `imgui` | Latest published 0.12.0, 2024-05-05. Established wrapper with later repository activity | Its published generation script pins Dear ImGui 1.89.2, predating the texture/font path used here; its README's Windows instructions require the MSVC Rust ABI. No tested llvm-mingw integration was established here. An older wrapper cannot safely access newer struct layouts just because both use C function names. [Published history][imgui-versions], [pinned generation script][imgui-pin], [repository and Windows notes][imgui-rs], `gpu/vk/vk_imgui.cpp:294–325` |
| Another UI or Rust renderer backend | Would replace at least drawing/backend behavior | No byte-identical pixel claim is justified when text metrics, atlas packing, vertex emission, blend state, or texture upload behavior changes. Current behavior is explicit in `gpu/vk/vk_ui.h:54–97` and `gpu/vk/vk_imgui.cpp:335–429` |

Keep stb_truetype rather than enabling FreeType, preserve font sources and rasterizer settings, glyph baking order, draw order, scale rounding, color conversion, and backend state. These are observable pixel inputs. Existing source-text tests cover menu behavior but cannot certify pixel equality. A later menu port needs synthetic-font draw-data comparisons and local rendered comparisons at the same resolution, DPI, timing, and HDR mode. Sources: `gpu/vk/vk_game_font.h:17–31`, `gpu/vk/vk_imgui.cpp:45–87`, `:335–400`, `tests/test_menu_utf8.py:1–6`.

## FidelityFX

This checkout contains FidelityFX SDK 1.1.4 GPU headers and GLSL sources, pinned to `c6efa6bf7f2027b3ec94f28578bb5965eabb9e55`. It deliberately includes no runtime library. Those C-looking headers feed shader compilation, not a host ABI. The first-party C++ implements resource selection and pass recording itself. Sources: `third_party/fidelityfx-sdk/PROVENANCE.md:5–26`, `scripts/fsr3_shaders.py:19–38`, `gpu/vk/vk_fsr3.cpp:1–6`.

No SDK C shim is needed. During the strangler, retain the existing first-party FSR recorder behind a C operation. When it moves, consume the exact checked-in SPIR-V words in Rust through `ash`; generate Rust static data or link a small C asset getter. Avoid recompilation merely to change the host language. The generator's existing `--check` mode is the later source-to-asset check, not a command run during this research. Sources: `scripts/fsr3_shaders.py:1–9`, `gpu/vk/vk_fsr3.cpp:28–36`, `gpu/vk/vk_fsr3fi.cpp:26–42`, `tests/test_fsr3_shaders.py:1–7`.

The nontrivial boundary is the GPU data layout. `vk_fsr3.cpp` asserts a 148-byte constant block. Interpolation asserts 176 bytes, and optical flow and its downsampler each assert 32. Preserve individual field offsets, not only total size, descriptor ordering, formats, history reset/rotation, dispatch dimensions, barriers, and floating-point calculation order. A Rust `repr(C)` host struct alone does not establish a GLSL std140 layout. The current manual layouts and `spd_setup` calculations are the parity oracle. Sources: `gpu/vk/vk_fsr3.cpp:111–145`, `gpu/vk/vk_fsr3fi.cpp:164–229`, `:604–617`.

AMD continues SDK development, but current upstream releases use a newer SDK/API organization than the pinned 1.1.4 source route. Their availability is not a reason to change shader assets in this port. The latest release observed is FSR SDK 2.4.0, 2026-10-28, whose release notes describe newer DLL-based ML technologies. [AMD releases][ffx-releases]. There is no missing Rust crate blocking today's use: this dependency is checked-in shader data. Sources: [AMD repository][ffx], `third_party/fidelityfx-sdk/PROVENANCE.md:5–26`.

Shader bytes can remain identical by retaining the assets. Pixels additionally depend on the recorder and its constants. Replacing this with a crate that calls AMD's newer host runtime would change architecture, pass scheduling, and potentially shader output; it is outside this parity port. Sources: `third_party/fidelityfx-sdk/PROVENANCE.md:20–26`, `gpu/vk/vk_fsr3.cpp:1–6`, `gpu/vk/vk_fsr3fi.cpp:1–8`.

## NVIDIA NGX

NGX is optional and absent from the repository. Hookshot compiles only `vk_dlss.cpp` against the local SDK, otherwise building its stubs. The fetch script pins v310.9.16 and individual file hashes. Windows links NVIDIA's static MSVC-runtime library with existing compatibility shims; Linux links `libnvsdk_ngx.a`. These are existing integration obligations, not problems that Rust wrappers remove. Sources: `nx2/build.py:181–209`, `:846–848`, `:894–896`, `scripts/ngx_sdk.py:25–77`, `gpu/vk/vk_ngx_msvc_rt.cpp:1–7`.

Recommend C wrappers around the existing integration, including init/capability reporting, feature creation/evaluation/release, and the same no-SDK stubs. Pass Vulkan handles, scalar options, and explicit resource descriptions. Keep SDK helper functions and parameter objects in C++. The four `NGX_*` helpers build/query/evaluate DLSS, while `NVSDK_NGX_*` expressions access parameters, construct image resources, and manage feature lifetime. Binding only external symbols would omit important header-helper behavior. Sources: `gpu/vk/vk_dlss.cpp:276–278`, `:398–446`, `:750–810`, `:1166`.

Runner-up is generated bindings to that narrow C wrapper header, not direct binding of the entire SDK C++ interface. `bindgen` is useful for C declarations but its exception, template, and inline-method limitations still apply. `cxx` offers little here because the rest of the renderer should not acquire ownership of NGX's C++ parameter objects. Sources: [bindgen limitations][bindgen-cpp], [CXX bindings][cxx-bindings], `gpu/vk/vk_dlss.cpp:178–202`.

NVIDIA's latest formal SDK release observed is v310.9.16, 2026-10-16, matching Hookshot's pin. The proprietary runtime is loaded separately and Hookshot picks the newest one in its configured directories, so pinning the static SDK alone does not freeze DLSS pixels. Preserve preset K/default/other choices, resource flags, motion/depth convention, history reset, output conversion, and lifetime delays. Sources: [NVIDIA releases][ngx], `scripts/ngx_sdk.py:25–77`, `gpu/vk/vk_dlss.cpp:25–32`, `:223–245`, `:778–810`.

The outstanding integration constraints are the Windows runtime shims, Vulkan device lineage, SDK/runtime availability, and Linux's NGX path behavior. Hookshot already works around non-ASCII data paths through `/proc/self/fd` and separately diagnoses an un-loadable runtime real path. Retain these implementations. A C facade can preserve current behavior on the same driver/runtime/input sequence; neither C nor Rust can promise byte-identical DLSS output across driver or runtime upgrades. Sources: `gpu/vk/vk_ngx_msvc_rt.cpp:1–7`, `gpu/vk/vk_dlss.cpp:327–340`, `:420–443`; [Vulkan device-pointer rules][vk-proc].

## zstd

Hookshot vendors the zstd 1.5.7 decoder only, with `ZSTD_DISABLE_ASM`, `ZSTD_LEGACY_SUPPORT=0`, and `ZSTD_MULTITHREAD=0`. The font reader checks frame content size, rejects unknown/error sizes and values above 256 MiB, creates a decode context per operation, and optionally uses a prebuilt dictionary. It requires the decoded size to equal the declared size. Sources: `third_party/zstd/PROVENANCE.md:1–6`, `nx2/build.py:869–878`, `gpu/vk/vk_game_font.h:49–59`, `:104–129`.

Recommend leaving this native decoder and either exposing the font reader as a C operation or declaring its eight used C functions in Rust. The native C API already is the desired seam; no C++ binding system is necessary. Preserve dictionary ownership/destruction and the exact size/error policy. Sources: `gpu/vk/vk_game_font.h:50–59`, `:110–119`, `third_party/zstd/lib/zstd.h:112–115`.

The runner-up, `zstd`, is an established Rust wrapper over native zstd through `zstd-safe` and `zstd-sys`. Latest published versions observed are `zstd` 0.14.3 and `zstd-sys` 2.1.2+zstd.1.5.7, both 2026-10-16. That native version matches Hookshot, but the wrapper is not automatically a binding to Hookshot's existing objects. Its package/build can compile another native library or select system zstd. Sources: [zstd package history and features][zstd-versions], [zstd-sys package/build documentation][zstd-sys], [repository][zstd-rs].

Defaults matter. Current wrapper docs enable legacy decoding, arrays, and dictionary training by default; `no_asm` disables assembly, and `pkg-config` selects a system library. Use a bounded one-shot decompressor with an explicit dictionary, rather than unbounded `decode_all`, and retain the content-size precheck and exact-size postcheck. Otherwise a malformed, unknown-size, or concatenated input can take a different allocation/error path even if ordinary fonts decode identically. Sources: [published zstd feature documentation][zstd-versions], [zstd-sys build options][zstd-sys], `gpu/vk/vk_game_font.h:50–59`.

Allocation parity also requires preserving context/dictionary reuse and native allocation callbacks. The current `with-rust-allocator` option changes allocation; upstream's experimental `rust-backend` replaces the C implementation and changes allocation and feature behavior. Neither belongs in this port. Valid decompressed bytes can match with the same decoder and dictionary, but high-level wrappers do not promise the same heap traffic, errors, or input acceptance. Sources: [zstd allocator and backend documentation][zstd-rs], [published allocator feature][zstd-versions], `gpu/vk/vk_game_font.h:50–59`.

The upstream release page still lists zstd 1.5.7, 2025-02-19, as latest. [Native zstd releases][zstd-releases]. No documented C API obstacle blocks these eight calls. The relevant unresolved choice is build/link ownership, so that a crate does not silently supply a second decoder or enable different features. Sources: [zstd-sys linking documentation][zstd-sys], `nx2/build.py:869–878`.

## Implementation gates for the planning map

These are proposed follow-up checks, not checks executed during research:

1. Freeze the native dependency identities and build flags, including the actual VMA header hash. Compile each new C header as C on both platforms, with native-owned opaque objects and no exceptions escaping. The existing layout crash and explicit C++ build dependencies justify this first step. Sources: `nx2/build.py:835–878`, `gpu/vk/vk_shader.cpp:1020–1031`; [bindgen exception rules][bindgen-cpp].
2. Establish one device, allocator, ImGui context, and NGX feature owner, with destruction after queued GPU work. Export fixed-width POD records and paired destruction functions. Preserve the original device's resolver and the native library lifetime. Sources: `gpu/vk/vk_main.cpp:27–73`, `gpu/vk/vk_vma.cpp:15–30`, `gpu/vk/vk_imgui.cpp:290`, `gpu/vk/vk_dlss.cpp:223–245`; [Vulkan resolver rules][vk-proc].
3. For Eden/Sirit, compare synthetic input translation and probe word arrays exactly, plus ordered reflection and handle evaluation. For FFX, compare shader asset hashes and serialized constant bytes. For zstd, use synthetic dictionaries/frames covering the existing limits and errors. Existing source and fixtures provide these seams. Sources: `gpu/vk/vk_shader.cpp:605–643`, `:1490–1506`, `gpu/vk/vk_coopmat.cpp:63–106`, `tests/test_fsr3_shaders.py:1–7`, `tests/test_game_font.py:1–19`.
4. Compare ImGui vertex/index/command data and synthetic-font atlas bytes with fixed timing and scale, then use the map's local game checks for pixels, allocation behavior, and NGX history. Keep game-derived artifacts local. Source-text tests alone do not cover those outputs. Sources: `gpu/vk/vk_imgui.cpp:335–429`, `tests/test_menu_utf8.py:1–6`, `.scratch/rust-port/map.md:25–29`, `:41–46`, `CONTRIBUTING.md:11–25`.

## Reproducing the source counts

These commands only read the named source directories. The method filter is deliberately explicit and approximate: it counts call-shaped source matches, including inline code, rather than using a C++ parser. It does not measure performance or execute dependencies.

```sh
rg -c 'Shader::' gpu/vk/*.cpp
rg -c 'Sirit::' gpu/vk/*.cpp
rg -l 'vk_menu\.cpp' tests/test_*.py
```

```python
from collections import Counter
from pathlib import Path
import re

root = Path("gpu/vk")
cpp = sorted(root.glob("*.cpp"))
headers = sorted(root.glob("*.h"))
for label, pattern, files in [
    ("Vulkan", r"\bvk[A-Z]\w*\s*\(", cpp),
    ("VMA", r"\bvma[A-Z]\w*\s*\(", cpp + headers),
    ("NGX", r"\b(?:NVSDK_NGX_|NGX_)\w*\s*\(", cpp + headers),
    ("zstd", r"\bZSTD_\w*\s*\(", cpp + headers),
    ("FidelityFX", r"\bffx[A-Z]\w*\s*\(", cpp + headers),
]:
    counts = Counter()
    for path in files:
        matches = re.findall(pattern, path.read_text())
        if matches:
            print(label, path.name, len(matches))
            counts.update(match.strip().rstrip("(").strip() for match in matches)
    print(label, "total", counts.total(), "distinct", len(counts))

methods = r"(?:->|\.)(Add\w+|Push\w+|Pop\w+|Path\w+|Prim\w+|Calc\w+|GetFontBaked|GetCustomRect|FontBaked\w+|SetTexID|GetTexID|Clear|_\w+)\s*\("
for name in ["vk_ui.h", "vk_menu.cpp", "vk_imgui.cpp"]:
    print(name, "drawing/font method matches", len(re.findall(methods, (root / name).read_text())))
```

## Primary web sources

All linked sources were read through Firecrawl on the research date. Published package pages take precedence over a repository's stale README version or GitHub release badge.

[cxx-repo]: https://github.com/dtolnay/cxx
[bindgen-repo]: https://github.com/rust-lang/rust-bindgen
[imgui-native]: https://github.com/ocornut/imgui/releases
[imgui-pin]: https://docs.rs/crate/imgui-sys/0.12.0/source/third-party/update-imgui.sh
[cxx-bindings]: https://cxx.rs/bindings.html
[cxx-errors]: https://cxx.rs/binding/result.html
[cxx-versions]: https://docs.rs/crate/cxx/1.0.205
[autocxx]: https://google.github.io/autocxx/
[autocxx-types]: https://google.github.io/autocxx/cpp_types.html
[autocxx-features]: https://google.github.io/autocxx/other_features.html
[autocxx-versions]: https://docs.rs/crate/autocxx/0.30.0
[bindgen-cpp]: https://rust-lang.github.io/rust-bindgen/cpp.html
[bindgen-versions]: https://docs.rs/crate/bindgen/0.73.2
[ash-repo]: https://github.com/ash-rs/ash
[ash-versions]: https://docs.rs/crate/ash/0.38.0+1.3.281
[ash-920]: https://github.com/ash-rs/ash/issues/920
[ash-source]: https://docs.rs/ash/0.38.0+1.3.281/src/ash/instance.rs.html#442-452
[ash-entry]: https://docs.rs/ash/0.38.0+1.3.281/ash/struct.Entry.html
[vk-proc]: https://docs.vulkan.org/refpages/latest/refpages/source/vkGetDeviceProcAddr.html
[vk-mem-release]: https://github.com/gwihlidal/vk-mem-rs/releases/tag/0.5.0
[vk-mem-versions]: https://docs.rs/crate/vk-mem/0.5.0
[vk-mem-repo]: https://github.com/gwihlidal/vk-mem-rs
[vk-mem-source]: https://docs.rs/vk-mem/0.5.0/src/vk_mem/lib.rs.html
[vk-mem-build]: https://docs.rs/crate/vk-mem/0.5.0/source/build.rs
[vma-config]: https://gpuopen-librariesandsdks.github.io/VulkanMemoryAllocator/html/configuration.html
[vma-releases]: https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/releases
[gpu-allocator-versions]: https://docs.rs/crate/gpu-allocator/0.28.0
[gpu-allocator]: https://github.com/Traverse-Research/gpu-allocator
[cimgui]: https://github.com/cimgui/cimgui
[dear-imgui-versions]: https://docs.rs/crate/dear-imgui-rs/0.22.0
[dear-imgui]: https://github.com/Latias94/dear-imgui-rs
[imgui-versions]: https://docs.rs/crate/imgui/0.12.0
[imgui-rs]: https://github.com/imgui-rs/imgui-rs
[ffx]: https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK
[ffx-releases]: https://github.com/GPUOpen-LibrariesAndSDKs/FidelityFX-SDK/releases/tag/v2.4.0
[ngx]: https://github.com/NVIDIA/DLSS/releases/tag/v310.9.16
[zstd-versions]: https://docs.rs/crate/zstd/0.14.3
[zstd-sys]: https://docs.rs/crate/zstd-sys/2.1.2+zstd.1.5.7
[zstd-rs]: https://github.com/gyscos/zstd-rs
[zstd-releases]: https://github.com/meta/zstd/releases/tag/v1.5.7
