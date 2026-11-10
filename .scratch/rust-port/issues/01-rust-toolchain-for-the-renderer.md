# Rust toolchain for the renderer's two targets

Type: research
Status: resolved
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

## Answer

Yes, with conditions. Findings: branch `research/rust-toolchain` (fc1f26f), `.scratch/rust-port/research/01-rust-toolchain-for-the-renderer.md`.

- **Targets:** `x86_64-pc-windows-gnullvm`, which matches UCRT llvm-mingw (`windows-gnu` defaults to MSVCRT and GCC), and
  `x86_64-unknown-linux-gnu`.
- **Build split:** `nx2/build.py` keeps compiling the retained C++ (Eden, Sirit, ImGui, VMA). Cargo drives the final
  link through the same clang driver, carrying today's link recipe forward unchanged. That recipe includes the NGX MSVC
  shims (`vk_ngx_msvc_rt.cpp`, `host/mingw_msvc_shims.c`). FidelityFX is shaders only, with no host library.
- **Exports:** the ABI is unchanged at version 12, with both version checks kept. Export `mkwvk_init` and also
  `mkwvk_close_asked`, which the ticket missed (`vk_main.cpp:964-987`, looked up in `host/rx_backend.c:280-285`). Only
  one side may define each export while the port is partial.
- **Failures:** `panic=abort`. C++ exceptions (Eden throws) are caught in C++ wrappers and turned into results before
  they reach Rust. A foreign exception crossing `extern "C"` is undefined behavior even under abort.
- **Linux:** the `dlsym(RTLD_DEFAULT, "hos_posix_*")` back-channel works unchanged from Rust.
- **CI:** a pinned minimal rustup in the `vulkan`, `vulkan-dlss` and AppImage jobs. The added minutes are unmeasured, so
  the pilot measures them.
- **New fact:** Windows players build `hookshot_vk.dll` on their own PC; CI only checks that it builds
  (`.github/workflows/windows-builder.yml:5`). The gate's "nothing new for users to install" therefore needs a prebuilt
  Windows DLL delivered to players. That is ticket 10. `HOOKSHOT_VK_PREBUILT` (`nx2/build.py:817`) already accepts one.
- **Pilot spec:** before any port work, add a game-free smoke check on both platforms. A stub C host loads the Rust
  library and checks the ABI layout, the version rejection, the NGX variant and the Linux back-channel.
