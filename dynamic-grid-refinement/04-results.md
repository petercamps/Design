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
depth at 0.55 µm reaches several thousand. Raising the maximum level to 40 or 60 lets the mass criterion continue: the
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
at level 60. Both runs at level 60, with 30 and 32 packets per cell, converged to a lower self-absorption, which
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
matters much more for the emission. The `logarithmic` flag and the noise test (`significance`), added after this test,
address both observations: the tests on a slab in the Dust destruction section below show that Monte Carlo noise drives
such runaway refinement, and that the noise test stops it. They have not been tried on these galaxies.

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
iterations. Merged iterations work as well and cost about the same. The dust temperature gradient criterion with an
absolute threshold is not recommended for this purpose; with the `logarithmic` flag and the noise test, as recommended
in the Dust destruction section below, it remains to be tried on these galaxies.

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

The second long-running test concerns a dust destruction front in a slab heated from outside. It proceeded in two
steps on the same model: first refining the temperature field without destruction, to find the criteria and iteration
settings that reach a converged reference with a minimal number of cells, and then adding a destruction front that
the refinement must resolve and the dust destruction recipe must converge on. The noise estimate, the noise test, the
`logarithmic` flag, the `RadiationFieldGradientCriterion`, the candidate guard, the photon packets per cell, and the
noise allowance of the recipe resulted from these tests (see the Features chapter). The main findings are:

- Monte Carlo noise is the central problem. A gradient criterion on a Monte Carlo field runs away: every run without a
  guard cascaded to the cell cap. The noise test stops the cascade without a level limit, and with it the refined grids
  reach, at each number of cells, the accuracy of a noise-free refinement on the reference field.
- The threshold must be relative: with the `logarithmic` flag, the dust temperature criterion and the radiation field
  criterion give nearly optimal grids, while an absolute threshold in kelvin is worse than a uniform grid.
- SEDs converge on every refined grid from about 150 000 cells on; edge-on images of the optically thin emission
  converge more slowly with the number of cells.
- A destruction front needs refinement on the front itself, down to the scale at which noise blurs it, and the recipe
  converges only with its noise allowance.
- For this nearly one-dimensional problem, the octree itself is the limit: hand-made Cartesian grids with a mesh fitted
  to the gradient need 2.5 to 3 times fewer cells for the same accuracy.

### Test case

The model is the slab of the TRUST benchmark: x and y from −5 to 5 pc, z from −5 to −2 pc, with uniform TRUST dust
(silicate, graphite, and PAH, as three fragments of a `FragmentDustMixDecorator`) and an optical depth of 3 at 1 µm
along z. A point source at the origin, 2 pc above the center of the top face, emits a blackbody of 10^4 K with 10^7
solar luminosities, so that the dust stays below 100 K. The temperature changes steeply in z near the top face (its
logarithm by about 2.5 per pc in the top 0.05 pc, against 0.2 to 0.6 per pc deeper down), and much more gently
horizontally.

The reference is a Cartesian grid with 128 × 128 cells horizontally (the width of octree level 7) and 256 cells in z on
a power-law mesh, growing from 3 mpc at the top face to 30 mpc at the bottom, with 4.2 million cells and 1.6·10^9
photon packets. Its temperatures are accurate to about 0.3% rms over the slab and 1% at most. The refinement runs
calculate temperatures only (an extinction-only simulation that stores the radiation field, which gives the same
temperatures as a run with dust emission at a fraction of the cost). They start from a uniform octree at level 2
(64 cells), refine in primary iterations, and end with a final pass of 10^9 photon packets, whose temperatures are
compared with the reference at 2.5 million random points: the relative rms error over the slab, in the top 0.1 pc,
and weighted by the sixth power of the temperature (a proxy for the absorbed luminosity, on which the SED depends),
and the maximum error. A script that refines an octree on the reference field itself, without noise, gives the
optimum that a criterion can reach for each threshold (the oracle).

### Refining the temperature field

#### Noise

The noise of the indicative temperature of a cell scales as the inverse of the cell width times the square root of the
number of photon packets: with 10^8 packets, it is 0.39% at level 6 and 0.75% at level 7. With 6·10^8 packets per
iteration, the noise of the difference between neighbors is then about 0.9% at level 8, 1.9% at level 9, and 3.7% at
level 10, against a threshold of 4%. Every run without a guard cascaded to the cap of 5 million cells, down to level 15
in the top skin of the outer slab, where the radiation arrives at grazing angles and many of the smallest cells
received no photon packet at all in the final pass. Averaging two iterations per decision cascaded just the same.
Keeping level 9 quiet with a threshold of 4% would take about 5·10^9 packets per iteration.

A level limit at level 8 prevents the cascade (602 000 cells), but requires knowing the depth in advance. The noise
test with a significance of 3 prevents it without that knowledge: the grid settles at 354 000 cells, on the oracle
curve, with about 2 600 cells at level 9 and 40 at level 10 where the gradient is significant. All runs with the noise
test end with a creep of small rounds (3 100, 2 300, 1 900, 1 500, 1 100, 1 000, 900, and 800 candidates), which add
15–25% of cells for a few percent in accuracy; a candidate guard of 1% ends them. With 3 000 instead of 1 200 packets
per cell, the grid and its errors are the same, so that about 1 000 packets per cell suffice.

#### Criteria

On the oracle, an absolute threshold on the indicative temperature, which is dominated by the PAH near the top face
(195 K, against 63 K for silicate), over-refines the hot top layer and leaves the cold regions coarse, which is worse
than a uniform grid for most metrics. With the logarithm, the indicative temperature and the temperatures of the
individual fragments give nearly the same grids, so that a criterion on a single fragment brings nothing. Relative to
uniform octrees, the refinement needs about 5 times fewer cells for the same error in the top layer, 2 to 3 times fewer
for the error weighted by the absorbed luminosity, and more cells for the rms error over the slab, where a uniform grid
spreads the error evenly over the bulk.

| Grid | Cells | rms slab | rms top 0.1 pc | max | rms weighted by T⁶ |
| --- | ---: | ---: | ---: | ---: | ---: |
| uniform octree, level 6 (oracle) | 262 000 | 0.78% | 2.10% | 7.0% | 1.35% |
| uniform octree, level 7 (oracle) | 2 097 000 | 0.35% | 1.04% | 3.3% | 0.66% |
| ln T 0.04, no guard (cap) | 5 000 000 | 0.77% | 0.78% | 7.2% | 0.80% |
| ln T 0.04, level ≤ 8 | 602 000 | 0.77% | 0.78% | 3.0% | 0.80% |
| ln T 0.04, noise test | 354 000 | 0.88% | 0.96% | 3.5% | 0.91% |
| ln T 0.028, noise test | 1 069 000 | 0.58% | 0.66% | 2.3% | 0.61% |
| ln J 0.2, level ≤ 8 | 341 000 | 0.95% | 1.04% | 4.0% | 1.06% |
| ln J 0.2, noise test | 142 000 | 1.15% | 1.31% | 5.7% | 1.21% |
| Cartesian 32 × 32 × 128, power-law z mesh | 131 000 | 0.94% | 0.80% | 4.6% | 0.88% |
| Cartesian 64 × 64 × 256, power-law z mesh | 1 049 000 | 0.42% | 0.38% | 2.0% | 0.40% |

Errors of the silicate temperature (graphite is similar). ln T: `DustTemperatureGradientCriterion` with `logarithmic`;
ln J: `RadiationFieldGradientCriterion` over all wavelengths with `logarithmic`; the number is `maxChange`. "Noise
test": `significance` 3. "Level ≤ 8": `maxExtraLevels` 6.

The radiation field criterion gives grids within about 5% of the accuracy of the temperature criterion for the same
number of cells. A threshold of 0.2 on the bolometric mean intensity is coarser than 0.04 on the temperature, probably
because the bolometric field includes the optical and near-infrared radiation, which penetrates deeper and varies less
than the ultraviolet that heats the dust; judging from the oracle, where the number of cells scales as the threshold
to the power −3, about 0.16 gives as many cells as 0.04 on the temperature. The Cartesian grids, designed with
knowledge of the solution, need 2.5 to 3 times fewer cells than the refined octrees for the same accuracy: the octree
ties the cell height to the cell width, so that resolving the thin top layer vertically also refines it horizontally.

![Temperature error versus number of cells on the slab](figures/results-slab.png ':size=100%')

The figure shows the four temperature errors versus the number of cells for the oracle (solid and dotted lines), the
refinement runs (markers), and the Cartesian grids (dashed line). The runs with a threshold that grows by √2 per level
used an option that was tried during these tests and later removed (see Image-driven refinement in the Open
questions).

#### Iterations and photon packets

In this model, the iterations serve only the refinement, and each recalculates the radiation field from scratch, so
that no iteration needs to be left out after a round and one iteration per decision suffices
(`numIterationsBeforeRefinement` 0, `numAveragedIterations` 1, `numIterationsAfterRefinement` 0). A slower schedule
with one iteration before refinement, two averaged iterations, and one left out after a round takes three iterations
per round, without a better decision. With `primaryIterationPacketsPerCell`, the first rounds, on 64 to 33 000 cells,
take a few seconds, where the geometric ramp reached 1.3·10^8 packets on 4 096 cells. Most of the iteration time goes
to the last iterations, at the maximum number of packets. For converged SEDs, the refinement iterations cost about as
much as one full run on the final grid.

#### Observables

Full runs with dust emission on the final grids of five refinement runs, read back with the `TopologyTreePolicy` and
compared with the reference, show the following:

- The SEDs are converged on every refined grid, from 142 000 cells on: their rms deviations equal the noise, and their
  systematic deviation is at most 0.1%. They depend on the absorbed luminosity of each cell, which is exact for any
  grid in this uniform medium, and on the temperature distribution only in second order.
- Face-on slit images are nearly converged from 142 000 cells on, at most about twice the noise.
- An edge-on vertical slit sees the optically thin mid- and far-infrared emission of the whole depth, and resolves the
  vertical steps between the large cells of the bulk. With a constant threshold, its deviation drops slowly with the
  number of cells (at 250 µm, 4.1% at 142 000, 3.1% at 602 000, and 2.2% at 1 069 000 cells, against a noise of 1.3%).
- The full runs on the refined grids take a fifth to a half of the time of a run on the Cartesian grid with
  64 × 64 × 256 cells with the same number of photon packets.

### The destruction front

A `LinearDustDestructionRecipe` sets the density fraction of each fragment from its equilibrium temperature in the
radiation field of the cell. The test uses a silicate step at 50 K (the minimum and maximum temperatures are equal),
with graphite and PAH never destroyed. The converged front is a flat bowl that holds 1.87% of the slab volume, lies
0.41 pc below the top face on the axis, and reaches the top face at a radius of 3.5 pc. It cannot get much deeper,
because graphite and PAH keep absorbing where silicate is gone. The reference front was calculated on the Cartesian
reference grid. The runs are judged by the destroyed volume, by the mismatch (the volume in which the run and the
reference disagree about the silicate fraction, relative to the destroyed volume of the reference, which is 1.5–2% for
the reference itself), and by the absorbed luminosity of each species. They use the iteration settings described
above and the noise allowance of the recipe (`significance` 5).

| Criteria | Cells | Iterations | Destroyed volume | Mismatch | Silicate luminosity |
| --- | ---: | ---: | ---: | ---: | ---: |
| uniform Cartesian 32 × 32 × 256, for comparison | 262 000 | 8 (max) | −0.02% | 6.7% | +0.01% |
| T_sil | 222 000 | 8 | −1.66% | 7.8% | +0.51% |
| T_sil + front ≤ level 7 | 240 000 | 8 | −1.19% | 4.5% | +0.33% |
| T_sil + front ≤ level 8 | 366 000 | 9 | −0.54% | 3.6% | +0.15% |
| T_sil + front ≤ level 9 | 1 040 000 | 10 | −0.79% | 4.8% | +0.23% |
| J + front ≤ level 8 | 336 000 | 8 | −0.52% | 3.7% | +0.14% |
| T_ind ≤ level 8 | 395 000 | 8 | −0.60% | 3.5% | +0.15% |
| T_ind, no level limit (cap) | 1 000 000 | 12 (max) | −0.69% | 4.3% | +0.21% |

T_sil: the dust temperature criterion on the temperature of the silicate fragment alone (an option used in these tests
and later removed), with `logarithmic`, `maxChange` 0.04, and `significance` 3; T_ind: the same on the indicative
temperature; J: the radiation field criterion with `logarithmic`, `maxChange` 0.16, and `significance` 3; front: a
`MediumStateGradientCriterion` on the silicate density fraction `fdyn_0` with `maxChange` 0.5 and a level limit.

- **A smooth criterion alone does not resolve the front.** The silicate temperature is continuous across the front, so
  that the temperature criterion leaves the front at level 6, a staircase of cells of 0.16 × 0.05 pc. Its mismatch is
  worse than that of a uniform grid with a similar number of cells, and the coarse cells straddling the front destroy
  1.7% too little silicate, which then absorbs 0.5% too much luminosity.
- **A criterion on the front resolves it, down to the noise.** At a step, the density fraction jumps from 0 to 1 at
  every level, so that the front criterion needs a level limit. At level 8, it adds a band of cells of 0.04 × 0.012 pc
  along the front, about 145 000 cells in all, and halves the mismatch. Level 9 is worse than level 8: its cells are
  smaller than the distance over which the noise blurs the front (about 0.007, 0.011, and 0.019 pc at levels 7, 8,
  and 9, against cell heights of 0.023, 0.012, and 0.006 pc), so that their fractions flip by noise, the front turns
  ragged, and the flips widen the refined band.
- **The default dust temperature criterion does both, with a level limit.** The indicative temperature of the
  fragmented mix jumps by about 20% at the front, so that the criterion flags the front at every level. With a limit
  at level 8, it refines the front like the front criterion, and the bulk like the silicate temperature criterion,
  with a slightly more accurate bulk. Without a limit, it ran away at the front to the cap of 1 million cells, with a
  worse front, and the recipe did not converge.
- **The radiation field criterion works as well** as the silicate temperature criterion with the front criterion, with
  8% fewer cells.
- The graphite and PAH temperatures, and their absorbed luminosities, hardly depend on the treatment of the front.

![Destroyed silicate fraction and cells through the source](figures/results-front.png ':size=80%')

The figure shows the destroyed silicate fraction and the cells in the vertical plane through the source, in the top
0.75 pc, for the reference, the silicate temperature criterion alone (`r_d50_ts`), with the front criterion at level 8
(`r_d50_ts_fr8e`), and the radiation field criterion with the front criterion at level 8 (`r_d50_rf_fr8`). The line is
the front of the reference.

### Convergence of the recipe

At a step, the density fraction of a cell is 0 or 1, and the cells whose temperature lies within the noise of 50 K
flip in every iteration, however many iterations are run. Their number grows with the resolution of the front: from
25–55 cells per iteration on a uniform octree at level 6 to 12 500–13 100 on a grid refined at the front at level 8.
With the absolute count of SKIRT 9, no fixed value suits grids whose front resolution is not known in advance: the run
refined at level 8 without a noise allowance, which allowed 10 000 cells, settled its grid after 10 iterations and
then spent 30 more iterations, 3.6 hours on xps, without converging. With the noise allowance and the same criteria,
the run converged after 9 iterations in 21 minutes on yoda, with the same results.

| Grid | `significance` | Cells not converged per iteration (last iterations) |
| --- | ---: | ---: |
| refined on T_sil alone, front at level 6, 222 000 cells | 3 | 14–28 |
| refined on T_sil and the front, front at level 8, 341 000–366 000 cells | 0 | 12 500–13 100 |
| same | 3 | 973–1 085 |
| same | 5 | 322, 192, 151 (converged) |
| refined on T_sil and the front, front at level 9, 0.72–1.04 million cells | 5 | 1 161, 1 447, 1 788 |

A significance of 3 is not enough for the recipe, because the noise estimate has heavy tails (see the Features
chapter); with 5, every run with a refined front converged as soon as its grid had settled. The remaining counts were
0.04% of the cells with the front at level 8 and 0.17% at level 9, so that a `maxFractionNotConvergedCells` of 0.002
lets both converge. A ramp instead of a step makes convergence harder: with a ramp from 48 to 52 K, the noise of the
silicate temperature in a cell (about 0.2 K at level 6) is 5% of the ramp, so that about 530 cells per iteration change
by more than the tolerance of 0.05, whereas a wide ramp moves the front.

### Recommended settings

For a smooth medium heated from outside, with or without a destruction front, on an octree:

| Aspect | Setting |
| --- | --- |
| Criterion | A `DustTemperatureGradientCriterion` with `logarithmic` and `significance` 3, without a level limit. The number of cells scales roughly as `maxChange` to the power −3: 0.04 gave about 350 000 cells, temperatures within 0.9% rms, and converged SEDs; 0.028 about 1 million cells, 0.6% rms, and edge-on far-infrared images within 1–2% per pixel. Without dust temperatures, a `RadiationFieldGradientCriterion` with `logarithmic`, `significance` 3, and `maxChange` 0.16 works as well. |
| Destruction front | Either the default `DustTemperatureGradientCriterion` with a `maxExtraLevels` that reaches the noise scale of the front, or the `RadiationFieldGradientCriterion` without a level limit plus a `MediumStateGradientCriterion` on the density fraction of the destroyed fragment (`fdyn_f`, `maxChange` 0.5) with a level limit. The cells at the front should not be smaller than the distance over which noise blurs the front, the relative noise of the temperature divided by its logarithmic gradient; here level 8. |
| Recipe | `significance` 5 and `maxFractionNotConvergedCells` 0.002; a step rather than a narrow ramp, unless the physics asks for a ramp. |
| Iterations | One iteration per decision (0, 1, and 0 for the three schedule options), `primaryIterationPacketsPerCell` about 1 000, a `primaryIterationPacketsMultiplier` that caps the iteration packets, and a `minCandidateFraction` of 0.01. Start from a uniform octree at level 2 (`minLevel` and `maxLevel` 2). |
| Grid | When the geometry is this simple, consider a Cartesian grid with a mesh fitted to the gradient, which needs 2.5 to 3 times fewer cells for the same accuracy. |

For example, for 10^9 photon packets in the final segment:

```xml
<IterationOptions minPrimaryIterations="1" maxPrimaryIterations="40" primaryIterationPacketsMultiplier="0.6"
                  primaryIterationInitialPacketsFraction="0.001" primaryIterationPacketsPerCell="1000"/>
<DynamicRefinementOptions maxCells="5000000" numIterationsBeforeRefinement="0" numAveragedIterations="1"
                          numIterationsAfterRefinement="0" minCandidateFraction="0.01">
```

With these settings, the refinement and the recipe converge together after 8 to 10 iterations.

## Photoionization
