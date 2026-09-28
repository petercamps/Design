# Implementation

## Responsibilities

The implementation distributes the work over existing classes, following the pattern already used
for dynamic medium state recipes: the configuration items describe what to do, and the medium
system orchestrates the work.

- `DynamicRefinementOptions` and the `RefinementCriterion` subclasses hold the configuration. A
  criterion tells whether its field changes in a given iteration loop, provides the current value
  of its field in a given cell, and, given the window-averaged field, returns the measure for a
  given cell. It keeps no per-cell data of its own. `GradientCriterion` implements the measure,
  the floor, and the normalization; its subclasses implement the field.
- `BinTreeSpatialGrid` and `OctTreeSpatialGrid` subdivide leaf cells in their flat node array and
  answer geometric queries: cell level, initial level, bounding box, and face neighbors.
- `MediumSystem` performs the refinement step: it accumulates the fields of the criteria, invokes
  the criteria, selects the cells to subdivide, and grows all per-cell data structures.
- `IterationHistory` holds all historical data of the refinement step: a scalar series with the
  number of cells subdivided in each iteration, keyed on the `DynamicRefinementOptions` item, and a
  cell window per criterion, keyed on the criterion. `MediumSystem` keeps no refinement state of
  its own.
- `MonteCarloSimulation` calls the refinement step from each of the three iteration loops.

## Module

The new classes, `DynamicRefinementOptions` and the criterion hierarchy, live in the `medium`
module, next to the spatial grids that hold the options and the medium system that performs the
refinement step. The criteria use the medium system, the material mixes and state variables in the
`material` module, and the iteration history in the `tools` module, all of which `medium` already
depends on. The changes to existing classes stay within their own modules or use existing
dependencies: `material` gains only self-contained additions (appending cells to the medium state,
short names for state variables), and the `simulation` and `probe` modules, which call the
refinement step or report the refined grid, already depend on `medium`. No new dependencies between
modules arise. In particular, the `MaterialMix` base class does not need to know about refinement,
because the criteria read the medium state from above.

## Iteration loop

The refinement step becomes an explicit step in `runPrimaryEmissionIterations()`,
`runSecondaryEmissionIterations()`, and `runMergedEmissionIterations()`. It is the last step of each
iteration, after all medium state updates and convergence checks, and before the probe system is
notified. For example, in the primary loop:

```cpp
converged = mediumSystem()->updatePrimaryDynamicMediumState();
converged &= mediumSystem()->updateDynamicRefinement();
```

The `updateDynamicRefinement()` function returns true if the refinement has settled, as defined in
the Features chapter. It first determines the criteria that take part in the current loop, by
passing the history's `loop()` to each criterion. If no criterion takes part, it returns true right
away. Otherwise,
it relies on the iteration history for all state that must survive from one iteration to the next:

- The **iteration index** within the current loop is provided by the history, so the function
  compares `iteration()` with `numIterationsBeforeRefinement` rather than keeping a counter.
- A **scalar series** with loop lifetime records the number of cells subdivided in each iteration,
  zero if there was no refinement round. An iteration directly follows a refinement round if the
  value one iteration back is nonzero. The series also serves to log the progress of the
  refinement.
- A **cell window** with loop lifetime holds the running sums of the field of each participating
  criterion. All windows are accumulated and reset together, so they share a single schedule, and
  the windows are complete when `numIterations()` reaches `numAveragedIterations`.

The series and windows are declared on demand, the first time the function needs them:

```cpp
auto& subdivided = _history->scalarSeries(options, 0, 2, IterationHistory::Lifetime::Loop,
                                          "number of subdivided cells");
subdivided.set(0);
bool followsRound = subdivided.has(1) && subdivided.value(1) > 0;

auto& window = _history->cellWindow(criterion, 0, _numCells, IterationHistory::Lifetime::Loop,
                                    "window-averaged " + criterion->fieldDescription());
if (!followsRound) window.accumulate([criterion](int m) { return criterion->value(m); });
```

Here `options` is the `DynamicRefinementOptions` item, and `criterion` is one of the participating
criteria. Because series are keyed on the item pointer, two criteria of the same type have separate
windows. Two criteria with the same field accumulate it separately, which costs a little time but
keeps the criteria independent.

Because all these data have loop lifetime, they are cleared automatically when a loop starts, so
that the refinement schedule starts afresh in each loop. Each iteration, the function first records
a zero in the subdivision series. Then, after the initial iterations, it proceeds as follows:

1. Unless this iteration directly follows a refinement round, accumulate the current field value
   of each participating criterion into its cell window.
2. If the windows are not yet complete, return false.
3. For each criterion that uses normalization, determine the percentile of its window-averaged
   field over all cells in which that field is positive.
4. Evaluate all cells in parallel. For each cell below its level limit, ask each participating
   criterion for its measure and compute the excess, the ratio of the measure to the criterion's
   threshold. The cell is a candidate if its largest excess is greater than one.
5. Sort the candidates by decreasing excess, breaking ties on cell index, and keep as many as fit
   under the cell cap. Each subdivision adds seven cells for an octree and one cell for a binary
   tree.
6. Subdivide the selected cells and grow the per-cell data structures (see below).
7. Reset all cell windows, record the number of subdivided cells in the subdivision series, which
   leaves the next iteration out of the averaging, record the aggregate series of the iteration
   history so that they reflect the refined grid, and log the result.

The function returns true only if a complete window produced no candidates, or if no candidate
could be subdivided because of the cell cap.

### Consistency between processes

The refinement decisions are made independently by each MPI process, without communication. This
is valid because the fields are derived from the medium state and the radiation field after both
have been synchronized between processes, so all processes see identical input and make identical
decisions. The same holds for
the cell windows, which the history accumulates in a single serial pass over all cells. Any data
that is local to a process, such as a radiation field before it has been communicated, must not
enter the decision. The explicit tie-breaking in the sort guarantees that the selection does not
depend on the sort implementation.

## Grid operations

In the SKIRT 10 design, the tree is constructed with pointer-based nodes, and then converted to a
flat, index-linked node array used for path segment generation, after which the pointer-based
nodes are discarded. Dynamic refinement operates directly on the flat array.

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

**Face neighbors.** The criteria need the leaf cells adjacent to a cell across each of its walls.
For a given wall, the grid takes the node linked across that wall. If that node is a leaf, it is
the single neighbor on that wall, at the same or a coarser level. Otherwise, the grid collects the
leaves in the node's subtree that touch the wall, by descending only into the children on the
facing side.

**Initial level.** When the flat array is established after construction, the grid records the
level of each cell in a byte array. Children inherit this value from their parent. The
`maxExtraLevels` limit compares a cell's current level with this initial level.

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
| `MediumState` | state variables for each cell | parent's values, which are intensive; volume recalculated from the grid |
| `MediumSystem` | radiation field tables `_rf1`, `_rf2`, `_rf2c` | parent's values times the child's volume fraction |
| `MediumSystem` | material mixes per cell, if applicable | parent's mixes |
| `IterationHistory` | cell windows, including those of the refinement criteria | parent's running sums |

The radiation field tables hold, for each cell, the sum of luminosity times path length of the
photon packets crossing the cell, and the mean intensity follows by dividing this sum by the cell
volume. The parent's values are therefore divided among the children in proportion to their
volume, including the first child, which keeps the parent's cell index. Each child then starts with
the parent's mean intensity, and the total absorbed luminosity is conserved. In the primary and
merged loops, the tables are cleared at the start of the next iteration, but probes performed after
the refining iteration, such as the `RadiationFieldProbe`, read them. In the secondary loop, the
primary field is never recalculated, and the secondary field of one iteration determines the
emission of the secondary sources in the next.

The iteration history grows all its cell windows in a single call to `appendCells()`, whichever
client declared them. The windows of the refinement criteria are reset after each refinement round
anyway. Other components need no changes: the secondary sources size their per-cell arrays and
obtain the cell library mapping each time they prepare for launch, probes query the number of cells
when they are performed, and instruments hold no per-cell data.

`MediumSystem` gathers the growth of all these structures in a single function that is called for
each refinement round, followed by a check that the size of each structure matches the new number
of cells. Forgetting to grow one of the structures is the most likely bug in this area, and the
check turns it into an immediate fatal error rather than memory corruption. If the number of
per-cell structures grows in the future, a registration mechanism can replace the explicit list.

Each growth operation reallocates and copies the affected arrays, so memory use temporarily
doubles for the largest structures, the medium state and the radiation field tables. With a small
number of refinement rounds, this seems acceptable. Reserving capacity up front based on the cell
cap would avoid the copies, but would waste memory whenever the cap is not reached.

## Criterion fields

`StateVariable::custom()` gains an optional short name, in addition to its description. The
`DiffuseIonizedGasMix` assigns the names listed in the Features chapter to the corresponding
custom variables. The standard variables have fixed names, such as `temperature`. The
documentation of `StateVariable::custom()` also states that custom variables must be intensive, so
that child cells can copy their parent's values (see the Features chapter).

At setup, the `MediumStateGradientCriterion` resolves its variable name to a medium component and
a state variable offset, taking the first component whose mix declares a variable with that name.
During the refinement step, it reads the value directly from the medium state. The loop in which
the variable changes follows from whether the component's mix has a primary or secondary dynamic
medium state.

The `DustTemperatureGradientCriterion` calls `MediumSystem::indicativeDustTemperature()`, which also
serves the `TemperatureProbe`. It takes part in all loops.

## Performance

The decision step is proportional to the number of cells times the average number of face
neighbors, and runs in parallel over the cells. It is performed once per window, so its cost is
small compared to the photon packet transport of an iteration. Accumulating the fields into the cell
windows costs one serial pass over the cells per iteration. This is cheap for medium state
variables, but the indicative dust temperature requires solving an energy balance equation for
each dust component in each cell. For that criterion, the values can first be calculated in
parallel into a temporary array, which the serial pass then accumulates. The windows need one
number per criterion per cell, which is negligible compared to the medium state of the
`DiffuseIonizedGasMix`.

Appended cells lose their spatial locality in the cell list. If this turns out to affect
performance, the cells could be renumbered once the last iteration loop has finished, which
requires permuting all per-cell structures, including the node array's cell indices. This is not
proposed until measurements show a need.

## Testing

The essential invariants of dynamic refinement are that the children tile the parent, existing cell
indices do not change, the mass of each medium component is conserved, neighbor links are
consistent, points are located in the correct cell, and paths traverse the new cells correctly. The
proposal is to verify them in functional tests in the `Functional9` repository, based on small
photoionization models with dynamic refinement.

A tree grid without policies and with equal `minLevel` and `maxLevel` builds a uniform tree, which
is a simple starting point for testing the subdivision of leaves. A test-only criterion that
deterministically refines the cells inside a given box, in the spirit of the existing
`TrivialGasMix`, allows testing the mechanics without Monte Carlo noise affecting the refinement
decisions.
