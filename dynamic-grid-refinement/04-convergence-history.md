# Convergence history

## Motivation

An iterative calculation needs a criterion to decide when to stop, and such a criterion almost
always compares the current iteration with one or more earlier ones. SKIRT keeps this historical
data in several unrelated places, with different granularities and lifetimes, and in some cases in
`mutable` data members of otherwise stateless configuration objects. Dynamic refinement adds more
instances of the same kind. This chapter catalogs the existing instances and proposes how to
centralize them.

## Catalog

The table lists the historical data currently kept to evaluate convergence, including the data
added by the reference implementation of dynamic refinement.

| # | Location | Data | Granularity and depth | Lifetime |
| --- | --- | --- | --- | --- |
| 1 | `MonteCarloSimulation`, helper class `DustAbsorptionConvergence` | dust-absorbed luminosity of secondary emission | one global value, one iteration back | a helper object per secondary or merged iteration loop |
| 2 | `MediumState`, aggregate cells | total volume, and the total of all variables of quantity type "numbervolumedensity", per medium component | global values, one iteration back | the simulation; shifted at the start of each iteration |
| 3 | medium state variables, compared before overwriting | e.g. temperature and ionization parameter (`DiffuseIonizedGasMix`), level populations (`NonLTELineGasMix`), dust density fractions (`DustDestructionRecipe`) | per cell, one iteration back | the simulation |
| 4 | `DiffuseIonizedGasMix`, `mutable` members | fraction of converged cells | one global value, three iterations back | cleared when converged, otherwise never |
| 5 | `MediumSystem` (reference implementation) | number of consecutive iterations each cell met a refinement criterion; iteration counter | per cell, counters | the simulation |
| 6 | `NeighborRefinementRecipe`, `mutable` members (reference implementation) | running sum and count of the refinement field per cell; transient flag; normalization scale | per cell, one window | the simulation; partially reset after subdivision |

Some remarks on individual instances:

- **(1)** The helper class is local to the source file of the iteration loops. It is cleanly
  scoped, but not reusable or observable elsewhere.
- **(2)** The aggregate states are stored as additional "fake" cells at the end of the per-cell
  data array. Which variables are aggregated is determined by their quantity type, so the
  `DiffuseIonizedGasMix` declares some of its diagnostics as "numbervolumedensity" specifically to
  have them aggregated. Only one previous state is kept. Dynamic state recipes do not receive the
  aggregate states, as noted in a comment in `MediumSystem::updateDynamicStateRecipes()`.
- **(3)** Comparing a new value with the old value before overwriting it in the medium state is
  the cheapest possible form of per-cell history, needing no storage at all. After a refinement
  round, a child cell compares with the value inherited from its parent, which is the desired
  behavior.
- **(4)** The history is updated in `isSpecificStateConverged()`, a `const` function of a material
  mix that is otherwise a stateless configuration object. Because the history is cleared only on
  convergence, it carries over from the primary iteration loop into the merged loop when the
  former ends without converging. The history depth is also stored as a `mutable` member, although
  it is never changed. Relatedly, the same source file keeps diagnostic counters and "logged once"
  flags at file scope, so that they are shared by all instances in the process rather than
  belonging to a single material mix.
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
- The depth of the history is hard-wired. The aggregate mechanism keeps exactly one previous
  state, which is why the `DiffuseIonizedGasMix` maintains a separate buffer for its three-step
  plateau criterion.
- Because the aggregate states are stored as fake cells, the per-cell data array is not purely
  per-cell, which complicates growing it during dynamic refinement.
- There is no uniform way to log or probe the convergence history.
- All instances are calculated from data synchronized between processes, so they are identical on
  all processes. Any redesign must preserve this property.

## Proposals

### Iteration history for global values

A new class, tentatively called `IterationHistory`, keeps a series of values per iteration for
each named quantity, up to a requested depth. `MediumSystem` owns one instance, which is reset at
the start of each iteration loop (primary, secondary, or merged):

- At setup, a client requests a named series and the depth it needs, for example "converged cell
  fraction" with a depth of three for the `DiffuseIonizedGasMix`.
- At the end of each iteration, the owner of a value records it in the corresponding series.
- Convergence criteria query the series through a few helper functions: the value a given number
  of iterations back, the relative change with respect to a previous iteration, and whether the
  series has remained within a given tolerance for a given number of iterations.

The convergence functions of material mixes and recipes, `isSpecificStateConverged()` and
`DynamicStateRecipe::endUpdate()`, receive access to the history as an argument, so that they no
longer need `mutable` members. The dust absorption helper of the iteration loops records its value
in the same history.

The aggregate states are still calculated by `MediumState`, but recorded as series in the history
instead of being stored as fake cells. This removes the fake cells from the per-cell data array,
allows any depth, and gives dynamic state recipes access to aggregate information. The
aggregation rule based on quantity type can remain in place for now. Later, state variables could
declare explicitly whether and how they are aggregated.

This proposal changes the signature of the convergence functions, and therefore affects every
material mix and recipe with a dynamic medium state: the `DiffuseIonizedGasMix`, the
`NonLTELineGasMix`, the `AbsorptionOnlyMaterialMixDecorator` (which forwards the call), and the
dynamic state recipes. Aggregate values would be retrieved by series name rather than through
`MaterialState` accessors for fake cells. The change is moderate in size and does not depend on
dynamic refinement, so it can be implemented first.

A uniform history also makes it straightforward to log all convergence quantities in a consistent
format, or to write them to a file with a probe performed after each iteration.

### Per-cell history

The running sums needed by dynamic refinement are kept by `MediumSystem` in a small, generic
per-cell history structure, rather than by the criteria. It holds one running sum per requested
field per cell, follows the global window schedule, and is one of the per-cell structures that grow
during refinement. This replaces instances 5 and 6.

The same structure could later serve other purposes, such as averaging a noisy quantity over
several iterations before using it in a state update. Until such a need arises, it is used only
for dynamic refinement.

The comparison of new and old values in the medium state (instance 3) remains unchanged. It needs
no storage, and moving it elsewhere would bring no benefit.

### Minimal alternative

If the proposals above are considered too extensive, the `mutable` members can be removed with a
much smaller change: make the convergence functions non-`const` and add a hook that is called at
the start of each iteration loop, so that the `DiffuseIonizedGasMix` can reset its history
properly. This resolves the most visible problem, but leaves the history scattered.

### Recommendation

Implement the iteration history first, independently of dynamic refinement, and implement the
per-cell history as part of dynamic refinement. Open points include which object owns the
iteration history (`MediumSystem`, or the simulation passing it to the medium system) and whether
the aggregate states move to the history in the same step.
