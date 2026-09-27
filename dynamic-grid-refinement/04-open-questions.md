# Open questions

## Design decisions

The following decisions need input. Each lists the options, followed by the option proposed in
this note.

**Placement of the configuration.** The refinement options can be a property of the tree grid, or
remain in `DynamicStateOptions` as in the reference implementation. The grid is proposed, because
the schema then offers dynamic refinement only where it is supported.

**Absolute level ceiling.** Dynamic refinement can be limited by an absolute ceiling on the cell
level, in addition to the `maxExtraLevels` limit relative to the initial grid and the `maxCells`
cap. The existing `maxLevel` property of the grid cannot serve as that ceiling, because
construction policies often rely on it as a stopping criterion, so a separate property would be
needed. In the reference galaxy run, the ceiling was equal to the grid's `maxLevel` and the
relative limit did the real work. The proposal therefore omits the ceiling. It can be added later as
an optional property if a use case arises.

**Initial level after reuse.** The `maxExtraLevels` limit is relative to each cell's level in the
initial grid. When a subsequent simulation loads a refined topology with the `TopologyTreePolicy`,
the refined grid becomes the initial grid, so that cells can gain another `maxExtraLevels` levels
beyond those already reached. This behavior can be accepted and documented, leaving it to the user
to lower `maxExtraLevels` when refining further. Alternatively, the topology file can store each
cell's initial level, which changes the file format and requires the `TopologyTreePolicy` to pass
these levels to the grid. Documenting the behavior is proposed, because the user configures
`maxExtraLevels` for the new simulation anyway.

**Selecting the refinement field.** A field can be selected by the name of a medium state variable,
or through an enumeration defined by the material mix as in the reference implementation. Names
are proposed, because they keep mix-specific knowledge out of the `MaterialMix` base class and make
any state variable usable. The downside is that neither the schema nor MakeUp can validate the name
or offer a list of choices; setup must report an error listing the available names. There does not
seem to be a mechanism in SMILE that combines both advantages.

**Media with multiple components.** A field can be taken from the first medium component that
offers it, as in the reference implementation, or from a component selected explicitly by index.
The first component is proposed for simplicity. An explicit index can be added when a use case with
several photoionized components arises.

**Measures and normalization.** The reference implementation also offers the plain largest
difference with the face neighbors as a measure, and a normalization that divides the measure by a
high percentile of the field over the grid, making a single threshold comparable across fields with
very different peak values. Only the gradient measure without normalization was used in production,
so the proposal omits the other options, reducing the number of options to document and test.
They can be added as properties of the criterion if a use case arises.

**Temporal filtering.** The reference implementation averages over per-cell windows in its main
comparison, and counts consecutive positive decisions in its other comparisons. The proposal uses
window averaging, with a single schedule for all cells. This makes the decision
points, and therefore the convergence rule, well defined, at the cost of delaying some decisions by
up to one window. The effect on the number of iterations in production runs has not been measured.

**Initial state of child cells.** Children can inherit their parent's complete state and material
mix, or the input model can be re-sampled for each child. Inheritance is proposed. It conserves
mass exactly, keeps the state consistent, and lets the next iteration start from the previous
solution. Re-sampling would resolve density structure within the parent cell, which is sometimes
the very reason the refinement is needed, but it changes the total mass and requires deciding
which state variables to take from the input model and which from the parent. This could be added
later as an option. The reference implementation takes an intermediate position: it re-evaluates
the material mix at the child's center, while inheriting the rest of the state from the parent.
This can make the mix inconsistent with the state variables that were initialized from the input
model for the parent.

**Refinement in later loops.** Refinement can take place in each iteration loop, and its schedule
starts afresh in each loop. In a simulation with primary and merged iterations, the merged loop
therefore cannot converge before `numIterationsBeforeRefinement + numAveragedIterations`
iterations, even if the grid refined in the primary loop needs no further refinement. Merged iterations are relatively
expensive, so this may matter. Alternatives are a property that selects the loops in which
refinement takes place, or a shorter schedule in a loop that follows a loop in which the
refinement settled. The proposal accepts the extra iterations until experience shows otherwise.

**Convergence rule.** The proposal requires a complete window without subdivision requests before
the loop can converge. The alternative is to accept convergence of the medium state in any
iteration without subdivision, as the reference implementation does, which risks ending the loop
while a decision is still pending.

**Cell cap behavior.** When the cap is reached, the proposal considers the refinement settled and
issues a warning. Alternatively, reaching the cap could be treated as a fatal error, forcing the
user to raise the cap or relax the criteria. A warning is proposed, matching the reference
implementation, since the resulting grid is still usable.

**Memory.** The medium state is replicated on each MPI process, which limits the number of cells a
refined grid can reach. There does not seem to be a good option within the scope of this note;
distributing the medium state over processes would be a major change affecting all of SKIRT's
parallelization.

**Growing the arrays.** The proposal reallocates and copies the per-cell arrays in each refinement
round, temporarily doubling their memory use. Reserving capacity based on the cell cap avoids the
copies, but wastes memory when the cap is not reached. Growing in chunks is a compromise, but it
complicates the classes that assume contiguous storage, such as the radiation field tables.

**Renumbering cells.** Cells created by refinement are appended at the end of the cell list. The
cells could be renumbered after the last iteration loop to restore spatial locality. This is not
proposed until measurements show a performance impact.

**Coarsening.** Merging cells whose fields have become smooth is not supported. It would require
removing cell indices, and therefore renumbering all per-cell data structures. No use case has been
identified.

**Testing.** Functional tests of dynamic refinement are affected by Monte Carlo noise in the
refinement decisions. A test-only criterion that refines deterministically can test the mechanics,
but testing the actual criteria requires tolerances in the comparison with reference output, which
may or may not be supported well by the current functional test procedure.

## Departures from the reference implementation

The following list summarizes the differences with the reference implementation that affect the
configuration or the behavior. Implementation differences are discussed in the Implementation
chapter.

1. The configuration moves from `DynamicStateOptions` to the tree grid, and the settings shared by
   all criteria move from the individual recipes to a single options item. In the reference
   implementation, each recipe carries its own copy of these settings, and the driver combines
   them (the smallest cell cap, the longest initial delay), which indicates that they are global in
   nature.
2. Temporal filtering uses window averaging on a single schedule for all cells. The reference
   implementation uses per-cell windows for its averaged gradient comparison.
3. Cells are prioritized by the ratio of their measure to the criterion's threshold, rather than
   by the raw measure. This keeps priorities comparable between criteria that watch fields with
   different units.
4. The iteration loop cannot converge before the refinement has settled. In the reference
   implementation, the loop can terminate during the initial iterations or while a window is
   still open.
5. Refinement can take place in all iteration loops, but only for the criteria whose field is
   updated in the loop. In the reference implementation, it takes place in the primary and merged
   loops, for all recipes.
6. Child cells inherit their parent's material mix. The reference implementation re-evaluates the
   mix at each child's center, while inheriting the rest of the state from the parent.
7. The radiation field of a parent cell is divided among its children in proportion to their
   volume. The reference implementation copies the parent's values to each child, which multiplies
   the mean intensity in the children by the ratio of the parent and child volumes.
8. Refinement fields are selected by the name of a medium state variable, rather than through an
   enumeration in the `MaterialMix` base class.
9. The reference implementation's `RefinedGridDumpProbe`, which combines grid geometry with fields
   specific to the `DiffuseIonizedGasMix`, is not included. Instead, the
   `SpatialCellPropertiesProbe` gains a column with the tree level of each cell.
10. Only the averaged gradient comparison is offered, without normalization, and there is no
    absolute ceiling on the level of refined cells. The reference implementation also offers the
    `MaxAbsoluteDifference` and `MaxNormalizedDifference` comparisons, and a `maxLevel` property.

The configuration properties correspond to those of the reference implementation as follows.

| Proposal | Reference implementation |
| --- | --- |
| — | `maxLevel` |
| `maxExtraLevels` (zero means no limit) | `maxLevelsAboveSeed` (a negative value means no limit) |
| `maxCells` (zero means no cap) | `maxCellCount` (smallest over recipes) |
| `numIterationsBeforeRefinement` | `numIterationsBeforeRefinement` (largest over recipes) |
| `numAveragedIterations` | `requiredPersistence` |
| `criteria` | `refinementRecipes` |
| `field` | `field`, `emittingIonAtomicNumber`, `emittingIonStage` |
| — (always the gradient measure) | `comparison` (only `AveragedGradient` is supported) |
| — | `normalizationPercentile` |
| `maxChange` | `maxNeighborDifference` |
| `minValue` | `minScalar` |

The refinement fields named by the `DiffuseIonizedGasMix` correspond to the values of the reference
implementation's `field` property as follows.

| Proposal | Reference implementation |
| --- | --- |
| `logU` | `IonizationParameter` |
| `x_HI` | `NeutralFraction` |
| `x_HII`, `x_NII`, `x_OI`, `x_OII`, `x_OIII`, `x_SII` | `EmittingIonFraction` with atomic number and stage |
| `temperature` | `Temperature` |

## Observations on the reference implementation

The following observations on the reference implementation are based on reading the code, and have
not been verified by running it:

- `MediumSystem::applyGridRefinement()` assumes that each subdivision adds seven cells when
  applying the cell cap, which is incorrect for binary trees.
- The refinement step is called from `updatePrimaryDynamicMediumState()`, which is also called in
  merged primary and secondary iterations. Refinement can therefore take place in those
  iterations, even though the recipes are configured as relevant for primary iterations.
- The radiation field tables are grown by copying the parent's values to each new child, while the
  first child keeps the parent's values. Because the tables hold sums over path lengths within the
  cell, the mean intensity in each child is overestimated by the ratio of the parent and child
  volumes. This is harmless for the next iteration, which clears the tables, but affects probes of
  the radiation field performed after a refining iteration.
- The iteration loop can converge in an iteration in which no cell happens to be subdivided,
  although candidates are still accumulating persistence or an averaging window is still open. It
  can also converge during the initial iterations, before refinement has been considered at all.
  Whether this happens in practice depends on how quickly the medium state converges after a
  refinement round.
- When several recipes are configured, the first recipe that asks for subdivision determines both
  the measure used for prioritizing and the required persistence, and the persistence counter per
  cell is shared by all recipes.
- `MediumSystem::refine()` re-evaluates the material mix at each child's center, while inheriting
  all state variables from the parent (see above).
- The seed level of each cell is captured on first use rather than at construction, which is
  correct only as long as no cell is subdivided before the first use.
- The documentation of `TreeSpatialGrid::subdivideLeaf()` states that the neighbor lists of the
  parent's external neighbors are not repaired, whereas `TreeNode::subdivide()`, which it calls,
  does repair them, as stated in the readme.
- `DiffuseIonizedGasMix::dynamicRefinementScalar()` translates atomic number and ionization stage
  into the solver's internal ion indices through a hard-coded table, which must be kept in sync
  with the solver.
- The default value of `minScalar` (-7) suits the ionization parameter logU, but not the fraction
  fields, for which a floor of -7 has no effect.
