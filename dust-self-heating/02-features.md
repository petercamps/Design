# Features

## Exact absorption

### Behavior

In every simulation with dust emission, the luminosity absorbed by the dust in each spatial cell is
accumulated while photon packets are traced, as the sum over all path segments crossing the cell of
the dust absorption opacity at the packet's wavelength, times the packet's luminosity at that point,
times the segment length. This is exactly the energy that the packets lose to dust absorption,
calculated with the same attenuation as the radiation field itself. Separate sums are kept for
primary and secondary emission, like the radiation field.

The dust emission of each cell is normalized to this absorbed luminosity. The shape of the emission
spectrum is still calculated from the mean intensity of the binned radiation field, so a library
entry, the equilibrium temperature of a grain, or the temperature distribution of a stochastically
heated grain is obtained exactly as before.

There is no configuration option. The current calculation is a discretization error rather than a
modeling choice, so there is no reason to keep it.

### Consequences

- **Energy conservation.** Up to Monte Carlo noise, the total luminosity absorbed by dust equals
  the luminosity lost by the photon packets to dust, for any radiation field wavelength grid.
- **Radiation field wavelength grid.** The grid only needs to resolve the shape of the radiation
  field. For the COLIBRE galaxies, a logarithmic grid with 50 bins from 0.09 to 2000 µm gives the
  same result as a grid with 200 bins over the same range. The documentation advice for strongly
  self-absorbing models changes accordingly.
- **Absorption outside the grid.** Photon packets with a wavelength outside the radiation field
  wavelength grid are not recorded in the radiation field, but their absorption by dust is counted.
  The emitted energy is therefore correct even if the grid misses part of the heating spectrum. The
  shape of the emission spectrum is not, because it is calculated from the binned radiation field,
  which lacks that part of the heating. After each emission segment, the simulation therefore logs
  the fraction of the dust-absorbed luminosity that was absorbed at wavelengths outside the grid,
  and issues a warning if that fraction exceeds 1%. A typical cause is a grid that covers the
  primary emission only, in a simulation that iterates over dust emission: nearly all of the
  absorbed secondary luminosity then lies outside the grid.
- **Results.** The dust-absorbed luminosity and the dust emission change in every simulation with
  dust emission. For a 50-bin grid like the one above, the absorbed primary luminosity of a galaxy
  decreases by about 0.4%. For coarser grids, the change is larger. In strongly self-absorbing
  models, the converged dust emission decreases by up to 10–15%.
- **Other media.** Exact absorption does not affect the extinction of photon packets by any medium,
  the radiation field seen by gas, electrons, or other media, gas emission, or the dynamic medium
  state. The heating of dust by the cosmic microwave background, if enabled, still enters only the
  shape of the emission spectrum, as today.

## Regional rebalance

### Behavior

The rebalance groups the cells into regions. Unlike the coarse spatial mesh of the classic
coarse-mesh rebalance in neutron transport, a region is not a contiguous part of the spatial
domain: it collects cells in similar physical conditions, which may be scattered throughout the
domain (see below for how the regions are formed). This poses no problem, because the transfer
fractions between regions are measured from the photon packets, rather than derived from the
geometry.

During each secondary emission iteration, the simulation records, for a partition of the cells
into regions, how much of the dust emission launched from each region is absorbed by the dust in
each other region, and how much escapes. At the end of the iteration, it solves a small linear
system for the dust emission of each region that is consistent with these transfer fractions and
with the absorbed primary luminosity of each region. It then rescales the secondary radiation field
and the absorbed secondary luminosity of all cells in each region by a common factor, so that the
next iteration starts from that luminosity. The spectral shape of the field in each cell is kept.
At the fixed point, all factors equal one, so the rebalance does not change the converged solution.

The regions are formed from two per-cell quantities: the ratio of the absorbed secondary to the
absorbed primary luminosity, which measures how deeply a cell sits in the self-absorbing core, and
the total absorbed luminosity per unit dust mass, which measures the heating. Both axes are divided
into bins on a logarithmic scale, with bin borders chosen so that each bin holds the same fraction
of the emitted dust luminosity, and the regions are the combinations of a bin on each axis. The
partition is recalculated at the start of each iteration, so that cells migrate between regions as
the solution develops, like they migrate between the entries of a cell library.

The rebalance starts with the second iteration, because the first iteration has no secondary field
from which the regions can be formed. It is performed at the end of every following iteration,
including the last one, so that the final dust emission is calculated from the rebalanced field.
Stopping the rebalance before the last iteration is not offered: the experiments show that the
slow mode of plain iteration then returns immediately.

Safeguards protect against noisy or degenerate tallies. The region factors are clamped to the range
0.5 to 5. A region without launched luminosity or without absorbed secondary luminosity, or with an
unphysical solution, receives the global factor, which is the factor of a rebalance with a single
region. If the linear system is singular or its solution is unphysical as a whole, the global
factor is applied to all cells.

### Applicability

The rebalance is available only if the simulation iterates over secondary emission, with or without
including primary emission in these iterations, and if thermal dust emission is the only form of
secondary emission. With gas emission, the dust absorbs radiation from sources that the rebalance
does not describe, so the rebalance and the luminosity deficit criterion are disabled, with a
message in the setup log.

## Convergence criteria

### Quantities

The convergence of the secondary emission loop is judged from the following quantities, calculated
in each iteration *n*:

- the absorbed primary luminosity *Lp*, the luminosity of the primary radiation absorbed by dust,
  which is recalculated in each iteration if primary emission is included in the loop;
- the absorbed secondary luminosity *Ls(n)*, the luminosity of the dust emission launched in
  iteration *n* that is absorbed by dust;
- the emitted dust luminosity *Le(n) = Lp + Ls'(n−1)*, where *Ls'(n−1)* is the absorbed secondary
  luminosity of the previous iteration after the rebalance;
- the **luminosity deficit** *δ(n) = (Ls(n) − Le(n) + Lp) / Lp*, the difference between the
  absorbed and the emitted secondary luminosity, relative to the absorbed primary luminosity. Since
  the escaping dust luminosity equals *Le(n) − Ls(n) = (1 − δ(n)) Lp*, this is the relative amount
  by which the escaping dust luminosity falls short of the absorbed primary luminosity, which it must
  equal once the solution has converged. The deficit can be negative because of noise or overshoot;
- the **self-absorbed fraction** *p(n) = Ls(n) / Le(n)*, the fraction of the dust emission that is
  reabsorbed by dust;
- the **secondary change** *c(n) = |Ls(n) − Ls(n−k)| / (k Ls(n))* with *k = min(w, n − 2)*, the mean
  relative change of the absorbed secondary luminosity per iteration over the last *w* iterations,
  excluding the first iteration of the loop. This is the quantity monitored by the current
  `maxFractionOfPrevious` criterion, averaged over *w* iterations instead of taken from a single
  iteration. The first iteration is excluded because its dust emission lacks any self-heating, so
  that the change from the first to the second iteration is large even in a model with little
  self-heating. Including it would delay convergence by an iteration or more: a functional test that
  reabsorbs 1.5% of its dust emission needed five instead of four iterations.

### Criteria

The secondary emission loop has converged in iteration *n* if one of the following conditions
holds:

- the absorbed secondary luminosity is smaller than the fraction `maxFractionOfPrimary` of the
  absorbed primary luminosity, which is typically the case in the first iteration of a model with
  little dust emission, as today; or
- at least *w + 1* iterations, and at least three, have been performed, the mean of the luminosity
  deficit over the last *w* iterations is at most `maxLuminosityDeficit` in absolute value, and the
  secondary change is at most `maxSecondaryChange`. The secondary change needs two iterations after
  the first one, which matters only for *w* = 1. While the last *w* iterations would include the
  first iteration, the secondary change is averaged over fewer iterations; the luminosity deficit
  never includes the first iteration once *w + 1* iterations have been performed.

Here *w* is the value of `numConvergenceIterations`. Averaging over several iterations suppresses
the Monte Carlo noise in both quantities, which is about 0.5% per iteration for the deficit of the
COLIBRE galaxies with 31.6 million photon packets. With the default *w* = 3, a model that does not
meet the first condition therefore needs at least four iterations, where the removed
`maxFractionOfPrevious` criterion often stopped after two.

Without the rebalance, the two conditions of the second criterion would be redundant. The emitted
dust luminosity of an iteration is then the absorbed primary luminosity plus the absorbed secondary
luminosity of the previous iteration, so that the deficit reduces to
*δ(n) = (Ls(n) − Ls(n−1)) / Lp*, and its mean over the last *w* iterations to
*(Ls(n) − Ls(n−w)) / (w Lp)*. From iteration *w* + 2 on, when the secondary change no longer needs
to skip the first iteration, this is the secondary change with the absorbed primary luminosity in
the denominator instead of the absorbed secondary luminosity. In a strongly self-absorbing model,
the absorbed secondary luminosity is many times the absorbed primary luminosity, so that the
deficit is by far the stricter of the two, which is why the current criterion is misleading (see
the Introduction).

The rebalance removes this redundancy. It rescales the secondary radiation field after each
iteration so that the energy budget closes for the transfer fractions measured in that iteration.
The deficit then no longer measures the change between iterations: it compares the absorbed
secondary luminosity of an iteration with the rescaled value of the previous one. It becomes small
as soon as the rebalance predicts the energy budget of the next iteration well, even while the
distribution of the emission over the cells is still changing. Only the secondary change still
tells whether the solution has stopped changing.

The two conditions thus address different errors. The deficit measures whether the escaping
luminosity, and hence the dust emission seen by an observer, is right. With the rebalance, the
deficit drops below 1% within about 10–15 iterations, provided that each
iteration launches enough photon packets per spatial cell; with too few, the deficit levels off
above the criterion (see Recommended settings). The secondary change measures whether the
distribution of the emission over the cells has settled. Even with the rebalance, the secondary
change drops toward zero more slowly than the deficit, so that this condition is usually the last
one to be met: the radiation trapped in the self-absorbing core keeps building up for some time
after the deficit has become small. This remaining build-up in the core affects the observed
spectrum much less than it affects the absorbed secondary luminosity: two runs with 100 and 900
regions differed by 2.7% in absorbed secondary luminosity after 40 iterations, while their dust
emission spectra agreed within 0.2%.

As today, the loop runs for at least `minSecondaryIterations` and at most `maxSecondaryIterations`
iterations, and a warning is issued if it ends without convergence. Without dust, the dust criteria
are skipped, and convergence is determined by the dynamic medium state alone, as today.

## Configuration

### Dust emission options

The new properties are added to `DustEmissionOptions`, next to the existing convergence criterion
that remains. All of them are relevant only if the simulation iterates over secondary emission.

| Property | Default | Description |
| --- | --- | --- |
| `maxFractionOfPrimary` | 0.01 | unchanged |
| `maxFractionOfPrevious` | — | removed |
| `maxLuminosityDeficit` | 0.01 | the maximum absolute value of the mean luminosity deficit over the last `numConvergenceIterations` iterations |
| `maxSecondaryChange` | 0.005 | the maximum mean relative change per iteration of the absorbed secondary luminosity over the last `numConvergenceIterations` iterations |
| `numConvergenceIterations` | 3 | the number of iterations over which the convergence quantities are averaged |
| `rebalanceSecondaryEmission` | true | rescale the secondary radiation field per region after each iteration |
| `numAbsorptionRatioBins` | 10 | the number of bins on the axis of the ratio of absorbed secondary to absorbed primary luminosity |
| `numHeatingRateBins` | 10 | the number of bins on the axis of the absorbed luminosity per unit dust mass |

The `rebalanceSecondaryEmission` property is displayed at the regular user level, and the numbers
of bins are displayed at the expert level only. The numbers of bins are relevant only if the
rebalance is enabled.

### Example

```xml
<DustEmissionOptions dustEmissionType="Stochastic" includeHeatingByCMB="false"
                     maxFractionOfPrimary="0.01" maxLuminosityDeficit="0.01"
                     maxSecondaryChange="0.005" numConvergenceIterations="3"
                     rebalanceSecondaryEmission="true" numAbsorptionRatioBins="10" numHeatingRateBins="10"
                     sourceWeight="1" wavelengthBias="0.5">
    <cellLibrary type="SpatialCellLibrary">
        <TemperatureWavelengthCellLibrary numTemperatures="100" numWavelengths="50"/>
    </cellLibrary>
    ...
</DustEmissionOptions>
```

### Incompatibilities

- The `maxFractionOfPrevious` property is removed. The PTS ski file upgrade removes it, and the new
  properties receive their default values. Its documentation already proved misleading: it compares
  the change with the secondary luminosity, which can be many times the primary luminosity. The
  `maxSecondaryChange` criterion monitors the same quantity, averaged over
  `numConvergenceIterations` iterations excluding the first iteration of the loop, but it no longer
  suffices by itself: the luminosity deficit criterion must be met as well.
- Exact absorption changes the results of all simulations with dust emission (see above).
- The rebalance and the new criteria change the number of iterations and the results of all
  simulations that iterate over secondary emission. For models that were run to convergence, the
  results change only within the convergence tolerance; for models that were not, the results move
  toward the converged solution. Models with little self-heating, which the removed criterion often
  stopped after two iterations, now need at least four with the defaults, unless their absorbed
  secondary luminosity stays below the fraction `maxFractionOfPrimary` of the absorbed primary
  luminosity.

These changes are also listed in the [Incompatibilities](skirt-10/04-incompatibilities.md) chapter
of the SKIRT 10 design note.

## Logging

Each secondary emission iteration logs the convergence quantities in a fixed format, replacing the
two lines on the dust-absorbed luminosity logged today. For example:

```
  Dust-absorbed luminosity: primary 2.511e12 Lsun, secondary 6.617e13 Lsun (26.35 x primary)
  Dust-emitted luminosity: 6.865e13 Lsun, self-absorbed fraction 96.38%, luminosity deficit 0.93%
  --> mean luminosity deficit over 3 iterations: 0.03% (criterion 1.00%)
  --> mean change of secondary luminosity over 3 iterations: 0.45% (criterion 0.50%)
  Rebalance: 100 regions, factors 0.984 to 1.044, effective factor 1.0083, no fallbacks
```

The effective factor is the ratio of the total absorbed secondary luminosity after and before the
rebalance. If some regions fall back to the global factor, the rebalance line gives their number
and the global factor instead of "no fallbacks"; if the solve fails, the line says so and reports
the global factor that was applied to all regions instead. In the first iteration of a loop,
which is not rebalanced, the rebalance line is omitted. Until the secondary change is available
(see Criteria), the lines with the means are replaced by a line stating the number of iterations
still needed for the criteria to apply. While the secondary change skips the first iteration, its
line states the smaller number of iterations over which it is averaged.

If the absorbed secondary luminosity is smaller than the absorbed primary luminosity, the first line
gives their ratio as a percentage, as in "(76.84% of primary)". If the loop converges because the
absorbed secondary luminosity is smaller than the fraction `maxFractionOfPrimary` of the absorbed
primary luminosity, a line such as "--> absorbed secondary luminosity is 0.52% of absorbed primary
luminosity (criterion 1.00%)" takes the place of the lines with the means.

Each emission segment of a simulation with dust emission that stores the radiation field, whether
or not it is part of an iteration, also logs the fraction of the dust-absorbed luminosity that was
absorbed at wavelengths outside the radiation field wavelength grid. The final secondary emission
after the iterations stores the radiation field only if the `storeEmissionRadiationField` option of
the secondary emission options is enabled. If the fraction exceeds 1%, the message is issued as
a warning that advises extending the grid. In an iteration loop, the warning is not repeated in
subsequent iterations.

## Probing

All quantities listed above, and a few more on the rebalance, are recorded as scalar series in the
iteration history, so that the history probe writes them to its secondary or merged iteration file,
with one row per iteration (see the Implementation chapter for the list). This gives a
machine-readable record of the convergence, which can be plotted directly, and which shows which
criterion holds up convergence.

The `SecondaryDustLuminosityProbe` reports the exact absorbed luminosity of each cell, since it uses
the same function that normalizes the dust emission. The `DustAbsorptionPerCellProbe` keeps
reporting the spectral absorbed luminosity on the radiation field wavelength grid, calculated from
the binned field. With a coarse grid, its integral over wavelength differs slightly from the exact
absorbed luminosity, as documented for the probe.

## Recommended settings

For models with strong dust self-heating, the recommended settings are as follows:

- a logarithmic radiation field wavelength grid with about 50 bins that covers both the primary
  emission and the dust emission, for example from 0.09 to 2000 µm; with exact absorption, the grid
  only needs to resolve the shape of the radiation field, and the logged fraction of the absorbed
  luminosity outside the grid shows whether it covers the heating spectrum;
- the `TemperatureWavelengthCellLibrary` with 100 temperature bins and 50 wavelength bins, which
  makes iterations much faster without a measurable effect on the spectrum of such models; in tests
  on COLIBRE galaxies at redshift 0, half or twice as many bins of either kind changed neither the
  spectrum nor the run time noticeably;
- the rebalance with the default 10 × 10 bins; more bins hardly change the observed spectrum, and
  they cannot make up for too few photon packets;
- enough photon packets in each secondary emission iteration for the number of spatial cells. With
  too few packets per cell, the luminosity deficit levels off above the criterion while the
  secondary change becomes small, so that the loop runs to its maximum number of iterations and
  ends with a warning. The solution is then biased: the absorbed secondary luminosity settles too
  low, and the escaping dust luminosity stays short of the absorbed primary luminosity by the
  deficit. In the tests, the level of the plateau fell by a factor of 2 to 2.4 for every threefold
  increase of the number of packets, and the number of packets per cell needed to bring it below
  the default criterion grew with the resolution of the grid: some 15 packets per cell sufficed on
  a coarse grid, whereas a grid of the same model with 40 times more cells still leveled off at a
  deficit of 1.3% with that number. A finer spatial grid therefore needs more photon packets, even
  more than in proportion to its number of cells. Even when the deficit meets the criterion, the
  absorbed secondary luminosity can still be several percent too low, although the observed
  spectrum is then hardly affected;
- a maximum of 40–60 secondary emission iterations, which leaves room for the convergence criteria
  to stop the loop;
- a history probe, which records the convergence quantities of each iteration, and thus shows which
  criterion holds up convergence, or whether the deficit has leveled off.

For other models with dust emission, the `TemperatureWavelengthCellLibrary` is an option to be
considered. The library saves most in models with many cells and many dust components, but in models
with less self-absorption it can change the integrated spectrum by a few percent compared with the
`AllCellsLibrary`. The logged ranges of the indicative temperatures and wavelengths hardly reveal
this difference, and more bins do not reduce it. Whether the library or the `AllCellsLibrary` is
closer to the converged spectrum is an open question (see Library or all cells in the Open
questions chapter).
