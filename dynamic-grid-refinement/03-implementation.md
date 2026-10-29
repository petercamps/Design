# Implementation

## Responsibilities

The implementation distributes the work over existing classes, following the pattern already used
for dynamic medium state recipes: the configuration items describe what to do, and the medium
system orchestrates the work.

- `DynamicRefinementOptions` and the `RefinementCriterion` and `RefinementCondition` subclasses
  hold the configuration. A criterion tells whether its field changes in a given iteration loop,
  provides the current value of its field in a given cell, and, given the window-averaged field,
  tells whether it flags a given cell and with which excess. A condition provides the current
  value of its variable in a given cell, and tells whether a window-averaged value lies in its
  range. Neither keeps per-cell data of its own. `GradientCriterion` implements the measure and
  the floor, and `ValueCriterion` implements the range test; their subclasses implement the field.
- `BinTreeSpatialGrid` and `OctTreeSpatialGrid` subdivide leaf cells in their flat node array and
  answer geometric queries: cell level, initial level, bounding box, and face neighbors.
- `MediumSystem` performs the refinement step: it accumulates the fields of the criteria and
  conditions, evaluates the criteria with their conditions and limits, selects the cells to
  subdivide, and grows all per-cell data structures.
- `IterationHistory` holds all historical data of the refinement step: scalar series with the
  number of cells subdivided in each iteration and the number of refinement rounds, keyed on the
  `DynamicRefinementOptions` item; and a cell window per criterion and per condition, keyed on that
  criterion or condition. `MediumSystem` keeps no refinement state of its own.
- `MonteCarloSimulation` calls the refinement step from each of the three iteration loops.

## Module

The new classes, `DynamicRefinementOptions` and the criterion and condition hierarchies, live in
the `medium` module, next to the spatial grids that hold the options and the medium system that
performs the refinement step. The criteria and conditions use the medium system, the material
mixes and state variables in the `material` module, and the iteration history in the `tools`
module, all of which `medium` already depends on. The changes to existing classes stay within
their own modules or use existing dependencies: `material` gains only a self-contained addition
(appending cells to the medium state), and the `simulation` and `probe` modules, which call the
refinement step or report the refined grid, already depend on `medium`.
No new dependencies between modules arise. In particular, the `MaterialMix` base class does not
need to know about refinement, because the criteria read the medium state from above.

## Iteration loop

The refinement step becomes an explicit step in `runPrimaryEmissionIterations()`,
`runSecondaryEmissionIterations()`, and `runMergedEmissionIterations()`. It is the last step of each
iteration, after all medium state updates and convergence checks, and before the probe system is
notified. For example, in the primary loop:

```cpp
converged = mediumSystem()->updatePrimaryDynamicMediumState();
converged &= mediumSystem()->updateDynamicRefinement(numPackets);
```

The argument is the number of photon packets launched in the iteration, which the refinement step
uses only to report the number of packets per cell. In the merged loop, it is the sum of the primary
and the secondary packets of the iteration. The `updateDynamicRefinement()` function returns true if
the refinement has settled, as defined in the Features chapter. It first determines the criteria
that take part in the current loop, by passing the history's `loop()` to each criterion. If no
criterion takes part, it returns true right away.

### Use of the iteration history

The refinement step is a client of the iteration history proposed in the
[Iteration history](iteration-history/01-introduction.md) design note, and relies on it for all
state that must survive from one iteration to the next. This section describes how the step gains
access to the history and how it uses it, as the Call sites chapter of that note does for the
other clients.

**Access.** The refinement step is performed by the medium system, which locates the history
during setup. It keys the series that concern the refinement as a whole on the
`DynamicRefinementOptions` item that configures the refinement, and the cell windows of each
refinement criterion, and of each condition that restricts a criterion, on that criterion or
condition. The criteria and conditions themselves keep no history.

**State.** The history provides the following:

- The **iteration index** within the current loop, so that the function compares `iteration()`
  with `numIterationsBeforeRefinement` rather than keeping a counter. The number of iterations
  that remain in the loop follows from the same index and the maximum number of iterations of the
  loop, which the configuration provides.
- A **scalar series** with loop lifetime records the number of cells subdivided in each iteration,
  zero if there was no refinement round. An iteration is left out of the averaging if one of the
  `numIterationsAfterRefinement` preceding values is nonzero. The series also serves to log the
  progress of the refinement.
- A second **scalar series** with loop lifetime holds the number of refinement rounds performed so
  far in the loop, carried over from one iteration to the next, for comparison with `maxRounds`.
- A **cell window** with loop lifetime holds the running sums of the field of each participating
  criterion, and of the variable of each of its conditions. All windows are accumulated and reset
  together, so they share a single schedule, and the windows are complete when `numIterations()`
  reaches `numAveragedIterations`.

**Accumulating and deciding.** At the end of each iteration, in any iteration loop, the refinement
step records the number of subdivided cells, accumulates the current field values unless the
iteration is one of those left out of the averaging after a refinement round, and evaluates the
criteria once the windows are complete. Because declaring is idempotent, the function declares
each series and window whenever it uses it, but it declares all of its scalar series in the first
iteration of the loop, including those that receive a value only later (see Series for logging and
probing below):

```cpp
int numLeftOut = options->numIterationsAfterRefinement();
auto& subdivided = _history->scalarSeries(options, Subdivided, numLeftOut + 1,
                                          IterationHistory::Lifetime::Loop, "number of subdivided cells");
subdivided.set(0);
bool leftOut = false;
for (int lag = 1; lag <= numLeftOut; ++lag) leftOut |= subdivided.has(lag) && subdivided.value(lag) > 0;

auto& window = _history->cellWindow(criterion, Field, _numCells, IterationHistory::Lifetime::Loop,
                                    "window-averaged " + criterion->fieldDescription());
if (!leftOut) window.accumulate([criterion](int m) { return criterion->value(m); });
if (window.numIterations() >= options->numAveragedIterations())
{
    // evaluate the criteria on the window means of each cell and its face neighbors,
    // subdivide the selected cells, reset the windows, and record the number of subdivided cells
}
```

Here `options` is the `DynamicRefinementOptions` item, and `criterion` is one of the participating
criteria. The ids, such as `Subdivided` and `Field`, distinguish the series and windows kept for
the same item. Because series are keyed on the item pointer, two criteria of the same type have
separate windows. The window of a condition is declared in the same way, keyed on the condition.
Two items with the same field accumulate it separately, which costs a little time and memory but
keeps the items independent.

Because all these data have loop lifetime, they are cleared automatically when a loop starts, so
that the refinement schedule starts afresh in each loop.

**Growing the grid.** When cells are subdivided, the medium system calls
`_history->appendCells(parentCells)` along with growing its other per-cell data structures (see
Growing per-cell data below), so that the cell windows of all clients remain consistent with the
grid without any client being notified. The call also causes the history to clear the series with
grid lifetime when the next iteration begins. This includes the fraction of converged cells that
the `DiffuseIonizedGasMix` keeps for its plateau criterion, which counts different cells before
and after a refinement round. The medium system also records the aggregate series again, so that
they describe the refined grid.

### Refinement step

Each iteration, the function first records a zero in the subdivision series, and carries over the
number of rounds. Then, after the initial iterations, it proceeds as follows:

1. Unless this iteration is left out of the averaging, accumulate the current field value of each
   participating criterion, and the current value of the variable of each of its conditions, into
   the corresponding cell window.
2. If the windows are not yet complete, return false.
3. Evaluate all cells in parallel, each cell on a single process, and sum the per-cell results over
   the processes (see Consistency between processes below). For each cell and each participating
   criterion, determine whether all conditions of the criterion hold and whether its test flags the
   cell, both on the window-averaged values. If so, check the limits of the criterion against the
   number of levels the cell has gained and against its column density. A flagged cell that has
   reached a limit is counted as blocked by each limit it has reached, so that a cell that has
   reached both limits counts for both. Otherwise the cell is a candidate, with the priority defined
   in the Features chapter. If several criteria flag the cell, the highest priority counts.
4. If there are candidates, decide whether to start a refinement round. No round is started if
   `maxRounds` rounds have been performed, or if fewer than
   `numIterationsAfterRefinement + numAveragedIterations` iterations remain in the loop.
5. If a round is started, sort the candidates by decreasing priority, breaking ties first on
   decreasing cell size and then on increasing cell index, and select as many as fit under the
   cell cap. Each subdivision adds seven cells for an octree and one cell for a binary tree.
6. Subdivide the selected cells, if any, and grow the per-cell data structures (see below).
7. Reset all field windows, record the number of subdivided cells in the subdivision series, which
   leaves the next iterations out of the averaging, increment the number of rounds if cells were
   subdivided, record the aggregate series of the iteration history so that they reflect the
   refined grid, and log the result. If candidates were left unsplit, issue a warning with their
   number.

The function returns true only in an iteration in which a decision was taken and no cells were
subdivided. The refinement has then settled, as defined in the Features chapter.

### Series for logging and probing

In addition to the series described above, the refinement step records a few scalar series that
serve only for logging and for the history probe proposed in the Iteration history design note.
Like the subdivision series, they are declared in `updateDynamicRefinement()`, with loop lifetime
and depth 1. The global series are keyed on the `DynamicRefinementOptions` item, and the
per-criterion series on the criterion, like the cell windows. All of them are declared in the
first iteration of each loop in which a criterion takes part, including those that are set only in
decision iterations: the `HistoryProbe` takes its columns from the series that exist when it is
first performed in a loop, and leaves out, with a warning, any series declared later.

| Series | Keyed on | Set in | Purpose |
| --- | --- | --- | --- |
| number of iterations accumulated in the current window | options | every iteration | shows the schedule: zero during the initial iterations and in the iterations left out after a refinement round, then counting up to `numAveragedIterations` |
| total number of cells | options | every iteration | shows the growth of the grid, to be compared with `maxCells` |
| number of photon packets per cell | options | every iteration | shows how the Monte Carlo noise per cell evolves as the grid grows |
| memory held by the per-cell data | options | every iteration | shows the memory use per process that depends on the number of cells (see Growing per-cell data) |
| refinement settled (0 or 1) | options | every iteration | shows whether the refinement or the medium state holds up convergence |
| number of candidate cells | options | decision iterations | shows how far the refinement is from settling and, compared with the number of subdivided cells, how many candidates were left unsplit |
| lowest priority among the subdivided cells | options | decision iterations in which the cell cap limits the round | shows how deep the cap cuts into the ranked candidates |
| number of candidate cells | criterion | decision iterations | shows which criterion drives the refinement |
| number of cells blocked by the level limit | criterion | decision iterations | shows whether the level limit of the criterion limits the refinement |
| number of cells blocked by the column density limit | criterion | decision iterations | shows how many of the cells flagged by the criterion have reached the target resolution |

The counts are obtained from the summed results of the parallel evaluation of the cells, and are
therefore identical on all processes. The memory series holds a value calculated from the number of
cells, rather than the measured memory use of the process, for the same reason. Series that are
set only in decision iterations have no value in other iterations, so that the decision points
stand out in the output of the history probe. If no criterion takes part in a loop, the refinement
step sets none of these series in that loop. The log messages of the refinement step are formed
from the same values.

### Consistency between processes

All MPI processes must make identical refinement decisions. The fields are derived from the medium
state and the radiation field after both have been synchronized between processes, so all processes
see identical input. The same holds for the cell windows, which the history accumulates in a single
serial pass over all cells. The evaluation of the cells in the decision step is distributed over the
processes with `parallelDistributed()`, as for the construction of the tree, so that each cell is
evaluated by exactly one process. The per-cell results, which are the highest priority of each cell
and the outcome of each criterion (candidate, or blocked by one or both limits), are then summed
over the processes. Because each cell contributes on a single process, the sums are exact, and all
processes continue with identical results: they rank the candidates, select the cells to subdivide,
and grow the per-cell data independently, without further communication. Any data that is local to a
process, such as a radiation field before it has been communicated, must not enter the decision. The
explicit tie-breaking in the sort guarantees that the selection does not depend on the sort
implementation.

## Grid operations

In the SKIRT 10 design, the tree is constructed as a list of index-linked nodes, which is then
copied into the flat node array used for path segment generation. Dynamic refinement operates
directly on the flat array.

**Subdividing a leaf.** To subdivide leaf node P holding cell m, the grid appends the children to
the end of the node array, sets P's first-child index, and marks P as a nonleaf node. The first
child takes over cell index m, and the other children receive new cell indices at the end of the
cell list. The child extents follow from P's extent and the splitting convention of the tree type.
Because nodes refer to each other by index, growing the node array does not invalidate any links.
This only happens between iterations, never during photon packet transport.

**Neighbor links.** The children's internal walls link to their siblings. Each external wall links
to P's neighbor across that wall, replaced by that neighbor's child at the children's level where
one exists, exactly as in the top-down pass that establishes the links after construction. The
links of other nodes that point to P remain valid: a neighbor link always points to a node at the
same or a coarser level, and the path segment generator descends into a neighbor that has
children. Subdivision is therefore a local operation, without a global rebuild of the neighbor
links. Optionally, the links of finer nodes adjacent to P can be redirected to P's children, which
saves one descent step for paths crossing those walls.

**Face neighbors.** The gradient criteria need the leaf cells adjacent to a cell across each of
its walls. For a given wall, the grid takes the node linked across that wall. If that node is a
leaf, it is the single neighbor on that wall, at the same or a coarser level. Otherwise, the grid
collects the leaves in the node's subtree that touch the wall, by descending only into the
children on the facing side.

**Initial level.** When the flat array is established after construction, the grid records the
level of each cell in a byte array. Children inherit this value from their parent. The level
limits compare a cell's current level with this initial level.

**Cell size.** The column density limits and the ordering of the candidates use the size of a
cell, which is the largest extent of its bounding box along the coordinate axes.

**Topology.** The `TreeSpatialGridTopologyProbe` writes the topology through a depth-first
traversal following the first-child indices, so the order in which nodes were appended does not
affect the output.

## Growing per-cell data

Because the first child of a subdivided cell reuses its parent's cell index and all other children
are appended, no existing cell index changes. Every per-cell data structure remains valid and only
needs to grow, with each new cell copying its parent's values. The following table lists the
per-cell structures, based on the current code.

| Owner | Structure | Values for the new cells |
| --- | --- | --- |
| tree grid | node array, cell-to-node map, initial levels | new nodes; parent's initial level |
| `MediumSystem` | cached number of cells | updated |
| `MediumState` | state variables for each cell | parent's values for intensive and categorical variables, the child's volume fraction of them for extensive variables; volume recalculated from the grid |
| `MediumSystem` | radiation field tables `_rf1`, `_rf2`, `_rf2c` | parent's values times the child's volume fraction |
| `MediumSystem` | dust-absorbed luminosity arrays `_labs1`, `_labs2`, `_labs2c`, in a simulation with dust emission | parent's values times the child's volume fraction |
| `MediumSystem` | material mixes per cell, if applicable | parent's mixes |
| `IterationHistory` | cell windows, including those of the refinement criteria | parent's running sums |

The radiation field tables hold, for each cell, the sum of luminosity times path length of the
photon packets crossing the cell, and the mean intensity follows by dividing this sum by the cell
volume. The parent's values are therefore divided among the children in proportion to their
volume, including the first child, which keeps the parent's cell index. Each child then starts with
the parent's mean intensity, and the total absorbed luminosity is conserved. The dust-absorbed
luminosity arrays hold, for each cell, the luminosity absorbed by the dust along the paths crossing
the cell, calculated with the dust opacity at the exact wavelength of each photon packet. They set
the luminosity of the dust emission of each cell (see the
[Dust self-heating](dust-self-heating/02-features.md) design note), and are divided among the
children in the same way, so that the luminosity emitted by the dust is conserved as well. In the
primary and merged loops, the primary tables and arrays are cleared at the start of the next
iteration, but probes performed after the refining iteration, such as the `RadiationFieldProbe`,
read them. In the secondary loop, the primary field is never recalculated, and the secondary field
of one iteration determines the emission of the secondary sources in the next.

The iteration history grows all its cell windows in a single call to `appendCells()`, whichever
client declared them (see Use of the iteration history above). The field windows of the
refinement criteria and conditions are reset after each decision anyway. Other components need no
changes: the secondary sources size their per-cell arrays and obtain the cell library mapping each
time they prepare for launch, the self-heating rebalancer assigns the cells to its regions at the
start of each iteration, probes query the number of cells when they are performed, and
instruments hold no per-cell data. The per-thread caches that the secondary sources use while
launching photon packets must not keep information about the cells, such as the number of cells,
from one launch preparation to the next.

`MediumSystem` gathers the growth of all these structures in a single function that is called for
each refinement round, followed by a check that the size of each structure matches the new number
of cells. Forgetting to grow one of the structures is the most likely bug in this area, and the
check turns it into an immediate fatal error rather than memory corruption. If the number of
per-cell structures grows in the future, a registration mechanism can replace the explicit list.

If a cell cap is configured, each per-cell structure reserves capacity for `maxCells` cells at the
end of setup, and the node array for the corresponding number of nodes, so that a refinement round
extends the structures in place. Otherwise, a round would allocate each grown array while the old
one still exists, so that the memory use during the round exceeds that of the refined grid. In
production runs, the cap is typically chosen to fill the available memory, and the round that
reaches the cap would then fail. On most platforms, reserved capacity that has not yet been used
occupies address space rather than physical memory, so that reserving costs little when the cap is
not reached. The per-cell containers must therefore support a capacity beyond their size. A
`std::vector` does; the `Array` and `Table` classes used for the medium state, the radiation field
tables, and the dust-absorbed luminosity arrays do not, so these data members need a container that
does. If no cap is configured, the structures are reallocated and copied in each round, with a
temporary increase in memory use.

The list of per-cell structures also yields the number of bytes held per cell, which serves two
purposes. During the iterations, the refinement step records the memory held by the per-cell data
in the iteration history. At the end of setup, if a cell cap is configured, the medium system logs
the amount of memory reserved for the per-cell data, which is the number of bytes per cell times
`maxCells`. It issues a warning if the memory of the process with all of that capacity in use,
which is its current memory use plus the number of bytes per cell times the number of cells that
can still be added, exceeds the physical memory reported by `System::availableMemory()`.

At the end of setup, the medium system also calculates the number of photon packets per cell for a
grid with `maxCells` cells, from the number of packets configured for each iteration loop in which a
criterion takes part (for the merged loop, the primary plus the secondary packets of an iteration),
and issues a warning if that number is below 100.

## Criterion fields

The criteria and conditions rely on the names and kinds of state variables proposed in the
[Medium state](medium-state/04-implementation.md) design note. The `DiffuseIonizedGasMix` assigns
the names listed in the Features chapter to the corresponding variables.

At setup, each medium state criterion and each `MediumStateCondition` resolves its variable name
to a medium component and a state variable index, through the variable list of each material mix,
taking the first component whose mix declares a variable with that name. During the refinement
step, it reads the value directly from the medium state. The loop in which the variable of a
criterion changes follows from whether the component's mix has a primary or secondary dynamic
medium state, and from `Configuration::hasDynamicStateRecipes()`: in a simulation with recipes, the
variable also changes in the primary and merged loops, whatever the component. The number density
for the column density limit of a medium state criterion is that of the same component, as
returned by `MediumSystem::numberDensity()`.

The `DustTemperatureGradientCriterion` calls `MediumSystem::indicativeDustTemperature()`, which also
serves the `TemperatureProbe`. It takes part in all loops. The number density for its column
density limit is the sum over the dust components.

A ski file cannot express an infinite value, so an omitted bound of a value criterion or a
condition is represented by the default value of the property, which is the lowest value the
property accepts for `minValue` (-1e300) and the highest for `maxValue` (1e300). The same holds
for the `minValue` floor of a gradient criterion. The bounds, the floor, and the `maxChange`
threshold are plain numbers in the units of the field, which are the internal units of SKIRT,
because the schema cannot know the quantity of a variable that is selected by name.

## Performance

The decision step is proportional to the number of cells times the average number of face neighbors,
and runs in parallel over the cells, distributed over the processes. It is performed once per
window, so its cost is small compared to the photon packet transport of an iteration. Accumulating
the fields into the cell windows costs one serial pass over the cells per window and per iteration.
This is cheap for medium state variables, but the indicative dust temperature requires solving an
energy balance equation for each dust component in each cell. For that criterion, the values are
first calculated in parallel into a temporary array, distributed over the processes in the same way
and summed over them, after which the serial pass accumulates them. The windows need one number per
cell for each criterion and each condition. For the example in the Features chapter, this amounts to
eight numbers per cell, which is small compared to the medium state of the `DiffuseIonizedGasMix`.

Appended cells lose their spatial locality in the cell list. If this turns out to affect
performance, the cells could be renumbered once the last iteration loop has finished, which
requires permuting all per-cell structures, including the node array's cell indices. This is not
proposed until measurements show a need.

## Testing

The essential invariants of dynamic refinement are that the children tile the parent, existing cell
indices do not change, the mass of each medium component is conserved, neighbor links are
consistent, points are located in the correct cell, and paths traverse the new cells correctly. The
proposal is to verify them in functional tests based on small
photoionization models with dynamic refinement.

A tree grid without policies and with equal `minLevel` and `maxLevel` builds a uniform tree, which
is a simple starting point for testing the subdivision of leaves. A test-only criterion that
deterministically refines the cells inside a given box, in the spirit of the existing
`TrivialGasMix`, allows testing the mechanics without Monte Carlo noise affecting the refinement
decisions. Because the conditions, the limits, and the priorities are deterministic functions of
the fields, the same approach verifies them, including the selection under the cell cap.
