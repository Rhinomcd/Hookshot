# Choose the pilot component and its seam

Type: grilling
Status: open
Blocked by: 02, 06

## Question

Which renderer component is the pilot, and which plain-C interface does its Rust crate sit behind?

Weigh each candidate on four things:

- its CPU share, from the profiling ticket;
- how cleanly it can be cut out (calls into Eden or `Shader::`, shared state in `vk_internal.h`);
- whether a game-free tier-A corpus can be built for it;
- how many text tests it takes down;
- whether it calls Vulkan from Rust, which brings in `ash` and the 1.3.281-versus-1.4.345 header gap (interop ticket).

A starting candidate is the CPU block-linear unswizzle and format conversion in `vk_texture.cpp`: it is a pure
function with existing stats, and it is tied to the scene-change stutter in the README's status table. The pick must
come out of the measurement, not this guess.
