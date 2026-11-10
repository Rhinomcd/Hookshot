# Which language owns the renderer's outer shell during the port

Type: grilling
Status: open
Blocked by: none

## Question

While the renderer is partly ported, is it a C++ library with Rust inside, or a Rust library with C++ inside?

- **Rust inside C++:** the C++ `hookshot_vk` keeps `mkwvk_init`, `mkwvk_close_asked` and the `MkwVkRenderer` table. Each
  ported component is a Rust `staticlib` behind a C header that the C++ calls. The link stays in `nx2/build.py`. The
  shell flips to Rust only when little C++ is left.
- **C++ inside Rust:** the toolchain research's `cdylib` shape. Rust exports the table from day one and calls the
  retained C++ through C facades. Cargo drives the final link.

Things to weigh:

- Only one side may define each export.
- Ownership: one native owner each for the device, VMA, ImGui and NGX, with Rust borrowing them (interop research).
- Which shape lets the pilot ship without touching `vk_api.h`.
- Where the panic and exception wrappers sit.
- What changes in the CI jobs and in the Windows prebuilt delivery (ticket 10).

Facts: the research for the toolchain ticket (`research/rust-toolchain`) and the interop ticket
(`research/vendored-interop`).
