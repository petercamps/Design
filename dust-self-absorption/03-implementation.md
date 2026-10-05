# Implementation

## Responsibilities

- `MediumSystem` stores the absorbed dust luminosity per cell next to the radiation field, traces
  the path segments of a photon packet to store both, provides the absorbed luminosity to the dust
  emission and the convergence test, and owns the rebalancer.
- `SelfAbsorptionRebalancer`, a new helper class, partitions the cells into regions, accumulates the
  transfer tallies, solves the rebalance system, and rescales the secondary radiation field.
- `PhotonPacket` remembers the spatial cell from which a secondary photon packet was emitted, and
  `DustSecondarySource` sets it.
- `MonteCarloSimulation` drives the rebalance from the secondary and merged iteration loops,
  evaluates the new convergence criteria, records the convergence series in the iteration history,
  and logs them.
- `DustEmissionOptions` holds the new properties, and `Configuration` offers the corresponding
  getters, plus a flag that tells whether the dust absorption cross sections are constant along a
  path.

## Module

The rebalancer lives in the `medium` module, next to the medium system that owns it and whose
per-cell data it reads and rescales. It uses `PhotonPacket` from `utils`, `ProcessManager` from
`mpi`, and `Log` from `tools`, all of which `medium` already depends on. The emission cell added to
`PhotonPacket` is a plain integer, so `utils` gains no dependency. The changes to
`MonteCarloSimulation` in `simulation`, and to `Configuration` and `ConfigurationSetup` in `tools`
and `simulation`, use existing dependencies. No new dependencies between modules arise.

The rebalancer is not a simulation item. It is not configured by the user, holds no references to
other items beyond the medium system, and needs no setup phase of its own. Its history series are
keyed on the `DustEmissionOptions` item, which configures it.

## Exact absorption

### Per-cell data

`MediumSystem` gains three arrays with one value per spatial cell, next to the radiation field
tables `_rf1`, `_rf2`, and `_rf2c`:

| Array | Contents | Allocated if |
| --- | --- | --- |
| `_labs1` | dust-absorbed luminosity from primary sources | the simulation has dust emission |
| `_labs2` | dust-absorbed luminosity from secondary sources, stable | the simulation has dust emission and a secondary radiation field |
| `_labs2c` | dust-absorbed luminosity from secondary sources, being accumulated | same as `_labs2` |

The arrays follow the life cycle of the corresponding radiation field tables. They are cleared
together with them, accumulated during the photon cycle with `LockFree::add()`, summed over
processes with `ProcessManager::sumToAll()` in `communicateRadiationField()`, and `_labs2c` is copied
to `_labs2` when `_rf2c` is copied to `_rf2`. The memory cost is 8 bytes per cell per array, which is
negligible compared to the radiation field tables.

In a simulation that stores the radiation field without calculating dust emission, for example to
probe the radiation field or to drive gas physics, the arrays are not allocated and the photon
cycle does not accumulate them.

### Storing the radiation field

Today, `MonteCarloSimulation::storeRadiationField()` loops over the path segments of a photon packet
and calls `MediumSystem::storeRadiationField(primary, m, ell, Lds)` for each segment. The proposal
moves this loop into the medium system, as a new public function that replaces the per-bin one:

```cpp
/** Stores the contribution of the specified photon packet along its current path to the radiation
    field and, if the simulation has dust emission, to the dust-absorbed luminosity of each cell
    crossed by the path. Also accumulates the transfer tallies of the rebalancer, if active. The
    path segments and their cumulative extinction optical depths must have been calculated. */
void storeRadiationField(bool primary, const PhotonPacket* pp);
```

The absorption tally needs the dust components, their cross sections, the number densities, the
material mixes of the cells, and the rebalancer, all of which are internal to the medium system.
Moving the loop avoids exposing these through new public functions. `performLifeCycle()` calls the
new function where it now calls its own `storeRadiationField()`, which is removed. The loop also
becomes the natural place for future per-path tallies, for example for photoionization, so the dust
tallies are written as one optional addition among possibly several.

For each segment, the function calculates the luminosity times path length `Lds` exactly as today,
from the extinction at the start and end of the segment. It adds `Lds` to the radiation field bin,
if the packet's wavelength lies within the radiation field wavelength grid, and adds `kabs * Lds` to
the absorbed luminosity of the cell, whether or not it lies within the grid, where `kabs` is the
dust absorption opacity of the cell at the packet's wavelength. The branch without kinematics
becomes:

```cpp
double luminosity = pp->luminosity();
int ell = _wavelengthGrid->bin(pp->wavelength());
DustAbsorptionOpacity dust(this, pp->wavelength(), pp);  // see below; inactive without dust emission
RebalanceTally tally = rebalanceTally(primary, pp);      // see below; inactive unless rebalancing
if (ell < 0 && !dust.active()) return;
if (tally && pp->numScatt() == 0) tally.launch(luminosity);
double lnExtBeg = 0.;
double extBeg = 1.;
for (const auto& segment : pp->segments())
{
    double lnExtEnd = -segment.tauExt();
    double extEnd = exp(lnExtEnd);
    int m = segment.m();
    if (m >= 0)
    {
        double Lds = luminosity * SpecialFunctions::lnmean(extEnd, extBeg, lnExtEnd, lnExtBeg) * segment.ds();
        if (ell >= 0) LockFree::add(primary ? _rf1(m, ell) : _rf2c(m, ell), Lds);
        if (dust.active())
        {
            double Labs = dust.opacity(m) * Lds;
            LockFree::add(primary ? _labs1[m] : _labs2c[m], Labs);
            if (tally) tally.absorb(m, Labs);
        }
    }
    lnExtBeg = lnExtEnd;
    extBeg = extEnd;
}
if (tally) tally.escape(luminosity * extBeg);
```

The branch with kinematics has the same structure, but calculates the perceived wavelength, the
radiation field bin, the perceived luminosity, and the opacity for each segment, as it does today
for the radiation field. The tally functions are described with the rebalancer below.

### Dust absorption opacity

The opacity is obtained through a small private helper class of the medium system, constructed once
per path, so that the work that does not depend on the cell is done once per path rather than once
per segment:

```cpp
/** Calculates the dust absorption opacity at a given wavelength in a given cell, for the photon
    packet traveling along the current path. */
class DustAbsorptionOpacity
{
public:
    DustAbsorptionOpacity(const MediumSystem* ms, double lambda, const PhotonPacket* pp);
    bool active() const;           // false if the simulation has no dust emission
    double opacity(int m) const;   // the dust absorption opacity in cell m
};
```

The helper works in one of two modes, chosen by a new flag of the configuration:

- **Constant cross sections.** If `Configuration::hasConstantDustAbsorptionSections()` returns true,
  the constructor obtains `mix(0, h)->sectionAbs(lambda)` for each dust component `h` in `_dust_hv`
  and stores them in a `ShortArray`. The opacity of a cell is then the sum over the dust components
  of the number density times the cross section, which for a single dust component is a single
  multiplication. This mirrors the fast paths of `getExtinctionOpticalDepth()`.
- **Variable cross sections.** Otherwise, the opacity of a cell is the sum over the dust components
  of `mix(m, h)->opacityAbs(lambda, &state, pp)`, as in the existing `opacityAbs()` overload that
  takes a photon packet. This handles material mix families, mixes with extra specific state
  variables, and the polarization-dependent absorption by aligned spheroidal grains.

The new flag is true if the simulation has a constant perceived wavelength, and no dust medium
component has a variable material mix, extra specific state variables, or polarized absorption. It
is set in `ConfigurationSetup` next to the existing flags for constant cross sections, which apply to
all media rather than to dust alone. With kinematics, the helper is constructed for each segment,
because the wavelength changes from segment to segment.

### Using the absorbed luminosity

- `dustLuminosity(m)` returns `_labs1[m] + _labs2[m]`, or `_labs1[m]` if there is no secondary
  radiation field. Its documentation changes accordingly. Through this function, the dust secondary
  source, its cell library mapping, and the `SecondaryDustLuminosityProbe` use the exact values.
- `totalDustAbsorbedLuminosity()` returns the sums of `_labs1` and `_labs2`. The arrays are
  identical on all processes after `communicateRadiationField()`, so the function no longer needs a
  parallel loop over all cells and wavelengths that evaluates the dust opacity for each, nor any
  communication.
- A new function `dustAbsorbedLuminosity(bool primary)` returns a reference to `_labs1` or `_labs2`,
  for the rebalancer.
- `meanIntensity()`, `dustEmissionSpectrum()`, the indicative temperatures, the cell libraries, the
  dynamic medium state updates, and the probes of the radiation field are not changed. They keep
  using the binned radiation field, for its spectral shape.

### Dynamic grid refinement

If the [Dynamic grid refinement](dynamic-grid-refinement/01-introduction.md) proposal is
implemented, the three arrays join the per-cell structures that grow when cells are subdivided. Like
the radiation field tables, they hold extensive quantities, so the parent's values are divided among
the children in proportion to their volume.

## Emission cell

`PhotonPacket` gains a data member `int _emissionCell{-1}`, with the functions `setEmissionCell(int
m)` and `emissionCell()`. The `launch()` function resets it to -1, so that it is -1 for packets from
primary sources and from secondary sources other than dust. `DustSecondarySource::launch()` sets it
to the cell from which the packet is launched, right after launching the packet. The value survives
scattering events, because a scattered packet carries the same emitted energy. Peel-off packets do
not need it. The memory cost is 4 bytes per packet, and the packet objects are reused for all
launches by a thread.

## Regional rebalance

### Activation

At setup, the medium system creates the rebalancer if the simulation iterates over secondary
emission, `rebalanceSecondaryEmission` is true, and dust emission is the only secondary emission,
and logs a message if the option is set but cannot be honored. The rebalancer is active during the
secondary emission segment of each iteration from the second iteration on, both in the secondary
loop and in the merged loop. It is controlled by two functions, which `MonteCarloSimulation` calls
through the medium system:

```cpp
/** Partitions the cells into regions based on the current absorbed luminosities, and clears the
    tallies. Must be called before the secondary photon packets of an iteration are launched. */
void beginRebalance();

/** Reduces the tallies over threads and processes, solves the rebalance system, and rescales the
    stable secondary radiation field and the stable absorbed secondary luminosity of each cell
    accordingly. Records the rebalance series in the iteration history and returns a summary for
    logging. Must be called after communicateRadiationField(false). */
RebalanceSummary endRebalance();
```

### Region partition

The partition of iteration *n* is calculated from the absorbed luminosities that determine the
emission of iteration *n*: the absorbed primary luminosity `A(m)` in `_labs1`, and the stable
absorbed secondary luminosity `S(m)` in `_labs2`, which is the rebalanced result of iteration *n−1*.
The emission of a cell is `E(m) = A(m) + S(m)`, and its dust mass `M(m)` is the sum of the mass
densities of the dust components times the cell volume.

1. Cells with `E(m) = 0` or `M(m) = 0` receive region -1. They emit nothing, and the dust they
   contain, if any, absorbed nothing in the previous iteration. Absorption in these cells during the
   current iteration, which is rare, is not tallied, so the transfer matrix treats it as a loss.
2. For the other cells, the coordinates are `u(m) = log10(S(m) / A(m))` and `v(m) = log10(E(m) /
   M(m))`. A cell without absorbed primary luminosity receives the largest finite value of `u` over
   all cells, and a cell without absorbed secondary luminosity the smallest one.
3. For each axis separately, the bin borders are the weighted quantiles of the coordinate, with the
   emission `E(m)` as weight, so that each of the `N` bins on that axis holds about `1/N` of the
   total emission. The quantiles are obtained from a fine histogram of 4096 equal bins between the
   smallest and largest coordinate, which costs one pass over the cells, rather than from sorting
   the cells. If all coordinates on an axis are equal, all cells go to the first bin of that axis,
   avoiding a division by a zero range.
4. The region index is `iu * numHeatingBins + iv`. Regions that hold no cell are dropped, and the
   remaining *K* regions are numbered consecutively. The result is stored in a `vector<int>` with
   one region index per cell.

The calculation is serial and deterministic. Since its inputs are identical on all processes, so is
the partition. Its cost is a few passes over the cells, which is small compared to an iteration.

### Tallies

The rebalancer accumulates three tallies during the secondary emission segment:

- `launched[s]`, the luminosity launched from region *s*;
- `absorbed[r][s]`, the luminosity launched from region *s* that is absorbed by dust in region *r*;
- `escaped[s]`, the luminosity launched from region *s* that leaves the spatial domain.

`MediumSystem::storeRadiationField()` obtains a small `RebalanceTally` handle, which binds the
tally object of the current thread to the region *s* of the packet's emission cell. The handle is
inactive, and the tallies cost nothing, unless the rebalancer is active, the packet is secondary,
and its emission cell lies in a region. Through the handle, the function adds the packet's
luminosity to `launched[s]` if the packet has not yet scattered, which identifies its first path,
adds the absorbed luminosity of each segment in a cell of region *r* to `absorbed[r][s]`, and adds
the luminosity that leaves the domain at the end of the path, `L exp(−τ)`, to `escaped[s]`. With
forced scattering, which is always enabled when the radiation field is stored, every path ends at
the boundary of the domain, so the sum of the escaped luminosities over all paths of all packets is
the escaping luminosity.

Updating a shared table with atomic additions would cause heavy contention, since most absorption
happens in a few regions. Each thread therefore accumulates into its own tally object, managed by
the existing `ThreadLocalMember<T>` template, which provides a separate copy of a data member for
each thread that uses it. `FluxRecorder` already uses this template in the same way, for the
thread-local contribution lists of its statistics. The rebalancer holds a
`ThreadLocalMember<RebalanceTallies>` data member, where `RebalanceTallies` holds the three tallies
and the number of regions *K* for which they are sized:

- `beginRebalance()` obtains the copies of all threads that used the member before through its
  `all()` function, and resizes and zeroes each of them for the new partition.
- The `RebalanceTally` handle obtains the copy of the current thread through `local()`, once per
  path, so that the cost of the lookup is negligible compared to the segment loop. A thread that
  uses the member for the first time after `beginRebalance()` receives a new, empty copy, which the
  handle sizes and zeroes when it does not match the current *K*.
- `endRebalance()` sums the copies returned by `all()` serially, and then sums the result over
  processes with `ProcessManager::sumToAll()`.

Both `beginRebalance()` and `endRebalance()` run between segments, when no worker thread accesses
the tallies, so that accessing the copies of other threads through `all()` needs no further
synchronization. The copies remain valid between iterations, because `ParallelFactory` reuses its
`Parallel` instances, and thus their worker threads, for the duration of the simulation. The memory
cost per thread is about 8 *K*² bytes.

### Rebalance system

With the summed tallies, `endRebalance()` calculates for each region *r*:

- the transfer fractions `T[r][s] = absorbed[r][s] / launched[s]`, with column *s* set to zero if
  `launched[s]` is zero;
- the absorbed primary luminosity `A[r]`, the sum of `_labs1` over the cells of the region;
- the absorbed secondary luminosity `C[r]`, the sum of `_labs2` over the cells of the region, which
  `communicateRadiationField(false)` has just set to the result of the current iteration.

At the fixed point, the emission `x[r]` of each region equals its absorbed primary luminosity plus
the luminosity it absorbs from the emission of all regions:

```
x = A + T x,    i.e.    (I − T) x = A
```

The rebalancer solves this dense system of *K* equations by Gaussian elimination with partial
pivoting, which costs about *K*³/3 operations, well under a second for *K* = 900. The new absorbed
secondary luminosity of region *r* is then `x[r] − A[r]`, and its factor is:

```
f[r] = (x[r] − A[r]) / C[r]
```

### Safeguards

- **Global factor.** The tallied self-absorbed fraction is `p = Σ absorbed / Σ launched`. If `p`
  is smaller than one, the total emission consistent with it is `X = ΣA / (1 − p)`, and the global
  factor is `(X − ΣA) / ΣC`. This is the factor of a rebalance with a single region. If `p` is not
  smaller than one, or `ΣC` is zero, the global factor is one.
- **Region fallback.** A region receives the global factor instead of its own if `launched[r]` or
  `C[r]` is zero, or if `x[r] < A[r]`.
- **Solve fallback.** If a pivot is smaller than 1e-12 times the largest element of its row, or if
  any `x[r]` is negative, the solve is rejected and all regions receive the global factor.
- **Clamping.** All factors, including the global one, are clamped to the range 0.5 to 5.

All divisions, including those in the bin borders of the partition, are guarded against zero
denominators, so that a degenerate model, such as one with a single emitting cell, cannot produce
NaN values.

### Rescaling

Finally, `endRebalance()` multiplies `_rf2(m, ell)` for all wavelength bins and `_labs2[m]` of each
cell with a region by the factor of its region. Cells in region -1 keep their values. Like the
partition, the rescaling is a serial pass over the cells, performed identically on all processes,
so that the stable secondary radiation field remains identical everywhere without communication.

The next iteration launches its dust emission from the rescaled field: the absorbed luminosity sets
the luminosity of each cell, and the rescaled mean intensity, which keeps its spectral shape, sets
the emission spectrum or library entry. Probes performed after the iteration see the rescaled field.

### Tally closure

For dust-emitted packets, the absorbed and escaped luminosity together must equal the launched
luminosity, apart from absorption by other media and the luminosity lost when packets are
terminated below the weight threshold. The rebalancer reports the smallest and largest ratio
`(Σr absorbed[r][s] + escaped[s]) / launched[s]` over the regions, as a diagnostic. With exact
absorption and dust as the only absorbing medium, this ratio was between 0.9998 and 1.0001 for all
regions of ID39321. A ratio well above one indicates an inconsistency between the attenuation of the
packets and the absorption tally, which is how the bias of the current calculation was found.

## Iteration loops

### Secondary loop

The secondary emission loop proposed in the Iteration history design note changes as follows:

```cpp
_history->beginLoop(IterationHistory::Loop::Secondary);
while (true)
{
    ++iter;
    _history->beginIteration();
    bool converged = true;
    {
        // ... time logger, clear the secondary radiation field (as today)

        // record the luminosity that the dust is about to emit
        double Lemitted = hasDust ? sumOf(mediumSystem()->totalDustAbsorbedLuminosity()) : 0.;

        // partition the cells and clear the tallies
        bool rebalance = mediumSystem()->hasRebalancer() && iter > 1;
        if (rebalance) mediumSystem()->beginRebalance();

        // ... prepare the secondary sources, launch the packets, wait, and communicate (as today)

        converged &= mediumSystem()->updateSecondaryDynamicMediumState();
        if (hasDust) converged &= isDustEmissionConverged(Lemitted);
        if (rebalance) logRebalance(mediumSystem()->endRebalance());
    }
    _history->scalarSeries(this, LoopConverged, ...).set(converged ? 1. : 0.);
    probeSystem()->probeSecondary(iter);
    if (logLoopConvergence(log(), converged, iter, minIters, maxIters)) break;
}
```

The convergence test uses the quantities measured in the iteration, before the rebalance. The
rebalance is performed in every iteration from the second one, including the last, so that the
final secondary emission starts from the rebalanced field.

### Merged loop

In the merged loop, the primary radiation field and `_labs1` are recalculated at the start of each
iteration. The emitted luminosity is recorded after the primary segment, and `beginRebalance()` is
called after the primary segment as well, so that the partition and the absorbed primary luminosity
per region reflect the current primary field. Otherwise, the loop changes like the secondary loop.
In particular, the dynamic medium state is always updated from the measured radiation field, before
`endRebalance()` rescales the secondary field, so that the medium state iteration does not depend
on the dust acceleration.

The merged loop requires one more change, which is needed regardless of this proposal. Today,
`clearRadiationField(true)` clears both the primary table and the stable secondary table. The merged
loop calls it at the start of each iteration, and `runPrimaryEmission()` calls it after the merged
loop. As a result, the dust emission of each merged iteration, and of the final secondary emission,
is calculated from the primary radiation field only, so that dust self-absorption is never iterated
in a simulation with merged iterations. The proposal is that
`clearRadiationField(true)` clears only `_rf1` and `_labs1`. The stable secondary table is zero
after allocation and is only replaced by `communicateRadiationField(false)` or rescaled by the
rebalancer. In a simulation without merged iterations, the primary emission is calculated before
any secondary emission, so this change has no effect there.

### Convergence test

The function `isDustEmissionConverged()` proposed in the Iteration history design note is
replaced by:

```cpp
bool MonteCarloSimulation::isDustEmissionConverged(double Lemitted)
{
    int w = _config->numConvergenceIterations();
    double Lp, Ls;
    std::tie(Lp, Ls) = mediumSystem()->totalDustAbsorbedLuminosity();

    auto lifetime = IterationHistory::Lifetime::Loop;
    auto& primary = _history->scalarSeries(this, DustAbsorbedPrimary, 1, lifetime,
                                           "dust-absorbed primary luminosity", "bolluminosity");
    auto& secondary = _history->scalarSeries(this, DustAbsorbedSecondary, w + 1, lifetime,
                                             "dust-absorbed secondary luminosity", "bolluminosity");
    auto& emitted = _history->scalarSeries(this, DustEmitted, 1, lifetime,
                                           "dust-emitted luminosity", "bolluminosity");
    auto& deficit = _history->scalarSeries(this, LuminosityDeficit, w, lifetime, "luminosity deficit");
    auto& fraction = _history->scalarSeries(this, SelfAbsorbedFraction, 1, lifetime, "self-absorbed fraction");
    auto& change = _history->scalarSeries(this, SecondaryChange, 1, lifetime, "secondary change");
    primary.set(Lp);
    secondary.set(Ls);
    emitted.set(Lemitted);

    if (Lp <= 0. || Ls <= 0.) return true;
    if (Ls / Lp < _config->maxFractionOfPrimary()) return true;

    bool dustOnly = _config->hasDustOnlySecondaryEmission();
    if (dustOnly) deficit.set((Ls - Lemitted + Lp) / Lp);
    if (Lemitted > 0.) fraction.set(Ls / Lemitted);
    if (!secondary.has(w)) return false;
    change.set(abs(Ls - secondary.value(w)) / (w * Ls));

    bool changeOk = change.value() <= _config->maxSecondaryChange();
    if (!dustOnly) return changeOk;
    double meanDeficit = 0.;
    for (int lag = 0; lag != w; ++lag) meanDeficit += deficit.value(lag);
    meanDeficit /= w;
    return changeOk && abs(meanDeficit) <= _config->maxLuminosityDeficit();
}
```

The deficit is evaluated only if dust emission is the only secondary emission, which is the
condition under which it is meaningful, whether or not the rebalance is enabled. The
deficit series has a value in each of the last *w* iterations once the secondary series has *w + 1*
values, since both are set in every iteration from the first. The logging, omitted here, is formed
from the same series, as shown in the Features chapter.

## History series

The convergence quantities are keyed on the simulation, and the rebalance quantities on the
`DustEmissionOptions` item, which configures the rebalancer. All series have loop lifetime, so they
are cleared at the start of the secondary or merged loop. The history probe writes all of them to its
file for the secondary or merged loop, one column per series and one row per iteration.

| Series | Keyed on | Quantity | Set in |
| --- | --- | --- | --- |
| dust-absorbed primary luminosity | simulation | bolometric luminosity | every iteration |
| dust-absorbed secondary luminosity | simulation | bolometric luminosity | every iteration |
| dust-emitted luminosity | simulation | bolometric luminosity | every iteration |
| luminosity deficit | simulation | — | every iteration, if dust is the only secondary emission |
| self-absorbed fraction | simulation | — | every iteration |
| secondary change | simulation | — | from iteration *w* + 1 |
| number of regions | options | — | rebalanced iterations |
| number of regions using the global factor | options | — | rebalanced iterations |
| solve accepted (0 or 1) | options | — | rebalanced iterations |
| global factor | options | — | rebalanced iterations |
| smallest region factor, largest region factor (two series) | options | — | rebalanced iterations |
| effective factor | options | — | rebalanced iterations |
| tallied self-absorbed fraction | options | — | rebalanced iterations |
| smallest tally closure, largest tally closure (two series) | options | — | rebalanced iterations |

The `DustAbsorbedLuminosity` id of the simulation's enumeration in the Iteration history design
note is replaced by the ids `DustAbsorbedPrimary`, `DustAbsorbedSecondary`, `DustEmitted`,
`LuminosityDeficit`, `SelfAbsorbedFraction`, and `SecondaryChange`.

## Changes to existing classes

**DustEmissionOptions.** The property `maxFractionOfPrevious` is removed, and the properties listed
in the Features chapter are added, with the attributes:

```cpp
PROPERTY_DOUBLE(maxLuminosityDeficit, "convergence requires the mean deficit of the escaping dust "
                                      "luminosity relative to the absorbed primary luminosity to be smaller")
ATTRIBUTE_MIN_VALUE(maxLuminosityDeficit, "]0")
ATTRIBUTE_MAX_VALUE(maxLuminosityDeficit, "1[")
ATTRIBUTE_DEFAULT_VALUE(maxLuminosityDeficit, "0.01")
ATTRIBUTE_RELEVANT_IF(maxLuminosityDeficit, "IterateSecondary")

PROPERTY_DOUBLE(maxSecondaryChange, "convergence requires the mean relative change per iteration "
                                    "of the absorbed secondary luminosity to be smaller")
ATTRIBUTE_MIN_VALUE(maxSecondaryChange, "]0")
ATTRIBUTE_MAX_VALUE(maxSecondaryChange, "1[")
ATTRIBUTE_DEFAULT_VALUE(maxSecondaryChange, "0.005")
ATTRIBUTE_RELEVANT_IF(maxSecondaryChange, "IterateSecondary")

PROPERTY_INT(numConvergenceIterations, "the number of iterations over which convergence is evaluated")
ATTRIBUTE_MIN_VALUE(numConvergenceIterations, "1")
ATTRIBUTE_MAX_VALUE(numConvergenceIterations, "10")
ATTRIBUTE_DEFAULT_VALUE(numConvergenceIterations, "3")
ATTRIBUTE_RELEVANT_IF(numConvergenceIterations, "IterateSecondary")

PROPERTY_BOOL(rebalanceSecondaryEmission, "rebalance the secondary radiation field over regions after each iteration")
ATTRIBUTE_DEFAULT_VALUE(rebalanceSecondaryEmission, "true")
ATTRIBUTE_RELEVANT_IF(rebalanceSecondaryEmission, "IterateSecondary")

PROPERTY_INT(numCoreBins, "the number of rebalance bins in the ratio of secondary to primary absorption")
ATTRIBUTE_MIN_VALUE(numCoreBins, "1")
ATTRIBUTE_MAX_VALUE(numCoreBins, "100")
ATTRIBUTE_DEFAULT_VALUE(numCoreBins, "10")
ATTRIBUTE_RELEVANT_IF(numCoreBins, "IterateSecondary&rebalanceSecondaryEmission")
ATTRIBUTE_DISPLAYED_IF(numCoreBins, "Level3")

// numHeatingBins: same attributes as numCoreBins
```

The maximum of 100 bins per axis limits the transfer matrix to 10000 regions, or 800 MB per
thread, which is far beyond any useful value.

**Configuration and ConfigurationSetup.** Getters for the new properties replace
`maxFractionOfPrevious()`. The new flag `hasConstantDustAbsorptionSections()` is added, as
described above, and a flag `hasDustOnlySecondaryEmission()`, true if the simulation has dust
emission and no gas emission, decides whether the rebalancer is created and the deficit applies.

**MediumSystem.** The per-bin `storeRadiationField()` is replaced by the per-path version; the
absorbed luminosity arrays are added; `clearRadiationField()`, `communicateRadiationField()`,
`totalDustAbsorbedLuminosity()`, and `dustLuminosity()` change as described; `clearRadiationField(true)`
no longer clears the stable secondary table; and the functions `dustAbsorbedLuminosity()`,
`hasRebalancer()`, `beginRebalance()`, and `endRebalance()` are added.

**MonteCarloSimulation.** `storeRadiationField()` is removed. The iteration loops and the
convergence test change as described.

**PhotonPacket and DustSecondarySource.** The emission cell is added and set, as described.

**SelfAbsorptionRebalancer.** New class in the `medium` module, with its tally objects and summary
structure.

**PTS.** The ski file upgrade for SKIRT 10 removes `maxFractionOfPrevious` from
`DustEmissionOptions`.

## Performance

The measurements below were made with the experimental implementation, for ID39321 with about
200 000 cells, 31.6 million photon packets per iteration, the cell library, and 20 threads on a
laptop.

| Configuration | Secondary iteration | Primary emission | Peak memory |
| --- | --- | --- | --- |
| current absorption, 50 bins, rebalance | 62 s | 80 s | 1.08 GB |
| current absorption, 200 bins, rebalance | 75 s | 85 s | 1.97 GB |
| exact absorption, 50 bins, rebalance | 70 s | 93 s | 1.02 GB |
| exact absorption, 200 bins, rebalance | 79 s | 106 s | 1.68 GB |

The experiment calculated the dust cross sections once per path, as proposed. An earlier variant
that obtained the opacity from the medium system for every segment made iterations about three times
slower, so this optimization is essential. The tallies of the rebalance cost about
1–2% of an iteration with 100 regions, once accumulated per thread; with a shared table and atomic
additions, an iteration took almost twice as long.

## Testing

The functional tests of all simulations with dust emission change, and must be inspected and
endorsed again. To separate the effects, the proposal is to implement and endorse the changes in
three steps, each verified on the complete functional test suite:

1. Exact absorption alone. Every change in the results must be explained by the absorbed
   luminosity: the relative change of the absorbed primary luminosity scales roughly with the square
   of the width of the radiation field bins, and the tally closure is one within noise.
2. The merged loop correction. Only tests with merged iterations change, and for tests with
   substantial dust self-absorption, the dust emission increases.
3. The rebalance and the new criteria. Tests that iterate over secondary emission change in the
   number of iterations and, for tests that did not converge, in their results.

New test cases in `Functional9/NEWTESTS` cover the cases that the experiments did not:

- a compact, optically thick dusty sphere with a central source, which exercises the rebalance in
  seconds, once with separate secondary iterations and once with merged iterations;
- the same sphere with a radiation field wavelength grid that does not cover the full source
  spectrum, to verify that absorption outside the grid is counted;
- the same sphere with a moving medium, to exercise the branch with kinematics;
- a model with a material mix family and a model with aligned spheroidal grains, to exercise the
  variable cross sections;
- a model with dust and gas emission, to verify that the rebalance is disabled and the deficit is
  not evaluated.

For the sphere, a converged reference follows from a long run with plain iteration, since its
self-absorption can be kept moderate. The rebalanced run must reproduce it within the noise, and
its tally closure must be one within noise.
