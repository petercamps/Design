# Features

## Overview

A simulation with dynamic refinement proceeds as follows:

1. The tree is constructed as usual by the configured tree policies. This initial grid should
   already be adequate for the density distribution and, where applicable, resolve the expected
   Strömgren spheres (see `ResolvedSpheresTreePolicy`).
2. The iterations start. After the medium state has been updated at the end of each iteration, the
   refinement step records the value of each configured refinement field in each cell.
3. At the end of each averaging window of a few iterations, each refinement criterion evaluates
   the window-averaged fields, and the cells across which a field changes too steeply are
   subdivided. The child cells inherit their parent's state.
4. The iterations continue on the refined grid. The loop terminates once the medium state has
   converged and a complete averaging window has passed without any cell asking for subdivision.

## Configuration

### Placement in the ski file

The refinement configuration is a new optional property of `TreeSpatialGrid`, called
`dynamicRefinementOptions`, next to the `policies` list that governs the initial construction. It
is relevant only if the simulation iterates over primary or secondary emission, and it is displayed
only at the expert user level, like the `iteratePrimaryEmission` flag.

There are three reasons for placing these options here. First, the schema then offers dynamic
refinement only for grids that support it, instead of failing at run time for other grids.
Second, the static and dynamic refinement of the same grid are configured in one place. Third, the
refinement is conceptually a property of the grid, even if it is driven by the medium state.

Dynamic refinement requires that the medium component providing each refinement field has a
dynamic medium state, primary or secondary, because otherwise the field does not change between
iterations. Setup reports a fatal error if this is not the case.

### Example

The following fragment shows an example configuration for a galaxy simulation that
refines on the layers of singly ionized nitrogen and doubly ionized oxygen:

```xml
<OctTreeSpatialGrid minLevel="3" maxLevel="12" ...>
    <policies type="TreePolicy">
        <ResolvedSpheresTreePolicy filename="sources.txt" numBins="2" reach="2"/>
    </policies>
    <dynamicRefinementOptions type="DynamicRefinementOptions">
        <DynamicRefinementOptions maxRefinementLevel="15" maxExtraLevels="4" maxCells="9000000"
                                  numIterationsBeforeRefinement="3" numAveragedIterations="4">
            <criteria type="RefinementCriterion">
                <GradientRefinementCriterion field="x_NII" maxChange="0.3" minValue="5e-3"/>
                <GradientRefinementCriterion field="x_OIII" maxChange="0.3" minValue="5e-3"/>
            </criteria>
        </DynamicRefinementOptions>
    </dynamicRefinementOptions>
</OctTreeSpatialGrid>
```

In this example, a cell can gain at most four levels beyond its level in the initial grid, and no
cell can exceed level 15. A cell at level 10 in the initial grid can thus reach level 14, whereas a
cell at the grid's maximum level 12 can reach only level 15.

### Refinement options

The `DynamicRefinementOptions` item holds the settings shared by all criteria.

| Property | Description |
| --- | --- |
| `maxRefinementLevel` | absolute maximum level of any cell created by dynamic refinement |
| `maxExtraLevels` | maximum number of levels a cell may gain beyond its level in the initial grid (zero means no limit) |
| `maxCells` | hard cap on the total number of cells |
| `numIterationsBeforeRefinement` | number of initial iterations of each iteration loop during which refinement is not considered |
| `numAveragedIterations` | number of iterations over which the refinement fields are averaged before each decision |
| `criteria` | the list of refinement criteria |

The `maxLevel` property of the grid itself continues to limit the initial construction only.
The `maxExtraLevels` limit is expressed relative to each cell's level in the initial grid. 
The `maxRefinementLevel` property imposes an absolute maximum level even after dynamic refinement.
For a binary tree, which splits a cell along one axis per level,
three levels correspond to one octree level.

When the `maxCells` cap prevents subdividing all cells that ask for it, the cells with the largest
excess over their threshold are subdivided first, and a warning reports how many cells were left
unsplit.

### Refinement criteria

`RefinementCriterion` is the abstract base class for criteria. Several criteria can be configured
at once, for example one per ion; a cell is subdivided if any of them asks for it. The proposal
offers a single concrete criterion, `GradientRefinementCriterion`, which compares a field between a
cell and its face neighbors.

| Property | Description |
| --- | --- |
| `field` | the name of the medium state variable that drives the criterion (see below) |
| `measure` | `Gradient` or `NeighborDifference` (see below) |
| `normalization` | `None`, or `Percentile` to divide by a percentile of the field over the grid |
| `normalizationPercentile` | the percentile, relevant for `Percentile` normalization |
| `maxChange` | the threshold; a cell is subdivided if its measure exceeds this value |
| `minValue` | cells with a field value below this floor are never subdivided |

The `Gradient` measure takes the larger of two quantities, both expressed as a change in the field
across the cell:

- the magnitude of the gradient obtained from central differences between the neighbors on
  opposite walls, multiplied by the cell size, which captures a front spread over several cells
  regardless of its orientation relative to the grid;
- the largest difference between the cell and the neighbors on a single wall, scaled to the cell
  size, which captures a layer only one cell thick, where central differences vanish.

When a wall has several (smaller) neighbors, their values are averaged. The `NeighborDifference`
measure simply takes the largest absolute difference between the cell and any face neighbor.

Normalization divides the measure by a high percentile of the field over all cells with a valid
value above the floor. This makes a single threshold comparable across fields with very different
peak values, such as N+ (peaking near 0.3) and O2+ (peaking near 1). A percentile is used rather
than the maximum, so that a single Monte Carlo outlier cannot set the scale.

### Refinement fields

A criterion's `field` property names a medium state variable. Any state variable of any material
mix can drive refinement, provided the mix gives it a short name. The standard `temperature`
variable is available for all mixes that store a temperature. The `DiffuseIonizedGasMix` names
the following custom variables:

| Field | Meaning |
| --- | --- |
| `logU` | the ionization parameter (resolves the ionized region) |
| `x_HI` | the neutral hydrogen fraction (resolves the ionization front) |
| `x_HII`, `x_NII`, `x_OI`, `x_OII`, `x_OIII`, `x_SII` | the fraction of H+, N+, O0, O+, O2+, or S+ (resolves that ion's emitting layer) |
| `temperature` | the gas temperature (resolves thermal transitions) |

The field is taken from the first medium component whose material mix offers a state variable
with the given name. Setup reports a fatal error listing the available names if no component
offers the requested one.

> Selecting a field by name keeps the `MaterialMix` base class free of mix-specific enumerations
> and ion index tables, and makes new fields available simply by naming a state variable. The
> price is that the schema cannot validate the name, so MakeUp cannot offer a list of choices.
> See Open questions.

## Iteration and convergence

Dynamic refinement can take place in each of the iteration loops: primary emission iterations,
secondary emission iterations, and merged primary and secondary emission iterations. In a given
loop, only the criteria whose field is updated in that loop take part. A field of a mix with a
primary dynamic medium state is updated in the primary and merged loops, and a field of a mix with
a secondary dynamic medium state in the secondary and merged loops. If no criterion takes part, the
refinement step does nothing in that loop.

In each loop, the refinement step starts afresh and goes through the following phases:

- During the first `numIterationsBeforeRefinement` iterations, refinement is not considered at
  all, so that the first decision is based on a settled radiation field.
- The refinement fields are then accumulated for `numAveragedIterations` iterations, after which
  each criterion evaluates the window-averaged values. Averaging over several iterations reduces the
  Monte Carlo noise that reaches the decision.
- If cells are subdivided, the next iteration is left out of the averaging, because the field is
  still adjusting to the changed grid. A new window starts after that.

The initial iterations and the window address different problems. The window reduces Monte Carlo
noise, which scatters around the current solution. At the start of a loop, however, the medium
state drifts systematically from its initial guess toward the solution, for example as ionization
fronts move outward, and the early primary iterations may use fewer photon packets. Averaging a
drifting field yields a value between the initial guess and the solution rather than the solution
itself. Because cells are never merged, a subdivision based on such transient structure is
permanent, whereas a missed subdivision is simply caught by a later window. A longer window could
also absorb the transient, but it would lengthen every refinement round, whereas the initial
iterations delay only the first decision in each loop. Skipping the iteration after a refinement
round follows the same reasoning.

The iteration loop is considered converged only if the medium state has converged and the
refinement has settled, meaning that a complete window after the initial iterations produced no
subdivision requests, or that no further subdivision is possible because of the cell cap or the
level limits. During the initial iterations and while a window is open, the refinement is not
settled. As a result, each refinement round costs at least `numAveragedIterations + 1` extra
iterations. If the loop ends because it reaches the maximum number of iterations while the
refinement has not settled, a warning is issued.

In a simulation with primary and merged iterations, refinement thus continues in the merged loop,
where the secondary radiation field may change the fields further. In a simulation with separate
secondary iterations, the primary emission has been completed before the secondary loop starts:
the primary radiation field is not recalculated, and the instruments have recorded the primary
emission. A cell created in the secondary loop receives its share of the parent's primary radiation
field, which is uniform over the parent, so that refinement in this loop resolves only structure
caused by the secondary radiation field.

## Initial state of child cells

A child cell inherits its parent's complete medium state, including the number density, the
temperature, and all custom variables such as the iterated ionization state, as well as its
parent's material mix. Only the cell volume is recalculated from the grid. The total mass of each
medium component is therefore conserved, and the next iteration starts from the previous solution.
The radiation field stored for the parent is divided among the children in proportion to their
volume, so that each child starts with its parent's mean intensity.

> Inheriting the state means that the refinement does not resolve any density structure within
> the parent cell. Re-sampling the input model for the child cells is a possible alternative (see
> Open questions).

## Output and reuse

Each refinement round logs the number of subdivided and added cells and the new total, and warns
when the cell cap prevents subdivision.

The `TreeSpatialGridTopologyProbe` gains a `probeAfter` option (`Setup` or `Run`), so that it can
record the topology of the refined grid at the end of the simulation. A subsequent simulation can
load this topology with the `TopologyTreePolicy`. It then starts from the refined grid, either
skipping dynamic refinement altogether, for example to calculate other diagnostics for the same
model, or refining further.

To follow the refinement from one iteration to the next, probes offering a `Primary` or `Secondary`
option for their `probeAfter` property can be used, such as the `CustomStateProbe`. Probes performed
after an iteration see the refined grid, where the new cells still hold their inherited state. The
`SpatialCellPropertiesProbe` gains a column with the tree level of each cell when used with a tree
grid.

## Limitations

- Only tree-based spatial grids support dynamic refinement, and cells are never merged.
- The medium state is replicated on each MPI process, so memory use per process grows with the
  number of cells. With the `DiffuseIonizedGasMix`, whose state is large, this amounts to about
  7 GB per million cells per process.
- Cells created by refinement are appended at the end of the cell list. This preserves all
  existing cell indices, but spatially adjacent cells no longer have nearby indices, which may
  reduce memory locality.
