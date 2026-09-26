# Current state

## Catalog

The table lists the historical data currently kept to evaluate convergence, followed by the data
that the implementation of dynamic grid refinement will require.

| # | Location | Data | Granularity and depth | Lifetime |
| --- | --- | --- | --- | --- |
| 1 | `MonteCarloSimulation`, helper class `DustAbsorptionConvergence` | dust-absorbed luminosity of secondary emission | one global value, one iteration back | a helper object per secondary or merged iteration loop |
| 2 | `MediumState`, aggregate cells | total volume, and the volume integral of all variables of quantity type "numbervolumedensity", per medium component | global values, one iteration back | the simulation; shifted at the start of each iteration |
| 3 | medium state variables, compared before overwriting | e.g. temperature and ionization parameter (`DiffuseIonizedGasMix`), level populations (`NonLTELineGasMix`), dust density fractions (`DustDestructionRecipe`) | per cell, one iteration back | the simulation |
| 4 | `DiffuseIonizedGasMix`, `mutable` members | fraction of converged cells | one global value, three iterations back | cleared when converged, otherwise never |
| 5 | `MediumSystem` (dynamic refinement) | number of consecutive iterations each cell met a refinement criterion; iteration counter | per cell, counters | the simulation |
| 6 | `NeighborRefinementRecipe`, `mutable` members (dynamic refinement) | running sum and count of the refinement field per cell; transient flag; normalization scale | per cell, one window | the simulation; partially reset after subdivision |

Remarks on individual instances:

- **(1)** The helper class is local to the source file of the iteration loops. It is cleanly
  scoped, but not reusable or observable elsewhere.
- **(2)** The aggregate states are stored as two additional cells at the end of the per-cell data
  array, allocated whenever the simulation has a dynamic medium state. The current aggregate is
  calculated at the end of setup and after each state update, and shifted to the previous aggregate
  at the start of each iteration. Which variables are aggregated is determined by their quantity
  type, so the `DiffuseIonizedGasMix` declares some of its diagnostics as "numbervolumedensity"
  specifically to have them aggregated. Conversely, all such variables are aggregated whether or not
  anybody uses the result: the `NonLTELineGasMix`, for example, only needs its level populations,
  but its collision partner densities are aggregated as well. Dynamic state recipes do not receive
  the aggregate states, as noted in a comment in `MediumSystem::updateDynamicStateRecipes()`.
- **(3)** Comparing a new value with the old value before overwriting it is the cheapest possible
  form of per-cell history, needing no storage at all. It remains unchanged by this proposal.
- **(4)** The history is updated in `isSpecificStateConverged()`, a `const` function. Because it is
  cleared only on convergence, it carries over from the primary iteration loop into the merged loop
  when the former ends without converging. The history depth is also a `mutable` member, although
  it never changes. Relatedly, the same source file keeps diagnostic counters and "logged once"
  flags at file scope, so that they are shared by all instances in the process.
- **(5, 6)** The dynamic refinement history is split between the medium system and the recipes,
  and the recipes must be notified of each subdivision so they can reset the history of the
  affected cells.

## Observations

- The history comes in three kinds: global values per iteration (1, 2, 4), per-cell values per
  iteration (3, 5, 6), and counters or flags (5, 6).
- There are five different owners, with no common notion of lifetime: some data lives for a single
  iteration loop, some for the whole simulation, and some is reset only on convergence.
- Two instances (4, 6) store history in `mutable` members updated from `const` functions. This
  works only because these functions are called once per iteration from a single thread.
- The depth is hard-wired. The aggregate mechanism keeps exactly one previous state, which is why
  the `DiffuseIonizedGasMix` maintains a separate buffer for its three-step plateau criterion.
- Aggregates are calculated for all eligible variables, rather than for those actually needed.
- Because the aggregate states are stored as fake cells, the per-cell data array is not purely
  per-cell, which complicates growing it during dynamic refinement.
- There is no uniform way to log or probe the convergence history.
- All instances are calculated from data synchronized between processes, so they are identical on
  all processes. The redesign must preserve this property.
