# Map: Rust port of Hookshot's runtime

## Destination

A staged spec for porting Hookshot's renderer (`gpu/vk`) and then its host runtime (`host/`) to Rust. Each stage names its
seam, its two evals and its council hill-climb. Stage 1 is a pilot whose measured result is the go/no-go gate for the
rest, and the spec is ready to hand to an executor.

## Notes

**Settled while charting (2026-11-10, with Ryan):**

- **Venue:** Ryan's fork, `Rhinomcd/Hookshot`. Ryan has read-only access upstream (NotRexed/Hookshot). The fork merges
  upstream and carries the drift cost (see the fork sync ticket).
- **Strategy:** strangler through the existing C ABI seams. One component at a time becomes a Rust crate behind a plain-C
  interface. The C/C++ version stays as the parity oracle until it is deleted, and the game runs at every step.
- **Order:** `gpu/vk` first (37K first-party lines). It ships as `hookshot_vk.dll`/`.so`, and the exe loads it by
  `mkwvk_init` with a version check (`MKWVK_API_VERSION`, `gpu/vk/vk_api.h:18`). A Rust renderer therefore drops in
  beside an installed game without rebuilding the 1.4 GB of lifted C. Linux players get the renderer prebuilt in the
  AppImage. Windows players compile it on their own PC (`.github/workflows/windows-builder.yml:5`), so a Rust renderer
  must reach them prebuilt (ticket 10). `host/` comes second (21K lines). It links into the exe that each user builds, so its Rust part has to ship
  as a prebuilt staticlib.
- **Pilot gate:** the pilot passes when (1) its output matches the C++ byte for byte on the eval corpus, (2) the straight
  port, before any hill-climb, is within measured noise of the C++ baseline, and (3) CI builds it for Windows and Linux
  with nothing new for users to install. Hill-climb gains are reported separately and do not gate.
- **Evals, two tiers:**
  - **Tier A:** synthetic microbenchmarks with a byte-exact check against the C++ oracle. No game is needed, so it runs
    in CI. It is the hill-climb's fitness score.
  - **Tier B:** a scripted headless run of the real game on Ryan's PC, producing frame-time p50/p99 (`RX_VK_FRAME_LOG`)
    and CPU ms (`RX_VK_BENCH`). Every candidate the climb keeps must pass it. Only numbers leave the PC.
- **Council hill-climb:**
  - Four members: Claude Opus 5.5, GPT-6.1 Sol, Muse Wonder Full 1.0 and Grok 4.7. Ryan ruled out Gemini 4 Argon.
  - Each round, every member implements one optimization in its own worktree.
  - Tier A scores the candidates. The best one that beats the incumbent by more than noise is kept, and must then pass
    tier B.
  - All members review the kept diff and score it out of 10.
  - The climb stops after two rounds without a gain.
  - If a member is unreachable, report it and do not substitute another model.

**Standing rules:**

- No game files, or anything made from them, leave the PC (CONTRIBUTING.md). Council members are outside providers,
  so they get repo source and a numbers-only eval JSON, never `work/`, `/home/rhino/Games/Hookshot`, run logs, frame
  dumps or captures.
- When a file is ported, the Python tests in `tests/` that slice that file's text are replaced in the same stage by Rust
  behavior tests covering the same behaviors. 194 tests read sources by path; `vk_draw.cpp` alone has 32.
- Vendored C/C++ (Eden, Sirit, ImGui, FidelityFX, VMA, NGX, zstd, opus) stays as it is, behind FFI.
- Rust is not installed on this machine yet.

**Skills for every session:**

- Grilling tickets: grilling and domain-modeling.
- Research tickets: research.
- Anything measured: perf-eval, benchmark-checklist and principle-explain-the-number.
- Climbs: council.
- Game runs and tests: test-box.

**Terms:**

- **Renderer:** `gpu/vk`, the Vulkan replay library.
- **Host:** `host/`, the reimplemented Horizon OS and NVN, linked into the exe.
- **Lifted C:** the game's code translated to C, generated on each user's PC.
- **Seam:** a plain-C interface a Rust crate can sit behind.
- **Parity oracle:** the C/C++ implementation a port is checked against.
- **Pilot:** stage 1's component.

## Decisions so far

<!-- one line per closed ticket: [title](issues/NN-slug.md): gist -->

- [Rust toolchain for the renderer's two targets](issues/01-rust-toolchain-for-the-renderer.md): gnullvm + linux-gnu `cdylib`, today's link recipe kept, `panic=abort`, C++ exceptions stop in C++ wrappers, both exports kept at ABI 12.

## Not yet specified

- **Renderer port order after the pilot.** Candidate units are replay and draw state (13K lines), memory and transfer
  (4.2K), textures and formats (4.3K), the shader glue to Eden (3.2K), DLSS, FSR3 and frame generation (5.5K), the
  ImGui menu (4.1K) and diagnostics (0.9K). The order hangs on the pilot's result and on the interop research.
- **`MkwVkHost`/`MkwVkRenderer` under a strangler.** Whether the Rust renderer keeps `vk_api.h` unchanged as the outer
  seam, and how Rust and C++ halves share one `VkDevice` while the renderer is partly ported.
- **Host stage.** This covers:
  - Shipping a prebuilt Rust staticlib into the user-PC link (llvm-mingw + lld, `-static`).
  - The HLE ABI: `imp_*` symbols bound by name by the lifter, and the `Ctx` struct that `nx2/rtgen.py` patches by text
    substitution.
  - `recomp_sj_*` setjmp/longjmp crossing Rust frames.
  - Guest-memory writes that must set `A64_MARK`.
  - Tier B for the host stage, which needs a full lifted-C build on Ryan's PC (25 to 45 minutes).
- **Release plumbing,** apart from Windows delivery (ticket 10): the changes to `windows-builder.yml`, `linux-appimage.yml` and `release.yml` once Rust ships.
- **The gate's verdict and what follows a failed pilot:** stop, re-scope, or port only the parts that measured well.

## Out of scope

- The `gpu/render` WebGPU fallback (26K lines), the `nx2/` Python lifter and build pipeline, the C# builder, `scripts/`,
  and the generated game C. Each stays in its current language.
- Rewriting vendored C/C++ in Rust.
- Upstream PRs to NotRexed/Hookshot. The venue is the fork.
