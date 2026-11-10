# Interop between a Rust renderer and the vendored C/C++

Type: research
Status: resolved
Blocked by: none
Research: branch `research/vendored-interop` on Rhinomcd/Hookshot, `.scratch/rust-port/research/02-interop-with-vendored-cpp.md`

## Question

For each library the renderer links, should a Rust renderer call the existing C/C++ through a shim, or switch to a Rust
crate? What does each choice cost?

- **Eden shader recompiler** (42.9K lines, about 26 files with local changes): `Shader::` is used 159 times in
  `vk_shader.cpp`, 40 times in `vk_draw.cpp` and 11 times in `vk_coopmat.cpp`. Compare a hand-written C API, `cxx`,
  `autocxx` and `bindgen`.
- **Vulkan:** `ash`, or raw bindings that match the current loader (`gpu/vk/vk_loader.h`).
- **VMA:** `vk-mem` bindings, or `gpu-allocator` (which would change allocation behavior, a parity risk).
- **Dear ImGui:** keep it in C++ (`cimgui` or `dear-imgui-rs`), or something else. The menu is `vk_menu.cpp`, with 19
  text tests.
- **FidelityFX** (pregenerated SPIR-V headers plus C headers) and **NGX** (a static, optional C++ SDK): C shims only?
- **zstd** (decoder used for the game font): the `zstd` crate, or keep the vendored decoder.

For each, report the binding approach, how mature it is, and whether its behavior can stay byte-identical to today's.

Primary sources: each crate's docs and repo, the cxx/autocxx books, and the repo's call sites.

## Answer

Keep every vendored library native and put the seam around whole renderer operations, not around each C++ method.
Findings: branch `research/vendored-interop` (fbad491), `.scratch/rust-port/research/02-interop-with-vendored-cpp.md`.

| Library | Binding | Why |
| --- | --- | --- |
| Eden + Sirit | A hand-written C facade over today's translation glue. It returns an opaque owner, borrowed SPIR-V words and flat reflection records. | `Shader::Info` holds vectors, maps and bitsets, so `repr(C)` cannot mirror it. Today's code relies on Eden's `Env` callbacks and the shader cache codec, which keep those in C++. `autocxx` is unmaintained. `cxx` is the runner-up. |
| Vulkan | `ash`, built from the existing resolver with `Entry::from_static_fn`. | Keeps today's loader search. Ash 0.38 covers Vulkan 1.3.281 headers, but the repo pins 1.4.345. |
| VMA | C wrappers over the one existing allocator. | `vk-mem` would create a second allocator and split pools and budgets. `gpu-allocator` changes where allocations land. |
| ImGui | Keep `vk_menu.cpp`, `vk_ui.h` and `vk_imgui.cpp` in C++ through the early stages, behind frame and menu operations. | Every Rust wrapper pins a different ImGui version than the vendored 1.92.9b. |
| FidelityFX | No runtime binding. The pass recorder stays C++ until ported; a port reuses the same checked-in SPIR-V. | It is shaders only. |
| NGX | A C facade over today's integration and its stubs. | — |
| zstd | Declare the eight C decoder functions in Rust. | — |

**Gates for the pilot spec:**

- One native owner each for the device, the allocator, the ImGui context and NGX. Rust borrows them.
- If the pilot calls Vulkan, inventory every symbol it uses against ash 0.38.
- Call fixed-array output queries through raw function pointers with zeroed structs (ash issue #920).
- Keep the `work_epoch` increments from `vk_work.h`, or a later barrier gets dropped.
- Ash fills missing functions with stubs that panic instead of null pointers, so keep the capability guards.
- Shader-cache files will not stay byte-identical if Rust replaces `Program`. Keep the cache codec in C++.
