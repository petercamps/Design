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
   the window-averaged fields. The cells that a criterion flags as too coarse, and that have not
   yet reached the limits of that criterion, are subdivided. The child cells inherit their
   parent's state.
4. The iterations continue on the refined grid. The loop terminates once the medium state has
   converged and the refinement has settled, meaning that an averaging window has passed without
   leading to further subdivision.

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
        <DynamicRefinementOptions maxCells="64000000"
                                  numIterationsBeforeRefinement="6" numAveragedIterations="4">
            <criteria type="RefinementCriterion">
                <MediumStateValueCriterion variable="x_NII" minValue="0.01"
                                           minColumnDensity="1.6e20 1/cm2">
                    <conditions type="RefinementCondition">
                        <MediumStateCondition variable="x_HI" minValue="0.01" maxValue="0.99"/>
                        <MediumStateCondition variable="logU" minValue="-6.5"/>
                    </conditions>
                </MediumStateValueCriterion>
                <MediumStateGradientCriterion variable="x_NII" maxChange="0.3"
                                              minColumnDensity="1.6e20 1/cm2">
                    <conditions type="RefinementCondition">
                        <MediumStateCondition variable="x_HI" minValue="0.01" maxValue="0.99"/>
                        <MediumStateCondition variable="logU" minValue="-6.5"/>
                    </conditions>
                </MediumStateGradientCriterion>
                <MediumStateGradientCriterion variable="x_OIII" maxChange="0.3" minValue="0.005"
                                              maxExtraLevels="2" minColumnDensity="1.6e20 1/cm2">
                    <conditions type="RefinementCondition">
                        <MediumStateCondition variable="logU" minValue="-6.5"/>
                    </conditions>
                </MediumStateGradientCriterion>
            </criteria>
        </DynamicRefinementOptions>
    </dynamicRefinementOptions>
</OctTreeSpatialGrid>
```

This example is modeled on the production runs of the reference implementation (see the
Introduction). The first criterion flags all cells that hold singly ionized nitrogen in partly
ionized gas, and the second flags the cells in such gas with an N+ fraction that differs by more
than 0.3 from that of a neighboring cell. Both keep subdividing a cell until its column density is
below their column density limit, however many levels that takes. The third criterion flags the
cells with an O2+ fraction that differs by more than 0.3 from that of a neighboring cell, ignoring
cells with an O2+ fraction below 0.005. It has the same column density limit, and in addition
subdivides a cell at most two levels beyond its level in the initial grid. A cell at level 10 in
the initial grid can thus reach level 12 through the third criterion, and any level through the
first two. The total number of cells never exceeds 64 million.

### Refinement options

The `DynamicRefinementOptions` item holds the settings shared by all criteria.

| Property | Description |
| --- | --- |
| `maxCells` | hard cap on the total number of cells (zero, the default, means no cap) |
| `numIterationsBeforeRefinement` | number of initial iterations of each iteration loop during which refinement is not considered (the default is five) |
| `numAveragedIterations` | number of iterations over which the refinement fields are averaged before each decision (the default is three) |
| `numIterationsAfterRefinement` | number of iterations after each refinement round that are left out of the averaging (the default is one) |
| `maxRounds` | maximum number of refinement rounds in each iteration loop (zero, the default, means no maximum) |
| `minCandidateFraction` | fraction of the number of cells below which the candidates do not lead to a refinement round (the default is 10^−3; displayed at the expert user level; see Monte Carlo noise below) |
| `levelsPerRound` | number of tree levels by which a refinement round subdivides each candidate (the default is one; displayed at the expert user level) |
| `criteria` | the list of refinement criteria |

The `maxLevel` property of the grid itself limits the initial construction only. During dynamic
refinement, the limits of each criterion tell how far that criterion may subdivide a cell (see
Limits below). If no limit applies to a criterion, only the `maxCells` cap and the `maxRounds`
maximum end the refinement of the cells it keeps flagging, unless the noise test of a gradient
criterion stops it where the field becomes too noisy to resolve (see Monte Carlo noise below).

A refinement round subdivides each selected cell by `levelsPerRound` levels, or by fewer levels
if the level limits of all criteria that flag the cell allow fewer. The cell is split into its
children, the children into theirs, and so on, so that a round turns the cell into up to c^k
cells, where c is the number of children of a tree node (8 for an octree, 2 for a binary tree)
and k the number of levels. With three levels, a binary tree thus splits a cell along all three
axes, as an octree does in a single level. More levels per round let a criterion that keeps
flagging the same cells, such as the dust luminosity criterion described below, reach a given
resolution in fewer rounds, and thus in fewer iterations. The column density limit and the cell
cap are applied to each cell before the round, so that a round of several levels may take a cell
beyond its column density limit.

When the `maxCells` cap prevents subdividing all cells that ask for it, the cells with the highest
priority are subdivided first (see Priority below), and a warning reports how many cells were left
unsplit. The cap counts the cells that each subdivision adds, which is c^k − 1 for a subdivision
by k levels, and the selection stops at the first candidate that no longer fits.

### Refinement criteria

`RefinementCriterion` is the abstract base class for criteria. Several criteria can be configured
at once, for example one per ion; a cell is subdivided if any of them asks for it. Like the
subclasses of `DynamicStateRecipe`, the subclasses omit part of the base class name from their own names.

A criterion consists of three parts:

- a **test** on a per-cell quantity, called the field of the criterion, which flags the cells that
  are too coarse;
- optional **conditions**, which restrict the criterion to the cells whose state lies in given
  ranges;
- **limits**, which tell how far the criterion may subdivide a cell.

A cell asks for subdivision, and is then called a candidate, if at least one criterion flags it
while all conditions of that criterion hold and the cell has not reached a limit of that
criterion.

The concrete criteria combine a test with a field:

| Criterion | Test | Field |
| --- | --- | --- |
| `MediumStateGradientCriterion` | the field differs strongly between the cell and a neighboring cell | a medium state variable |
| `MediumStateValueCriterion` | the field lies in a given range | a medium state variable |
| `DustTemperatureGradientCriterion` | the field differs strongly between the cell and a neighboring cell | the indicative dust temperature |
| `RadiationFieldGradientCriterion` | the field differs strongly between the cell and a neighboring cell | the mean intensity of the radiation field integrated over a wavelength range |
| `DustLuminosityCriterion` | the field exceeds a given fraction | the fraction of the dust-absorbed luminosity that is absorbed in the cell |

The gradient and value tests are implemented by the abstract subclasses `GradientCriterion` and
`ValueCriterion`. The `DustLuminosityCriterion` derives directly from `RefinementCriterion` and
implements its own test. All criteria offer the following properties.

| Property | Description |
| --- | --- |
| `conditions` | the list of conditions that restrict the criterion (the list may be empty) |
| `maxExtraLevels` | the level limit of this criterion: cells that have gained this number of levels beyond their level in the initial grid are not subdivided by the criterion (zero means no limit) |
| `minColumnDensity` | the column density limit of this criterion: cells with a column density below this value are not subdivided by the criterion (zero means no limit) |

### Gradient criteria

A gradient criterion compares the field between a cell and its face neighbors. All gradient
criteria offer the following properties.

| Property | Description |
| --- | --- |
| `logarithmic` | if true, the measure compares the natural logarithms of the field values rather than the values themselves (the default is false) |
| `maxChange` | the maximum difference in field value between a cell and a neighboring cell; a cell is flagged if its measure exceeds this value (see below) |
| `significance` | the number of standard errors by which a difference must exceed `maxChange` for the cell to be flagged (zero, the default, means no noise test; displayed from the regular user level on; see Monte Carlo noise below) |
| `minValue` | cells with a field value below this floor are never flagged (by default, there is no floor) |

The measure of a cell is the largest absolute difference between the window-averaged field value
of the cell and that of any of its face neighbors, which are the cells that share a wall, or part
of a wall, with it. A wall on the edge of the domain has no neighbors.

The measure, the `maxChange` threshold, and the `minValue` floor are expressed in the units of the
field. Fields with different peak values therefore need different values for `maxChange` and
`minValue`.

With the `logarithmic` flag, the measure is instead the largest absolute difference between the
natural logarithms of the window-averaged field values. For small differences, this is the
relative difference, so that the threshold is dimensionless (a value of 0.04 allows a change of
about 4% between neighboring cells) and serves fields with any peak value, such as a temperature
that varies by a large factor across the domain, or a mean intensity that varies by orders of
magnitude. The `minValue` floor is still expressed in the units of the field. A cell with a field
value that is not positive is then never flagged, and a neighbor with such a value is ignored. An
absolute threshold treats a jump from 20 to 25 K like one from 80 to 85 K, although the former
matters much more for the dust emission.

### Value criteria

A value criterion flags each cell in which the window-averaged field lies in a given range. It
does not look at the neighboring cells.

| Property | Description |
| --- | --- |
| `minValue` | the lower bound of the range: a cell is flagged only if its field value is at or above this value (by default, there is no lower bound) |
| `maxValue` | the upper bound of the range: a cell is flagged only if its field value is at or below this value (by default, there is no upper bound) |

Both bounds are inclusive, and are expressed in the units of the field. Either bound can be
omitted, which leaves the range open on that side.

A value criterion expresses that a given kind of gas must be resolved down to a given scale,
wherever it occurs. Because child cells inherit their parent's state, a cell flagged by a value
criterion remains flagged after it has been subdivided, at least until later iterations change
its state. The final resolution is therefore determined by the limits of the criterion, and not
by its test. Setup reports a fatal error if neither a level limit nor a column density limit
applies to a value criterion.

Value criteria complement gradient criteria in two ways. First, a gradient criterion compares a
cell with its neighbors, so that it can miss a coarse cell that holds a complete thin layer: the
state of such a cell is an average over the layer and its surroundings, and may differ little
from the state of the neighboring cells. Second, for some structures the required resolution
follows from a physical scale rather than from the smoothness of the field. In an H II region, for
example, low-ionization lines such as [N II] and [S II] form in a layer at the edge of the
ionized region. The thickness of this layer is set by the mean free path of the ionizing photons,
which scales as the inverse of the density, so that the layer holds a nearly fixed column density
of gas, whatever the size of the region. The appropriate target is then a column density per cell,
which a value criterion with a column density limit expresses directly.

### Conditions

A condition restricts a criterion to part of the medium, for example to partly ionized gas.
`RefinementCondition` is the abstract base class for conditions, and `MediumStateCondition` is
its only subclass for now. It requires a medium state variable to lie in a given range.

| Property | Description |
| --- | --- |
| `variable` | the name of the medium state variable |
| `minValue` | the lower bound of the range: the condition holds only if the value of the variable is at or above this value (by default, there is no lower bound) |
| `maxValue` | the upper bound of the range: the condition holds only if the value of the variable is at or below this value (by default, there is no upper bound) |

A criterion ignores every cell in which one of its conditions does not hold. As for a value
criterion, both bounds are inclusive and expressed in the units of the variable, and either can
be omitted.

Conditions are evaluated on window-averaged values, like the field of the criterion itself, so
that all quantities entering a decision are averaged over the same iterations. A value criterion
thus treats its own range exactly like a condition. The distinction serves to express intent: the
field of the criterion names the structure to be resolved, and the conditions delimit the gas in
which the criterion applies.

### Limits

Each criterion has two limits, which tell how far it may subdivide a cell: a level limit, set by
its `maxExtraLevels` property, and a column density limit, set by its `minColumnDensity`
property. The limits are set on each criterion separately, and for both properties a value of
zero, the default, means that there is no limit. A criterion never flags a cell that has reached
one of its limits. Another criterion with wider limits can still flag that cell.

With a `maxExtraLevels` value of N, the criterion does not flag a cell that has gained N or more
levels beyond its level in the initial grid, whichever criteria caused the earlier subdivisions.
For a binary tree, which splits a cell along one axis per level, three levels correspond to one
octree level.

With a nonzero `minColumnDensity` value, the criterion does not flag a cell with a column density
below that value. The column density of a cell is its number density multiplied by its size,
defined as follows:

- The number density is the one stored in the medium state for the medium component that provides
  the variable of the criterion or, for the dust temperature and dust luminosity criteria, the sum
  over the dust components. Its meaning depends on the material mix. For the
  `DiffuseIonizedGasMix`, it counts hydrogen plus helium nuclei, so that a limit intended as a
  hydrogen column density must be multiplied by one plus the helium abundance.
- The size is the largest extent of the cell along the coordinate axes.

Because child cells inherit their parent's number density, each octree subdivision halves the
column density of a cell. A `minColumnDensity` value therefore produces cells with a size
inversely proportional to the local density, rather than cells at a given level, which matches
the scaling of the layers described under Value criteria above. A limit on the number density
alone could not serve this purpose, because subdividing a cell does not change its number density.

In the example above, all three criteria specify the same `minColumnDensity` value, and only the
third criterion specifies a `maxExtraLevels` value.

### Priority

When the `maxCells` cap does not leave room to subdivide all candidates, the candidates are ranked
by priority. For each criterion that flags a candidate, the priority is the product of two
factors:

- the excess, which is the ratio of the measure to the `maxChange` threshold for a gradient
  criterion, one for a value criterion, and the ratio of the field to the `maxFraction` threshold
  for the dust luminosity criterion;
- the ratio of the cell's column density to the column density limit of the criterion, or one if
  the criterion has no such limit.

The priority of a candidate is the largest of these values over the criteria that flag it. The
candidates with the highest priority are subdivided first. Among candidates with equal priority,
the larger cells go first, and among those the cells with the lower cell index. As a result,
among the cells flagged by a value criterion, the cells that are coarsest for their density are
subdivided first.

### Medium state variables

The medium state criteria and the `MediumStateCondition` have a `variable` property that names a
medium state variable. Any state variable of any material mix can be used, provided the mix gives
it a short name, as proposed in the [Medium state](medium-state/03-features.md) design note. Every
custom variable of the `DiffuseIonizedGasMix` has a short name; these are the most relevant ones
for refinement:

| Variable | Meaning |
| --- | --- |
| `logU` | the ionization parameter (resolves the ionized region) |
| `x_HI` | the neutral hydrogen fraction (resolves the ionization front) |
| `x_HeI` | the neutral helium fraction (resolves the helium ionization front) |
| `x_HII`, `x_NII`, `x_OI`, `x_OII`, `x_OIII`, `x_SII` | the fraction of H+, N+, O0, O+, O2+, or S+ (resolves that ion's emitting layer) |
| `n_HII`, `n_e` | the ionized hydrogen and electron number densities (resolve the structure of the ionized gas) |
| `T` | the gas temperature (resolves thermal transitions) |

The other gas mixes and the `FragmentDustMixDecorator` name their variables as well, for example
the atomic hydrogen fraction `f_atom` of the `SpinFlipHydrogenGasMix` and the dynamic density
fraction `fdyn_f` of fragment f of the `FragmentDustMixDecorator`, which change between
iterations, and the kinetic temperature `T_kin` of the `NonLTELineGasMix`, which is imported and
can thus serve only in a condition.

The variable is taken from the first medium component whose material mix offers a state variable
with the given name. Setup reports a fatal error listing the available names if no component
offers the requested one. For the variable of a criterion, setup also reports a fatal error if
the variable does not change between iterations, that is, if that component has no dynamic medium
state, primary or secondary, and the simulation has no dynamic medium state recipes. A recipe may
update the state of any component in the primary and merged loops, so that in a simulation with
recipes, the variables of all components count as changing in those loops. This allows a criterion
on the dynamic density fraction `fdyn_f` of a `FragmentDustMixDecorator`, which has no dynamic
medium state of its own but is updated by the `DustDestructionRecipe`, as suggested in the
Introduction. The variable of a condition may be static.

### Dust temperature

The `DustTemperatureGradientCriterion` uses the indicative dust temperature of each cell as its
field, as calculated by `MediumSystem::indicativeDustTemperature()`. This temperature is obtained by
solving the energy balance equation for a representative grain of each dust component in the local
radiation field, averaged over the dust components weighted by their mass in the cell. It is not
stored in the medium state, but calculated from the radiation field when needed. The criterion has
no properties beyond those of all gradient criteria. Setup reports a fatal error if the simulation
has no dust or does not store the radiation field.

For a fragmented dust mix, the indicative temperature is the mean of the temperatures of the
fragments, each weighted by its weight factor times its dynamic density fraction (a plain mean for
a geometric medium without destruction), so that it changes where a dust destruction recipe
removes a fragment.

### Radiation field

The `RadiationFieldGradientCriterion` uses the mean intensity of the radiation field integrated over
a wavelength range as its field: the sum of the mean intensity times the bin width over the bins
of the radiation field wavelength grid whose characteristic wavelength lies in the range. Unlike
the dust temperature criterion, it does not involve the properties of the material, and its field
is cheap to calculate, so that it applies to any medium.

| Property | Description |
| --- | --- |
| `minWavelength` | the shortest wavelength of the integration range (by default, the range includes all bins) |
| `maxWavelength` | the longest wavelength of the integration range |

The field is expressed in the internal units of SKIRT (W m^−2 sr^−1). Because the mean intensity
varies by orders of magnitude, the criterion is normally used with the `logarithmic` flag. A
temperature in equilibrium with the radiation field varies roughly as the fourth to sixth root of
the absorbed intensity, so that a relative threshold on the mean intensity corresponds to a four to
six times smaller relative threshold on such a temperature, to the extent that the spectral shape
of the radiation field does not change between the cells. In tests, a threshold of 0.16 to 0.2 on
the bolometric mean intensity gave grids as accurate as a threshold of 0.04 on the indicative dust
temperature.

The criterion takes part in all iteration loops, as the dust temperature criterion does, and the
number density for its column density limit is the sum over all medium components. Setup reports a
fatal error if the simulation does not store the radiation field, or if the wavelength range
contains no bin of the radiation field wavelength grid. A narrow range has a drawback for the noise
test: the noise of the field is estimated from the radiation field summed over all wavelengths
(see Monte Carlo noise below), which underestimates the noise of a narrow range. In a test with a
range of 0.08 to 0.4 µm in an optically thick medium, the noise test did not prevent runaway
refinement, whereas a range up to 1 µm worked.

### Dust-absorbed luminosity

The `DustLuminosityCriterion` flags the cells that absorb more than a given fraction of the total
luminosity absorbed by dust, so that no cell emits more than that fraction of the dust emission.
It resolves the regions that dominate the dust emission, such as the dust near embedded sources or
the self-absorbing core of an optically thick galaxy, without resolving dust that hardly emits. It
is the counterpart, for the absorbed luminosity, of the `DensityTreePolicy`, which limits the
fraction of the dust mass in each cell when the grid is constructed.

| Property | Description |
| --- | --- |
| `maxFraction` | the maximum fraction of the dust-absorbed luminosity in each cell; a cell is flagged if its window-averaged fraction exceeds this value (the default is 10^−5) |
| `primaryOnly` | if true, the field is the fraction of the luminosity absorbed from the primary sources only (the default is false) |

The field is the luminosity absorbed by the dust in the cell divided by the total luminosity
absorbed by the dust, both as accumulated at the exact wavelength of each photon packet (see the
[Dust self-heating](dust-self-heating/02-features.md) design note), and includes the luminosity
absorbed during primary and secondary emission. With `primaryOnly`, both are limited to the
primary emission, so that the criterion resolves the regions where the radiation of the primary
sources is absorbed, without subdividing the cells that mostly absorb the radiation emitted by the
dust itself. The excess of a flagged cell is the ratio of its field to `maxFraction`.

Because a subdivided cell divides its absorbed luminosity among its children, the criterion limits
itself: it stops flagging a cell once the cell absorbs less than the threshold, and it therefore
needs neither a level limit nor a column density limit. In an optically thick cell, however, most
of the luminosity is absorbed by the children facing the radiation, so that these continue to be
subdivided, and around a bright source embedded in very opaque material the refinement follows the
absorbing layer to ever smaller cells. Monte Carlo noise also lets cells close to the threshold
cross it in later windows. In practice, a level limit or the `maxRounds` option therefore bounds
the refinement. Setup reports a fatal error if the simulation does not accumulate the luminosity
absorbed by dust, which requires dust emission and a dust medium.

### Other criteria

A value criterion and a condition based on the indicative dust temperature would follow the same
pattern as their medium state counterparts. They are not part of this proposal, because no use
case has been identified.

The radiation field criterion covers the mean intensity integrated over a wavelength range. Other
quantities derived from the radiation field, such as the ratio of two such integrals, could drive
refinement through further criteria. They are not part of this proposal, because no use case has
been identified.

## Iteration and convergence

Dynamic refinement can take place in each of the iteration loops: primary emission iterations,
secondary emission iterations, and merged primary and secondary emission iterations. In a given
loop, only the criteria whose field changes in that loop take part. A variable of a mix with a
primary dynamic medium state changes in the primary and merged loops, and a variable of a mix with
a secondary dynamic medium state in the secondary and merged loops. In a simulation with dynamic
medium state recipes, the variables of all components also change in the primary and merged loops.
The dust temperature changes in all loops, because each loop recalculates at least part of the
radiation field, and so does the dust-absorbed luminosity. With `primaryOnly`, the dust luminosity
criterion takes part only in the primary and merged loops, which recalculate the primary radiation
field. The conditions of a criterion do not influence whether it takes part. If no criterion takes
part, the refinement step does nothing in that loop.

A simulation may iterate over primary emission if the grid has dynamic refinement options, even if
it has no dynamic medium state. The primary iterations then serve only to refine the grid on the
primary radiation field, for example with the dust luminosity criterion and `primaryOnly`, before
the secondary emission starts. Because the primary radiation field is recalculated from scratch in
each iteration, these iterations may use fewer photon packets than the final primary emission.

The number of photon packets of a primary iteration can follow the growth of the grid. The
`IterationOptions` offer the `primaryIterationPacketsPerCell` property (zero by default, displayed
at the expert user level): if it is nonzero, each primary iteration launches that number of
packets per cell of the current grid, bounded below by the number for the first iteration (set by
`primaryIterationInitialPacketsFraction`) and above by the full number (set by
`primaryIterationPacketsMultiplier`), instead of following the geometric ramp. The noise per cell
then stays about constant while the grid grows, and the early iterations on a coarse grid cost
little. In tests, a refinement starting from 64 cells performed its
first rounds within seconds, and about 1000 packets per cell sufficed: more did not change the
grid. The option applies to the primary loop only, not to the merged loop.

In each loop, the refinement step starts afresh and goes through the following phases:

- During the first `numIterationsBeforeRefinement` iterations, refinement is not considered at
  all, so that the first decision is based on a settled radiation field.
- The refinement fields are then accumulated for `numAveragedIterations` iterations, after which
  the criteria evaluate the window-averaged values, and the refinement step decides which cells
  to subdivide. Averaging over several iterations reduces the Monte Carlo noise that reaches the
  decision.
- If cells are subdivided, which is called a refinement round, the next
  `numIterationsAfterRefinement` iterations are left out of the averaging, because the field is
  still adjusting to the changed grid. A new window starts after that. If no cells are
  subdivided, a new window starts right away.

The initial iterations and the window address different problems. The window reduces Monte Carlo
noise, which scatters around the current solution. At the start of a loop, however, the medium
state drifts systematically from its initial guess toward the solution, for example as ionization
fronts move outward, and the early primary iterations may use fewer photon packets. Averaging a
drifting field yields a value between the initial guess and the solution rather than the solution
itself. Because cells are never merged, a subdivision based on such transient structure is
permanent, whereas a missed subdivision is simply caught by a later window. A longer window could
also absorb the transient, but it would lengthen every refinement round, whereas the initial
iterations delay only the first decision in each loop. Leaving out the iterations after a
refinement round follows the same reasoning: the state of the new cells drifts from the inherited
values toward the solution on the finer grid, which can take more than one iteration.

A decision does not lead to a refinement round, even though cells ask for subdivision, in the
following cases:

- The loop has already performed `maxRounds` refinement rounds.
- The `maxCells` cap leaves no room for subdividing even the candidate with the highest priority,
  by its number of levels.
- Fewer than `numIterationsAfterRefinement + numAveragedIterations` iterations remain before the
  loop reaches its maximum number of iterations. The loop would then end before the new cells
  have been evaluated, and possibly before their state has adjusted to the changed grid.
- The number of candidates is below the fraction `minCandidateFraction` of the number of cells
  (see Monte Carlo noise below).

In each of the first three cases, a warning reports the number of candidates that were left
unsplit. In the last case, an informational message reports it, because a few remaining candidates
are expected once the refinement has done its work.

The iteration loop is considered converged only if the medium state has converged and the
refinement has settled. The refinement has settled in an iteration if a decision was taken in
that iteration and did not lead to a refinement round. During the initial iterations and while a
window is open, the refinement is not settled. As a result, each refinement round costs at least
`numIterationsAfterRefinement + numAveragedIterations` extra iterations. If the loop ends because
it reaches the maximum number of iterations while the refinement has not settled, a warning is
issued.

A refinement round changes the set of cells, so that convergence measures that count cells cannot
be compared across a round. Series in the iteration history that depend on the grid, such as the
fraction of converged cells used by the plateau criterion of the `DiffuseIonizedGasMix`, are
therefore cleared after each round (see the
[Iteration history](iteration-history/03-design.md) design note).

In a simulation with primary and merged iterations, refinement thus continues in the merged loop,
where the secondary radiation field may change the fields further. In a simulation with separate
secondary iterations, the primary emission has been completed before the secondary loop starts:
the primary radiation field is not recalculated, and the instruments have recorded the primary
emission. A cell created in the secondary loop receives its share of the parent's primary radiation
field and of the primary luminosity absorbed by the dust in the parent, which are uniform over the
parent, so that refinement in this loop resolves only structure caused by the secondary radiation
field. For a field that depends mostly on the primary radiation field, the jump in value at the
boundary of a subdivided cell does not diminish with further subdivision, so that cells along that
boundary may keep asking for subdivision until a limit or the cell cap stops them. Structure caused
by the primary radiation field is therefore best resolved in primary iterations before the
secondary loop, as described above.

## Monte Carlo noise

The fields that drive refinement are calculated from the radiation field, directly or through the
medium state, and therefore carry Monte Carlo noise. The noise in a cell grows as the cell gets
smaller, because fewer photon packets cross it. Without precautions, the noise affects the
refinement in three ways:

- **Runaway refinement.** Noise pushes the measure of the many cells whose difference is just
  below the threshold of a gradient criterion over it. Subdividing such a cell reduces its true
  difference with its neighbors but increases its noise, so that its children are flagged even more
  often, and so on, down to cells that hardly receive any photon packets.
- **Creep.** Even if the refinement does not run away, a few cells cross the threshold in nearly
  every decision, so that small rounds follow each other, each costing at least one iteration for
  little gain.
- **Recipe convergence.** At a dust destruction front, for example, the cells whose temperature
  lies within the noise of the destruction temperature change their density fraction in every
  iteration, so that the dust destruction recipe never converges. The number of such cells grows
  with the resolution of the front.

A level limit on each criterion prevents the runaway, but it requires knowing the needed depth in
advance, which is what dynamic refinement is meant to find out. The following features address
these effects instead: an estimate of the noise of the radiation field in each cell, the noise test
of the gradient criteria and the noise allowance of the dust destruction recipes, which use that
estimate, and the candidate guard.

### Noise estimate

The noise of the radiation field in a cell is estimated by dividing the photon packets of each
emission segment into 8 interleaved batches, by their history index, and accumulating the
radiation field of the cell, summed over all wavelengths, separately for each batch. All
contributions of a photon packet, including those of its paths after each scattering event, belong
to the same batch, so that the 8 batch sums are independent estimates of the radiation field of
the cell. If one of the batches received no contributions, the estimate is unreliable, and the cell
counts as infinitely noisy.

The number of batches is a trade-off. The figure shows its effect from statistics alone, assuming
that each batch sum is roughly Gaussian, so that the estimate has one degree of freedom less than
the number of batches.

![Noise estimate with 8 and 16 batches](figures/features-batches.png ':size=100%')

- Panel (a): in 90% of the cells, a single estimate lies between 0.56 and 1.42 times the true
  standard error with 8 batches, and between 0.70 and 1.29 times with 16 batches.
- Panel (b): because of this spread, noise alone exceeds k estimated standard errors more often
  than k true standard errors. For the criteria, a significance of 3 stays well clear of the
  cascade limit with either number of batches (48 times the one-sided probability equals one
  there; see Noise test below). The recipe needs a significance of 5 with 8 batches, where 16
  batches would reach the same rate at 3.85.
- Panel (c): each batch receives a share of the photon packets that cross the cell, so that more
  batches leave more cells with an empty batch, and thus without a usable estimate: with 40
  packets crossing a cell, 5% of the cells with 8 batches, but 75% with 16.

Doubling the number of batches would mainly allow a lower significance for the recipe, at the
cost of about twice as many photon packets per cell for a usable estimate and twice the memory for
the batches. Using 8 batches thus seems a fair middle ground. For the criteria, the tails
are moreover lighter than shown, because the noise test combines the estimates of two cells over
the iterations of the averaging window.

An important caveat is that the estimate refers to the radiation field summed over all wavelengths.
For a quantity that depends on a narrow wavelength range, it is too low (see Radiation field above).

### Noise test

A gradient criterion with a nonzero `significance` k reduces the difference between a cell and each
face neighbor by k times its estimated standard error before comparing it with the threshold. The
standard error combines the noise of the window-averaged field values of both cells. Each criterion
converts the relative noise of the radiation field into the noise of its field, and the noise of a
window average is the root mean square of the noise of its iterations divided by the square root
of their number; with the `logarithmic` flag, the noise is taken relative to the field value. A cell
is thus flagged only if its difference exceeds the threshold with a confidence set by k. Where the
noise is small compared with the threshold, the test changes little; where it is not, the effective
threshold rises, so that a cell that is too noisy to resolve the gradient is not subdivided
further. The test thereby makes the trade-off between photon packets and resolution explicit: with
fewer packets, the refinement stops at coarser cells.

| Criterion | Noise of the field |
| --- | --- |
| `DustTemperatureGradientCriterion` | the change of the indicative dust temperature when the radiation field is multiplied by one plus its relative noise |
| `RadiationFieldGradientCriterion` | the field value times the relative noise of the radiation field |
| `MediumStateGradientCriterion` | the difference between the values of the variable for the radiation field multiplied by 1 + kσ and by 1 − kσ, divided by 2k, with σ the relative noise of the radiation field, if a dynamic medium state recipe can recalculate the variable (see below); otherwise not known, so that the criterion ignores its `significance` property, and setup issues a warning if it is nonzero |

A medium state variable is calculated by the material mix or the recipe that sets it, so that the
criterion cannot convert the noise of the radiation field by itself. It looks for a dynamic medium
state recipe that sets the variable and can recalculate it for a scaled radiation field. The
`DustDestructionRecipe` does so for the dynamic density fractions `fdyn_f` of a
`FragmentDustMixDecorator`, from the equilibrium temperature of the fragment in the scaled field, as
for its noise allowance (see below). For a variable that varies smoothly with the strength of the
radiation field, the difference divided by 2k is its standard error. For a variable that jumps, such
as the density fraction at a sharp destruction front, it is 1/(2k) in a cell whose radiation field
lies within k standard errors of the jump, and zero elsewhere. The difference of 1 between two cells
across the front is then reduced to 0.5 or less if either cell could flip by noise, so that with a
`maxChange` of 0.5, the pair is not flagged: the refinement of the front stops where its cells
become too noisy to resolve it, without a level limit. No recalculation is available for other
variables, such as the ionization fractions of the `DiffuseIonizedGasMix` (see Noise of medium state
variables in the Open questions).

A value of 3 is the natural choice. A cell has six face neighbors, each of which can push it over
the threshold by noise, and each flagged cell becomes eight children, so that a cascade dies out
only if 48 times the one-sided Gaussian tail probability beyond k is below one, which requires k
above 2. With 3, the refinement settled in all tests.

### Noise allowance of dust destruction recipes

The `DustDestructionRecipe`, and thus the `LinearDustDestructionRecipe`, has a `significance`
property as well (zero by default). A cell counts as not converged if the density fraction of a
fragment changes by more than the `densityFractionTolerance`. With a nonzero significance k, the
recipe widens this tolerance in each cell by the change that the noise could cause: the difference
between the density fractions for the radiation field of the cell multiplied by 1 − kσ and by
1 + kσ, where σ is the relative noise of the radiation field in the cell. This assumes that the
fraction varies monotonically with the strength of the radiation field. The solution does not
change, since every cell is still updated in every iteration; only the convergence test ignores the
changes that the noise explains. Because the noise estimate has heavy tails, a value of 5 is
needed. In tests with 3, a significant fraction of cells at a refined front still changed in every
iteration.

The recipes allow a fraction of the number of cells to remain not converged
(`maxFractionNotConvergedCells`), rather than a number of cells as in SKIRT 9 (see the
[Incompatibilities](skirt-10/04-incompatibilities.md) chapter of the SKIRT 10 note), so that the
allowance grows with a refined grid.

### Candidate guard

The `minCandidateFraction` option ends the creep: a decision with fewer candidates than this
fraction of the number of cells does not lead to a refinement round, and the refinement counts as
settled. It guards against the trickle of noise candidates and small rounds near the end of the
refinement, not against a cascade: once a cascade has started, each decision finds many candidates.
In tests, the creep remained even when the number of packets per cell was fixed, so it is not
caused by the decreasing number of packets per cell alone. Presumably, each round creates finer
cells whose values differ more from those of the coarser cells next to them, which moves the
refined region on by a few cells per round. A guard of 1% ended the creep. The default of 10^−3
is more cautious: it ends a tail of rounds of a few cells each, but leaves alone a round that
still splits a few hundred cells in a grid of 100 000. In the photoionization tests of the
Results chapter, it ended such a tail at no measurable cost.

## Initial state of child cells

A child cell inherits its parent's complete medium state, including the number density, the
temperature, and all custom variables such as the iterated ionization state, as well as its
parent's material mix. Only the cell volume is recalculated from the grid. The total mass of each
medium component is therefore conserved, and the next iteration starts from the previous solution.
The radiation field stored for the parent is divided among the children in proportion to their
volume, so that each child starts with its parent's mean intensity. In a simulation with dust
emission, the medium system also records the luminosity absorbed by the dust in each cell, which
sets the luminosity of the dust emission of the cell (see the
[Dust self-heating](dust-self-heating/02-features.md) design note). The parent's absorbed
luminosity is divided among the children in the same way, so that the total luminosity absorbed,
and thus emitted, by the dust is conserved.

How the value of a state variable is passed on depends on its kind, which the material mix
declares as proposed in the [Medium state](medium-state/03-features.md) design note. An intensive
variable does not depend on the size of the cell, and is copied to each child, as is a categorical
variable. This covers the number density, the bulk velocity, the magnetic field, and the variables
of all current material mixes, which store quantities such as temperatures, ionization fractions,
level populations per unit volume, and mean intensities. An extensive variable scales with the
size of the cell, as a mass, a number of particles, or a luminosity does, and is divided among the
children in proportion to their volume.

Inheriting the state means that the refinement does not resolve any density structure within
the parent cell.

## Output and reuse

Each decision logs the number of candidates and, for a refinement round, the number of subdivided
cells, the number of levels per round if it is more than one, the number of added cells, and the
new total number of cells, together with the number of photon packets per cell launched in the
iteration, counting both the primary and the secondary packets in a merged iteration. Warnings
report the candidates that were left unsplit, as described above.

The number of photon packets per cell decreases as the grid grows. This increases the Monte Carlo
noise in the medium state, and thereby in the refinement fields and in the convergence measures.
At setup, the number of packets per cell is therefore also calculated for a grid with `maxCells`
cells, for each iteration loop in which refinement takes place, and a warning is issued if it is
below 100. In primary iterations, the `primaryIterationPacketsPerCell` option keeps the number of
packets per cell constant as the grid grows, within the bounds of the configured numbers (see
Iteration and convergence above).

The refinement step records its progress in the iteration history, so that the `HistoryProbe`
proposed in the Iteration history design note lists it for each iteration: the number of cells,
candidates, and subdivided cells, the cells blocked by each limit, the number of photon packets
per cell, and the memory held by the per-cell data.

The `TreeSpatialGridTopologyProbe` gains a `probeAfter` option (`Setup`, `Primary`, `Secondary`, or
`Run`), so that it can record the topology of the refined grid at the end of the simulation. A
subsequent simulation can load this topology with the `TopologyTreePolicy`. It then starts from the
refined grid, either skipping dynamic refinement altogether, for example to calculate other
diagnostics for the same model, or refining further.

The reloaded grid has the same cells as the refined grid, but not in the same order: the
`TopologyTreePolicy` numbers the cells level by level, as for any tree, whereas a refinement round
keeps the existing cell indices and appends the new cells (see the Implementation chapter).
Per-cell output of the refined simulation, such as the level populations from which a
`NonLTELineGasMix` can restart, therefore does not match the reloaded grid (see Renumbering cells
in the Open questions).

To follow the refinement from one iteration to the next, probes offering a `Primary` or `Secondary`
option for their `probeAfter` property can be used, such as the `CustomStateProbe`. Probes performed
after an iteration see the refined grid, where the new cells still hold their inherited state. The
`SpatialCellPropertiesProbe` gains a column with the tree level of each cell when used with a tree
grid.

## Memory use

The per-cell data are replicated on each MPI process, so memory use per process grows with the
number of cells. The amount per cell depends on the material mixes, which determine the number of
state variables, and on the number of bins of the radiation field wavelength grid. In the
reference production runs, with the `DiffuseIonizedGasMix` and a radiation field wavelength grid
of 75 bins, each process used about 4.4 GB per million cells, in addition to about 30 GB that
does not depend on the number of cells.

If a criterion or a dust destruction recipe uses the noise of the radiation field, the medium
system keeps 8 values per cell for the batches of the primary radiation field, and 16 more in a
simulation with a secondary radiation field. Each criterion with the noise test also keeps a
second cell window, which holds the squared noise of its field.

If a cell cap is configured, the memory for `maxCells` cells is reserved at setup, so that the
memory need of the run is fixed from the start, and refinement rounds do not need additional
memory while they grow the per-cell data. Setup logs the amount of memory reserved for each
process, and issues a warning if the memory of the process then exceeds the physical memory of
the host.
