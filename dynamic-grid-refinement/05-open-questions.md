# Open questions

## Future design considerations

**Initial level after reuse.** A level limit is relative to each cell's level in the
initial grid. When a subsequent simulation loads a refined topology with the `TopologyTreePolicy`,
the refined grid becomes the initial grid, so that cells can gain another `maxExtraLevels` levels
beyond those already reached. This behavior can be accepted and documented, leaving it to the user
to lower `maxExtraLevels` when refining further.

**Shared windows.** Each criterion and each condition accumulates its own cell window, even if
several of them use the same variable, as in the example of the Features chapter, where eight
field windows hold four distinct variables. The refinement step could keep one window per distinct
field instead. This saves memory and time, but requires a notion of field identity that the
criteria and conditions do not need otherwise.

**Open-ended ranges.** A ski file cannot express an infinite value, and the property system has
no notion of an unspecified number. The proposal represents an omitted bound by a default value at
the edge of the range accepted by the property. Alternatives are to let ski files accept infinite
values, or to offer separate properties or classes for ranges with one and with two bounds.

**Column density.** The proposal defines the column density of a cell through the number density
stored in the medium state and the largest extent of the cell. The number density is the only
density that every material mix defines, but its meaning differs between mixes. For the
`DiffuseIonizedGasMix` it includes helium, whereas the paper on the reference implementation
states its target as a hydrogen column density. For cells that are not cubical, the largest extent
is the conservative choice; the cube root of the volume and the diagonal are alternatives. For the
dust temperature criterion, a limit on the optical depth of the cell, as used by the
`OpticalDepthTreePolicy`, may be more natural than a number column density.

**Levels per round.** By default, each refinement round subdivides a candidate by one level, so
that a cell that needs several levels takes as many rounds, each costing at least
`numIterationsAfterRefinement + numAveragedIterations` iterations. In the production runs of the
reference implementation, the finest grid reached its cell cap in the fourth round, while part of
its cells needed a fifth. The `levelsPerRound` option lets a round subdivide each candidate by a
fixed number of levels instead, reduced only by the level limits of the criteria that flag it. In
the [Results](dynamic-grid-refinement/04-results.md), three levels per round on a binary tree,
which split a cell along all three axes as an octree level does, worked well for the dust
luminosity criterion. A fixed number of levels is a blunt instrument, however. It refines all
descendants of a cell, although many of them may turn out to lie outside the layer being resolved
once their state has been calculated, and because the column density limit is checked only before
the round, it may take a cell beyond that limit. For a criterion with a column density limit, the
number of levels that a cell needs to reach the limit is known in advance, so that a round could
instead subdivide each cell by exactly that number, perhaps capped by `levelsPerRound`.

**Media with multiple components.** A medium state variable can be taken from the first
medium component that offers it, or from a component selected explicitly by index. The
first component is proposed for simplicity. An explicit index can be added when a use
case with several medium components arises.

**Initial state of child cells.** Children can inherit their parent's complete state and material
mix, or the input model can be re-sampled for each child. Inheritance is proposed. It conserves
mass exactly, keeps the state consistent, and lets the next iteration start from the previous
solution. Re-sampling would resolve density structure within the parent cell, which is sometimes
the very reason the refinement is needed, but it changes the total mass and requires deciding
which state variables to take from the input model and which from the parent. This could be added
later as an option.

**Refinement in later loops.** Refinement can take place in each iteration loop, and its
schedule starts afresh in each loop. In a simulation with primary and merged iterations,
the merged loop therefore cannot converge before `numIterationsBeforeRefinement +
numAveragedIterations` iterations, even if the grid refined in the primary loop needs no
further refinement. Merged iterations are relatively expensive, so this may matter.
Alternatives are a property that selects the loops in which refinement takes place, or a
shorter schedule in a loop that follows a loop in which the refinement settled. The
proposal accepts the extra iterations until experience shows otherwise.

**Refinement in the secondary loop.** The secondary loop does not recalculate the primary
radiation field, and new cells receive a uniform share of their parent's primary field. For a field
that depends mostly on the primary radiation field, such as the dust temperature in many models,
the jump at the boundary of a subdivided cell therefore persists at every level, and the cells along
that boundary may keep asking for subdivision until a limit or the cell cap stops them.
Alternatives are to exclude the dust temperature criterion from the secondary loop, which removes
dust temperature refinement from simulations without primary iterations, or to recalculate the
primary radiation field after each refinement round in the secondary loop, which costs a primary
emission segment per round. The proposal accepts the behavior and documents it, relying on the
limits, until experience shows otherwise.

**Dust composition.** The indicative dust temperature averages the temperatures of the dust
components weighted by their mass in the cell, so that it depends on the local dust composition as
well as on the radiation field. In the COLIBRE galaxies of the Results chapter, where the
composition varies between cells, the dust temperature criterion did not settle, presumably because
neighbors in the same radiation field differ in temperature and their children inherit that
difference. Refinement cannot resolve such a jump, so the criterion gains nothing from following it.
The cell library of the [Dust self-heating](dust-self-heating/02-features.md) design note had the
same problem, and solves it by weighting each dust component by its total mass in the simulation
rather than by its mass in the cell (see Cell library mapping in its Features chapter). The
criterion could use the same weights, so that its field depends on the radiation field only, while
the fragments of a fragmented dust mix keep their weights per cell, so that the criterion still sees
a destruction front. Trying this on the COLIBRE galaxies would also verify the explanation.
Meanwhile, the radiation field criterion, which does not depend on the dust and settled for ID33341,
can take the place of the dust temperature criterion.

**Convergence rule.** The proposal requires a decision that does not lead to a refinement round
before the loop can converge, so that the loop can converge only in a decision iteration. The
alternative is to accept convergence of the medium state in any iteration without subdivision,
which risks ending the loop while a decision is still pending. Once the cell cap or the maximum
number of rounds rules out any further round, the decisions serve only the diagnostics. The
refinement could then be reported as settled in every iteration after those left out of the
averaging, which lets the loop converge up to one window earlier.

**Rounds near the end of a loop.** The proposal starts no round when fewer than
`numIterationsAfterRefinement + numAveragedIterations` iterations remain in the loop, so that the
loop does not end on cells whose state has not adjusted. The candidates are then left unsplit, and
a warning is issued. A shorter margin, such as only the iterations left out of the averaging,
would allow one more round at the risk of ending the loop on a grid that still asks for
refinement.

**Packets per cell.** The warning for a low number of photon packets per cell uses a threshold of
100, which is a placeholder. The production runs of the reference implementation use a budget of
250 packets per iteration for each cell of the maximum number of cells, the galaxies of the Results
chapter needed 10 to 40 packets per cell in their secondary iterations, and the gradient criteria
with the noise test on the slab about 1 000 in their primary iterations. The threshold thus
depends on the criteria and on what the iterations must converge, and a single value may not
serve. Below about 40 packets per cell, the noise estimate itself becomes unreliable, because a
batch then receives only a few packets. The `primaryIterationPacketsPerCell` option scales the
packets of primary iterations with the number of cells; the secondary and merged iterations have no
such option yet.

**Noise estimate.** The estimate refers to the
radiation field summed over all wavelengths, which underestimates the noise of a quantity that
depends on part of the spectrum: the dust temperature depends mostly on the ultraviolet, and the
noise test of a radiation field criterion over the ultraviolet alone did not prevent runaway
refinement. Batches per wavelength range, or batches of the luminosity absorbed by the dust, would
be more faithful, at a further cost in memory.

**Noise of medium state variables.** The `MediumStateGradientCriterion` estimates the noise of its
variable only if a dynamic medium state recipe can recalculate the variable for a scaled radiation
field, as the `DustDestructionRecipe` does for the density fractions of a fragmented dust mix.
Otherwise, it ignores its `significance` property. For a variable that a material mix calculates
from the radiation field of the cell, such as an ionization fraction, the noise could be estimated
in the same way if the mix offered a way to evaluate its update for a scaled radiation field. For
the `DiffuseIonizedGasMix`, whose update depends on the ionizing part of the spectrum, the noise of
the radiation field summed over all wavelengths would then be too low (see Noise estimate above).

**Resolution of a front.** With the noise test, a criterion on a destruction front stops by itself
near the scale at which noise blurs the front, but at a level that varies along the front, because
the outcome of the test for a pair of cells depends on where the front lies between their centers.
The front is then less accurate than with a level limit at that scale, which has to be chosen by
hand. A rule per cell could do better: subdivide a cell at the front only while its size exceeds the
distance over which noise blurs the front, the relative noise of the temperature divided by its
logarithmic gradient, which does not depend on the position of the front within the cell.

**Recipe convergence.** A dust destruction recipe allows a fraction of the number of cells to remain
not converged. In the tests, the cells that remain not converged lie at the front, and their number
grows by a factor of 4 per level of the front, while the rest of the grid stays the same, so that
the fraction still grows with the resolution of the front (0.04% at level 8, 0.17% at level 9). A
fraction weighted by the dust mass would not depend on the resolution, since the mass of the
layer at the front halves with each level, and it measures the material whose opacity is still
uncertain. It requires a weight per cell in the count of cells that are not converged, and for a
recipe that clears a cell, the mass before the update.

**Image-driven refinement.** Images that see the whole volume of a model, such as the edge-on
images of the optically thin emission of the slab in the Results chapter, resolve the steps between
the large cells of the bulk, and converge slowly with a constant threshold. During the tests, an
option that lets the threshold of a gradient criterion grow by a given factor per level beyond the
initial level was tried: with a factor of √2, it shifted cells from the top layer to the bulk, and
brought the edge-on images close to the noise with 658 000 cells, as close as a Cartesian grid with
1 million cells, where a constant threshold left them 1.4 to 2 times above the noise with 1.07
million cells. The option was removed again, because it
targets the images only indirectly. A criterion that targets the pixel scale of the instruments
would make this explicit.

**Anisotropic refinement.** An octree ties the height of a cell to its width, so that resolving a
thin layer in one direction also refines it in the other two, which limits the gain of refinement
for a nearly one-dimensional structure: on the slab of the Results chapter, Cartesian grids with a
mesh fitted to the gradient needed 2.5 to 3 times fewer cells. A binary tree that splits a cell
along the axis of its largest gradient, rather than cycling through the axes, would avoid most of
this. It requires a split axis per node in the tree, in the path segment generator, and in the
topology input and output.

**Defaults.** The defaults of `numIterationsBeforeRefinement` (5) and `numAveragedIterations` (3)
suit a medium state that evolves over the iterations, as in photoionization, but not a field that
each iteration recalculates from scratch, for which a decision in every iteration works better. The
`logarithmic` flag, the noise test, and the candidate guard are off by default, so that existing
configurations keep their behavior, although the tests suggest `logarithmic` with a significance of
3 for a temperature or a mean intensity, and a guard of 1%. Defaults that depend on the criteria are
possible, but make the configuration harder to understand.

**Candidates probe.** A new `RefinementCandidatesProbe` could write the cells that asked for
subdivision at the most recent decision but were left unsplit, with for each cell its index and its
priority for each criterion. It would show where the cell cap, or any other reason for not
subdividing a candidate, keeps the grid coarser than the criteria ask for. The probe could be
performed at the end of the run, or after each iteration of a loop, and its cell indices would match
those of the other per-cell probes, such as the `SpatialCellPropertiesProbe`, which provides the
position and size of each cell. It would require keeping the outcome of the most recent decision in
a second cell window per criterion. This window would hold a single value per cell rather than a
running sum: the refinement step would reset it and then accumulate it exactly once at each
decision, before subdividing the selected cells, storing the priority that the criterion gave to
each candidate that is left unsplit, and zero for all other cells. The children of a subdivided cell
would thus inherit a value of zero, and the outcome would grow with the grid and be saved in
checkpoints like any other cell window. Alternatives are to record only the highest priority of each
cell and the criterion that gave it, which needs two windows whatever the number of criteria, or to
also report the cells that were subdivided or blocked by a limit. The probe is not proposed, because
the log messages and the history series already give the number of candidates per criterion, the
number blocked by each limit, and the number left unsplit, while the existing per-cell probes show
where the grid was refined, from which the candidates can be reconstructed approximately. The probe
could be added later, when a use case needs to know which cells were left unsplit.

**Growing the arrays.** If a cell cap is configured, the proposal reserves capacity for that
number of cells at setup. The alternative, reallocating and copying the per-cell arrays in each
refinement round, temporarily needs memory for both the old and the new arrays, at the moment when
the grid approaches the size that was chosen to fill the available memory. Reserving relies on the
platform not committing physical memory to capacity that is not yet used; where it does, memory is
wasted whenever the cap is not reached. It also requires containers with a capacity beyond their
size for the medium state and the radiation field tables. Growing in chunks is a compromise, but
it complicates the classes that assume contiguous storage, such as the radiation field tables.

**Renumbering cells.** Cells created by refinement are appended at the end of the cell list. The
cells could be renumbered after the last iteration loop to restore spatial locality. Renumbering
them level by level, as the `TopologyTreePolicy` numbers the cells of a reloaded topology, would
also make the per-cell output at the end of the refined simulation, such as the level populations
from which a `NonLTELineGasMix` can restart, match the reloaded grid. This is not proposed until
measurements show a performance impact, or a use case needs per-cell output that matches a
reloaded grid.

**Coarsening.** Merging cells whose fields have become smooth is not supported. It would require
removing cell indices, and therefore renumbering all per-cell data structures. No use case has been
identified.
