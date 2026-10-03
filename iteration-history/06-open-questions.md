# Open questions

**End of a loop.** The history has no `endLoop()` function. Loop-lifetime series are cleared when
the next loop begins, so they remain available after a loop has finished, for example to a probe
performed at the end of the run. Also, each iteration loop has several exit points, including
early returns, all of which would need to call `endLoop()`. The drawback is that `loop()` and
`iteration()` keep reporting the last loop after it has finished. If a client needs to know that
no loop is running, each iteration function can create a small guard object whose constructor
calls `beginLoop()` and whose destructor resets the loop to `None` without clearing any series,
covering all exit paths.

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
could be offered by adding an aggregation type to the aggregation rule, and a corresponding
optional argument to `aggregated()`, when a client needs them.

**Parallel accumulation.** Cell windows are accumulated in a serial loop over all cells, because
`ParallelFactory` offers no task mode in which each process performs all tasks with multiple
threads. Adding such a mode would allow multithreaded accumulation. The serial loop is proposed
until measurements show that it matters.

**Recipe signature.** Recipes locate the history themselves, rather than receiving a scope as an
argument of `endUpdate()` like material mixes do. Passing a scope would make the two kinds of
clients symmetric, but would change the signature of `endUpdate()` for no current benefit.

**Series declared late.** The history probe fixes its columns at the first iteration of a loop.
Almost all series exist by then: aggregate series are declared during setup, and other series in
the first iteration in which their client runs, before the probe is performed. A series first
declared in a later iteration of the loop is left out of that loop's file, and the probe logs a
warning. Alternatively, the probe could write one row per iteration and series, which accommodates
any series at any time, but is much less convenient for plotting.
