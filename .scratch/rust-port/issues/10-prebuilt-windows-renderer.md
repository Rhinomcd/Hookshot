# Deliver a prebuilt Windows renderer to players

Type: grilling
Status: open
Blocked by: none

## Question

Windows players compile `hookshot_vk.dll` on their own PC with llvm-mingw. The `vulkan` job in
`.github/workflows/windows-builder.yml` only checks that it builds, and the package job does not consume its artifact. A
Rust renderer must not make players install Rust. How does a prebuilt DLL reach them?

- **Options:**
  - Ship the DLL in the builder's update parts and point `HOOKSHOT_VK_PREBUILT` (`nx2/build.py:817`) at it. The Linux
    AppImage already works this way.
  - Bundle a pinned Rust toolchain as a builder part.
  - Keep both paths.
- **DLSS:** the DLSS build links NVIDIA's NGX SDK, and `vulkan-dlss.yml` forbids uploading the SDK. May a prebuilt DLL
  that has NGX linked in be shipped, or does DLSS keep a build-on-PC path?
- **Versions:** how a prebuilt DLL stays in step with an exe built on the player's PC from a different commit, given
  the `MKWVK_API_VERSION` check.

Facts: research for the toolchain ticket, branch `research/rust-toolchain`, section 4.
