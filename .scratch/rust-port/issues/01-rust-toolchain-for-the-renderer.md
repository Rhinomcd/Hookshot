# Rust toolchain for the renderer's two targets

Type: research
Status: open
Blocked by: none
Research: branch `research/rust-toolchain` on Rhinomcd/Hookshot, `.scratch/rust-port/research/01-rust-toolchain-for-the-renderer.md`

## Question

Can a Rust `cdylib` replace `hookshot_vk.dll` (llvm-mingw, `x86_64-w64-mingw32`, lld, `-static`, C++20 static libc++)
and `hookshot_vk.so` (Linux, clang + lld)?

1. Which Rust target matches llvm-mingw: `x86_64-pc-windows-gnullvm` or `x86_64-pc-windows-gnu`? Can the crate link
   statically with Eden, ImGui, VMA, FidelityFX and NGX built by the same llvm-mingw, in one DLL?
2. How does it export `mkwvk_init` with the C ABI (`gpu/vk/vk_api.h:185-190`), and how is the version check kept?
3. Panic strategy at the C boundary (`panic=abort` versus `catch_unwind` per entry point), and how C++ exceptions
   (Eden throws) cross a Rust frame.
4. What has to change in `windows-builder.yml`, `vulkan-dlss.yml` and `linux-appimage.yml`: the rustup install, a
   cache, and the added build minutes.
5. The Linux back-channel: the `.so` finds `hos_posix_*` with `dlsym(RTLD_DEFAULT)` (`gpu/vk/vk_memory.cpp:265`).
   Does that still work from Rust?

Primary sources: rustc target docs, the llvm-mingw README, cargo/rustc linker docs, the repo's `nx2/build.py:835-1048`
and `.github/workflows/`.
