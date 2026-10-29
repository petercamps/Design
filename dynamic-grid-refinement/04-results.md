# Results

Dynamic grid refinement was implemented in the SKIRT 10 beta in October 2026. This chapter reports long-running tests
of the implementation with realistic models, one section per application: the settings that produce converged
results in the least run time, the effect of the refinement on the results, and the recommended settings. The
additions to the implementation that resulted from these tests are described in the Features and Implementation
chapters.

## Dust self-heating

The first long-running test outside photoionization concerns the dust radiative transfer of the three COLIBRE
galaxies of the [Dust self-heating](dust-self-heating/04-results.md) design note, which are dominated by dust
self-heating. The goal was to find the settings — the grid built at setup, the dynamic refinement options, the
iteration options, and the number of photon packets — that produce converged results in the least run time, and in
the process to determine the converged results as well as possible. The main findings are:

- The resolution that matters most for these galaxies is that of the dust density in their dense cores, which is
  known at setup. The original grid stops subdividing the core at its maximum level, long before each cell holds less
  than the requested fraction of the dust mass. Lifting that limit raises the emission at 5–10 µm by 1 to 10% and at
  10–20 µm by up to 8%, and a finer mass resolution raises it further: the mid-infrared emission of these galaxies is
  not converged at the resolutions that fit in a few million cells. The emission beyond 40 µm, which carries most of
  the dust luminosity, changes by 1 to 2% at most.
- Dynamic refinement driven by the radiation field changes the spectrum by 1 to 2% at most. It cannot replace the
  resolution of the dense core, because the cells that matter absorb little luminosity and hardly stand out in
  temperature.
- The largest savings in run time come from a coarser grid outside the core and from launching fewer photon packets
  in the iterations than in the final emission segments: together, these make a run about four times faster than in
  the original configuration, with the same results.

### Test case

The three galaxies, ID33341, ID5642, and ID39321, are described in the Results chapter of the dust self-heating note.
They are compact and dusty, with V-band optical depths along the axes of the domain of up to several thousand, and the
dust absorbs 6, 9, and 34 times more of its own emission than of the radiation of the primary sources. Their dust
half-mass radii are 528, 294, and 219 pc. All runs used the input files and the settings of the original runs by
Andrea Gebek, as summarized in that note, with the following exceptions:

- three SED instruments viewing along the x, y, and z axes instead of one along z;
- the default convergence criteria of the SKIRT 10 beta (`maxSecondaryChange` 0.01);
- the spatial grid, the iteration scheme, the dynamic refinement options, and the number of photon packets, as
  described below.

The results are compared through the SED summed over the three instruments, integrated over the wavelength bands
5–10, 10–20, 20–40, 40–80, 80–160, 160–400, and 400–2000 µm, and over 5–1000 µm as a whole. The emission below 5 µm
hardly depends on any of the settings studied here: it agrees within 0.5% between all runs. Unless stated otherwise,
differences are quoted relative to a run on the coarse grid described below, with 100 million photon packets for the
primary emission and the final secondary emission, and 10 million per secondary iteration. Run times are compared
only between runs on the same computer with the same number of threads, and are quoted relative to each other.

### The setup grid

#### The outskirts

Andrea's grid is a binary tree subdivided from level 15 to at most level 36 until each cell holds less than 10^−6 of
the dust mass. For ID33341, a quarter of its cells lie more than 4 kpc from the center, where they hold 4% of the dust
mass. The coarse grid used in the rest of this section subdivides the whole domain until each cell holds less than
10^−5 of the dust mass, and the cube of 2 kpc around the center (with a `BoxTreePolicy`) until each cell holds less
than 10^−6, as before:

| Galaxy | Cells, original grid | Cells, coarse grid | Dust mass within the box |
| --- | ---: | ---: | ---: |
| ID33341 | 402 341 | 265 742 (−34%) | 89% |
| ID5642 | 316 954 | 295 700 (−7%) | 98% |
| ID39321 | 197 895 | 162 021 (−18%) | 97% |

For ID33341, the SED on the coarse grid agrees with that on the original grid within 0.6% in all bands, and the run
with 31.6 million packets is 22% faster, because both the photon paths and the per-cell work shrink with the number
of cells.

#### The maximum level

Both grids stop at level 36, where a cell measures 12.6 pc. In the dense core, many cells at that level hold far more
than 10^−6 of the dust mass, so that the density structure within them is not resolved, and their diagonal optical
depth at 0.55 µm reaches several thousand. Raising the maximum level lets the mass criterion continue: at level 60, the
deepest cells are at level 47 or 48, about 1 pc, and the largest optical depth of a cell falls to 500 to 800. Relative
to the coarse grid at level 36:

| Galaxy | Cells, level 36 | Cells, level 40 | Cells, level 60 | 5–10 µm, level 40 | 5–10 µm, level 60 | 10–20 µm, level 40 | 10–20 µm, level 60 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| ID33341 | 265 742 | 706 046 | 1 341 783 | +2.2% | +3.2% | +1.0% | +1.2% |
| ID5642 | 295 700 | 958 003 | 1 458 781 | +9.0% | +10.3% | +4.9% | +7.8% |
| ID39321 | 162 021 | 615 941 | 1 445 346 | +1.1% | +1.4% | +1.5% | +0.6% |

For ID39321, the values are the mean of the comparisons of two runs at each level with two runs at level 36;
individual runs of this galaxy scatter by 1 to 2% in the mid-infrared, as much as the effect itself. The emission
beyond 40 µm changes by less than 1%, except for ID39321, where it rises by about 1% at level 40 and falls by up to 2%
at level 60; both runs at level 60, with 30 and 32 packets per cell, converged to a lower self-absorption, which
suggests that they had too few packets per cell for this galaxy (see Photon packets). The effect is largest for
ID5642, the galaxy with the most compact dust. Level 40 (5 pc) captures most of the change at 5–10 µm, but only part
of it at 10–20 µm. A grid with an additional `OpticalDepthTreePolicy` limiting the diagonal optical depth to 100, as
in the refined grid of the dust self-heating note, yields the same mid-infrared emission as the grid at level 60
without it: along the z axis of ID33341, both are 3.0% and 2.1% brighter at 5–10 and 10–20 µm. The differences
reported in that note for its refined grid thus stem from the resolution of the density structure, not from the
optical depth limit itself, nor from the low number of packets per cell of those runs.

![Spectrum of ID5642 relative to the coarse grid at level 36](figures/results-levels.png ':size=90%')

### Photon packets

The noise in the SED comes from the final primary and secondary emission segments, which feed the instruments, rather
than from the iterations, which only determine the dust temperatures. Two runs of ID33341 that differ only in the
random seed, with 3.16 million packets per iteration and 31.6 million packets in the final segments, agree within 0.3%
in all bands except 10–20 µm (0.8%), whereas two runs with 3.16 million packets throughout differ by up to 2.7%.

The iterations therefore need far fewer packets than the final segments, as long as each cell receives enough of them.
The `secondaryIterationPacketsMultiplier` (and `primaryIterationPacketsMultiplier` for primary or merged iterations)
sets the number of packets per iteration relative to the final segments. For ID33341 on the coarse grid:

| Packets per iteration | Packets per cell | Iterations | 5–10 µm | 40–80 µm | 160–400 µm | 5–1000 µm |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 31.6 million | 119 | 14 | +0.5% | +0.1% | +0.2% | +0.1% |
| 3.16 million | 12 | 14–15 | +0.4…0.5% | 0.0…+0.1% | −0.2% | 0.0% |
| 1 million | 3.8 | 14 | 0.0% | −0.6% | −0.8% | −0.6% |
| 0.32 million | 1.2 | 50 | −0.1% | −3.1% | −2.4% | −2.8% |

With about 10 packets per cell or more, the iterations converge to the same SED within the noise, in the same number
of iterations. With fewer, the far-infrared emission settles too low, as the luminosity deficit levels off above zero,
and the iterations take longer. The mid-infrared emission hardly depends on the number of packets per cell. The galaxy
with the strongest self-absorption needs more: on a grid with 250 000 cells, ID39321 needed 41 iterations with 13
packets per cell, and its far-infrared emission came out 4% low, whereas with 40 packets per cell it converged in 23
iterations. With 5 packets per cell on 1.4 million cells, its luminosity deficit was still 8% after 25 iterations,
and even its emission at 10–20 µm came out 10% low.

### Dynamic refinement

#### Dust temperature gradient

The `DustTemperatureGradientCriterion` compares the indicative dust temperature of a cell with that of its face
neighbors. For ID33341, the largest differences between neighbors are 10 to 20 K; a threshold of 5 K flags about
33 000 of the 266 000 cells, 10 K about 1 600, and 20 K about 30, with little change once self-heating builds up. With
a threshold of 5 K, the refinement does not settle: each round flags more cells than the previous one (22 000, 30 000,
43 000, 61 000), presumably because a refined region creates new differences at its border and because smaller cells
receive fewer photon packets. The SED did not change measurably after three rounds. The criterion compares absolute
differences, so that a single threshold treats a jump from 20 to 25 K like one from 80 to 85 K, although the former
matters much more for the emission.

#### Dust-absorbed luminosity

The absorbed luminosity is highly concentrated. On the original grid of ID33341, 137 cells each absorb more than 10^−3
of the total, together 42%, and 1 000 cells absorb 63%. These are the opaque cells of the core: cells with a diagonal
optical depth above 100 absorb 31% of the primary and 85% of the secondary luminosity. The `DustLuminosityCriterion`
targets these cells, and its behavior depends on which luminosity it considers:

- Counting all absorbed luminosity, it mostly subdivides the self-absorbing core, where the dust absorbs its own
  emission. With a threshold of 10^−5 in merged iterations, the grid of ID33341 grew from 266 000 to 655 000 cells in
  11 rounds and had not settled after 40 iterations.
- Counting only the primary luminosity, it subdivides the layers in which the radiation of the sources is absorbed.
  With a threshold of 10^−5, the first round adds 160 000 cells and the following rounds fewer and fewer, but the
  refinement does not stop: around bright sources embedded in very opaque dust, it follows the absorbing layer to cells
  of 0.4 pc, and Monte Carlo noise lets a few hundred cells near the threshold cross it in every averaging window.

With a threshold of 3·10^−5, at most four rounds (`maxRounds`), and three levels per round, the primary-only criterion
adds 66 000, 61 000, and 87 000 cells to the coarse grids of ID33341, ID5642, and ID39321. The averaging window can be
short (one iteration before the first decision, two averaged iterations, none left out after a round), because the
primary field does not drift as a dynamic medium state does: each iteration recalculates it from scratch. Refining in
11 primary iterations with a tenth of the packets, followed by the secondary iterations on the refined grid, costs
about as much as refining in merged iterations, which recalculate the primary field in every iteration while the
self-heating builds up: for ID33341, the run with primary iterations took 6% less time, with the same grid and the
same SED.

#### Effect on the spectrum

Relative to the coarse grid at level 36, the refinement raises the emission at 5–10 µm by 1.5 to 2.0% for ID33341,
1.1 to 1.3% for ID5642, and 1.4% for ID39321, and changes the other bands by about 1% at most. The result hardly
depends on the threshold: for ID33341, a grid refined with 10^−5 to 712 000 cells and run with 44 packets per cell
gives +1.95% at 5–10 µm, and the grid refined with 3·10^−5 to 332 000 cells +1.55%. Refining on the total instead of
the primary luminosity gives the same result: for ID5642, a grid refined on the total luminosity with a threshold of
10^−5, to 591 000 cells, gives +1.6% at 5–10 µm. On a setup grid at level 40, which already resolves the core, the
refinement adds only about 20 000 cells and changes the emission by less than 1.5% (ID33341 and ID5642).

The refinement thus cannot replace the resolution of the dense core: for ID5642, it adds 1.3 to 1.6% at 5–10 µm,
where raising the maximum level adds 9 to 10%. Presumably, the dense clumps that the mass criterion resolves matter
as absorbers rather than as emitters: on a coarse grid, the dust of a clump is spread over a cell of 12.6 pc, which
then blocks the mid-infrared radiation over its whole cross section, whereas the resolved clump covers only part of it.
Such clumps absorb little luminosity, and their temperature hardly stands out from that of their neighbors, so that
neither the luminosity criterion nor the temperature gradient criterion selects them.

![Spectrum of ID33341 on refined grids, relative to the coarse grid](figures/results-refined.png ':size=90%')

### Converged results

The emission beyond 40 µm, which carries most of the dust luminosity, is converged to about 1% on the coarse grid at
level 36, provided that the iterations launch at least 10 photon packets per cell (30 to 40 for ID39321) and that the
convergence criteria are met: neither the maximum level, nor a finer mass resolution, nor dynamic refinement changes it
by more than 1 to 2%.

The emission at 5–40 µm is not converged, because it depends on the resolution of the dust density in the dense core,
down to scales well within the smoothing kernels of the dust particles. Raising the maximum level until each cell of
the core holds less than 10^−6 of the dust mass (61 solar masses for ID5642, a fiftieth of the mean dust mass of a
particle) is not enough: for ID5642, a mass resolution of 3·10^−7 (4.8 million cells) raises the emission at 5–10 µm
by another 2.6% and at 10–20 µm by another 4.1%; for ID33341 (4.4 million cells), the increase is 2.4% and 0.9%.
Relative to the coarse grid at level 36:

| Galaxy | Resolution of the core | Cells | 5–10 µm | 10–20 µm | 20–40 µm | 40–2000 µm |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| ID33341 | 10^−6, level 40 | 706 046 | +2.2% | +1.0% | −0.1% | within 0.5% |
| ID33341 | 10^−6, level 60 | 1 341 783 | +3.2% | +1.2% | +0.3% | within 0.2% |
| ID33341 | 3·10^−7, level 60 | 4 355 630 | about +6% | about +2% | about +0.4% | about 0% |
| ID5642 | 10^−6, level 40 | 958 003 | +9.0% | +4.9% | +0.6% | −0.6…−0.3% |
| ID5642 | 10^−6, level 60 | 1 458 781 | +10.3% | +7.8% | +1.5% | −0.7…+0.2% |
| ID5642 | 3·10^−7, level 60 | 4 766 656 | about +13% | about +12% | about +3% | about 0% |
| ID39321 | 10^−6, level 40 | 615 941 | +1.1% | +1.5% | 0.0% | +0.7…1.2% |
| ID39321 | 10^−6, level 60 | 1 445 346 | +1.4% | +0.6% | +0.8% | −2.0…+0.8% |

The mid-infrared emission of these galaxies thus reflects how the particles' smoothing kernels distribute the dust
mass at scales of a few parsec, a structure that the hydrodynamical simulation does not resolve. Converging it further
would require grids of 10 million cells or more; whether that is meaningful depends on the science question.

The rows for a mass resolution of 3·10^−7 combine the run at 10^−6 with the difference between two runs at the two
resolutions with about 4 packets per cell, since runs with enough packets per cell on more than 4 million cells did
not fit in the time available. The mid-infrared emission hardly depends on the number of packets per cell: at
10^−6 and level 60, the run of ID33341 with 5 packets per cell agrees with that with 24 within 0.3% at 5–20 µm.

### Recommended settings

For dust radiative transfer in compact, self-absorbing galaxies like these, the test suggests the following settings:

| Aspect | Setting |
| --- | --- |
| Setup grid, outskirts | A `DensityTreePolicy` with `maxFraction` 10^−5 for the whole domain. |
| Setup grid, core | A `BoxTreePolicy` around the region holding most of the dust (here the central 2 kpc, 89–98% of the dust mass) with a nested `DensityTreePolicy` with `maxFraction` 10^−6. |
| Maximum level | High enough that it does not stop the mass criterion in the core: at least 40 for a binary tree here (5 pc), 48 or more to let the criterion run its course. |
| Iterations | Primary emission once, followed by secondary emission iterations with the default convergence criteria. |
| Photon packets | `numPackets` of about 30 million for an SED noise of about 0.5% per band; `secondaryIterationPacketsMultiplier` such that each iteration launches at least 10 packets per cell, or 30 to 40 for a galaxy with self-absorption as strong as in ID39321. |
| Dynamic refinement | Not needed for the SED. To resolve the radiation field near the sources, for example for maps of the dust temperature, use the `DustLuminosityCriterion` described below. |

For dynamic refinement in such simulations, the following settings worked well: a `DustLuminosityCriterion` with
`maxFraction` 3·10^−5 and the `primaryOnly` flag; `maxRounds` 4 and `levelsPerRound` 3 (for a binary tree); one
iteration before refinement, two averaged iterations, and none left out after a round; primary emission iterations
(`iteratePrimaryEmission`) with a `primaryIterationPacketsMultiplier` of 0.1, followed by the secondary emission
iterations. Merged iterations work as well and cost about the same. The dust temperature gradient criterion is not
recommended for this purpose in its current form.

The following table compares the run time of the original configuration with that of the coarse grid at level 36 and
at level 40, each with 31.6 million packets in the final segments and 3.16 million (level 36) or 10 to 19 million
(level 40) per iteration, as a fraction of the run time of the original configuration. The original runs of ID5642
and ID39321 date from the dust self-heating note and used a stricter convergence criterion (`maxSecondaryChange`
0.005), which added a few iterations.

| Galaxy | Original grid, 31.6 million packets throughout | Coarse grid, level 36 | Coarse grid, level 40 |
| --- | ---: | ---: | ---: |
| ID33341 | 100% (14 iterations) | 27% (14 iterations) | 66% (18 iterations) |
| ID5642 | 100% (21 iterations) | 23% (17 iterations) | 68% (23 iterations) |
| ID39321 | 100% (26 iterations) | 22% (22 iterations) | 99% (27 iterations) |

On the coarse grid at level 36, the SED of ID33341 and ID5642 agrees with the reference within 0.4% in all bands, at a
quarter of the run time of the original configuration. The grid at level 40 takes 2.4 to 4.5 times as long as the
coarse grid at level 36, because it has 2.7 to 3.8 times as many cells and needs more packets per iteration, and it
raises the mid-infrared emission as described above.

## Dust destruction

## Photoionization
