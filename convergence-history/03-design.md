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
implementation detail; and the mix's own series live alongside the aggregates of the same
component.

A key also carries a *kind*, distinguishing series chosen by a client from the aggregates managed
by the medium system. For an aggregate, the id is the custom index of the state variable, so the
kind keeps these ids from colliding with the ids chosen by the client for its own series.

Identification by name was considered and rejected: names are not unique when several instances of
the same class are present, unless they include some instance identification, which leads back to
using the item pointer. Names remain useful for logging and probing, so each series also carries a
human-readable description.

## On-demand creation

All series are created on demand. If no client asks for a series, the history holds no data and
costs nothing. Declaring a series is idempotent: declaring it again with the same key returns the
existing series, and if the requested depth is larger, the series grows to that depth. A client can
therefore simply declare its series each time it needs it, without separate setup code.

Aggregates are the one case where the timing of the request matters. The first value of an
aggregate series describes the initial medium state, and must be recorded at the end of the medium
system setup, before any iteration starts. Aggregates must therefore be requested during setup:

- A material mix requests an aggregate by marking the corresponding state variable as aggregated in
  the list it returns from `specificStateVariableInfo()`. The medium system declares the aggregate
  series when it initializes the medium state.
- Any other client, such as a dynamic state recipe, declares an aggregate series directly with the
  history during its own setup, which precedes the final setup phase of the medium system.

The medium system calculates only the requested aggregates, so the fake aggregate cells disappear
from the medium state.

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
makes it possible to log all convergence quantities in a consistent format, or to write them to a
text column file with a probe performed after each iteration. Such a probe is a natural extension,
but is not part of this proposal.
