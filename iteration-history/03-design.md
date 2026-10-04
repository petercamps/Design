# Design

## Central object

A new class, `IterationHistory`, holds all historical data used by iteration loops. The
`MonteCarloSimulation` creates a single instance as one of its children, in the same way it
creates its `Configuration` object. Like the configuration, the history is a simulation item that
is not configured by the user, and any simulation item can locate it with
`find<IterationHistory>()`.

The history knows nothing about the quantities it stores or about the criteria that use them. It
offers storage with a well-defined lifetime, a few helper functions for common convergence tests,
and a listing of its contents for logging and probing.

## Series

The history holds two kinds of series:

- A **scalar series** holds the values of a single global quantity over the most recent iterations,
  up to a depth requested by the client. Examples are the total dust-absorbed luminosity, the
  fraction of converged cells, and the aggregate of a medium state variable. It replaces instances
  1, 2, and 4 in the catalog.
- A **cell window** holds, for each spatial cell, the running sum of a quantity over the iterations
  of the current averaging window. It replaces instances 5 and 6, and is used only by the proposed
  dynamic grid refinement.

The per-cell comparisons of new and old state values (instance 3) are not moved into the history.

## Identification

A series is identified by a key consisting of a simulation item pointer and an integer id:

- The **item** is the simulation item on whose behalf the series is kept. For most clients, this
  is the client itself. Because each item in the simulation hierarchy is a distinct object, two
  clients of the same type, such as two recipes of the same class, automatically have distinct
  keys.
- The **id** is a small integer chosen by the client, typically from a private enumeration, so
  that a client can keep several series. It defaults to zero.

Material mixes are an exception: their series are keyed on the medium component rather than on the
mix itself. The medium system, which calls the mix's convergence function for a given component,
hands the mix a *scope* bound to that component. The mix then only chooses the id. This has three
advantages: the mix does not need to know which component it belongs to; the key remains stable
when a component uses a material mix family, where the mix object in a given cell is an
implementation detail; and all series of the mix, including its aggregates (see below), share a
single id space that the mix controls.

Identification by name was considered and rejected: names are not unique when several instances of
the same class are present, unless they include some instance identification, which leads back to
using the item pointer. Names remain useful for logging and probing, so each series also carries a
human-readable description.

## On-demand creation

All series are created on demand. If no client asks for a series, the history holds no data and
costs nothing. Declaring a series is idempotent: declaring it again with the same key returns the
existing series, and if the requested depth is larger, the series grows to that depth. Declaring an
existing series with a different lifetime or aggregation rule is a fatal error. A client can
therefore simply declare its series each time it needs it, without separate setup code.

## Aggregates

An aggregate is the volume integral of a medium state variable over all cells, for a given medium
component. Only the medium system can calculate it: it requires a pass over the complete,
synchronized medium state, and it must be recalculated whenever the state changes, including after
updates by dynamic state recipes, which the material mix is not aware of.

The medium system does not need to *own* the aggregates, however. An aggregate series is an
ordinary scalar series, owned by the client that requests it and keyed like any other series of
that client, with an *aggregation rule* attached: the medium component and the custom state
variable to integrate. The medium system sets the value of every series carrying such a rule. As a
result, the client chooses all ids in its own key space, and no separate kind of key is needed.

Aggregates are the one case where the timing of the request matters. The first value of an
aggregate series describes the initial medium state, and must be recorded at the end of the medium
system setup, before any iteration starts. Aggregates must therefore be requested during setup:

- A material mix requests an aggregate by marking the corresponding state variable as aggregated,
  with a series id of its choice, in the list it returns from `specificStateVariableInfo()`. When
  it initializes the medium state, the medium system declares the aggregate series on behalf of
  the mix, keyed on the medium component and the id chosen by the mix.
- Any other client, such as a dynamic state recipe, declares an aggregate series directly with the
  history during its own setup, keyed on itself. The setup of all such clients precedes the final
  setup phase of the medium system.

If two clients request the same aggregate, each gets its own series, and the medium system
calculates the same value twice. This costs one extra number per iteration, which is negligible
compared to the simplicity gained. The medium system calculates only the requested aggregates, so
the fake aggregate cells disappear from the medium state.

## Iteration alignment and lifetime

All scalar series advance together, one slot per iteration, so that "one iteration back" means the
same thing for every series. The iteration loops of `MonteCarloSimulation` drive this with two
calls to the history:

- At the start of each loop (primary, secondary, or merged), `beginLoop()` clears the series with
  loop lifetime.
- At the start of each iteration, `beginIteration()` advances all scalar series by one slot. It
  replaces the current `MediumSystem::beginDynamicMediumStateIteration()`, which shifts the
  aggregate states.

During an iteration, a client sets the value for the current slot. It may set it more than once;
the last value counts. This matches the current aggregate behavior, which recalculates the current
aggregate after the recipe update and again after the material mix update in the same iteration. A
slot that is not set in a given iteration remains empty, and the helper functions report missing
values rather than silently using stale ones.

Each series has one of two lifetimes:

- **Loop** lifetime: the series is cleared at the start of each iteration loop. This is the
  natural choice for most client series, such as the dust absorption and the fraction of converged
  cells.
- **Simulation** lifetime: the series persists for the complete simulation. This is the choice for
  aggregates, so that the first iteration of the primary loop compares with the initial state
  recorded at setup, and the first iteration of a merged loop compares with the last iteration of
  the preceding primary loop, as is the case today.

## Parallelization

The history is replicated on each MPI process. All processes declare the same series and record the
same values, so the histories remain identical without any communication. This imposes one rule on
clients: a recorded value must be identical on all processes, which is the case for values
calculated from synchronized data, such as the medium state after synchronization or the radiation
field after it has been communicated.

Scalar series are declared and set from serial code, once per iteration. Cell windows are
accumulated in a single call that covers all cells, so that the accumulation is identical on all
processes. The loops of the medium state update use a task mode that distributes the cells over
processes, which cannot be used here, and `ParallelFactory` currently offers no mode in which each
process performs all tasks with multiple threads. The accumulation is therefore a serial loop over
the cells, which is cheap compared to an iteration.

When dynamic grid refinement subdivides cells, the medium system asks the history to grow all cell
windows along with the other per-cell data structures. A new cell starts with its parent's running
sum, consistent with inheriting the parent's medium state.

## Logging and probing

Because every series carries its key and a description, the history can list its contents. This
makes it possible to log all convergence quantities in a consistent format, and to write them to a
text column file with a probe performed after each iteration. A scalar series can optionally carry
a physical quantity, as a quantity name known to the `Units` class, so that its values can be
converted to output units. Most convergence metrics, such as fractions and relative changes, are
dimensionless and need no quantity.

The proposal includes such a probe, the `HistoryProbe`. It writes one text column file per
iteration loop, with a row per iteration and a column per scalar series, holding the value set in
that iteration. A simulation that iterates thus produces a file for the primary loop, a file for
the secondary or merged loop, or both. This gives a compact, machine-readable record of how each loop converged, which
serves several purposes:

- finding out which criterion keeps a loop from converging when it reaches its maximum number of
  iterations;
- tuning iteration parameters, such as the minimum and maximum number of iterations, convergence
  fractions, packet ramps, and the parameters of dynamic grid refinement, based on data rather than
  trial and error;
- following dynamic grid refinement, with the number of subdivided cells next to the convergence
  metrics it disturbs;
- estimating the number of iterations, and thus the cost, for a class of models;
- plotting convergence behavior directly, rather than parsing the log;
- comparing convergence behavior between code versions in functional tests, catching changes in
  the number of iterations that the final output would not reveal.

Cell windows are not written. They hold per-cell partial results, potentially millions of numbers
per iteration, and the refined grid and its state can be inspected with existing per-cell probes.
