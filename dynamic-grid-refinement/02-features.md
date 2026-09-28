# Features

## Overview

A simulation with dynamic refinement proceeds as follows:

1. The tree is constructed as usual by the configured tree policies. This initial grid should
   already be adequate for the density distribution and, where applicable, resolve the expected
   areas of special interest (e.g. through `ResolvedSpheresTreePolicy`).
2. The iterations start. After the radiation field and medium state have been updated at the
   end of each iteration, the refinement step records, for each configured refinement criterion,
   the value of its field in each cell.
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
refinement is conceptually a property of the grid, even if it is driven by the medium state or the
radiation field.

### Example

The following fragment shows an example configuration for a galaxy simulation that
refines on the layers of singly ionized nitrogen and doubly ionized oxygen:

```xml
<OctTreeSpatialGrid minLevel="3" maxLevel="12" ...>
    <policies type="TreePolicy">
        <ResolvedSpheresTreePolicy filename="sources.txt" numBins="2" reach="2"/>
    </policies>
    <dynamicRefinementOptions type="DynamicRefinementOptions">
        <DynamicRefinementOptions maxExtraLevels="4" maxCells="9000000"
                                  numIterationsBeforeRefinement="3" numAveragedIterations="4">
            <criteria type="RefinementCriterion">
                <MediumStateGradientCriterion variable="x_NII" maxChange="0.3" minValue="5e-3"/>
                <MediumStateGradientCriterion variable="x_OIII" maxChange="0.3" minValue="5e-3"/>
            </criteria>
        </DynamicRefinementOptions>
    </dynamicRefinementOptions>
</OctTreeSpatialGrid>
```

In this example, a cell can gain at most four levels beyond its level in the initial grid. A cell at
level 10 in the initial grid can thus reach level 14, and a cell at the grid's maximum level 12 can
reach level 16.

### Refinement options

The `DynamicRefinementOptions` item holds the settings shared by all criteria.

| Property | Description |
| --- | --- |
| `maxExtraLevels` | maximum number of levels a cell may gain beyond its level in the initial grid (zero means no limit) |
| `maxCells` | hard cap on the total number of cells (zero means no cap) |
| `numIterationsBeforeRefinement` | number of initial iterations of each iteration loop during which refinement is not considered |
| `numAveragedIterations` | number of iterations over which the refinement fields are averaged before each decision |
| `criteria` | the list of refinement criteria |

The `maxLevel` property of the grid itself limits the initial construction only.
The `maxExtraLevels` limit is expressed relative to each cell's level in the initial grid. If it is
zero, only the `maxCells` cap limits the refinement, and if both are zero, refinement continues as
long as the criteria ask for it. For a binary tree, which splits a cell along one axis per level,
three levels correspond to one octree level.

When the `maxCells` cap prevents subdividing all cells that ask for it, the cells with the largest
excess over their threshold are subdivided first, and a warning reports how many cells were left
unsplit.

### Refinement criteria

`RefinementCriterion` is the abstract base class for criteria. Several criteria can be configured
at once, for example one per ion; a cell is subdivided if any of them asks for it. Like the
subclasses of `DynamicStateRecipe`, the subclasses omit part of the base class name from their own names.

`GradientCriterion` is an abstract subclass for criteria that compare a per-cell quantity, called
the field of the criterion, between a cell and its face neighbors. Its concrete subclasses define
the field:

| Criterion | Field |
| --- | --- |
| `MediumStateGradientCriterion` | a medium state variable |
| `DustTemperatureGradientCriterion` | the indicative dust temperature |

### Gradient criteria

All gradient criteria offer the following properties.

| Property | Description |
| --- | --- |
| `maxChange` | the threshold; a cell is subdivided if its measure exceeds this value (see below) |
| `minValue` | cells with a field value below this floor are never subdivided |
| `normalizationPercentile` | the percentile of the field over the grid by which the field is divided (zero means no normalization) |

The measure is the larger of two quantities, both expressed as a change in the field across the
cell:

- the magnitude of the gradient, with along each axis the difference between the values on the two
  opposite walls divided by the distance between them, multiplied by the cell size, which captures
  a front spread over several cells regardless of its orientation relative to the grid;
- the largest difference between the cell and a single wall, scaled to the cell size, which
  captures a layer only one cell thick, where the values on opposite walls are similar and the
  first quantity is therefore close to zero.

When a wall has several (smaller) neighbors, their values and center positions are averaged. At the
edge of the domain, the cell itself takes the place of a missing neighbor.

Without normalization, the measure, the threshold, and the floor are expressed in the units of the
field, so fields with very different peak values, such as N+ (peaking near 0.3), O2+ (peaking near
1), or a dust temperature of tens of kelvin, need very different values for `maxChange` and
`minValue`. If `normalizationPercentile` is nonzero, the window-averaged field is divided by that
percentile of the field over all cells in which it is positive, before the floor and the measure
are evaluated. The threshold and the floor then become fractions of the field's typical peak
value, so that similar values can be used for all criteria. A high percentile, such as 99, is used
rather than the maximum, so that a single Monte Carlo outlier cannot set the scale. Normalization
makes sense only for fields with positive values, and not, for example, for the logarithmic
ionization parameter.

### Medium state variables

The `MediumStateGradientCriterion` adds a `variable` property that names a medium state variable.
Any state variable of any material mix can drive refinement, provided the mix gives it a short
name. The standard `temperature` variable is available for all mixes that store a temperature. The
`DiffuseIonizedGasMix` names the following custom variables:

| Variable | Meaning |
| --- | --- |
| `logU` | the ionization parameter (resolves the ionized region) |
| `x_HI` | the neutral hydrogen fraction (resolves the ionization front) |
| `x_HII`, `x_NII`, `x_OI`, `x_OII`, `x_OIII`, `x_SII` | the fraction of H+, N+, O0, O+, O2+, or S+ (resolves that ion's emitting layer) |
| `temperature` | the gas temperature (resolves thermal transitions) |

The variable is taken from the first medium component whose material mix offers a state variable
with the given name. Setup reports a fatal error listing the available names if no component
offers the requested one, and also if that component has no dynamic medium state, primary or
secondary, because the variable then does not change between iterations.

> Selecting a variable by name keeps the `MaterialMix` base class free of mix-specific enumerations
> and ion index tables, and makes new fields available simply by naming a state variable. The
> price is that the schema cannot validate the name, so MakeUp cannot offer a list of choices.

### Dust temperature

The `DustTemperatureGradientCriterion` uses the indicative dust temperature of each cell as its
field, as calculated by `MediumSystem::indicativeDustTemperature()`. This temperature is obtained by
solving the energy balance equation for a representative grain of each dust component in the local
radiation field, averaged over the dust components weighted by their mass in the cell. It is not
stored in the medium state, but calculated from the radiation field when needed. The criterion has
no properties beyond those of all gradient criteria. Setup reports a fatal error if the simulation
has no dust or does not store the radiation field.

### Other criteria

Other quantities derived from the radiation field, such as the mean intensity integrated over a
given wavelength range, or the ratio of two such integrals, could drive refinement through a
`RadiationFieldGradientCriterion`. Such a criterion is not part of this proposal, because the
properties that select the quantity need further thought.

## Iteration and convergence

Dynamic refinement can take place in each of the iteration loops: primary emission iterations,
secondary emission iterations, and merged primary and secondary emission iterations. In a given
loop, only the criteria whose field changes in that loop take part. A variable of a mix with a
primary dynamic medium state changes in the primary and merged loops, and a variable of a mix with
a secondary dynamic medium state in the secondary and merged loops. The dust temperature changes in
all loops, because each loop recalculates at least part of the radiation field. If no criterion
takes part, the refinement step does nothing in that loop.

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
level limit. During the initial iterations and while a window is open, the refinement is not
settled. As a result, each refinement round costs at least `numAveragedIterations + 1` extra
iterations. If the loop ends because it reaches the maximum number of iterations while the
refinement has not settled, a warning is issued.

In a simulation with primary and merged iterations, refinement thus continues in the merged loop,
where the secondary radiation field may change the fields further. In a simulation with separate
secondary iterations, the primary emission has been completed before the secondary loop starts:
the primary radiation field is not recalculated, and the instruments have recorded the primary
emission. A cell created in the secondary loop receives its share of the parent's primary radiation
field, which is uniform over the parent, so that refinement in this loop resolves only structure
caused by the secondary radiation field. For a field that depends mostly on the primary radiation
field, the jump in value at the boundary of a subdivided cell does not diminish with further
subdivision, so that cells along that boundary may keep asking for subdivision until the level
limit or the cell cap stops them (see Open questions).

## Initial state of child cells

A child cell inherits its parent's complete medium state, including the number density, the
temperature, and all custom variables such as the iterated ionization state, as well as its
parent's material mix. Only the cell volume is recalculated from the grid. The total mass of each
medium component is therefore conserved, and the next iteration starts from the previous solution.
The radiation field stored for the parent is divided among the children in proportion to their
volume, so that each child starts with its parent's mean intensity.

Copying the state is correct only because all state variables other than the volume are intensive:
their values do not depend on the size of the cell. This holds for the standard variables, such as
the number density, the metallicity, the temperature, the bulk velocity, and the magnetic field. It
also holds for the custom variables of all current material mixes, which store quantities such as
ionization fractions, level populations per unit volume, and mean intensities. Custom state
variables must therefore be intensive. A quantity that scales with the size of the cell, such as a
mass, a number of particles, or a luminosity, must be stored per unit volume or per unit mass
instead.

Inheriting the state means that the refinement does not resolve any density structure within
the parent cell. Re-sampling the input model for the child cells is a possible alternative (see
Open questions).

## Output and reuse

Each refinement round logs the number of subdivided and added cells and the new total, and warns
when the cell cap prevents subdivision.

The `TreeSpatialGridTopologyProbe` gains a `probeAfter` option
(`Setup`, `Primary`, `Secondary`or `Run`), so that it can
record the topology of the refined grid at the end of the simulation. A subsequent simulation can
load this topology with the `TopologyTreePolicy`. It then starts from the refined grid, either
skipping dynamic refinement altogether, for example to calculate other diagnostics for the same
model, or refining further.

To follow the refinement from one iteration to the next, probes offering a `Primary` or `Secondary`
option for their `probeAfter` property can be used, such as the `CustomStateProbe`. Probes performed
after an iteration see the refined grid, where the new cells still hold their inherited state. The
`SpatialCellPropertiesProbe` gains a column with the tree level of each cell when used with a tree
grid.

## Memory use

The medium state is replicated on each MPI process, so memory use per process grows with the
number of cells. With the `DiffuseIonizedGasMix`, whose state is large, this amounts to about
7 GB per million cells per process.
