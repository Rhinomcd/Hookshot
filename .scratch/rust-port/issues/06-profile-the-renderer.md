# Where the renderer's CPU time goes on the tier-B route

Type: task
Status: open
Blocked by: 04

## Question

On the tier-B route, which renderer components take the CPU time? Rank them by the cost a pilot could move.

- **Tools:** `HOS_PROFILE` + `HOS_SAMPLE` on the render threads (`scripts/prof.py`), `RX_VK_PROF_ALL` and
  `prof_ns[PROF_*]` (`gpu/vk/vk_internal.h:270`), `RX_VK_PASS_TIMING`, and the texture upload and unswizzle stats.
- **Output:** each component's share of render-thread CPU, in steady gameplay and across a scene change.

Keep the profiles on the PC. Commit the ranked table only, with no symbol names from the lifted code.
