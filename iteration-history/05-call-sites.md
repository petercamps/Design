# Call sites

For each client, this chapter describes how it gains access to the history and how it uses it.
The code fragments are illustrative; they omit logging and other details that do not change. The
proposed dynamic grid refinement would be a further client, and the only one that uses cell
windows; the Implementation chapter of the
[Dynamic grid refinement](dynamic-grid-refinement/03-implementation.md) design note describes how
it uses the history.

## MonteCarloSimulation

**Access.** The simulation owns the history, next to its configuration:

```cpp
Configuration* _config{new ConfigurationSetup(this)};
IterationHistory* _history{new IterationHistory(this)};
```

**Series ids.** The simulation keys its own series on itself, and distinguishes them by id with a
private enumeration:

```cpp
enum HistoryId { DustAbsorbedLuminosity, NumPrimaryPackets, NumSecondaryPackets, LoopConverged };
```

**Iteration events.** Each of the three iteration functions starts its loop, and each iteration
starts by advancing the history, replacing the call to `beginDynamicMediumStateIteration()`:

```cpp
_history->beginLoop(IterationHistory::Loop::Primary);  // or Secondary, Merged
while (true)
{
    ++iter;
    _history->beginIteration();
    ...
}
```

**Dust emission convergence.** The `DustAbsorptionConvergence` helper class and its `_prevLabsseco`
member are replaced by a private member function of the simulation, which also receives the
iteration index for its log messages:

```cpp
bool MonteCarloSimulation::isDustEmissionConverged(int iter, double fractionOfPrimary,
                                                   double fractionOfPrevious)
{
    auto [Labsprim, Labsseco] = mediumSystem()->totalDustAbsorbedLuminosity();
    auto& series = _history->scalarSeries(this, DustAbsorbedLuminosity, 2, IterationHistory::Lifetime::Loop,
                                          "dust-absorbed secondary luminosity", "bolluminosity");
    series.set(Labsseco);
    return Labsprim <= 0. || Labsseco <= 0. || Labsseco / Labsprim < fractionOfPrimary
           || (series.has(1) && series.relativeChange() < fractionOfPrevious);
}
```

The loop lifetime reproduces the current behavior, where each loop creates a new helper object. In
the first iteration of a loop, there is no previous value, so the relative change criterion does
not apply. Currently, the previous value is initialized to zero, which yields a relative change of
100%; the outcome is the same.

**Loop information.** For the benefit of the history probe, each iteration function also records
the number of photon packets launched in the iteration, which varies with the packet ramp in the
primary loop, and whether the loop has converged, as 0 or 1. These series have depth 1 and loop
lifetime:

```cpp
_history->scalarSeries(this, NumPrimaryPackets, 1, IterationHistory::Lifetime::Loop,
                       "number of primary photon packets").set(Npp);
...
_history->scalarSeries(this, LoopConverged, 1, IterationHistory::Lifetime::Loop,
                       "loop converged").set(converged ? 1. : 0.);
```

The convergence flag is set before the probe system is notified, so that the probe sees it.

## MediumSystem

**Access.** The medium system locates the history during setup, like it locates the
configuration, and keeps the pointer:

```cpp
_config = find<Configuration>();
_history = find<IterationHistory>();
```

**Declaring aggregates for material mixes.** When initializing the medium state, the medium system
declares an aggregate series on behalf of the material mix for each state variable that the mix
marked for aggregation. The series is keyed on the medium component, with the id chosen by the mix:

```cpp
for (auto medium : _media)
{
    const auto& variables = medium->mix()->specificStateVariableInfo();
    _state.initSpecificStateVariables(variables);
    for (const StateVariable& variable : variables)
        if (variable.isAggregated())
            for (int k = 0; k != variable.count(); ++k)
                _history->aggregateSeries(medium, variable.aggregateSeriesId() + k,
                                          AggregationRule{medium, variable.customIndex() + k}, 2,
                                          variable.elementDescription(k),
                                          integratedQuantity(variable.quantity()));
}
```

An array variable has a custom index for each element, and the medium system declares a series for
each element, with consecutive ids, described by the description of the variable and the index of
the element. A scalar variable has a single element, described by the description of the variable.

The quantity of the series follows from the quantity of the variable: the volume integral of a
number density is a pure number, which needs no quantity, and that of a mass density is a mass. The
helper function `integratedQuantity()` performs this mapping, returning the empty string for
quantities it does not know. It is a local function in the implementation file of the medium
system.

**Recording aggregates.** The private function `recordAggregates()` retrieves all aggregate series
from the history, whoever declared them. For each series, it maps the medium in the aggregation
rule to its component index, calculates all volume integrals in a single pass through
`MediumState::volumeIntegrals()`, and sets the current value of each series.
It is called where `calculateAggregate()` is called today: at the end of setup (recording the
initial state), after the dynamic state recipes update the state, and after the material mixes
update the state. With dynamic grid refinement, it is also called after each refinement round.

**Convergence of material mixes.** When collecting convergence information, the medium system
passes each material mix a scope bound to its medium component:

```cpp
for (int h : (primary ? _pdms_hv : _sdms_hv))
{
    HistoryScope scope(_history, _media[h]);
    converged &= mix(0, h)->isSpecificStateConverged(_numCells, numUpdated, numNotConverged, scope);
}
```

## DiffuseIonizedGasMix

**Access.** Through the scope passed to `isSpecificStateConverged()`. The mix keeps no history of
its own: the `mutable` members `_convergedFractionHistory` and `_convergenceHistorySize` are
removed.

**Series ids.** The mix defines the ids of all its series, including its aggregates, in a single
enumeration:

```cpp
enum HistoryId { ConvergedFraction, IonizedHydrogen, FirstIonFraction };
```

**Aggregates.** The mix marks its ionized hydrogen density and its ion fraction aggregates for
aggregation when it adds them to its variable list, keeping their current quantity type:

```cpp
auto variable = StateVariable::custom("ionized hydrogen number density", "numbervolumedensity");
_nHIonized = _variables.add(variable.aggregated(IonizedHydrogen));
```

The ion fraction aggregates are marked with the ids `FirstIonFraction + i`.

**Convergence.** The plateau criterion uses a series with grid lifetime, and the global criterion
reads the aggregate series:

```cpp
constexpr int plateauLength = 3;

auto& fractions = history.scalarSeries(ConvergedFraction, plateauLength,
                                       IterationHistory::Lifetime::Grid, "fraction of converged cells");
fractions.set(convergedFraction);
bool stabilityConverged = fractions.isStable(plateauLength, stabilityConvergenceThreshold());

const auto& ionizedH = history.series(IonizedHydrogen);
bool globalConverged = ionizedH.relativeChange() <= maxChangeInGlobalIonizedH();
```

The former formula also divided by the current value, with a small minimum (1e-12) to avoid a
division by zero. Because the aggregate is the number of ionized hydrogen atoms, that minimum
matters only for a total that is exactly zero, where both formulas report the mix as not
converged, so `relativeChange()` gives the same numbers and decisions. The logged plateau, oldest
value first, is taken from the series, and the logged ion fractions use
`history.series(FirstIonFraction + i)`.
In the first primary iteration, the previous value of each aggregate is the initial state recorded
at setup, as today.

**Behavioral change.** Currently, the plateau history is cleared when the mix reports convergence,
and otherwise never. With grid lifetime, it is cleared at the start of each loop instead. As a
result, the history no longer carries over from the primary loop into a subsequent merged loop.
Conversely, when the loop continues after convergence because the minimum number of iterations
has not been reached, the plateau is no longer restarted. Both changes seem to be improvements.
With dynamic grid refinement, the history is also cleared after each refinement round, because
the fraction of converged cells before the round and the fraction after it count different
cells. The plateau criterion then needs three iterations on the refined grid. In the functional
tests, the change alters only the plateau listed in the log of the cases that use the mix, not
their convergence or their number of iterations.

## NonLTELineGasMix

**Access.** Through the scope passed to `isSpecificStateConverged()`.

**Aggregates.** The mix marks only its level populations for aggregation. They form a single array
variable, marked with the id `FirstLevelPopulation` (zero, since the mix keeps no other series), so
that the population of level `p` has the id `FirstLevelPopulation + p`. Its collision partner
densities, which are aggregated today without being used, are no longer aggregated.

**Convergence.** The global criterion takes the largest relative change of the aggregate series:

```cpp
double changeInGlobalLevelPops = 0.;
for (int p = 0; p != _model.numLevels(); ++p)
{
    double change = history.series(FirstLevelPopulation + p).relativeChange();
    if (std::isnan(change) || change > changeInGlobalLevelPops) changeInGlobalLevelPops = change;
}
```

**Behavioral change.** The change is now relative to the current value rather than the previous
one, consistent with the criterion per cell of the mix, which already divided by the new value,
and with the other global criteria. The special cases for a zero population disappear: a population
that appears from zero is still a change of 100%, and one that stays zero no change, while one that
drops to zero is an infinite rather than a 100% change. A missing value is kept as NaN, so that it
blocks convergence rather than being skipped. Near convergence, the two conventions differ in
second order only, but after a large drop the new value can exceed 100%. In the functional tests,
the logged changes differ, for example 215%, 40%, and 12% instead of 133%, 29%, and 11% for the
three iterations of one case, but the decisions, and thus all output files, are unchanged. The
documentation of the `maxChangeInGlobalLevelPopulations` property shows the new formula.

## AbsorptionOnlyMaterialMixDecorator

The decorator forwards `specificStateVariableInfo()` to the decorated mix, so the aggregation marks
are forwarded as well. It forwards the scope to the decorated mix's `isSpecificStateConverged()`
unchanged.

## Dynamic state recipes

The existing recipes need no history and remain unchanged. A future recipe that needs history
locates the history during setup, keeps the pointer, and keys all its series, including any
aggregates, on itself:

```cpp
enum HistoryId { NotConverged, TotalFragmentDensity };

void SomeRecipe::setupSelfAfter()
{
    DynamicStateRecipe::setupSelfAfter();
    _history = find<IterationHistory>();
    _history->aggregateSeries(this, TotalFragmentDensity, AggregationRule{medium, customIndex});
}

bool SomeRecipe::endUpdate(int numCells, int numUpdated, int numNotConverged)
{
    auto& series = _history->scalarSeries(this, NotConverged, 2, IterationHistory::Lifetime::Loop,
                                          "number of not-converged cells");
    series.set(numNotConverged);
    double change = _history->series(this, TotalFragmentDensity).relativeChange();
    ...
}
```

Recipes are set up before the final setup phase of the medium system, so the aggregates they
declare are recorded from the initial state onward. This gives recipes access to aggregate
information, as envisioned by the comment in `MediumSystem::updateDynamicStateRecipes()`.

## HistoryProbe

**Access.** The probe locates the history during setup, like any other client:

```cpp
_history = find<IterationHistory>();
```

**Availability.** The wizard offers the probe only for simulations that iterate over primary or
secondary emission (`ATTRIBUTE_TYPE_DISPLAYED_IF` with `IteratePrimary|IterateSecondary`).

**Timing.** The probe returns `When::Iterations`, so that it is performed after each iteration of
each loop, through `ProbeSystem::probePrimary()` in the primary loop and
`ProbeSystem::probeSecondary()` in the secondary and merged loops. The history's `loop()` and
`iteration()` tell the probe which loop and iteration it is looking at.

**Output.** The probe writes a separate text column file for each iteration loop. When it is first
performed in a loop, it opens the file for that loop, named after the simulation prefix, the probe
name, and the loop, for example `prefix_history_primary.dat`, `prefix_history_secondary.dat`, or
`prefix_history_merged.dat`. Each loop runs at most once in a simulation, so a file is never
reopened. The first column holds the iteration index within the loop. The other columns follow the
scalar series listed by `allScalarSeries()` at that time, so that each file has its own set of
columns. A series with simulation lifetime, such as an aggregate, appears in the file of each loop
in which it exists, so that its evolution across loops can be followed by combining the files. Each
column header combines the description of the series with a description of its item, such as its
type and, for a medium component, its index, so that series of different items of the same type can
be told apart, as in `population of level 1 -- medium 0` or
`loop converged -- MonteCarloSimulation`. Values are converted to output units according to the
quantity of each series. For each iteration, the probe appends a row with the current value of each
series, or NaN if the value was not set in that iteration, and flushes the file, so that the
progress of a long run can be followed while it executes, and the output survives an aborted run.
Only the root process writes the files; the series hold identical values on all processes.

**End of the run.** The probe overrides `probeRun()`, which the probe system calls at the end of
the run, to close the file of the last loop. If the output path of the simulation names an HDF5
file, it then stores each of its files as a text column file bundle through
`ColumnOutFile::copyToBundle()`, leaving the regular file in place. The functional test procedure
of PTS accepts such a bundle next to the regular file when both hold the same information, whereas
it counts any other file written both ways as a failure.

## Removed code

The refactoring removes:

- the `DustAbsorptionConvergence` helper class in `MonteCarloSimulation.cpp`;
- the aggregate cells in `MediumState`, together with the third argument of `initConfiguration()`,
  and the functions `calculateAggregate()` and `pushAggregate()`;
- `MediumSystem::beginDynamicMediumStateIteration()`, and the construction of `MaterialState`
  objects for the aggregate cells in `MediumSystem::updateDynamicStateMedia()`;
- the `mutable` members `_convergedFractionHistory` and `_convergenceHistorySize` of the
  `DiffuseIonizedGasMix`.
