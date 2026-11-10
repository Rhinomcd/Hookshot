# A Linux tier-B harness that emits numbers only

Type: task
Status: open
Blocked by: none

## Question

Make a scripted headless run of the installed game (`/home/rhino/Games/Hookshot/bin`) on Linux that writes one JSON of
numbers and nothing else. `scripts/run-vk.ps1` exists only for Windows.

- **Inputs:** `HOS_HEADLESS`, `HOS_AUDIO=0`, a fixed `HOS_INPUT_SCRIPT` route (title screen to Kokiri Forest
  gameplay, plus one scene change), and `RX_VK_FRAME_LOG` and `RX_VK_BENCH` (`gpu/vk/vk_replay.cpp:1542-1600`, `:3542`).
- **Swapping the renderer:** the harness must also run with a different renderer library pointed in, since tier B
  compares C++ against Rust.
- **Output:** frame-time p50/p99, `RX_VK_BENCH` CPU ms, upload stats per route (`gpu/vk/vk_texture.cpp:1218-1240`),
  and the run's wall time.
- **Noise:** run the unchanged build ten times and report the spread of each metric, per benchmark-checklist. The
  hill-climb's "beats the incumbent by more than noise" threshold comes from these numbers.

Run it through test-box. The game, its logs and its frames stay on the PC; only the JSON is committed.
