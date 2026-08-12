# AXIOM technical specification

AXIOM is a Rust crate that searches Particle-Lenia interaction laws. A law is a bounded flat vector of floats. The engine decodes one into a simulation, runs it, and measures what it produced; the tuner scores the measurement and proposes the next law.

This is the surface-level account. The code is the specification, and where the two disagree the code is right.

## Design invariants

Six constraints hold everywhere, enforced by the module graph rather than by convention.

- **The engine is downstream of nothing.** `engine` imports only `engine` and `util`, so the simulation can never acquire a dependency on the thing measuring it.
- **The tuner touches no simulation.** It builds no `Sim`. `harness::rollout` is the only place one is constructed to be measured.
- **One trust boundary.** `harness::protocol` is where an outside caller's numbers get checked. Past it, an out-of-range value is a bug rather than an input.
- **Dimensionality is data.** No constant fixes how many axes a genome has.
- **Periodic bounds throughout.** Minimum-image distance and wrapping on every step, so structures cross every face and no boundary is terminal.
- **Norms are measured, never assumed.** Normalized potential means the same thing across genomes because the reference is measured on a fixed uniform field.

## How a run works

```
genome  →  decode  →  Sim  →  rollout  →  descriptor  →  gates  →  criteria  →  score
                                                                                  ↓
                                                                            next genome
```

A **genome** is a bounded flat vector with a documented layout. It decodes into an interaction matrix, a set of traits, and a box whose size is derived from a coordination gene rather than set freely, which is what keeps density controlled.

A **rollout** runs one genome for a fixed schedule and returns one descriptor. It is the only path from a law to a number.

**Gates** are cheap rejections that run before scoring: non-finite state, collapse, dispersal. **Criteria** are the scored objectives. A genome that fails a gate never reaches them.

**Search** is a population loop over that pipeline, running on its own thread and reporting per generation.

## Measurement

Every spatial metric reads one shared periodic density-and-trait grid, computed once per rollout. On top of it sit the radial distribution function, the trait-by-radius histogram, and the structural summaries the criteria consume. Building the grid once is the reason adding a metric is cheap.

## Where the physics lives twice

The Particle-Lenia step exists in two places: `engine/lenia.rs` on the CPU and `engine/gpu/step.wgsl` as a compute shader. That duplication is deliberate and is the one place in the crate where a change has to be made in two files to be correct.

## Module map

| Area | What it owns |
|---|---|
| `engine/` | Substrate, traits, kernels, genome layout and decode, the interaction matrix, the box, and both step implementations |
| `harness/` | The session loop, the frontend protocol, the catalog handed to a frontend as data, the live playground, search, and rollout |
| `tuner/` | The metric plan, the shared field, the radial distribution, and the scoring |
| `util.rs` | Deterministic xorshift, distance, the non-finite guard |

Full per-file responsibilities are in the module headers, which are kept current because they sit next to the code they describe.

## Limits

- The measurement grid is three-axis. A search rejects a genome shaped any other way.
- Scores are comparable within a search, not across searches with different criteria.
- GPU and CPU rollouts are not bit-identical. Treat a cross-backend comparison as approximate.
- Nothing here claims a law that scores well is interesting. Scoring is a filter, not a verdict.

## Reading further

[README.md](README.md) covers the same ground for someone running the crate rather than modifying it, including the command surface and what the code deliberately does not contain.
