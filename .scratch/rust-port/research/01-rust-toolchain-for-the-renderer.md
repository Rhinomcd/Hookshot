# Rust toolchain for the renderer's two targets

Research date: 2026-11-10. Repository evidence is from `f9f8cf53758e982ef573003394981dc2cd121ac9`.
This is a reading-only finding. No Rust installation, compilation, runtime check, or inspection of `work/` or `/home/rhino/Games/` took place.
Official web sources were read with the Firecrawl CLI. Toolchain recipes below are proposals, not executed commands.

## 1. Target and native-library compatibility

**Answer:** Use `x86_64-pc-windows-gnullvm` for Windows and `x86_64-unknown-linux-gnu` for Linux. A Rust `cdylib` can contain the retained native objects. The Windows link needs the existing llvm-mingw runtime selection and NGX shims, not just the right target name.

The Rust target guide explicitly describes `windows-gnullvm` as UCRT plus LLVM tools and libraries, names llvm-mingw as a supported C toolchain, supports native and cross compilation, and supplies both `std` and host tools. It also specifies the Windows C calling convention. The platform table places x86-64 gnullvm in Tier 2 with host tools. In contrast, the `windows-gnu` guide names GCC, GNU Binutils, and **MSVCRT as its default**. Choosing `windows-gnu` because the C compiler says `mingw32` would select the wrong defaults for this project. [R1], [R2], [R3]

The ticket's `nx2/toolchain.py` does not exist at this revision. `nx2/build.py:38-44` imports `kit` before `toolchain`; `nx2/kit.py:10-15` adds the vendored tools directory to `sys.path`. The implementation to preserve is `third_party/a64-kit/tools/toolchain.py`:

- Lines 16-26 select `HOOKSHOT_MINGW` and its LLVM tools.
- Lines 54-57 select `--target=x86_64-w64-mingw32`.
- Lines 60-76 select lld, `-static`, and suppression of MSVC default libraries through lld's MinGW driver.
- `scripts/make_setup_parts.py:28-30` pins `llvm-mingw-20261104-ucrt-x86_64`.

The llvm-mingw README supports C++ exceptions, but warns that its CRT and C++ standard library differ from other MinGW toolchains. It explicitly rejects general object and static-library compatibility with other toolchains. Across DLLs, it requires CRT resources to stay with their owning allocator. That supports rebuilding the retained C++ with the same pinned llvm-mingw and keeping C++ objects and ownership behind C wrappers. It does **not** prove that arbitrary GCC or MSVC archives can join the link. [R4]

The actual dependency situation is more specific than the ticket's list:

| Dependency | Current evidence | Consequence for Rust |
| --- | --- | --- |
| Eden and Sirit | `nx2/build.py:212-223,831-845,882-891` compile their sources as C++20 with the renderer's compiler and flags. | Keep those objects behind C wrappers, with opaque handles and explicit destruction. |
| ImGui | `nx2/build.py:857-868` compiles its five translation units with the same compiler and renderer flags. | Retain that build and wrap its C++ interface. |
| VMA | `gpu/vk/vk_vma.cpp:1-8` defines `VMA_IMPLEMENTATION` and includes `vk_mem_alloc.h`; the include directory comes from `nx2/build.py:845`. | Keep exactly one implementation translation unit, or deliberately replace it. |
| FidelityFX | `gpu/vk/vk_fsr3.h:1-5` and `gpu/vk/vk_fsr3fi.h:1-8` describe vendored shaders compiled to SPIR-V and Hookshot's own dispatches. No AMD host library is linked. | Preserve the shader assets and dispatch contracts. There is no existing FidelityFX host archive to make ABI-compatible. |
| NGX | `nx2/build.py:181-184,203-209,892-897` uses NVIDIA's prebuilt MSVC static-runtime `nvsdk_ngx_s.lib` on Windows, with `vk_ngx_msvc_rt.cpp` and `host/mingw_msvc_shims.c`. | NGX is an existing, narrow MSVC compatibility exception. Keep both shims and every default-library exclusion. It is not built by llvm-mingw. |

The Rust Reference defines `cdylib` for a dynamic system library loaded by another language, and permits native libraries through `-L` and `-l`, and native objects through `-C link-arg=file.o`. Cargo build scripts provide link-search, link-library, and cdylib-specific link-argument directives. Their ordering can affect archive resolution. [R5], [R6]

Proposed build integration:

1. Keep `nx2/build.py` as the coordinator for retained C++ compilation and its object cache. Give Cargo the resulting objects or archives and their dependency stamps. Avoid compiling them twice.
2. Let Cargo/rustc perform the `cdylib` link through a driver from the same toolchain. Set `CARGO_TARGET_X86_64_PC_WINDOWS_GNULLVM_LINKER` to llvm-mingw's target-prefixed clang driver, or a wrapper around the current clang invocation. Set `CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER` to the selected Linux `clang++`. Cargo documents these overrides as `-C linker`; rustc distinguishes a compiler driver from the underlying linker. [R7], [R8]
3. For Windows, carry forward `-fuse-ld=lld`, the current static native-library selection, `-luser32`, `-lc++`, `-lunwind`, and the four MSVC exclusions from `native_link()`. With NGX, also carry its archive, `-ladvapi32`, `/nodefaultlib:libcpmt`, and both shim objects. These are the current requirements in `nx2/build.py:203-209,892-897,1048-1049` and `third_party/a64-kit/tools/toolchain.py:60-76`. Pass driver flags with `-C link-arg`, not directly as rustc flags. [R8]
4. For Linux, preserve PIC native objects, clang plus lld, `-Bsymbolic`, garbage collection, `-static-libstdc++`, `-static-libgcc`, optional NGX, and `-ldl -lpthread -lm`. Preserve `HOOKSHOT_SYSROOT` in compiler and linker arguments. These come from `nx2/build.py:56-64,106-108,831,1042-1046`. Keep glibc dynamic. The current Linux link does not use Windows' global `-static`.
5. Use ordinary native object files for the first interop check. Cross-language LLVM bitcode LTO adds compiler and linker version constraints that the rustc LTO guide explicitly documents. Matching the C ABI does not establish bitcode compatibility. [R9]

Do not equate the existing Windows `-static` with a completely static Windows C runtime. The existing DLSS workflow accepts UCRT API-set DLL imports, and rejects other non-system imports, in `.github/workflows/vulkan-dlss.yml:108-117`. Likewise, `-C target-feature=+crt-static` is a separate Rust runtime control, whose support depends on the target. It is not a replacement for the native-link recipe. Rust's Reference recommends inspecting the resulting binary rather than assuming the requested runtime linkage took effect. [R5]

**Open risk:** The sources establish a supported target and a viable linking mechanism, not a successful link of this exact crate and toolchain. A later game-free smoke build must check exports, imports, native archive ordering, compiler runtime resolution, and the NGX variant. Do not promise a single DLL with no extra runtime dependencies until that check passes. Native allocation and destruction must remain paired even inside a mixed Rust and C++ library. [R4]

## 2. Export and version handshake

**Answer:** Keep the header and its layout unchanged. Define the Rust entry point with an unmangled symbol and the C ABI:

```rust
#[unsafe(no_mangle)]
pub unsafe extern "C" fn mkwvk_init(host: *const MkwVkHost) -> *const MkwVkRenderer
```

This is a signature sketch, not a complete implementation. `no_mangle` gives the symbol its identifier and exports it publicly from the produced library. `extern "C"` chooses the platform C calling convention. `#[repr(C)]` gives the two tables, their nested by-value records, and other shared records the C field-layout rules. The FFI bindings must also preserve the C field types, pointer signatures, and nullable callback representations. Rust's default struct representation is insufficient. [R10], [R11], [R12]

The existing contract is `gpu/vk/vk_api.h:14-18,59-60,137-139,184-190`. Version 12 is current. Keep the two-sided check exactly:

- Before reading later host fields, reject a null host or a first-field `version` different from `MKWVK_API_VERSION`, as `gpu/vk/vk_main.cpp:987-989` does.
- On initialization failure, return null. On success, return a stable, library-owned table with `version = MKWVK_API_VERSION`, as `gpu/vk/vk_main.cpp:995,1015-1020` does.
- The host still resolves the string `mkwvk_init`, calls it with `vk_host`, and rejects null or a mismatched returned version in `host/rx_backend.c:241-251`.

A language-only implementation change needs no version bump if the complete ABI stays identical. A change to either table does, per `AGENTS.md:12-13`, "The renderer's interface is versioned." Generate or mechanically check bindings against the header so the version and layouts cannot drift. A later synthetic C host should check sizes, alignments, field offsets, null and wrong-version rejection, and callback calls on both targets.

There is also an export absent from the ticket's description. `gpu/vk/vk_main.cpp:964-987` exports `mkwvk_close_asked`, and `host/rx_backend.c:280-285` looks it up by name to stop shader preparation during shutdown. Preserve it for current behavior even though the host tolerates its absence. There must be one definition of each export during the staged port, so the retained C++ implementation and Rust must not both export `mkwvk_init`.

Cargo's conventional Linux artifact name also needs adaptation. Preserve the consumer-facing basename `hookshot_vk.so`, not a new `libhookshot_vk.so` path. `nx2/build.py:63-64,1034-1039` and `scripts/linux/appimage.sh:24-27,72` name the required files. Stage Cargo's output at the existing DLL or SO path.

**Open risk:** A matching version number cannot detect a wrong Rust layout or a dangling table or string pointer. The proposed C host and a real host load remain acceptance checks. This research did not execute either.

## 3. Panics and C++ exceptions

**Answer:** Use `panic = "abort"` for the first shipping renderer build. Keep all externally callable Rust functions and callbacks `extern "C"`. Catch C++ exceptions inside C++ wrappers before returning to Rust. Recoverable native failures must return an error code or result record rather than unwind.

The Rust Reference's ABI table is decisive. [R11], [R13]

| Boundary | Specified behavior |
| --- | --- |
| Rust panic with `panic=abort` | Terminates the process without unwinding. |
| Rust panic escaping an `extern "C"` function with `panic=unwind` | Aborts at the non-unwinding boundary. |
| C++ exception entering Rust through an `extern "C"` declaration | Undefined behavior, including when Rust uses `panic=abort`. |
| Native exception crossing a `C-unwind` boundary with `panic=unwind` | Unwinding is permitted. All crossed foreign boundaries must permit it too. |
| Native exception crossing a `C-unwind` boundary with `panic=abort` | Aborts, according to the Reference's ABI table. |

`panic=abort` therefore handles Rust's failure policy. It does **not** make a throwing C++ call declared `extern "C"` safe. Keep C++ exceptions enabled in retained native code and add `extern "C"` wrappers that catch `std::exception` and `...`, translate the failure, and return normally. Avoid Rust callbacks inside throwing native operations unless their exception behavior has also been audited.

Eden really throws. For example, `gpu/vk/eden/shader_recompiler/frontend/maxwell/translate/impl/floating_point_range_reduction.cpp:36-37` throws `NotImplementedException`. The current renderer catches translation failures and marks the program failed in `gpu/vk/vk_shader.cpp:1575-1587`. It catches both standard and other worker exceptions in `gpu/vk/vk_jobs.cpp:29-40`. Preserve those recovery semantics when the translation or scheduling logic moves to Rust. A wrapper must return enough information for Rust to maintain the same failed or retryable state.

The alternative is `panic=unwind` with `catch_unwind` **inside every exported entry point and callback**, before a panic reaches its C boundary. Thread roots also need an explicit failure policy. This can return null from `mkwvk_init` or a documented failure from fallible operations. However, many renderer callbacks return `void` in `gpu/vk/vk_api.h:137-181`. Catching a panic there does not restore partially changed GPU or replay state. Recovery would need a documented renderer-failed state or process termination. The `abort` handler is non-recoverable, so `catch_unwind` cannot convert an abort-mode panic into an error. [R13]

`C-unwind` is an option for an intentional C++ → Rust → C++ exception path with an unwind build and compatible native unwind machinery. It is unnecessary for wrappers that catch locally. It is not a Rust C++ exception handler: the Reference says catching a foreign unwind with `catch_unwind` may abort or return an opaque error, with the choice unspecified. Do not depend on decoding or recovering an Eden exception that way. [R11], [R13]

**Open risk:** Abort mode makes a Rust panic fatal to the whole game. This is a deliberate initial policy, not parity for recoverable Eden errors. Wrapper error translation must preserve existing shader and worker recovery, and later synthetic tests must exercise both ordinary errors and exceptions. This reading-only task does not prove cross-language unwinder interoperability.

## 4. CI installation, cache, packaging, and minutes

**Answer:** Install a pinned minimal Rust toolchain only in jobs that build the renderer, select its target explicitly, and add a Cargo cache separate from native dependencies. Preserve the existing release-cache and NGX restrictions. Added build minutes are unknown until measured.

Rustup documents noninteractive installation and minimal profiles. Its cross-compilation guide documents `rustup target add` and notes that a target standard library does not supply every required native linker tool. The gnullvm guide supplies both host and target tools, so a Windows gnullvm-host compiler is an available choice. No custom target or nightly-only `build-std` is required by this design. Pin a released toolchain that supplies the selected target rather than a moving `stable` label. [R1], [R14], [R15]

Proposed workflow changes, not changes made by this ticket:

| Workflow | Existing build and evidence | Required addition |
| --- | --- | --- |
| `windows-builder.yml` | The `vulkan` job runs on `windows-2022`, has a 45-minute timeout, installs llvm-mingw, runs `RX_VK_ONLY=1 RX_RELEASE=1`, and uploads `hookshot_vk.dll`. Lines 133-193. Its separate `package` job has a 30-minute timeout, lines 28-34. | Install rustup if absent and a pinned minimal toolchain before the renderer build. A gnullvm host toolchain avoids making host build scripts depend on MSVC by default. Explicitly select `--target x86_64-pc-windows-gnullvm`, configure llvm-mingw as the linker, put its `bin` directory on PATH for host tools too, and invoke the Cargo-aware `nx2/build.py` path. Keep the DLL artifact path. Rust compilation belongs in `vulkan`; the package job needs only the eventual prebuilt-renderer delivery changes. |
| `vulkan-dlss.yml` | `windows-2022`, 45-minute timeout, the same native toolchain, a temporary build-only NGX SDK, renderer-only build, and a system-DLL import allowlist. Lines 53-117. SDK upload and caching are explicitly forbidden, lines 1-5. | Add the same pinned Rust and linker setup. Extend **both** path filters, lines 11-39, to include the Rust crate, Cargo manifest and lockfile, toolchain configuration, and shared wrappers. Keep the NGX shims and import audit. Do not cache the DLSS target directory, native objects containing NGX, linked output, or SDK. Cache only Cargo downloads that do not contain the SDK. |
| `linux-appimage.yml` | Builds in `ubuntu:22.04`, documented as the glibc 2.35 floor, with a 60-minute timeout. Lines 1-4,19-26. Dependencies are installed before `scripts/linux/appimage.sh`, lines 74-98. | Install minimal Rust **inside this container**, put Cargo on subsequent steps' PATH, and select `--target x86_64-unknown-linux-gnu`. Configure the existing clang++ and lld, plus the same sysroot policy. Build and stage `hookshot_vk.so` before packaging. Keep compilation on the oldest supported environment. The AppImage already builds the renderer once and copies it, `scripts/linux/appimage.sh:22-27,72`. |

For a Windows MSVC-host Rust installation instead, adding the gnullvm target is sufficient for the renderer's target code, but build scripts and proc macros remain host programs. Their own native dependencies must not accidentally introduce an undeclared Visual Studio requirement. Prefer explicit `--target` and target-specific linker settings so host and target builds are distinguishable. Cargo documents separate host handling for build scripts and proc macros under explicit `--target`. [R7], [R15]

Cache proposal:

- Cache Cargo registry and Git dependency downloads. On non-DLSS development builds, optionally cache the dedicated renderer `target` directory too. Never cache credential files or either forbidden game directory.
- Key compiled outputs by runner OS, host and target triples, exact Rust version, llvm-mingw version or Linux native toolchain and sysroot identity, profile and panic strategy, relevant Cargo files, wrappers, native flags, and ABI headers. Track changes to retained native inputs and relevant environment variables through the coordinator or Cargo's `rerun-if-*` directives. Cargo explicitly supports those invalidation directives. [R6]
- Follow existing release policy. Windows tag builds skip unpacked llvm-mingw caches in `.github/workflows/windows-builder.yml:54-59`. Linux tags cache dependency downloads rather than compiled dependencies in `.github/workflows/linux-appimage.yml:61-72`. Reuse downloaded sources on releases if desired, but rebuild compiled renderer outputs. Do not restore ordinary or DLSS compiled caches into a release.

CI installation alone does not satisfy the map's "nothing new for users to install" gate. Today the Windows workflow describes its renderer build as a check and says players build the renderer locally, `.github/workflows/windows-builder.yml:1-7`. Its package job runs independently of the renderer job and does not consume the renderer artifact, lines 28-41,112-121,189-193. Either deliver the prebuilt Rust DLL in the Windows builder/update parts and teach the build to use it, or bundle a Rust toolchain. Prefer the prebuilt DLL, matching Linux's existing approach. The coordinator already accepts `HOOKSHOT_VK_PREBUILT` and copies the named file, `nx2/build.py:817-819,1034-1036`. Windows package wiring is a dependency for the release-plumbing ticket, not completed by adding rustup to CI.

### Added build minutes

There is no defensible measured delta in the available sources. The Linux workflow's "about 4 minutes" is a comment about the existing dependency-cached job, not a Rust baseline, `.github/workflows/linux-appimage.yml:25`. Timeouts of 45, 45, and 60 minutes are limits, not build durations. Quoting an added-minute estimate as a finding would invent evidence.

For the pilot, record baseline and candidate renderer jobs with the same native dependencies, runner class, job count, and release flags. Separate toolchain installation, Cargo dependency fetch, retained native compilation, Rust compilation, link, and packaging. Compare a cold run, a restored-cache run, and an ABI-preserving Rust-only edit. Added minutes are the candidate elapsed job time minus the matched C++ job time; summed job minutes and wall time are different when jobs run concurrently. Keep the present timeouts until those observations justify a change. No timing or speedup claim is made here.

**Open risk:** Actual CI time, cache size, and an appropriate release toolchain pin remain unmeasured or unselected. Windows prebuilt delivery must be specified. The optional Linux NGX link exists in `nx2/build.py:1046`, but these three workflows provide a dedicated NGX link check only on Windows.

## 5. Linux back-channel lookup

**Answer:** Yes. A Rust `cdylib` can call `dlsym(RTLD_DEFAULT, "hos_posix_guest_memfd")` and invoke the result as `unsafe extern "C" fn() -> c_int`. This is an ELF loader lookup, independent of which language produced the calling shared library.

The Linux man-pages project's `dlsym(3)` specifies that `RTLD_DEFAULT` searches global symbols in the executable, its dependencies, and shared objects opened with `RTLD_GLOBAL`. The executable already exports `hos_posix_*` through `-Wl,--export-dynamic-symbol=hos_posix_*` in `nx2/build.py:974-978`. The renderer's current lookup is `gpu/vk/vk_memory.cpp:260-266`, and the shared-library link includes `-ldl` in `nx2/build.py:1042-1046`. [R16]

Preserve that executable export flag. Keep ordinary `dlopen` loading in the same namespace. `host/posix/win32_posix.c:749-771` currently maps the DLL name to an SO and opens it with `RTLD_NOW | RTLD_LOCAL`. `RTLD_LOCAL` on the renderer does not prevent it from looking up the executable's global exports. It controls whether the renderer's own symbols enter the global lookup scope. No switch to `RTLD_GLOBAL` is required for this back-channel. [R16], [R17]

Use the Linux bindings' `RTLD_DEFAULT`, a NUL-terminated symbol name, a null check, and a function pointer with the exact C signature. Do not create a Rust reference or function pointer from a null result. Use `dlerror` when reporting lookup failures. The manual documents the clear-error, lookup, read-error sequence because a null symbol address alone does not always imply a lookup error. [R16]

Preserve optional behavior: `RX_VK_NO_DMABUF` disables the lookup, a missing function produces `fd = -1`, and a missing memfd or import capability prevents this path rather than preventing library load. The current logic is in `gpu/vk/vk_memory.cpp:262-288`. Other retained POSIX shims look up `hos_posix_guest_discard`, `hos_posix_keys`, `hos_posix_window`, and `hos_posix_post_message` in `host/posix/win32_posix.c:640,1197,1208,1223`; keep that export family intact.

**Open risk:** A future change to executable symbol visibility, loader namespace, or host exports can break discovery. A later game-free C executable should export a stub `hos_posix_guest_memfd`, load the actual Rust SO using the current flags, and check both lookup success and the missing-symbol fallback. This research did not run that check.

## Primary web sources

[R1]: https://doc.rust-lang.org/rustc/platform-support/windows-gnullvm.html
[R2]: https://doc.rust-lang.org/rustc/platform-support/windows-gnu.html
[R3]: https://doc.rust-lang.org/rustc/platform-support.html
[R4]: https://github.com/mstorsjo/llvm-mingw/blob/master/README.md#known-issues
[R5]: https://doc.rust-lang.org/reference/linkage.html
[R6]: https://doc.rust-lang.org/cargo/reference/build-scripts.html
[R7]: https://doc.rust-lang.org/cargo/reference/config.html
[R8]: https://doc.rust-lang.org/rustc/codegen-options/index.html
[R9]: https://doc.rust-lang.org/rustc/linker-plugin-lto.html
[R10]: https://doc.rust-lang.org/reference/abi.html#the-no_mangle-attribute
[R11]: https://doc.rust-lang.org/reference/items/functions.html#unwinding
[R12]: https://doc.rust-lang.org/reference/type-layout.html#the-c-representation
[R13]: https://doc.rust-lang.org/reference/panic.html#unwinding-across-ffi-boundaries
[R14]: https://rust-lang.github.io/rustup/installation/index.html
[R15]: https://rust-lang.github.io/rustup/cross-compilation.html
[R16]: https://man7.org/linux/man-pages/man3/dlsym.3.html
[R17]: https://man7.org/linux/man-pages/man3/dlopen.3.html

- [R1] Rust target guide, windows-gnullvm.
- [R2] Rust target guide, windows-gnu.
- [R3] Rust platform support and target tiers.
- [R4] llvm-mingw README, status and known issues.
- [R5] Rust Reference, cdylib, mixed-language linking, and C runtime linkage.
- [R6] Cargo build scripts, link directives, ordering, and invalidation.
- [R7] Cargo configuration, target linker and host-versus-target rustflags.
- [R8] rustc code generation options, linker, link-arg, link-self-contained, and panic.
- [R9] rustc cross-language linker-plugin LTO requirements.
- [R10] Rust Reference, no_mangle and symbol export.
- [R11] Rust Reference, function ABIs and the unwinding behavior table.
- [R12] Rust Reference, C representation and layout.
- [R13] Rust Reference, panic recovery and foreign unwinding.
- [R14] Rustup installation, noninteractive installation and minimal profiles.
- [R15] Rustup cross-compilation, target installation and native linker requirements.
- [R16] Linux man-pages, dlsym and RTLD_DEFAULT.
- [R17] Linux man-pages, dlopen and RTLD_LOCAL.

## Recommendation

Proceed with a gnullvm Windows cdylib and a GNU Linux cdylib behind the unchanged C tables. Keep native compilation in the current coordinator, let Rust drive the final link through the existing toolchains, and preserve the native link recipes. Start with abort-mode Rust panics and C++ wrappers that translate every exception into an ordinary result. Gate implementation on game-free load and ABI checks for both platforms, including Windows NGX and the Linux back-channel. Measure CI cost during that pilot.

Later tickets depend on these facts:

- Windows uses UCRT llvm-mingw. Its matching Rust target is `x86_64-pc-windows-gnullvm`, Tier 2 with host tools. [R1], [R3]
- NGX is a prebuilt MSVC exception requiring the existing shims. FidelityFX is currently shaders, not a linked host SDK. `nx2/build.py:181-209,892-897`; `gpu/vk/vk_fsr3.h:1-5`.
- The C ABI is version 12. Keep both the input and output version checks, table layouts, lifetimes, and `mkwvk_close_asked`. `gpu/vk/vk_api.h:18`; `gpu/vk/vk_main.cpp:973-989,1015-1020`.
- Abort-mode Rust does not make foreign exceptions safe. Native exceptions must stop in C++ wrappers for the recommended design. [R11], [R13]
- Linux must retain dynamic glibc compatibility with the Ubuntu 22.04 build floor and exported `hos_posix_*` symbols. `.github/workflows/linux-appimage.yml:1-4,24`; `nx2/build.py:974-978,1042-1046`.
- The Windows prebuilt-DLL delivery path and measured CI minutes remain release and pilot dependencies. CI rustup setup alone does not remove users' renderer build requirement. `.github/workflows/windows-builder.yml:1-7,112-121,181-193`.
