# Results

The proposal was implemented in the SKIRT 10 beta in October 2026. This chapter reports tests of the
implementation with the three COLIBRE galaxies that triggered the proposal. It shows how the new
code converges where SKIRT 9 did not, which grid and how many photon packets produce converged
results in the least run time, and how accurately the cell library reproduces the dust emission
calculated for every cell.

## The COLIBRE galaxies

The three galaxies come from snapshot 56 of the COLIBRE simulations, at redshift 4. They are compact
and dusty: half of their dust mass lies within a few hundred parsec of their center, and the V-band
optical depth along the axes of the domain reaches thousands along at least one axis. Each galaxy is
described by three particle files:

- stellar particles, with their initial mass, metallicity, and age, emitting with the BPASS SED
  family;
- star-forming gas particles, emitting with the Toddlers SED family, normalized to their star
  formation rate over the last 10 Myr;
- dust particles, with the masses of large and small graphite and silicate grains, distributed over
  seven dust medium components (large graphite, large silicate, and small silicate grains, and small
  graphite grains in four parts), with stochastic heating.

| Property | ID33341 | ID5642 | ID39321 |
| --- | ---: | ---: | ---: |
| Stellar particles | 28 525 | 17 852 | 19 243 |
| Star-forming gas particles | 3 225 | 5 897 | 7 213 |
| Dust particles | 15 194 | 21 443 | 22 607 |
| Initial mass of the stellar particles (Msun) | 5.9 × 10<sup>10</sup> | 3.6 × 10<sup>10</sup> | 3.7 × 10<sup>10</sup> |
| Star formation rate (Msun/yr) | 63 | 94 | 280 |
| Dust mass (Msun) | 4.0 × 10<sup>7</sup> | 6.1 × 10<sup>7</sup> | 6.9 × 10<sup>7</sup> |
| Dust half-mass radius (pc) | 528 | 294 | 219 |
| Source luminosity (Lsun) | 9.8 × 10<sup>11</sup> | 1.09 × 10<sup>12</sup> | 2.83 × 10<sup>12</sup> |
| V-band optical depth along the x, y, and z axes | 15, 61, 3101 | 2091, 273, 321 | 1471, 2951, 801 |
| Fraction of the source luminosity absorbed by dust | 84% | 98% | 95% |
| Absorbed secondary / absorbed primary luminosity | 6.1 | 9.4 | 34 |
| Fraction of the dust emission reabsorbed by dust | 86% | 90% | 97% |

The half-mass radius is measured from the dust center of mass, ignoring the smoothing lengths. The
optical depths are those of the particles along the full axes of the domain. The last three rows
come from converged runs on the spatial grid of the SKIRT 9 runs by Andrea Gebek; they depend
somewhat on the grid and on the number of photon packets.

## Test setup

Unless stated otherwise, all runs share the following settings: a cubic domain of 51.4 kpc on a
side; a radiation field wavelength grid with 50 logarithmic bins from 0.09 to 2000 µm; a dust
emission wavelength grid with 100 logarithmic bins over the same range and a finer subgrid from 1 to
30 µm; the `TemperatureWavelengthCellLibrary` with 100 × 50 entries; and the rebalance with 10 × 10
bins. The primary emission is calculated once, followed by secondary emission iterations.

The spatial grids are binary trees, subdivided from level 15 until each cell holds less than a given
fraction of the dust mass, up to a maximum level:

- the *original grid*, the grid of the SKIRT 9 runs by Andrea Gebek, with a mass fraction of
  10<sup>−6</sup> and a maximum level of 36;
- the *coarse grid*, with a mass fraction of 10<sup>−5</sup> for the whole domain and, through a
  `BoxTreePolicy`, 10<sup>−6</sup> in the cube of 2 kpc around the center, which holds 89%, 98%, and
  97% of the dust mass of the three galaxies, also with a maximum level of 36;
- variants of the coarse grid with a maximum level of 40 or 60, or with a mass fraction of
  3 × 10<sup>−7</sup> in the central cube and a maximum level of 60;
- the *optical depth grid*, the original grid with a maximum level of 60 and an
  `OpticalDepthTreePolicy` that subdivides each cell until the optical depth of its diagonal at
  0.55 µm is at most 100.

| Grid | ID33341 | ID5642 | ID39321 |
| --- | ---: | ---: | ---: |
| Original | 402 341 | 316 954 | 197 895 |
| Coarse | 265 742 | 295 700 | 162 021 |
| Coarse, level 40 | 706 046 | 958 003 | 615 941 |
| Coarse, level 60 | 1 341 783 | 1 458 781 | 1 445 346 |
| Coarse, 3 × 10<sup>−7</sup> in the center, level 60 | 4 355 630 | 4 766 656 | — |
| Optical depth | 3 463 685 | 3 409 889 | 8 612 202 |

The comparison with SKIRT 9 and the runs on the optical depth grid use one SED instrument viewing
along the z axis, at a distance of 10 Mpc, and 31.6 million photon packets per emission segment,
unless stated otherwise. The other runs use three SED instruments viewing along the x, y, and z
axes, and are compared through the SED summed over the three, integrated over the wavelength bands
5–10, 10–20, 20–40, 40–80, 80–160, 160–400, and 400–2000 µm, and over 5–1000 µm as a whole. Unless
stated otherwise, these differences are quoted relative to a run on the coarse grid with 100 million
photon packets for the primary emission and the final secondary emission, and 10 million per
secondary emission iteration. The emission below 5 µm hardly depends on any of the settings studied
here: it agrees within 0.5% between all runs. Run times are compared only between runs on the same
computer with the same number of threads.

## Comparison with SKIRT 9

The SKIRT 9 runs by Andrea Gebek used the original grid with the `AllCellsLibrary` and the binned
absorption, and they suffered from the optical depth underflow (see the Implementation chapter).
They performed 60 plain iterations, because a `maxFractionOfPrevious` of 10<sup>−5</sup> kept them
from stopping earlier. The v10beta runs repeat them with the same input files and grid, with exact
absorption, the underflow fix, the rebalance, and the convergence criteria, with a
`maxSecondaryChange` of 0.005, stricter than the default.

### Convergence

![Convergence on the original grid](figures/results-convergence.png ':size=90%')

The figure shows, for each iteration, the luminosity deficit, the ratio of the absorbed secondary to
the absorbed primary luminosity, and the relative change of the absorbed secondary luminosity. For
SKIRT 9, these quantities are calculated from the luminosities in the log files, and the change is
relative to the previous iteration, as in its convergence criterion; for v10beta, they are recorded
by the history probe, and the change is the secondary change, averaged over three iterations.

| Galaxy | Run | Iterations | Deficit, mean of the last three | Absorbed secondary / primary | Absorbed primary luminosity (Lsun) | Spectrum 5–1000 µm, relative to v10beta |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| ID33341 | SKIRT 9 | 60 | 1.8% | 4.4 | 8.01 × 10<sup>11</sup> | 0.989 |
| ID33341 | v10beta | 17 | 0.0% | 6.1 | 8.22 × 10<sup>11</sup> | 1 |
| ID5642 | SKIRT 9 | 60 | 3.4% | 6.9 | 1.046 × 10<sup>12</sup> | 0.968 |
| ID5642 | v10beta | 21 | 0.1% | 9.4 | 1.071 × 10<sup>12</sup> | 1 |
| ID39321 | SKIRT 9 | 60 | 18.4% | 18.0 | 2.52 × 10<sup>12</sup> | 0.837 |
| ID39321 | v10beta | 26 | −0.2% | 34.3 | 2.70 × 10<sup>12</sup> | 1 |

Plain iteration converges slowly, and its criterion is misleading. With the SKIRT 9 default
`maxFractionOfPrevious` of 3%, the runs would have stopped after 15, 16, and 24 iterations, at a
deficit of 8%, 12%, and 29%, and with an absorbed secondary luminosity of less than half of the
converged value. After 60 iterations, the deficit of ID39321 was still 18%, and its absorbed
secondary luminosity still grew by 1% per iteration.

The rebalance removes this problem. The first rebalance multiplies the absorbed secondary luminosity
by a factor of 2.4 to 4.3, and the deficit falls below 1% from iteration 8, 8, and 11. The secondary
change is the last criterion to be met: after the deficit has become small, the absorbed secondary
luminosity still grows by 9 to 21% while the radiation trapped in the core builds up.

### Spectrum

The spectrum below 5 µm agrees within 0.2% for all three galaxies, since the transport of the photon
packets did not change. Between 5 and 1000 µm, the v10beta spectrum is 1.1%, 3.3%, and 19.5%
brighter. Three effects contribute, partly in opposite directions:

- **Convergence.** At the end of the SKIRT 9 runs, the escaping dust luminosity fell short of the
  absorbed primary luminosity by the deficit of 1.8%, 3.4%, and 18%. For ID39321, this is the main
  effect.
- **Optical depth underflow.** The v10beta runs absorb 2.6%, 2.4%, and 6.9% more primary luminosity:
  the energy that the SKIRT 9 runs lost in segments that take the optical depth past 745.
- **Binned absorption.** With 50 bins, the binned absorption overestimates the absorbed luminosity,
  by 0.6% for the primary emission of ID39321. In SKIRT 9, this excess recurs in every pass and is
  amplified by the self-absorption, which partly offsets the two effects above.

### The optical depth of the cells

The underflow matters for these galaxies because the original grid does not resolve the optical
depth: the density criterion stops subdividing a cell once it holds less than 10<sup>−6</sup> of the
dust mass, however opaque the cell is. The table gives the optical depth of the cell diagonals at
0.55 µm, as reported by the `ConvergenceInfoProbe`, and the fraction of the dust mass in cells with
a diagonal optical depth above a given value:

| Galaxy | Largest | Average | Mass-weighted average | Dust mass with τ > 10 | τ > 100 | τ > 150 | τ > 500 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| ID33341 | 4076 | 6.2 | 231 | 73% | 40% | 31% | 10% |
| ID5642 | 4738 | 13.5 | 210 | 81% | 39% | 30% | 11% |
| ID39321 | 4603 | 23.6 | 486 | 90% | 65% | 57% | 32% |

The plain average is low because most cells lie in the nearly empty outskirts; the average weighted
with the dust mass shows the conditions that the radiation meets. Since the extinction in the far
ultraviolet is up to about five times the extinction at 0.55 µm, segments in cells with a diagonal
optical depth above about 150 can take the optical depth past 745 in the ultraviolet. For all three
galaxies, about a third or more of the dust mass lies in such cells, and the star-forming regions,
which emit most of the ultraviolet, sit in the densest dust. On the optical depth grid, no segment
can reach the underflow threshold, and the absorbed primary luminosity agrees with that of the
v10beta runs on the original grid within 0.07%: the underflow fix recovers the lost energy.

### Run time

Exact absorption costs time: in a timing test with ID39321 and a single secondary emission
iteration, it makes the primary emission 22% slower and a secondary emission iteration 14% slower.
The final secondary emission, which stores no radiation field, is not slower.

The three galaxies were run on the original grid with the public SKIRT 9 (commit `4c8330f`) and with
v10beta (commit `12c1f50`), three times each, alternating between the two builds, with 20 threads on
a machine with 20 logical cores. Both builds used the cell library and performed exactly four
secondary emission iterations; the v10beta runs included exact absorption and, from the second
iteration on, the rebalance. The table lists the mean run time of the primary emission and of a
secondary emission iteration, averaged over iterations 2 to 4. The repetitions agree within a few
percent.

| Galaxy | Primary emission, SKIRT 9 | Primary emission, v10beta | Secondary iteration, SKIRT 9 | Secondary iteration, v10beta |
| --- | ---: | ---: | ---: | ---: |
| ID33341 | 220 s | 140 s (−36%) | 120 s | 87 s (−28%) |
| ID5642 | 156 s | 104 s (−33%) | 107 s | 77 s (−28%) |
| ID39321 | 145 s | 94 s (−35%) | 93 s | 65 s (−30%) |

The v10beta runs are about a third faster in both phases, despite the cost of exact absorption. Most
of the gain comes from the restructured tree grids (see the
[Tree-based spatial grids](tree-based-spatial-grids/01-introduction.md) design note), helped by the
known initial cell of a path and link-time optimization: the final secondary emission, which stores
no radiation field and thus measures the cost of the photon paths alone, is 49–54% faster. The
rebalance costs little: forming the regions and solving the system take less than 0.1 s per
iteration, and the tallies along the photon paths add 1–4% to an iteration. The peak memory use is
16–24% lower, which corresponds to about 500 bytes less per node of the tree.

## Spatial grid

The resolution that matters most for these galaxies is that of the dust density in their dense
cores, which is known at setup. The outskirts, in contrast, can be resolved more coarsely than in
the original grid.

### The outskirts

On the original grid, a quarter of the cells of ID33341 lie more than 4 kpc from the center, where
they hold 4% of the dust mass. The coarse grid has 34%, 7%, and 18% fewer cells than the original
grid. For ID33341, the SED on the coarse grid agrees with that on the original grid within 0.6% in
all bands, and the run with 31.6 million photon packets is 22% faster, because both the photon paths
and the per-cell work shrink with the number of cells.

### The core

Both the original and the coarse grid stop at level 36, where a cell measures 12.6 pc. In the dense
core, many cells at that level hold far more than 10<sup>−6</sup> of the dust mass, so that the
density structure within them is not resolved, and their diagonal optical depth reaches several
thousand. Raising the maximum level to 40 or 60 lets the mass criterion continue: the deepest cells
are at level 47 or 48, about 1 pc, and the largest diagonal optical depth falls to 500 to 800. A
mass fraction of 3 × 10<sup>−7</sup> in the central cube resolves the core further. Relative to the
coarse grid at level 36:

| Galaxy | Grid | 5–10 µm | 10–20 µm | 20–40 µm | 40–2000 µm |
| --- | --- | ---: | ---: | ---: | ---: |
| ID33341 | Level 40 | +2.2% | +1.0% | −0.1% | within 0.5% |
| ID33341 | Level 60 | +3.2% | +1.2% | +0.3% | within 0.2% |
| ID33341 | 3 × 10<sup>−7</sup>, level 60 | about +6% | about +2% | about +0.4% | about 0% |
| ID5642 | Level 40 | +9.0% | +4.9% | +0.6% | −0.6…−0.3% |
| ID5642 | Level 60 | +10.3% | +7.8% | +1.5% | −0.7…+0.2% |
| ID5642 | 3 × 10<sup>−7</sup>, level 60 | about +13% | about +12% | about +3% | about 0% |
| ID39321 | Level 40 | +1.1% | +1.5% | 0.0% | +0.7…1.2% |
| ID39321 | Level 60 | +1.4% | +0.6% | +0.8% | −2.0…+0.8% |

For ID39321, the values are the mean of the comparisons of two runs at each level with two runs at
level 36; individual runs of this galaxy scatter by 1 to 2% in the mid-infrared, as much as the
effect itself. The rows for a mass fraction of 3 × 10<sup>−7</sup> add to the run at 10<sup>−6</sup>
and level 60 the difference between two runs at the two mass fractions with about 4 photon packets
per cell, since runs with more packets on more than 4 million cells did not fit in the time
available; at 10<sup>−6</sup> and level 60, the mid-infrared emission of ID33341 with 5 packets per
cell agrees with that with 24 within 0.3% at 5–20 µm.

![Spectrum of ID5642 relative to the coarse grid at level 36](figures/results-levels.png ':size=90%')

The mid-infrared emission is not converged. Level 40 (5 pc) captures most of the change at 5–10 µm,
but only part of it at 10–20 µm, and the finer mass fraction raises the emission further: at 5–10
and 10–20 µm by another 2.6% and 4.1% for ID5642, and by 2.4% and 0.9% for ID33341. The effect is
largest for ID5642, the galaxy with the most compact dust. The emission beyond 40 µm, which carries
most of the dust luminosity, changes by 1 to 2% at most.

The mid-infrared emission of these galaxies thus reflects how the smoothing kernels of the particles
distribute the dust mass at scales of a few parsec, a structure that the hydrodynamical simulation
does not resolve. Presumably, the dense clumps matter as absorbers rather than as emitters: on a
coarse grid, the dust of a clump is spread over a cell of 12.6 pc, which then blocks the
mid-infrared radiation over its whole cross section, whereas the resolved clump covers only part of
it. The optical depth of the cells does not matter by itself: along the z axis of ID33341, the
optical depth grid and the coarse grid at level 60 are both 3.0% brighter at 5–10 µm and 2.1% at
10–20 µm than the coarse grid at level 36.

## Photon packets

### Packets per iteration

The noise in the SED comes from the final primary and secondary emission segments, which feed the
instruments, rather than from the iterations, which only determine the dust temperatures. Two runs
of ID33341 that differ only in the random seed, with 3.16 million packets per iteration and
31.6 million packets in the final segments, agree within 0.3% in all bands except 10–20 µm (0.8%),
whereas two runs with 3.16 million packets throughout differ by up to 2.7%.

The iterations therefore need far fewer packets than the final segments, as long as each cell
receives enough of them. The `secondaryIterationPacketsMultiplier` sets the number of packets per
iteration relative to the final segments. For ID33341 on the coarse grid, with 31.6 million packets
in the final segments:

| Packets per iteration | Packets per cell | Iterations | 5–10 µm | 40–80 µm | 160–400 µm | 5–1000 µm |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 31.6 million | 119 | 14 | +0.5% | +0.1% | +0.2% | +0.1% |
| 3.16 million | 12 | 14–15 | +0.4…0.5% | 0.0…+0.1% | −0.2% | 0.0% |
| 1 million | 3.8 | 14 | 0.0% | −0.6% | −0.8% | −0.6% |
| 0.32 million | 1.2 | 50 | −0.1% | −3.1% | −2.4% | −2.8% |

With about 10 packets per cell or more, the iterations converge to the same SED within the noise, in
the same number of iterations. With fewer, the far-infrared emission settles too low and the
iterations take longer, whereas the mid-infrared emission hardly changes. The galaxy with the
strongest self-absorption needs more: on a grid with 250 000 cells, ID39321 needed 41 iterations
with 13 packets per cell, and its far-infrared emission came out 4% low, whereas with 40 packets per
cell it converged in 23 iterations. The runs of ID39321 at level 60, with 30 and 32 packets per
cell, converged to a lower self-absorption than those at level 36, which probably explains their
lower far-infrared emission (see The core).

### Too few packets per cell

With too few packets per cell, the luminosity deficit levels off above the criterion while the
secondary change becomes small, so that the loop runs to its maximum number of iterations. On the
optical depth grid, the run of ID39321 with 31.6 million packets, fewer than 4 per cell and
iteration, did not converge in 60 iterations: its deficit leveled off at about 5%. Runs with 3.16,
10, and 150 million packets show that the number of packets per cell sets the level of this plateau:

![Luminosity deficit and self-absorption versus the number of photon packets](figures/results-packets.png ':size=90%')

The plateau is a bias rather than slow convergence: with 150 million packets, the mean deficit is
1.36%, 1.32%, and 1.30% over iterations 16–30, 31–45, and 46–60, while the absorbed secondary
luminosity changes by less than 0.1% per iteration. That luminosity settles too low: it still rises
steeply with the number of packets. The plateau falls by a factor of 2 to 2.4 for every threefold
increase of the number of packets; a plateau below 1% would take some 250 million packets, about 30
per cell. On the original grid, 16 packets per cell suffice to converge. A finer grid thus needs
more packets per cell, not just more packets in proportion to its number of cells.

The spectrum along a line of sight does not reveal the deficit. Over all directions, the escaping
dust luminosity of the run with 150 million packets is 1.8% below that of the run on the original
grid, in line with the difference of their deficits, whereas the flux between 5 and 1000 µm along
the z axis is 1.7% higher. The luminosity deficit, logged and recorded by the history probe, rather
than the spectrum, therefore shows whether a run has converged.

## Cell library

The `TemperatureWavelengthCellLibrary` maps the cells to its entries with the dust components
weighted by their total mass in the simulation, so that the mapping depends on the radiation field
only (see Cell library mapping in the Features chapter). The SKIRT 9 mapping calculates the
properties for the dust in each cell instead. In tests by Andrea Gebek on 300 COLIBRE galaxies at
redshift 0, the library with the SKIRT 9 mapping and 100 × 50 entries changed the integrated
spectrum by up to 3.5% compared with the `AllCellsLibrary`, with a deficit around 50 µm and an
excess beyond 100 µm, most pronounced for stellar masses between 10<sup>9.5</sup> and
10<sup>10.5</sup> solar masses. Half or twice as many temperature or wavelength bins gave
practically the same spectra, and ten times more photon packets did not change the difference
either.

The comparison was repeated for one of these galaxies, ID4981185 (997 000 cells, with the input
files and spatial grid of Andrea Gebek and 10<sup>8</sup> photon packets), and for ID39321 (the
original grid, with 3.16 × 10<sup>7</sup> photon packets). To isolate the library, the dust emission
was calculated once from the primary radiation field, without iterations. In ID4981185, small grains
hold 5.5 to 41% of the dust mass between the 10th and 90th percentiles of the particles, weighted by
dust mass, against 0.6 to 3.5% in ID39321. A copy of each galaxy in which every particle has the
average composition of the galaxy tests whether the composition causes the difference. The table
gives the logarithm of the ratio of the dust emission with the library to that with all cells, for
the SKIRT 9 mapping with the dust in each cell, for a mapping with a fixed `MeanInterstellarDustMix`
as reference, and for the mapping with the component weights, each with the galaxy's own composition
or with the uniform copy:

| Galaxy | Mapping | Composition | 40 µm | 160 µm | 250 µm | 1000 µm |
| --- | --- | --- | ---: | ---: | ---: | ---: |
| ID4981185 | dust in the cell | galaxy | −0.024 | +0.017 | +0.021 | +0.025 |
| ID4981185 | dust in the cell, 500 × 250 entries | galaxy | −0.021 | +0.016 | +0.021 | +0.024 |
| ID4981185 | dust in the cell | uniform | −0.004 | +0.002 | +0.001 | −0.002 |
| ID4981185 | `MeanInterstellarDustMix` | galaxy | +0.002 | −0.001 | −0.003 | −0.005 |
| ID4981185 | component weights | galaxy | +0.002 | −0.001 | −0.004 | −0.004 |
| ID39321 | dust in the cell | galaxy | 0.000 | +0.005 | +0.005 | +0.003 |
| ID39321 | `MeanInterstellarDustMix` | galaxy | 0.000 | −0.022 | −0.031 | −0.037 |
| ID39321 | `MeanInterstellarDustMix` | uniform | +0.001 | −0.011 | −0.016 | −0.019 |
| ID39321 | component weights | galaxy | −0.002 | −0.003 | −0.006 | −0.008 |

- **The composition causes the difference.** With uniform composition, the mapping with the dust in
  each cell agrees with all cells within the noise, whereas more entries leave the difference
  unchanged. Two other possible causes, the shape of the radiation field within an entry and the
  Monte Carlo noise of the all-cells result, play no role, because they would persist with uniform
  composition.
- **A fixed reference mix is not good enough.** The `MeanInterstellarDustMix` removes the difference
  for ID4981185, but lowers the far-infrared emission of ID39321 by up to 8%, and by up to 4% even
  with uniform composition, whereas the mapping with the dust in each cell is accurate for this
  galaxy. Its absorption curve is close to that of the COLIBRE dust in the near-infrared, so the
  reason remains unclear.
- **The component weights work for both galaxies.** Beyond 3 µm, the library agrees with all cells
  within 0.005 dex for ID4981185 and within 0.008 dex for ID39321.

Below 3 µm, where the dust emission is weak compared with the starlight, an excess of 0.01 to 0.05
dex remains in all variants.

## Converged results and recommended settings

The emission beyond 40 µm, which carries most of the dust luminosity, is converged to about 1% on
the coarse grid at level 36, provided that the iterations launch at least 10 photon packets per cell
(30 to 40 for ID39321) and that the convergence criteria are met: neither the maximum level nor a
finer mass fraction changes it by more than 1 to 2%. The emission at 5–40 µm is not converged at the
resolutions that fit in a few million cells, because it depends on the dust density structure in the
core at scales within the smoothing kernels of the particles. Converging it further would require
grids of 10 million cells or more; whether that is meaningful depends on the science question.

For dust radiative transfer in compact, self-absorbing galaxies like these, the tests suggest the
following settings, in addition to the general recommendations of the Features chapter:

| Aspect | Setting |
| --- | --- |
| Setup grid, outskirts | A `DensityTreePolicy` with `maxFraction` 10<sup>−5</sup> for the whole domain. |
| Setup grid, core | A `BoxTreePolicy` around the region holding most of the dust (here the central 2 kpc, 89–98% of the dust mass) with a nested `DensityTreePolicy` with `maxFraction` 10<sup>−6</sup>, or less if the mid-infrared emission matters. |
| Maximum level | High enough that it does not stop the mass criterion in the core: at least 40 for a binary tree here (5 pc), 48 or more to let the criterion run its course. |
| Iterations | Primary emission once, followed by secondary emission iterations with the default convergence criteria. |
| Photon packets | `numPackets` of about 30 million for an SED noise of about 0.5% per band; `secondaryIterationPacketsMultiplier` such that each iteration launches at least 10 packets per cell, or 30 to 40 for a galaxy with self-absorption as strong as in ID39321. |

The following table compares the run time on the original grid with 31.6 million packets throughout
with that on the coarse grid at level 36 and at level 40, each with 31.6 million packets in the
final segments and 3.16 million (level 36) or 10 to 19 million (level 40) per iteration, as a
fraction of the run time on the original grid. The runs on the original grid of ID5642 and ID39321
used the stricter `maxSecondaryChange` of 0.005, which added a few iterations.

| Galaxy | Original grid | Coarse grid, level 36 | Coarse grid, level 40 |
| --- | ---: | ---: | ---: |
| ID33341 | 100% (14 iterations) | 27% (14 iterations) | 66% (18 iterations) |
| ID5642 | 100% (21 iterations) | 23% (17 iterations) | 68% (23 iterations) |
| ID39321 | 100% (26 iterations) | 22% (22 iterations) | 99% (27 iterations) |

On the coarse grid at level 36, the SED of ID33341 and ID5642 agrees with that of the reference run
within 0.4% in all bands, at about a quarter of the run time on the original grid. The grid at
level 40 takes 2.4 to 4.5 times as long as the coarse grid at level 36, because it has 2.7 to 3.8
times as many cells and needs more packets per iteration, and it raises the mid-infrared emission as
described in The core.
