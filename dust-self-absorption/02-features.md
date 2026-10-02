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
modeling choice, so there is no reason to keep it, except for comparing results (see Open
questions).

### Consequences

- **Energy conservation.** Up to Monte Carlo noise, the total luminosity absorbed by dust equals
  the luminosity lost by the photon packets to dust, for any radiation field wavelength grid.
- **Radiation field wavelength grid.** The grid only needs to resolve the shape of the radiation
  field. For the COLIBRE galaxies, a logarithmic grid with 50 bins from 0.09 to 2000 µm gives the
  same result as a grid with 200 bins over the same range. The documentation advice for strongly
  self-absorbing models changes accordingly.
- **Absorption outside the grid.** Photon packets with a wavelength outside the radiation field
  wavelength grid are not recorded in the radiation field, but their absorption by dust is counted.
  The emitted energy is therefore correct even if the grid misses part of the heating spectrum.
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
does not describe, so the rebalance is disabled with a message in the setup log (see Open questions).

### Cost

The tallies cost about 1–2% of an iteration with 100 regions. The memory needed is one transfer
matrix per execution thread, which is 80 kB for 100 regions and 6.5 MB for 900 regions. The linear
system is solved by Gaussian elimination, which takes well under a second even for 900 regions.

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
- the **secondary change** *c(n) = |Ls(n) − Ls(n−w)| / (w Ls(n))*, the mean relative change of the
  absorbed secondary luminosity per iteration over the last *w* iterations.

### Criteria

The secondary emission loop has converged in iteration *n* if one of the following conditions
holds:

- the absorbed secondary luminosity is smaller than the fraction `maxFractionOfPrimary` of the
  absorbed primary luminosity, which is typically the case in the first iteration of a model with
  little dust emission, as today; or
- at least *w + 1* iterations have been performed, the mean of the luminosity deficit over the
  last *w* iterations is at most `maxLuminosityDeficit` in absolute value, and the secondary change
  is at most `maxSecondaryChange`.

Here *w* is the value of `numConvergenceIterations`. Averaging over several iterations suppresses
the Monte Carlo noise in both quantities, which is about 0.5% per iteration for the deficit of the
COLIBRE galaxies with 31.6 million photon packets.

The two conditions of the second criterion address different errors. The deficit measures whether
the escaping luminosity, and hence the dust emission seen by an observer, is right. With the
rebalance, the deficit drops below 1% within about 10–15 iterations, even for ID39321. The
secondary change measures whether the distribution of the emission over the cells has settled. With
the rebalance, it decays more slowly, because the radiation trapped in the core keeps building up
for some time. This affects the observed spectrum much less than it affects the absorbed secondary
luminosity: two runs with 100 and 900 regions differed by 2.7% in absorbed secondary luminosity
after 40 iterations, while their dust emission spectra agreed within 0.2%.

As today, the loop runs for at least `minSecondaryIterations` and at most `maxSecondaryIterations`
iterations, and a warning is issued if it ends without convergence. Without dust, the dust criteria
are skipped, and convergence is determined by the dynamic medium state alone, as today.

### Example

Applied to the logs of the ID39321 runs with 100 regions and exact absorption, the default values
proposed below stop the loop after 24–26 iterations, when the absorbed secondary luminosity is
within 5% of its value after 40 iterations. For a run with 900 regions (with the current absorption
calculation and 200 bins), they stop the loop after 20 iterations, within 3%.

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
| `numCoreBins` | 10 | the number of bins on the axis of the ratio of absorbed secondary to absorbed primary luminosity |
| `numHeatingBins` | 10 | the number of bins on the axis of the absorbed luminosity per unit dust mass |

The `rebalanceSecondaryEmission` property is displayed at the regular user level, and the numbers
of bins are displayed at the expert level only. The numbers of bins are relevant only if the
rebalance is enabled.

### Example

```xml
<DustEmissionOptions dustEmissionType="Stochastic" includeHeatingByCMB="false"
                     maxFractionOfPrimary="0.01" maxLuminosityDeficit="0.01"
                     maxSecondaryChange="0.005" numConvergenceIterations="3"
                     rebalanceSecondaryEmission="true" numCoreBins="10" numHeatingBins="10"
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
  the change with the secondary luminosity, which can be many times the primary luminosity.
- Exact absorption changes the results of all simulations with dust emission (see above).
- The rebalance and the new criteria change the number of iterations and the results of all
  simulations that iterate over secondary emission. For models that were run to convergence, the
  results change only within the convergence tolerance; for models that were not, the results move
  toward the converged solution.

These changes are also listed in the [Incompatibilities](skirt-10/04-incompatibilities.md) chapter
of the SKIRT 10 design note.

## Logging

Each secondary emission iteration logs the convergence quantities in a fixed format, replacing the
two lines on the dust-absorbed luminosity logged today. For example, for iteration 24 of ID39321:

```
  Dust-absorbed luminosity: primary 2.511e12 Lsun, secondary 6.617e13 Lsun (26.35 x primary)
  Dust-emitted luminosity: 6.865e13 Lsun, self-absorbed fraction 96.38%, luminosity deficit 0.93%
  --> mean luminosity deficit over 3 iterations: 0.03% (criterion 1.00%)
  --> mean change of secondary luminosity over 3 iterations: 0.45% (criterion 0.50%)
  Rebalance: 100 regions, factors 0.984 to 1.044, effective factor 1.0083, no fallbacks
```

The effective factor is the ratio of the total absorbed secondary luminosity after and before the
rebalance. If the solve fails, the rebalance line says so and reports the global factor that was
applied instead. As long as fewer than *w + 1* iterations have been performed, the lines with the
means are replaced by a line stating the number of iterations still needed for the criterion to
apply.

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
absorbed luminosity, as documented for the probe (see Open questions).

## Recommended settings

For strongly self-absorbing models, the documentation recommends:

- a logarithmic radiation field wavelength grid with about 50 bins from 0.09 to 2000 µm, which
  suffices with exact absorption;
- the `TemperatureWavelengthCellLibrary` with 100 temperature bins and 50 wavelength bins, which
  makes iterations about 25 times faster without a measurable effect on the spectrum, with the
  advice to increase the number of temperature bins if the logged temperature range is very wide;
- the rebalance with the default 10 × 10 regions;
- some 30 million photon packets per iteration for the COLIBRE galaxies tested, which have a few
  hundred thousand cells, since ten times more packets changed the global quantities by only 0.1%;
- a maximum of 40–60 secondary emission iterations, which leaves room for the convergence criteria
  to stop the loop.
