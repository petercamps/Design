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

**Convergence rule.** The proposal requires a decision that does not lead to a refinement round
before the loop can converge, so that the loop can converge only in a decision iteration. The
alternative is to accept convergence of the medium state in any iteration without subdivision,
which risks ending the loop while a decision is still pending. Once the cell cap or the maximum
number of rounds rules out any further round, the decisions serve only the diagnostics. The
refinement could then be reported as settled in every iteration after those left out of the
averaging, which lets the loop converge up to one window earlier.

**Settling.** In the proposal, a single candidate causes a refinement round. A few cells that
Monte Carlo noise moves in and out of a criterion can thus cause one round after another, each
costing at least `numIterationsAfterRefinement + numAveragedIterations` iterations. The `maxRounds`
property bounds this, but its default of zero means no maximum. A nonzero default would bound the
loop automatically, at the risk of silently cutting off the refinement of a model that needs more
rounds. A tolerance, which considers the refinement settled as long as no more than a given number
of cells ask for subdivision, was considered and rejected. Such a number has no natural scale, it
cannot tell noise from a small structure that really needs refinement, and the cells within the
tolerance are still subdivided as soon as a later decision exceeds it.

**Rounds near the end of a loop.** The proposal starts no round when fewer than
`numIterationsAfterRefinement + numAveragedIterations` iterations remain in the loop, so that the
loop does not end on cells whose state has not adjusted. The candidates are then left unsplit, and
a warning is issued. A shorter margin, such as only the iterations left out of the averaging,
would allow one more round at the risk of ending the loop on a grid that still asks for
refinement.

**Packets per cell.** The warning for a low number of photon packets per cell uses a threshold of
100, which is a placeholder. The production runs of the reference implementation use a budget of
250 packets per iteration for each cell of the maximum number of cells. The threshold needs to be
calibrated, and may depend on the criteria.

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
