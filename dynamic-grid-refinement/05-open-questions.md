# Open questions

## Design decisions

The following decisions need input. Each lists the options, followed by the option proposed in
this note.

**Placement of the configuration.** The refinement options can be a property of the tree grid, or
remain in `DynamicStateOptions` as in the reference implementation. The grid is proposed, because
the schema then offers dynamic refinement only where it is supported.

**Absolute level ceiling.** The ceiling for dynamic refinement can be a separate `maxLevel`
property of the refinement options, or the existing `maxLevel` property of the grid. A separate
property is proposed, because construction policies often rely on the grid's `maxLevel` as a
stopping criterion, and a single value would force the user to raise the construction limit in
order to allow deeper dynamic refinement.

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

**Measures and normalization.** Only the `Gradient` measure without normalization was used in
production. The proposal includes the `NeighborDifference` measure and percentile normalization
because they are simple. Alternatively, they can be omitted until there is a use case, reducing the
number of options to document and test.

**Temporal filtering.** The reference implementation averages over per-cell windows in its main
comparison, and counts consecutive positive decisions in its other comparisons. The proposal uses
window averaging for all criteria, with a single schedule for all cells. This makes the decision
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

**Merged iterations.** Refinement could continue during merged primary and secondary iterations,
but this would require growing the data structures of the secondary sources during these
iterations, and has not been tested. The proposal limits refinement to the primary iteration loop.

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
cells could be renumbered after the primary iterations to restore spatial locality. This is not
proposed until measurements show a performance impact.

**Coarsening.** Merging cells whose fields have become smooth is not supported. It would require
removing cell indices, and therefore renumbering all per-cell data structures. No use case has been
identified.

**Testing.** Functional tests of dynamic refinement are affected by Monte Carlo noise in the
refinement decisions. A test-only criterion that refines deterministically can test the mechanics,
but testing the actual criteria requires tolerances in the comparison with reference output, which
may or may not be supported well by the current functional test procedure.

## Observations on the reference implementation

The following observations on the reference implementation are based on reading the code, and have
not been verified by running it:

- `MediumSystem::applyGridRefinement()` assumes that each subdivision adds seven cells when
  applying the cell cap, which is incorrect for binary trees.
- The refinement step is called from `updatePrimaryDynamicMediumState()`, which is also called in
  merged primary and secondary iterations. Refinement can therefore take place in those
  iterations, even though the recipes are configured as relevant for primary iterations.
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
