# Keeping the fork in step with upstream during the port

Type: grilling
Status: open
Blocked by: none

## Question

Upstream (NotRexed/Hookshot) merges often; #1551 merged on 2026-11-10. How does `Rhinomcd/Hookshot` follow it once
part of the renderer is in Rust?

- When an upstream PR changes a C++ file whose behavior is already ported, who re-ports the change, and in which
  merge?
- Does the parity oracle stay buildable after deletion, so upstream fixes can be diffed against it?
- What is the cadence for merging upstream?
- What happens to the upstream text-slicing tests that target files the fork has deleted?
