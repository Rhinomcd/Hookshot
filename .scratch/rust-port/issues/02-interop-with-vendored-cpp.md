# Interop between a Rust renderer and the vendored C/C++

Type: research
Status: open
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
