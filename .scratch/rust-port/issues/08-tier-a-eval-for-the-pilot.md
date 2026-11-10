# Tier-A eval for the pilot

Type: grilling
Status: open
Blocked by: 07

## Question

What exactly does the pilot's tier-A eval run and measure?

- **Corpus:** synthetic inputs and their size and format mix, chosen so the mix matches what the profiling ticket saw
  in the game, without using game data.
- **Oracle:** how the C++ implementation is linked into the eval harness as the reference.
- **Parity:** the byte-exact comparison.
- **Fitness metric and its unit.**
- **CI:** how the eval runs in CI on both platforms.
- **Gaming:** how it resists overfitting, for example a held-out corpus the council never sees.
