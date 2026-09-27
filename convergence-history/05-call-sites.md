# Call sites

For each client, this chapter describes how it gains access to the history and how it uses it.
The code fragments are illustrative; they omit logging and other details that do not change.

## MonteCarloSimulation

**Access.** The simulation owns the history, next to its configuration:

```cpp
Configuration* _config{new ConfigurationSetup(this)};
IterationHistory* _history{new IterationHistory(this)};
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
member are replaced by a private member function of the simulation, which keys its series on the
simulation itself:

```cpp
bool MonteCarloSimulation::isDustEmissionConverged(double fractionOfPrimary, double fractionOfPrevious)
{
    double Labsprim, Labsseco;
    std::tie(Labsprim, Labsseco) = mediumSystem()->totalDustAbsorbedLuminosity();
    auto& series = _history->scalarSeries(this, 0, 2, IterationHistory::Lifetime::Loop,
                                          "dust-absorbed secondary luminosity");
    series.set(Labsseco);
    return Labsprim <= 0. || Labsseco <= 0. || Labsseco / Labsprim < fractionOfPrimary
           || (series.has(1) && series.relativeChange() < fractionOfPrevious);
}
```

The loop lifetime reproduces the current behavior, where each loop creates a new helper object. In
the first iteration of a loop, there is no previous value, so the relative change criterion does
not apply. Currently, the previous value is initialized to zero, which yields a relative change of
100%; the outcome is the same.

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
    auto variables = medium->mix()->specificStateVariableInfo();
    _state.initSpecificStateVariables(variables);
    for (const auto& variable : variables)
        if (variable.isAggregated())
            _history->aggregateSeries(medium, variable.aggregateSeriesId(),
                                      AggregationRule{medium, variable.customIndex()}, 2,
                                      variable.description());
}
```

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
aggregation, keeping their current quantity type:

```cpp
result.push_back(StateVariable::custom(index++, "ionized hydrogen number density",
                                       "numbervolumedensity").aggregated(IonizedHydrogen));
```

The ion fraction aggregates are marked with the ids `FirstIonFraction + i`.

**Convergence.** The plateau criterion uses a series with loop lifetime, and the global criterion
uses the aggregate series:

```cpp
constexpr int plateauLength = 3;

auto& fractions = history.scalarSeries(ConvergedFraction, plateauLength,
                                       IterationHistory::Lifetime::Loop, "fraction of converged cells");
fractions.set(convergedFraction);
bool stabilityConverged = fractions.isStable(plateauLength, stabilityConvergenceThreshold());

const auto& ionizedH = history.series(IonizedHydrogen);
bool globalConverged = ionizedH.relativeChange() <= maxChangeInGlobalIonizedH();
```

The logged ion fractions use `history.series(FirstIonFraction + i)` in the same way.
In the first primary iteration, the previous value of each aggregate is the initial state recorded
at setup, as today.

**Behavioral change.** Currently, the plateau history is cleared when the mix reports convergence,
and otherwise never. With loop lifetime, it is cleared at the start of each loop instead. As a
result, the history no longer carries over from the primary loop into a subsequent merged loop.
Conversely, when the loop continues after convergence because the minimum number of iterations
has not been reached, the plateau is no longer restarted. Both changes seem to be improvements.

## NonLTELineGasMix

**Access.** Through the scope passed to `isSpecificStateConverged()`.

**Aggregates.** The mix marks only its level populations for aggregation, using the level index `p`
as the series id, since it keeps no other series. Its collision partner densities, which are
aggregated today without being used, are no longer aggregated.

**Convergence.** The global criterion keeps its own formula, which is relative to the previous
value rather than the current one, and reads the values from the aggregate series:

```cpp
const auto& population = history.series(p);
double currentPop = population.value(0);
double previousPop = population.value(1);
```

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

## Possible future dynamic grid refinement

The [Dynamic grid refinement](dynamic-grid-refinement/01-introduction.md) design note proposes
subdividing cells between iterations, based on per-cell fields, such as medium state variables or
the indicative dust temperature, averaged over several iterations. If that proposal is implemented,
its refinement step would use the history as follows.

**Access.** The refinement step is performed by the medium system, which locates the history
during setup. It keys the scalar series with the number of subdivided cells on the
`DynamicRefinementOptions` item that configures the refinement, and the cell window of each
refinement criterion on that criterion. The criteria themselves keep no history.

**Accumulating and deciding.** At the end of each iteration, in any iteration loop, the refinement
step records the number of subdivided cells, accumulates the current field values unless the
iteration directly follows a refinement round, and evaluates the criteria once the windows are
complete:

```cpp
auto& subdivided = _history->scalarSeries(options, 0, 2, IterationHistory::Lifetime::Loop,
                                          "number of subdivided cells");
subdivided.set(0);
bool followsRound = subdivided.has(1) && subdivided.value(1) > 0;

auto& window = _history->cellWindow(criterion, 0, _numCells, IterationHistory::Lifetime::Loop,
                                    "window-averaged " + criterion->fieldDescription());
if (!followsRound) window.accumulate([criterion](int m) { return criterion->value(m); });
if (window.numIterations() >= options->numAveragedIterations())
{
    // evaluate the criteria on the window means of each cell and its face neighbors,
    // subdivide the selected cells, reset the windows, and record the number of subdivided cells
}
```

Because the series have loop lifetime, the refinement schedule starts afresh in each loop. The
history's `iteration()` tells the refinement step when the initial iterations without refinement
have passed.

**Growing the grid.** When cells are subdivided, the medium system calls
`_history->appendCells(parentCells)` along with growing its other per-cell data structures, so that
the cell windows of all clients remain consistent with the grid without any client being notified.
The medium system also records the aggregate series again, so that they describe the refined grid.

**Convergence.** The refinement has settled when a complete window produced no subdivision
requests. The subdivision series also serves to log the progress of the refinement.

## Removed code

The refactoring removes:

- the `DustAbsorptionConvergence` helper class in `MonteCarloSimulation.cpp`;
- the aggregate cells in `MediumState`, together with the third argument of `initConfiguration()`,
  and the functions `calculateAggregate()` and `pushAggregate()`;
- `MediumSystem::beginDynamicMediumStateIteration()`, and the construction of `MaterialState`
  objects for the aggregate cells in `MediumSystem::updateDynamicStateMedia()`;
- the `mutable` members `_convergedFractionHistory` and `_convergenceHistorySize` of the
  `DiffuseIonizedGasMix`.
