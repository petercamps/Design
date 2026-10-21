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
extra data member per series. The convergence criteria implemented so far keep no references; the
history probe keeps pointers to the series of its current file, which remain valid for the lifetime
of the history.

**Relative change convention.** The helper function computes the change relative to the current
value, and all three global criteria use it: the dust emission criterion; the
`DiffuseIonizedGasMix`, whose own formula gave the same numbers; and the `NonLTELineGasMix`, whose
formula divided by the previous value and now divides by the current one, as its criterion per cell
already did. This changes the logged values of that criterion, but in the functional tests not its
decisions (see the Call sites chapter).

**Plateau reset.** The lifetime of the plateau series in the `DiffuseIonizedGasMix` changes
when its history is cleared (see the Call sites chapter). The implementation accepts this change:
in the functional tests, it alters only the plateau listed in the log, not the convergence or the
number of iterations. Should the previous behavior be needed after all, the mix can clear the series
itself on convergence, which requires a public `clear()` function on scalar series.

**Per-cell iterates.** Accelerators for fixed-point iterations, such as the Anderson and Ng
acceleration discussed in the Dust self-heating design note, combine the last few iterates of a
state vector in each cell. The history could hold these iterates in a third kind of series, next
to the scalar series and the cell windows, and a helper next to the history could perform the
acceleration step, so that a material mix supplies only its per-cell state and residual. Such a
series would have grid lifetime, because iterates obtained on different grids do not belong to
the same sequence. It is not part of this proposal.

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

**Series declared late.** The history probe fixes the columns of a loop's file when it is first
performed in that loop.
Almost all series exist by then: aggregate series are declared during setup, and other series in
the first iteration in which their client runs, before the probe is performed. A series first
declared in a later iteration of the loop is left out of that loop's file, and the probe logs a
warning. Alternatively, the probe could write one row per iteration and series, which accommodates
any series at any time, but is much less convenient for plotting.
