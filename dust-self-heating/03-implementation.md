# Implementation

## Responsibilities

- `MediumSystem` stores the absorbed dust luminosity per cell next to the radiation field, traces
  the path segments of a photon packet to store both, provides the absorbed luminosity to the dust
  emission and the convergence test, and owns the rebalancer.
- `SelfHeatingRebalancer`, a new helper class, partitions the cells into regions, accumulates the
  transfer tallies, solves the rebalance system, determines the factor of each region, by which the
  medium system rescales the secondary radiation field, and logs a summary of each rebalance.
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
per-cell data it uses. It uses `ThreadLocalMember` from `utils`, `ProcessManager` from `mpi`, and
`IterationHistory` and `Log` from `tools`, all of which `medium` already depends on. The emission
cell added to `PhotonPacket` is a plain integer, so `utils` gains no dependency. The changes to
`MonteCarloSimulation` in `simulation`, and to `Configuration` and `ConfigurationSetup` in `tools`
and `simulation`, use existing dependencies. No new dependencies between modules arise.

The rebalancer is not a simulation item. It is not configured by the user and needs no setup phase
of its own. It does not even hold a reference to the medium system: the medium system passes it the
absorbed luminosities and the dust mass of each cell, and applies the factors it determines. The
rebalancer holds only the iteration history, the logger, the numbers of bins on its two axes, which
the medium system takes from the configuration, and the `DustEmissionOptions` item, which serves
only as the key of its history series.

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
dust absorption opacity of the cell at the packet's wavelength. If the wavelength lies outside the
grid, it also adds the absorbed luminosity to a sum for the path (see Absorption outside the grid,
below). The branch without kinematics becomes:

```cpp
SelfHeatingRebalancer::Tally tally;  // see below; inactive unless rebalancing
if (_rebalancer && !primary) tally = _rebalancer->tally(pp->emissionCell());
if (tally && pp->numScatt() == 0) tally.launch(pp->luminosity());
double luminosity = pp->luminosity();
int ell = _wavelengthGrid->bin(pp->wavelength());
DustAbsorptionOpacity dust(this, pp->wavelength(), pp);  // see below; inactive without dust emission
if (ell < 0 && !dust.active()) return;
double lnExtBeg = 0.;
double extBeg = 1.;
double LabsOutside = 0.;
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
            if (ell < 0) LabsOutside += Labs;
            if (tally) tally.absorb(m, Labs);
        }
    }
    lnExtBeg = lnExtEnd;
    extBeg = extEnd;
}
if (LabsOutside > 0.) LockFree::add(primary ? _labsOutside1 : _labsOutside2c, LabsOutside);
if (tally) tally.escape(luminosity * extBeg);
```

The tally handle is obtained, and the launched luminosity recorded, before the choice between the
two branches, so that both share them. An active handle implies active dust absorption, so the early
return never skips a tallied path. The branch with kinematics has the same structure, but calculates
the perceived wavelength, the radiation field bin, the perceived luminosity, and the opacity for
each segment, as it does today for the radiation field. The tally functions are described with the
rebalancer below.

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
- The medium system passes `_labs1` and `_labs2` to the rebalancer directly (see Activation below),
  so no public getter for the arrays is needed.
- `meanIntensity()`, `dustEmissionSpectrum()`, the indicative temperatures, the cell libraries, the
  dynamic medium state updates, and the probes of the radiation field are not changed. They keep
  using the binned radiation field, for its spectral shape.

### Absorption outside the grid

`MediumSystem` keeps two scalars next to the absorbed luminosity arrays. `_labsOutside1` and
`_labsOutside2c` hold the part of the luminosity in `_labs1` and `_labs2c` that was absorbed at
wavelengths outside the radiation field wavelength grid, summed over all cells. They are cleared
together with the corresponding array, and accumulated once per path, as shown above, so that the
threads do not contend for them in the loop over the segments. With kinematics, the radiation
field bin is determined for each segment, so that only part of the segments of a path may
contribute. The cost is one comparison per segment.

In `communicateRadiationField()`, after summing the scalar over the processes, the medium system
divides it by the sum of the corresponding array and keeps the resulting fraction. This happens
before the rebalancer rescales the stable secondary array. A new function returns the fraction:

```cpp
/** Returns the fraction of the dust-absorbed luminosity of the most recent primary or secondary
    emission segment that was absorbed at wavelengths outside the radiation field wavelength grid,
    or zero if no luminosity was absorbed. */
double dustAbsorbedFractionOutsideGrid(bool primary) const;
```

`MonteCarloSimulation` reports the fraction after each emission segment that stores the radiation
field in a simulation with dust emission, through a private helper function. The final secondary
emission after the iterations stores the field, and thus reports the fraction, only if the
`storeEmissionRadiationField` option of the secondary emission options is enabled. The function
logs the fraction, records it in the iteration history if it is called from an iteration loop, and
issues the message as a warning if the fraction exceeds 1%. In an iteration loop, the warning is
issued only if the previous value of the history series is missing or did not exceed the threshold,
so that it is not repeated in every iteration.

The scalars and the fraction serve only until the fraction has been reported at the end of the
segment, so they are not stored in a checkpoint.

### Optical depth underflow

While the implementation was tested, a long-standing loss of energy came to light in the storage
loop, which affects the radiation field as well as the absorbed luminosity. For each segment, the
loop calculates the mean extinction over the segment as the logarithmic mean of the extinction
factors at its beginning and end, `lnmean(e^−τend, e^−τbeg, −τend, −τbeg)`. In double precision,
`e^−τ` underflows to zero for an optical depth above about 745, and `SpecialFunctions::lnmean()`
returned zero as soon as one of its values was zero, even though the logarithms were available. A
segment that takes the cumulative optical depth past 745 therefore stored nothing, although it
absorbs nearly all of the luminosity `L e^−τbeg` that the packet still carries when it enters the
segment. Later segments are not affected in a meaningful way, because the remaining luminosity is
negligible. The function now returns `x2 / (ln x2 − ln x1)` when only the smaller value has
underflowed. The transport of the photon packets itself was never affected, so the escaping
radiation was always correct; only the radiation field and the absorbed luminosity in the cells
concerned were too low, and with them the heating and the emission of the dust in those cells.

The loss is significant only if a single segment takes the optical depth from a small value to above
745 by itself: a cell crossing with an extinction optical depth of several hundred or more at the
packet's wavelength, near the start of the path. This mostly concerns primary photon packets emitted
inside, or entering after little extinction, cells that are extremely opaque at the source
wavelengths. Dust emission is not affected in practice, because the dust opacity at infrared
wavelengths is some hundred times smaller. The largest optical depth of a cell diagonal at 0.55 µm,
reported by the `ConvergenceInfoProbe`, offers a quick test, keeping in mind that the extinction in
the far ultraviolet is up to about five times the extinction at 0.55 µm, and that a segment starting
inside a cell crosses on average less than half of its diagonal. The following thresholds are
estimates on that basis:

- below about 150, no segment can reach the threshold, and the issue cannot arise;
- between about 150 and 500, segments can reach it in the ultraviolet, but only along the longest
  chords of the most opaque cells, so that the effect is expected to stay well below a percent;
- above about 500, the effect becomes significant when primary sources are embedded in these cells,
  and the lost fraction of the source luminosity is roughly the fraction emitted inside them.

The COLIBRE galaxies at redshift 4 fall in the last category, with cell diagonals up to an optical
depth of 4600 at 0.55 µm. For ID39321, 6.6% of the source luminosity was lost: the dust absorbed
88.6% of it instead of 95.2%, so that the absorbed primary luminosity, and thus the escaping dust
luminosity at convergence, came out 7% too low. For ID5642, a run with the fix absorbs 2.4% more
primary luminosity than the SKIRT 9 run of Andrea Gebek, which suggests an effect of about 3%, since
exact absorption by itself lowers the absorbed luminosity slightly. None of the functional tests
that existed before is affected; the test `SelfHeating/SelfHeatingOpaqueCloud`, a dusty cloud with
an optical depth of 20 000 at 0.55 µm along its diameter, illuminated in the far ultraviolet only,
absorbs no energy at all without the fix.

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
scattering events, because a scattered packet carries the same emitted energy. The exception is a
scattering event that emulates secondary emission, such as X-ray fluorescence in the X-ray gas
mixes: `setEmulatedSecondaryOrigin()` resets the emission cell to -1, because the outgoing packet is
considered to be emitted by the gas, so that the tallies no longer attribute it to dust emission.
Peel-off packets do not need the emission cell. The memory cost is 4 bytes per packet, and the
packet objects are reused for all launches by a thread.

## Regional rebalance

### Activation

At setup, the medium system creates the rebalancer if the simulation iterates over secondary
emission, `rebalanceSecondaryEmission` is true, and dust emission is the only secondary emission.
The setup summary logged by the configuration mentions the rebalance, or that it is omitted because
the simulation also has gas emission. The rebalancer is controlled by two functions, which
`MonteCarloSimulation` calls through the medium system in every iteration, both in the secondary
loop and in the merged loop:

```cpp
/** Partitions the cells into regions based on the current absorbed luminosities and dust masses,
    and clears and activates the tallies. Must be called before the secondary photon packets of
    each iteration are launched. If the stable absorbed secondary luminosity is zero in all cells,
    as in the first iteration of a loop, the regions cannot be formed, and nothing is rebalanced in
    this iteration. In any case, declares the rebalance series in the iteration history. */
void beginRebalance();

/** Unless nothing is rebalanced in this iteration, reduces the tallies over threads and processes,
    solves the rebalance system, and rescales the stable secondary radiation field and the stable
    absorbed secondary luminosity of each cell by the factor of its region. The rebalancer records
    the rebalance series in the iteration history and logs a summary. Must be called after
    communicateRadiationField(false). */
void endRebalance();
```

The rebalance thus starts with the second iteration of each loop, without the loop having to count
iterations. The rebalancer declares its history series in every iteration, including the first,
because the history probe fixes the columns of its file in the first iteration of a loop. The medium
system passes `_labs1`, `_labs2`, and the dust mass of each cell to the rebalancer at the start of
the iteration, and `_labs1` and `_labs2` at the end; it then multiplies the stable secondary
radiation field and absorbed luminosity of each cell by the factor that the rebalancer returns for
that cell (see Rescaling below).

### Region partition

The partition of iteration *n* is calculated from the absorbed luminosities that determine the
emission of iteration *n*: the absorbed primary luminosity `A(m)` in `_labs1`, and the stable
absorbed secondary luminosity `S(m)` in `_labs2`, which is the rebalanced result of iteration *n−1*.
The emission of a cell is `E(m) = A(m) + S(m)`, and its dust mass `M(m)` is the sum of the mass
densities of the dust components times the cell volume.

1. Cells with `E(m) = 0` or `M(m) = 0` receive region -1. They emit nothing, and the dust they
   contain, if any, absorbed nothing in the previous iteration. Absorption in these cells during the
   current iteration, which is rare, is not tallied, so the transfer matrix treats it as a loss. As
   a safeguard, a cell whose `E(m)` or `M(m)` is not finite receives region -1 as well, so that the
   coordinates of all other cells are finite.
2. For the other cells, the coordinates are `u(m) = log10(S(m) / A(m))` and `v(m) = log10(E(m) /
   M(m))`. A cell without absorbed primary luminosity receives the largest finite value of `u` over
   all cells, and a cell without absorbed secondary luminosity the smallest one.
3. For each axis separately, the bin borders are the weighted quantiles of the coordinate, with the
   emission `E(m)` as weight, so that each of the `N` bins on that axis holds about `1/N` of the
   total emission. The quantiles are exact: the cells are sorted on the coordinate, with ties
   broken by cell index so that the result does not depend on the sort implementation, and each
   group of cells with equal coordinates goes to the bin holding the cumulative emission at the
   center of the group. Cells with equal coordinates, such as all cells without absorbed secondary
   luminosity, thus always share a bin, and if all coordinates on an axis are equal, all cells go
   to a single bin of that axis. A fine histogram of the coordinates, as in the first
   implementation, avoids the sort, but a few cells with tiny absorbed luminosities can stretch the
   coordinate range over hundreds of decades and leave bins empty: for ID39321 on a spatial grid of
   8.6 million cells, 20 × 20 bins gave 285 regions with a histogram of 4096 bins, and 400 with
   exact quantiles.
4. The region index is `iu * numHeatingRateBins + iv`. Regions that hold no cell are dropped, and the
   remaining *K* regions are numbered consecutively. The result is stored in a `vector<int>` with
   one region index per cell.

The calculation is serial and deterministic. Since its inputs are identical on all processes, so is
the partition. Its cost is dominated by the two sorts: for the 8.6 million cells of ID39321, the
partition takes about 1.6 s per iteration, against about 0.5 s with the histogram; the difference
is about 0.3% of an iteration with 31.6 million packets.

### Tallies

The rebalancer accumulates three tallies during the secondary emission segment:

- `launched[s]`, the luminosity launched from region *s*;
- `absorbed[r][s]`, the luminosity launched from region *s* that is absorbed by dust in region *r*;
- `escaped[s]`, the luminosity launched from region *s* that leaves the spatial domain.

`MediumSystem::storeRadiationField()` obtains a small `Tally` handle, a nested class of
`SelfHeatingRebalancer`, from the rebalancer's `tally()` function, which binds the tally object of
the current thread to the region *s* of the packet's emission cell. The handle is
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
`ThreadLocalMember<Tallies>` data member, where the nested structure `Tallies` holds the three
tallies and the number of regions *K* for which they are sized:

- `beginRebalance()` obtains the copies of all threads that used the member before through its
  `all()` function, and resizes and zeroes each of them for the new partition.
- The `tally()` function obtains the copy of the current thread through `local()`, once per
  path, so that the cost of the lookup is negligible compared to the segment loop. A thread that
  uses the member for the first time after `beginRebalance()` receives a new, empty copy, which the
  function sizes and zeroes when it does not match the current *K*.
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
        // ... time logger, clear the secondary radiation field, prepare the secondary sources (as today)

        // record the luminosity that the dust is about to emit
        double Lemitted = hasDust ? totalLuminosity(mediumSystem()->totalDustAbsorbedLuminosity()) : 0.;

        // partition the cells and clear the tallies (nothing is rebalanced in the first iteration)
        bool rebalance = mediumSystem()->hasRebalancer();
        if (rebalance) mediumSystem()->beginRebalance();

        // ... launch the packets, wait, and communicate (as today)

        converged &= mediumSystem()->updateSecondaryDynamicMediumState();
        if (hasDust) converged &= isDustEmissionConverged(Lemitted);
        if (rebalance) mediumSystem()->endRebalance();
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
is calculated from the primary radiation field only, so that dust self-heating is never iterated
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
    bool dustOnly = _config->hasDustOnlySecondaryEmission();
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
    if (dustOnly && Lp > 0.) deficit.set((Ls - Lemitted + Lp) / Lp);
    if (Lemitted > 0.) fraction.set(Ls / Lemitted);

    // the secondary change excludes the first iteration of the loop, whose dust emission lacks self-heating
    int n = _history->iteration();
    int k = min(w, n - 2);
    if (Ls > 0. && n > w && k > 0) change.set(abs(Ls - secondary.value(k)) / (k * Ls));

    if (Lp <= 0. || Ls <= 0.) return true;
    if (Ls / Lp < _config->maxFractionOfPrimary()) return true;
    if (!change.has()) return false;

    bool changeOk = change.value() <= _config->maxSecondaryChange();
    if (!dustOnly) return changeOk;
    double meanDeficit = 0.;
    for (int lag = 0; lag != w; ++lag) meanDeficit += deficit.value(lag);
    meanDeficit /= w;
    return changeOk && abs(meanDeficit) <= _config->maxLuminosityDeficit();
}
```

The deficit is evaluated only if dust emission is the only secondary emission, which is the
condition under which it is meaningful, whether or not the rebalance is enabled. All quantities are
recorded before the tests, so that the history has a value for each of them in every iteration in
which it is defined. The secondary change is available from iteration max(*w* + 1, 3); by then, the
deficit series has a value in each of the last *w* iterations, since it is set in every iteration
from the first. The logging, omitted here, is formed from the same series, as shown in the Features
chapter.

## History series

The convergence quantities are keyed on the simulation, and the rebalance quantities on the
`DustEmissionOptions` item, which configures the rebalancer. All series have loop lifetime, so they
are cleared at the start of the secondary or merged loop. The history probe writes all of them to
its file for the secondary or merged loop, one column per series and one row per iteration. Because
the probe takes its columns from the series that exist in the first iteration of the loop, the
rebalancer declares its series in every iteration, also in the first, which is not rebalanced; their
values are set in the rebalanced iterations only.

| Series | Keyed on | Quantity | Set in |
| --- | --- | --- | --- |
| dust-absorbed primary luminosity | simulation | bolometric luminosity | every iteration |
| dust-absorbed secondary luminosity | simulation | bolometric luminosity | every iteration |
| dust-emitted luminosity | simulation | bolometric luminosity | every iteration |
| luminosity deficit | simulation | — | every iteration, if dust is the only secondary emission |
| self-absorbed fraction | simulation | — | every iteration |
| secondary change | simulation | — | from iteration max(*w* + 1, 3) |
| fraction of absorbed primary luminosity outside the radiation field grid | simulation | — | iterations that calculate the primary radiation field |
| fraction of absorbed secondary luminosity outside the radiation field grid | simulation | — | every iteration |
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
`LuminosityDeficit`, `SelfAbsorbedFraction`, `SecondaryChange`, `PrimaryOutsideGrid`, and
`SecondaryOutsideGrid`.

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

PROPERTY_INT(numAbsorptionRatioBins, "the number of rebalance bins in the ratio of secondary to primary absorption")
ATTRIBUTE_MIN_VALUE(numAbsorptionRatioBins, "1")
ATTRIBUTE_MAX_VALUE(numAbsorptionRatioBins, "100")
ATTRIBUTE_DEFAULT_VALUE(numAbsorptionRatioBins, "10")
ATTRIBUTE_RELEVANT_IF(numAbsorptionRatioBins, "IterateSecondary&rebalanceSecondaryEmission")
ATTRIBUTE_DISPLAYED_IF(numAbsorptionRatioBins, "Level3")

// numHeatingRateBins: same attributes as numAbsorptionRatioBins
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
no longer clears the stable secondary table; and the functions `dustAbsorbedFractionOutsideGrid()`,
`hasRebalancer()`, `beginRebalance()`, and `endRebalance()` are added.

**MonteCarloSimulation.** `storeRadiationField()` is removed. The iteration loops and the
convergence test change as described, and each emission segment reports the fraction of the dust
absorption outside the radiation field wavelength grid.

**PhotonPacket and DustSecondarySource.** The emission cell is added and set, as described.

**SelfHeatingRebalancer.** New class in the `medium` module, with its tally objects and the handle
through which the medium system adds to them.

**PTS.** The ski file upgrade for SKIRT 10 removes `maxFractionOfPrevious` from
`DustEmissionOptions`.

## Testing

The functional tests of all simulations with dust emission change, and must be inspected and
endorsed again. To separate the effects, the proposal is to implement and endorse the changes in
three steps, each verified on the complete functional test suite:

1. Exact absorption alone. Every change in the results must be explained by the absorbed
   luminosity: the relative change of the absorbed primary luminosity scales roughly with the square
   of the width of the radiation field bins, and the tally closure is one within noise.
2. The merged loop correction. Only tests with merged iterations change, and for tests with
   substantial dust self-heating, the dust emission increases.
3. The rebalance and the new criteria. Tests that iterate over secondary emission change in the
   number of iterations and, for tests that did not converge, in their results.

New functional tests cover the cases that the experiments did not:

- a compact, optically thick dusty sphere with a central source, which exercises the rebalance in
  seconds, once with separate secondary iterations and once with merged iterations;
- the same sphere with a radiation field wavelength grid that does not cover the full source
  spectrum, to verify that absorption outside the grid is counted, and that the reported fraction
  and the warning are as expected;
- the same sphere with a moving medium, to exercise the branch with kinematics;
- a model with a material mix family and a model with aligned spheroidal grains, to exercise the
  variable cross sections;
- a model with dust and gas emission, to verify that the rebalance is disabled and the deficit is
  not evaluated.

For the sphere, a converged reference follows from a long run with plain iteration, since its
self-heating can be kept moderate. The rebalanced run must reproduce it within the noise, and
its tally closure must be one within noise.
