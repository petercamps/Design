# API

This chapter proposes the interface of the new classes, and the changes to existing classes. The
declarations show the essential functions only; the documentation comments state their contracts.

## Series key

```cpp
/** A SeriesKey identifies a series in the IterationHistory. */
struct SeriesKey
{
    const SimulationItem* item;  // the item on whose behalf the series is kept
    int id;                      // an id chosen by the client, unique for that item

    bool operator<(const SeriesKey& other) const;  // lexicographic, so that keys can index a map
};
```

## Aggregation rule

```cpp
/** An AggregationRule specifies the value that the medium system records in an aggregate series:
    the volume integral over all cells of the custom state variable with the specified index, for
    the specified medium component. */
struct AggregationRule
{
    const Medium* medium;  // the medium component
    int customIndex;       // the index of the custom state variable
};
```

## Scalar series

```cpp
/** A ScalarSeries holds the values of a single global quantity over the most recent iterations,
    up to its depth. All scalar series in the history advance together, one slot per iteration. */
class ScalarSeries
{
public:
    /** Sets the value for the current iteration, replacing any value set earlier in the same
        iteration. The value must be identical on all processes. */
    void set(double value);

    /** Returns true if a value was set \em lag iterations back, where zero indicates the current
        iteration. Returns false if the series was cleared since then, or if lag >= depth(). */
    bool has(int lag = 0) const;

    /** Returns the value set \em lag iterations back, or NaN if has(lag) returns false. */
    double value(int lag = 0) const;

    /** Returns the change of the current value relative to the value \em lag iterations back,
        i.e. |v(0) - v(lag)| / |v(0)|. Returns zero if both values are equal, infinity if v(0) is
        zero and v(lag) is not, and NaN if either value is missing. */
    double relativeChange(int lag = 1) const;

    /** Returns true if the \em count most recent values are all available and each of them
        differs from the next by at most \em tolerance, and false otherwise. */
    bool isStable(int count, double tolerance) const;

    /** Returns the key, description, and depth of the series. */
    const SeriesKey& key() const;
    const string& description() const;
    int depth() const;

    /** Returns the aggregation rule if this is an aggregate series, or null otherwise. */
    const AggregationRule* aggregationRule() const;
};
```

## Cell window

```cpp
/** A CellWindow holds, for each spatial cell, the running sum of a quantity over the iterations
    accumulated since the last reset. */
class CellWindow
{
public:
    /** Adds the value returned by \em valueInCell to the running sum of each cell, and increments
        the number of accumulated iterations. The callback is invoked serially for all cells in
        index order, so that the result is identical on all processes. */
    void accumulate(const std::function<double(int m)>& valueInCell);

    /** Returns the number of iterations accumulated since the last reset. */
    int numIterations() const;

    /** Returns the mean value for cell \em m over the accumulated iterations, or NaN if no
        iterations have been accumulated. */
    double mean(int m) const;

    /** Clears the running sums and the number of accumulated iterations. */
    void reset();

    /** Returns the number of cells, the key, and the description of the window. */
    int numCells() const;
    const SeriesKey& key() const;
    const string& description() const;
};
```

## Iteration history

```cpp
/** IterationHistory holds the historical data used by the iteration loops of a simulation. All
    series are created on demand; if no client requests a series, the object holds no data. The
    MonteCarloSimulation creates a single instance as a child, so that any simulation item can
    locate it with find<IterationHistory>(). The class is not discoverable because it is not
    intended to be configured by the user. */
class IterationHistory : public SimulationItem
{
public:
    /** The iteration loops. */
    enum class Loop { None, Primary, Secondary, Merged };

    /** The lifetime of a series: cleared at the start of each iteration loop, or kept for the
        complete simulation. */
    enum class Lifetime { Loop, Simulation };

    /** Creates an empty history hooked up as a child to the specified parent. */
    explicit IterationHistory(SimulationItem* parent);

    //======== Iteration events, called by MonteCarloSimulation ========

    /** Starts a new iteration loop: clears all series with loop lifetime, and resets the iteration
        index to zero. */
    void beginLoop(Loop loop);

    /** Starts a new iteration: increments the iteration index and advances all scalar series by
        one slot, leaving the new current slot empty. */
    void beginIteration();

    /** Returns the current loop, and the one-based index of the current iteration within that loop
        (zero before the first iteration). */
    Loop loop() const;
    int iteration() const;

    //======== Declaring series ========

    /** Returns the scalar series with the key (item, id), creating it if needed. If the series
        exists with a smaller depth, it grows to the requested depth, preserving its values. Throws
        a fatal error if the series exists with a different lifetime or with an aggregation rule.
        The returned reference remains valid for the lifetime of the history. */
    ScalarSeries& scalarSeries(const SimulationItem* item, int id, int depth, Lifetime lifetime,
                               string description);

    /** Returns the aggregate series with the key (item, id), creating it if needed with the
        specified aggregation rule and simulation lifetime. The medium system records the value of
        the series after setup and after each medium state update. The series must therefore be
        declared during setup, before the medium system records the initial values. Throws a fatal
        error if the series exists with a different aggregation rule or without one. */
    ScalarSeries& aggregateSeries(const SimulationItem* item, int id, const AggregationRule& rule,
                                  int depth = 2, string description = "");

    /** Returns the cell window with the key (item, id), creating it for the specified number of
        cells if needed. The returned reference remains valid for the lifetime of the history. */
    CellWindow& cellWindow(const SimulationItem* item, int id, int numCells, Lifetime lifetime,
                           string description);

    //======== Retrieving series ========

    /** Returns the existing scalar series with the key (item, id), for example an aggregate series
        declared during setup. Throws a fatal error if there is no such series. */
    ScalarSeries& series(const SimulationItem* item, int id);

    /** Returns all aggregate series, so that the medium system can record their values. */
    vector<ScalarSeries*> aggregates();

    //======== Dynamic grid refinement ========

    /** Grows all cell windows by one cell for each element of \em parentCells. The new cells
        receive consecutive indices following the existing cells, and start with the running sum of
        the indicated parent cell. */
    void appendCells(const vector<int>& parentCells);

    //======== Listing ========

    /** Return all scalar series and all cell windows, for logging and probing. */
    vector<const ScalarSeries*> allScalarSeries() const;
    vector<const CellWindow*> allCellWindows() const;

private:
    Loop _loop{Loop::None};
    int _iteration{0};
    std::map<SeriesKey, ScalarSeries> _scalars;  // node-based, so references remain valid
    std::map<SeriesKey, CellWindow> _windows;
};
```

## History scope

```cpp
/** A HistoryScope gives a client access to the series kept on behalf of a given simulation item,
    without the client knowing that item. The medium system passes a scope bound to a medium
    component to the convergence function of that component's material mix. */
class HistoryScope
{
public:
    /** Creates a scope for the specified history and item. */
    HistoryScope(IterationHistory* history, const SimulationItem* item);

    /** Returns the scalar series with the key (item, id), as described for
        IterationHistory::scalarSeries(). */
    ScalarSeries& scalarSeries(int id, int depth, IterationHistory::Lifetime lifetime,
                               string description) const;

    /** Returns the existing scalar series with the key (item, id), as described for
        IterationHistory::series(). A material mix uses this function to retrieve the aggregate
        series declared on its behalf by the medium system. */
    const ScalarSeries& series(int id) const;

    /** Returns the current loop and iteration index. */
    IterationHistory::Loop loop() const;
    int iteration() const;
};
```

## Changes to existing classes

**StateVariable.** A custom variable can be marked for aggregation:

```cpp
/** Returns a copy of this state variable marked for aggregation into the series with the specified
    id. When it initializes the medium state, the medium system declares an aggregate series with
    the key (medium component, id) for the variable. The material mix chooses the id, which must be
    unique among the ids of all its series, and retrieves the series through HistoryScope::series().
    Only custom variables can be marked. The aggregate is meaningful for densities; this is not
    enforced. */
StateVariable aggregated(int seriesId) const;

/** Return true if this state variable is marked for aggregation, and the id of its series. */
bool isAggregated() const;
int aggregateSeriesId() const;
```

**MaterialMix.** The convergence function receives a history scope instead of the current and
previous aggregate material states:

```cpp
virtual bool isSpecificStateConverged(int numCells, int numUpdated, int numNotConverged,
                                      const HistoryScope& history) const;
```

**MediumState.** The aggregate cells are removed: `initConfiguration()` loses its third argument,
and `calculateAggregate()` and `pushAggregate()` are removed. A new function calculates requested
volume integrals:

```cpp
/** Returns the volume integral over all cells of each of the specified variables, identified by
    medium component index and custom variable index, calculated in a single pass over the cells. */
vector<double> volumeIntegrals(const vector<std::pair<int, int>>& variables) const;
```

**MediumSystem.** `beginDynamicMediumStateIteration()` is removed, and a private function
`recordAggregates()` records the values of all aggregate series.

**DynamicStateRecipe.** No change. A recipe that needs history locates the history itself (see the
Call sites chapter).
