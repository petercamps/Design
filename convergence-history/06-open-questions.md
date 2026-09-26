# Open questions

**Owner of the history.** The history can be a child of `MonteCarloSimulation`, locatable by any
simulation item, or a member of `MediumSystem`, passed to other clients. A child of the simulation
is proposed, because the iteration loops are clients too, and because the configuration object
provides a precedent for a helper item that is not configured by the user.

**Name.** `IterationHistory` describes what the object holds; `ConvergenceHistory` describes its
main purpose, but fits the cell windows of dynamic grid refinement less well, since these feed
refinement decisions rather than convergence tests.

**Key design.** The proposal identifies a series by an item pointer, a kind, and an id. The kind
exists only to keep the ids of aggregates apart from the ids chosen by a client for the same item.
Alternatives are to reserve negative ids for aggregates, which avoids the extra field but encodes
meaning in a number, or to key aggregates on a different item, such as the medium system with an id
combining the component and variable indices, which complicates the lookup by the material mix.

**Keeping references.** Because declaration is idempotent and cheap, the proposal lets clients
declare their series whenever they need them. Clients with a setup phase could instead keep the
returned references. This saves a map lookup per iteration, which is negligible, at the cost of an
extra data member per series.

**Relative change convention.** The helper function computes the change relative to the current
value, which matches the dust emission and the `DiffuseIonizedGasMix` criteria. The
`NonLTELineGasMix` computes the change relative to the previous value, and keeps its own formula.
Unifying the conventions would change the behavior of that criterion slightly, which is outside the
scope of a refactoring.

**Plateau reset.** The loop lifetime of the plateau series in the `DiffuseIonizedGasMix` changes
when its history is cleared (see the Call sites chapter). If the current behavior must be preserved
exactly, the mix can clear the series itself on convergence, which requires a `clear()` function on
scalar series.

**Aggregates of standard variables.** Only custom variables can be marked for aggregation. The
current mechanism also aggregates the total volume and the number density of each component, but
no client uses these values. Support for standard variables can be added when a client needs it.

**Aggregation types.** The proposal supports a single type of aggregation, the volume integral,
which is meaningful for densities. Other types, such as a volume-weighted or mass-weighted average,
could be offered by giving `aggregated()` an argument, when a client needs them.

**Parallel accumulation.** Cell windows are accumulated in a serial loop over all cells, because
`ParallelFactory` offers no task mode in which each process performs all tasks with multiple
threads. Adding such a mode would allow multithreaded accumulation. The serial loop is proposed
until measurements show that it matters.

**Recipe signature.** Recipes locate the history themselves, rather than receiving a scope as an
argument of `endUpdate()` like material mixes do. Passing a scope would make the two kinds of
clients symmetric, but would change the signature of `endUpdate()` for no current benefit.

**History probe.** A probe that writes all scalar series to a text column file after each
iteration would make convergence behavior easy to inspect. It is not part of this proposal, but the
listing functions of the history are designed to support it.
