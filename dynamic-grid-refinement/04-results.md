# Results

Dynamic grid refinement was implemented in the SKIRT 10 beta in October 2026. This chapter reports long-running tests
with realistic models, one section per application. Each section shows which refinement settings work and why, what the
refinement changes in the results, and which settings are recommended. The tests use the features described in the
Features chapter.

## Dust self-heating

The first long-running test outside photoionization concerns the three COLIBRE galaxies of the
[Dust self-heating](dust-self-heating/04-results.md) design note, which are dominated by dust self-heating. The Results
chapter of that note shows that the SED of these galaxies depends mostly on the resolution of the dust density in their
dense cores and on the number of photon packets per cell, and it recommends a setup grid that is coarse in the outskirts
and resolves the dust mass in the core. The idea behind the current test is to start from a coarse setup grid, and to
let dynamic refinement decide where the grid must be finer. For the SED, this does not work: the refinement changes it
by 2% at most, whereas resolving the core at setup changes it by up to 10%, because no refinement criterion selects the
cells that matter. The refinement does resolve the radiation field near the sources, which should benefit maps of the
dust temperature.

### Setup

The setup grid is the coarse grid of the dust self-heating note: a binary tree, subdivided up to level 36 until each
cell holds less than 10^−5 of the dust mass, and less than 10^−6 in the central 2 kpc, with 265 742, 295 700, and
162 021 cells for ID33341, ID5642, and ID39321. Its maximum level stops the mass criterion in the dense core, so that it
does not resolve the dust mass there. The other settings are those of that note. The refinement takes place in primary
emission iterations, followed by secondary emission iterations on the refined grid, or in merged iterations; both give
the same grid and the same SED at about the same cost. Unless stated otherwise, differences are quoted relative to a run
on the setup grid without refinement, with 100 million photon packets for the primary emission and the final secondary
emission and 10 million per iteration, for the SED summed over three instruments viewing along the x, y, and z axes.

### Refinement criteria

The luminosity absorbed by the dust is highly concentrated. On the grid of the SKIRT 9 runs of ID33341, 137 cells each
absorb more than 10^−3 of the total, together 42%, and cells with a diagonal optical depth above 100 absorb 31% of the
primary and 85% of the secondary luminosity. The `DustLuminosityCriterion` targets these cells, and its behavior depends
on which luminosity it counts:

- Counting all absorbed luminosity, it mostly subdivides the self-absorbing core, where the dust absorbs its own
  emission, and it does not settle: with a threshold of 10^−5, the grid of ID33341 grew from 266 000 to 655 000 cells in
  11 rounds.
- Counting only the primary luminosity (`primaryOnly`), it subdivides the layers in which the radiation of the sources
  is absorbed. With a threshold of 3·10^−5, the first decision flags 2.0 to 4.4% of the cells, the second 0.6 to 1.7%,
  and later decisions 0.1 to 0.5%, a trickle of cells close to the threshold that cross it by noise. A candidate guard
  of 1% therefore ends the refinement after one round for ID33341 and ID5642 and after two for ID39321, adding 43 000,
  42 000, and 75 000 cells. Without a guard, the refinement does not stop: with a threshold of 10^−5, it follows the
  absorbing layer around bright sources embedded in very opaque dust to cells of 0.4 pc.

The `RadiationFieldGradientCriterion` settles as well. With the `logarithmic` flag, a `maxChange` of 0.16, a
`significance` of 3, and a candidate guard of 1%, the first two decisions flagged 5.8% and 2.9% of the cells of ID33341,
so that the grid grew from 266 000 to 374 000 and 450 000 cells; the seven decisions that followed flagged 0.8 to 0.9%
each, which the guard left unsplit.

The `DustTemperatureGradientCriterion` does not settle. With the `logarithmic` flag, a `maxChange` of 0.04, a
`significance` of 3, and a candidate guard of 1%, it flagged 19%, 12%, and 12% of the cells of ID33341 in three
successive decisions, so that the grid grew from 266 000 to 620 000 and 1.16 million cells, and then to the cap of
2 million. The noise test did not slow the refinement, although the number of photon packets per cell fell from 414 to
95, so the flagged differences are not noise. The likely cause is the dust composition, which varies between cells. The
indicative temperature is the mean of the temperatures of the seven dust components weighted by their mass in the cell,
and in ID33341 the mass fraction of the small grains, which are hotter than the large grains in the same radiation
field, ranges from 1 to 10% (10th and 90th percentiles of the particles, weighted by dust mass). Neighbors that differ
in composition then differ in indicative temperature in the same radiation field. Because child cells inherit their
parent's composition, subdivision does not remove such a jump: it recurs between the children on either side of the
common wall, which the criterion flags again in the next round. The radiation field criterion, which does not depend on
the dust properties, settles with a threshold of 0.16 on the mean intensity, which corresponds to 2.5 to 4% on a
temperature in equilibrium with the radiation field (see Radiation field in the Features chapter). This correspondence
assumes that neighboring cells have radiation fields of the same spectral shape, so a threshold that is too fine for a
clumpy galaxy may contribute as well.

### Effect on the spectrum

The luminosity criterion raises the emission at 5–10 µm by 1 to 2% and changes the other bands by about 1% at most,
regardless of the threshold, of the luminosity it counts, and of the number of rounds: for ID33341, grids refined to
332 000 and 712 000 cells give +1.55% and +1.95% at 5–10 µm, and for ID5642, refinement on the total luminosity to
591 000 cells gives +1.6%. The candidate guard, which ends the refinement after one or two rounds, keeps about half of
this rise. The radiation field criterion changes no band of ID33341 by more than 0.8% (+0.3% at 5–10 µm), close to the
noise of these runs. On a setup grid at level 40, which already resolves the core, the luminosity criterion adds only
about 20 000 cells and changes the emission by less than 1.5%.

![Spectrum of ID33341 on refined grids, relative to the coarse grid](figures/results-refined.png ':size=90%')

Resolving the core at setup, in contrast, raises the emission of ID5642 at 5–10 µm by 9 to 10% (see the dust
self-heating note), so dynamic refinement cannot replace it. The dust self-heating note attributes this emission to the
density structure of the core, presumably to dense clumps that act as absorbers of the mid-infrared radiation. Dynamic
refinement misses these clumps for two reasons. A child cell inherits the density of its parent (see Initial state of
child cells in the Features chapter), so that subdividing a cell adds no density structure. Moreover, the clumps absorb
little luminosity, and their temperature and radiation field hardly stand out from those of their neighbors, so that no
criterion selects them in the first place.

### Temperature maps

The luminosity criterion with `primaryOnly` resolves the layers in which the radiation of the sources is absorbed, where
the dust is hottest and its temperature changes over the shortest distances. These tests did not examine maps of the
dust temperature, but such maps, and other results that depend on the local radiation field near the sources, are
expected to benefit from this refinement, even though the SED does not.

### Recommended settings

For the SED of compact, self-absorbing galaxies like these, dynamic refinement is not needed; the setup grid recommended
in the dust self-heating note, which resolves the dust mass in the core, matters far more. To resolve the radiation
field near the sources, for example for maps of the dust temperature, the following settings refine the grid in one or
two rounds at a modest cost:

- a `DustLuminosityCriterion` with `maxFraction` 3·10^−5 and the `primaryOnly` flag;
- a `minCandidateFraction` of 0.01 (or `maxRounds` 4, which refines somewhat further) and `levelsPerRound` 3 (for a
  binary tree);
- one iteration before refinement, two averaged iterations, and none left out after a round; the window can be short
  because each iteration recalculates the primary radiation field from scratch, so that it does not drift as a dynamic
  medium state does;
- primary emission iterations (`iteratePrimaryEmission`) with a `primaryIterationPacketsMultiplier` of 0.1, followed by
  the secondary emission iterations on the refined grid; merged iterations work as well and cost about the same.

The dust temperature gradient criterion is not recommended when the dust composition varies between cells, because it
does not settle.

## Dust destruction

The second long-running test concerns a dust destruction front in a slab heated from outside. The refinement must
resolve both the smooth temperature field and the front, with a minimal number of cells, and the dust destruction recipe
must converge on the front. The main findings are:

- Monte Carlo noise is the central problem. Without the noise test, a gradient criterion on a Monte Carlo field runs
  away. With it, the refinement stops by itself, and the refined grids reach, at each number of cells, the accuracy of a
  noise-free refinement on the reference field.
- The threshold must be relative: with the `logarithmic` flag, the dust temperature criterion and the radiation field
  criterion give nearly optimal grids, while an absolute threshold in kelvin is worse than a uniform grid.
- SEDs converge on every refined grid from about 150 000 cells on; edge-on images of the optically thin emission
  converge more slowly with the number of cells.
- A destruction front needs a criterion on the front itself, and is resolved best down to the scale at which noise blurs
  it. A level limit at that scale works best; the noise test on the front, with a significance of 3, stops near that
  scale by itself, but at a level that varies along the front, which makes the front less accurate. The recipe converges
  only with its noise allowance.
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
a power-law mesh, growing from 3 mpc at the top face to 30 mpc at the bottom, with 4.2 million cells and 1.6·10^9 photon
packets. Its temperatures are accurate to about 0.3% rms over the slab and 1% at most. The refinement runs calculate
temperatures only (an extinction-only simulation that stores the radiation field, which gives the same temperatures as a
run with dust emission at a fraction of the cost). They start from a uniform octree at level 2 (64 cells), refine in
primary iterations, and end with a final pass of 10^9 photon packets, whose temperatures are compared with the reference
at 2.5 million random points: the relative rms error over the slab, in the top 0.1 pc, and weighted by the sixth power
of the temperature (a proxy for the absorbed luminosity, on which the SED depends), and the maximum error. A script that
refines an octree on the reference field itself, without noise, gives the optimum that a criterion can reach for each
threshold (the oracle).

### Refining the temperature field

The runs in this section have no dust destruction: all fragments keep their full density, in the reference as well
as in the refined grids.

#### Noise

The noise of the indicative temperature of a cell scales as the inverse of the cell width times the square root of the
number of photon packets: with 10^8 packets, it is 0.39% at level 6 and 0.75% at level 7. With 6·10^8 packets per
iteration, the noise of the difference between neighbors is then about 0.9% at level 8, 1.9% at level 9, and 3.7% at
level 10, against a threshold of 4%. Without the noise test, every run cascaded to the cap of 5 million cells, down to
level 15 in the top skin of the outer slab, where the radiation arrives at grazing angles and many of the smallest cells
received no photon packet at all in the final pass. Averaging two iterations per decision does not help, and keeping
level 9 quiet with photon packets alone would take about 5·10^9 packets per iteration.

A level limit at level 8 prevents the cascade (602 000 cells), but requires knowing the depth in advance. The noise test
with a significance of 3 prevents it without that knowledge: the grid settles at 354 000 cells, on the oracle curve,
with about 2 600 cells at level 9 and 40 at level 10 where the gradient is significant. The refinement then ends with a
creep of small rounds (from 3 100 down to 800 candidates), which add 15–25% of cells for a few percent in accuracy; the
candidate guard of 1% ends it. With 3 000 instead of 1 200 packets per cell, the grid and its errors are the same, so
that about 1 000 packets per cell suffice.

#### Criteria

On the oracle, an absolute threshold on the indicative temperature, which is dominated by the PAH near the top face
(195 K, against 63 K for silicate), over-refines the hot top layer and leaves the cold regions coarse, which is worse
than a uniform grid for most metrics. With the logarithm, the indicative temperature and the temperatures of the
individual fragments give nearly the same grids. Relative to uniform octrees, the refinement needs about 5 times fewer
cells for the same error in the top layer, 2 to 3 times fewer for the error weighted by the absorbed luminosity, and
more cells for the rms error over the slab, where a uniform grid spreads the error evenly over the bulk.

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
than the ultraviolet that heats the dust; judging from the oracle, where the number of cells scales as the threshold to
the power −3, about 0.16 gives as many cells as 0.04 on the temperature. The Cartesian grids, designed with knowledge of
the solution, need 2.5 to 3 times fewer cells than the refined octrees for the same accuracy: the octree ties the cell
height to the cell width, so that resolving the thin top layer vertically also refines it horizontally.

![Temperature error versus number of cells on the slab](figures/results-slab.png ':size=90%')

The figure shows the four temperature errors versus the number of cells for the oracle (solid and dotted lines), the
refinement runs (markers), and the Cartesian grids (dashed line). The runs marked √2 use a threshold that grows by a
factor of √2 per level, a test-only variant (see Image-driven refinement in the Open questions).

#### Iterations and photon packets

Each iteration recalculates the radiation field from scratch, so that one iteration per decision suffices and no
iteration needs to be left out after a round (`numIterationsBeforeRefinement` 0, `numAveragedIterations` 1,
`numIterationsAfterRefinement` 0); a slower schedule with three iterations per round gives no better decisions. With
`primaryIterationPacketsPerCell`, the first rounds, on 64 to 33 000 cells, take a few seconds, and most of the iteration
time goes to the last iterations, at the maximum number of packets. For converged SEDs, the refinement iterations cost
about as much as one full run on the final grid.

#### Observables

Full runs with dust emission on the final grids of five refinement runs, read back with the `TopologyTreePolicy` and
compared with the reference, show the following:

- The SEDs are converged on every refined grid, from 142 000 cells on: their rms deviations equal the noise, and their
  systematic deviation is at most 0.1%. They depend on the absorbed luminosity of each cell, which is exact for any grid
  in this uniform medium, and on the temperature distribution only in second order.
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
the reference itself), and by the absorbed luminosity of each species. They use the iteration settings described above.

#### Convergence of the recipe

At a step, the density fraction of a cell is 0 or 1, and the cells whose temperature lies within the noise of 50 K flip
in every iteration, however many iterations are run. Their number grows with the resolution of the front: from 25–55
cells per iteration on a uniform octree at level 6 to 12 500–13 100 on a grid with cells at level 8 along the front.
Without the noise allowance, the recipe therefore does not converge on a refined front: a run with the front at level 8
settled its grid after 10 iterations and then did not converge in 30 more. The allowance removes the flips that the
noise explains from the convergence test:

| Front | `significance` | Cells not converged per iteration (last iterations) |
| --- | ---: | ---: |
| at level 6, 222 000 cells | 3 | 14–28 |
| at level 8, 341 000–366 000 cells | 0 | 12 500–13 100 |
| at level 8 | 3 | 973–1 085 |
| at level 8 | 5 | 322, 192, 151 (converged) |
| at level 9, 0.72–1.04 million cells | 5 | 1 161, 1 447, 1 788 |

A significance of 3 is not enough, because the noise estimate has heavy tails (see the Features chapter); with 5, every
run with a refined front converged as soon as its grid had settled. The remaining counts are 0.04% of the cells with the
front at level 8 and 0.17% at level 9, so that a `maxFractionNotConvergedCells` of 0.002 lets both converge. A ramp
instead of a step makes convergence harder: with a ramp from 48 to 52 K, the noise of the silicate temperature in a cell
(about 0.2 K at level 6) is 5% of the ramp, so that about 530 cells per iteration change by more than the tolerance of
0.05, whereas a wide ramp moves the front. The refinement runs below use a significance of 5 and a
`maxFractionNotConvergedCells` of 0.002.

#### Resolving the front

| Criteria | Cells | Iterations | Destroyed volume | Mismatch | Silicate luminosity |
| --- | ---: | ---: | ---: | ---: | ---: |
| uniform Cartesian 32 × 32 × 256, for comparison | 262 000 | 8 (max) | −0.02% | 6.7% | +0.01% |
| T_sil | 222 000 | 8 | −1.66% | 7.8% | +0.51% |
| T_sil + front ≤ level 7 | 240 000 | 8 | −1.19% | 4.5% | +0.33% |
| T_sil + front ≤ level 8 | 366 000 | 9 | −0.54% | 3.6% | +0.15% |
| T_sil + front ≤ level 9 | 1 040 000 | 10 | −0.79% | 4.8% | +0.23% |
| J + front ≤ level 8 | 337 000 | 8 | −0.67% | 3.7% | +0.18% |
| J + front, noise test, k = 2 | 1 618 000 | 16 (max) | −0.71% | 4.3% | +0.21% |
| J + front, noise test, k = 3 | 336 000 | 10 | −0.79% | 4.5% | +0.23% |
| J + front, noise test, k = 5 | 260 000 | 9 | −1.09% | 5.6% | +0.31% |
| T_ind ≤ level 8 | 395 000 | 8 | −0.60% | 3.5% | +0.15% |
| T_ind, no level limit (cap) | 1 000 000 | 12 (max) | −0.69% | 4.3% | +0.21% |

T_sil: the dust temperature criterion on the temperature of the silicate fragment alone, a test-only variant that,
unlike the indicative temperature, is continuous across the front, with `logarithmic`, `maxChange` 0.04, and
`significance` 3; T_ind: the same on the indicative temperature; J: the radiation field criterion with `logarithmic`,
`maxChange` 0.16, and `significance` 3; front: a `MediumStateGradientCriterion` on the silicate density fraction
`fdyn_0` with `maxChange` 0.5, with a level limit, or with the noise test and no level limit (`significance` k).

- **A smooth criterion alone does not resolve the front.** The silicate temperature is continuous across the front, so
  that the temperature criterion leaves the front at level 6, a staircase of cells of 0.16 × 0.05 pc. Its mismatch is
  worse than that of a uniform grid with a similar number of cells, and the coarse cells straddling the front destroy
  1.7% too little silicate, which then absorbs 0.5% too much luminosity.
- **A criterion on the front resolves it, down to the noise.** At a step, the density fraction jumps from 0 to 1 at
  every level, so that the front criterion without the noise test needs a level limit. At level 8, it adds a band of
  cells of 0.04 × 0.012 pc along the front, about 145 000 cells in all, and halves the mismatch. Level 9 is worse than
  level 8: its cells are smaller than the distance over which the noise blurs the front (about 0.007, 0.011, and
  0.019 pc at levels 7, 8, and 9, against cell heights of 0.023, 0.012, and 0.006 pc), so that their fractions flip by
  noise, the front turns ragged, and the flips widen the refined band.
- **The radiation field criterion works as well** as the silicate temperature criterion together with the front
  criterion, with 8% fewer cells. The runs with the noise test on the front use it.
- **With the noise test, the front criterion stops by itself, but unevenly.** With k = 3 and no level limit, the
  refinement settles at as many cells as with the limit at level 8, and resolves most of the front at level 8. Along the
  front, however, the level varies from 6 to 11, because the outcome of the pair test at a step depends on where the
  front lies between the two cell centers, and on the spread of the noise estimate. The front is then only as accurate
  as with a limit at level 7 or 9. With k = 2, the refinement runs away: below the noise scale, the front is a band in
  which neighbors differ at random, and flips by more than two estimated standard errors are frequent enough that each
  subdivision creates more candidates than it removes (see Noise test in the Features chapter). At the iteration limit,
  the band was filled with cells down to level 15, but was no narrower than with the limit at level 8. With k = 5, the
  refinement stops a level too early, with half of the front at level 7.
- **The default dust temperature criterion does both, with a level limit.** The indicative temperature of the fragmented
  mix jumps by about 20% at the front, so that the criterion flags the front at every level. With a limit at level 8, it
  refines the front like the front criterion, and the bulk like the silicate temperature criterion, with a slightly more
  accurate bulk. Without a limit, it runs away at the front to the cell cap, with a worse front, so that the recipe does
  not converge either: unlike the noise of the front criterion, the noise of the indicative temperature covers the
  temperature change of each fragment in a scaled radiation field, but not the jump when a fraction flips.
- The graphite and PAH temperatures, and their absorbed luminosities, hardly depend on the treatment of the front.

![Destroyed silicate fraction and cells through the source](figures/results-front.png ':size=80%')

The figure shows the destroyed silicate fraction and the cells in the vertical plane through the source, in the top
0.75 pc, for the reference, the silicate temperature criterion alone (`r_d50_ts`), with the front criterion at level 8
(`r_d50_ts_fr8e`), and the radiation field criterion with the front criterion at level 8 (`r_d50_rf_fr8`). The line is
the front of the reference.

### Recommended settings

For a smooth medium heated from outside, with or without a destruction front, on an octree:

| Aspect | Setting |
| --- | --- |
| Criterion | A `DustTemperatureGradientCriterion` with `logarithmic` and `significance` 3, without a level limit. The number of cells scales roughly as `maxChange` to the power −3: 0.04 gave about 350 000 cells, temperatures within 0.9% rms, and converged SEDs; 0.028 about 1 million cells, 0.6% rms, and edge-on far-infrared images within 1–2% per pixel. Without dust temperatures, a `RadiationFieldGradientCriterion` with `logarithmic`, `significance` 3, and `maxChange` 0.16 works as well. |
| Destruction front | Either the default `DustTemperatureGradientCriterion` with a `maxExtraLevels` that reaches the noise scale of the front, or the `RadiationFieldGradientCriterion` without a level limit plus a `MediumStateGradientCriterion` on the density fraction of the destroyed fragment (`fdyn_f`, `maxChange` 0.5) with a level limit. The cells at the front should not be smaller than the distance over which noise blurs the front, the relative noise of the temperature divided by its logarithmic gradient; here level 8. If that scale is not known in advance, the `MediumStateGradientCriterion` with `significance` 3 and no level limit finds it, at the cost of a less accurate front. |
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

This section returns to the motivating case, with the two examples on which the reference implementation was developed:
an H II region around a single cluster in a uniform medium, and a group of star-forming regions cut out of a
cosmological simulation. The refinement must resolve the ionization front, and with it the thin layer at its outer edge
in which the low-ionization lines form. The main findings are:

- Without refinement, the front lies in coarse cells that mix ionized and neutral gas. These cells overestimate the
  ionized hydrogen by about half, and make the [N II], [S II], and [O I] lines several times too strong, [O I] by more
  than an order of magnitude.
- The refined grids agree with those of the reference implementation.
- The column density limit sets the resolution of the front, and thus the accuracy of the low-ionization lines.
  Hydrogen, the Balmer lines, and [O III] are converged at the limit of the reference setup; the low-ionization lines
  are not.
- The refinement settles by itself. Monte Carlo noise adds only a tail of small rounds at the outer edge of the front,
  which the candidate guard ends.
- The number of rounds depends on how far the initial grid is from the final one, and an averaging window of two
  iterations suffices.

### Test cases

**Single cluster.** A point source at the center of a uniform cube of 16.75 pc emits 2.0·10^50 ionizing photons per
second into gas with a hydrogen density of 339 cm^−3 and a metallicity of 0.0044, described by a
`DiffuseIonizedGasMix`, and mixed with dust. The dust absorbs a large part of the ionizing photons, so that the front
lies at about 2.75 pc, well inside the dust-free Strömgren radius of 3.5–3.8 pc. The simulation iterates over primary
emission, with 3·10^7 photon packets per iteration at the end of a ramp. The initial grid is an octree at level 3,
refined by a `ResolvedSpheresTreePolicy` around a sphere of 1 pc out to twice its radius, with 3 872 cells, so that the
front starts in cells of 2.09 pc.

**Star-forming group.** A cube of 263 pc cut out of a halo of the MEGATRON simulation holds 213 star particles, and gas
and dust imported from an adaptive mesh, with a metallicity, a temperature, and abundances per cell. The simulation
includes dust and gas emission and iterates over primary and secondary emission; the refinement takes place in the
primary loop. The initial grid follows the imported mesh and is refined around 83 seeded spheres, with 96 111 cells.

**Criteria.** Both examples use the criteria of the reference setup: a `MediumStateValueCriterion` that flags cells
with an N+ fraction `x_NII` of at least 0.01 and a `MediumStateGradientCriterion` on `x_NII` with a `maxChange` of 0.3,
both restricted to partly ionized gas (`x_HI` between 0.01 and 0.99, `logU` at least −6.5), and a
`MediumStateGradientCriterion` on the O2+ fraction `x_OIII` with a `maxChange` of 0.3 and at most two extra levels,
restricted by the same bound on `logU`. All three have the same column density limit, 1.6·10^20 cm^−2 in the single
cluster and 2.56·10^21 cm^−2 in the star-forming group. The first decision follows six initial iterations, each
decision averages two or four iterations, and the iteration after each round is left out. The single cluster is
compared on four grids: the initial grid without refinement, the grids refined with the limit of the reference setup
(with both windows), and the grid refined with half that limit, which allows one more level in this uniform medium.

### The ionization front

Coarse cells at the front hold ionized and neutral gas in a single partly ionized state, with a diluted electron
density that lowers the recombination rate. They therefore overestimate the ionized volume, and with it the Balmer
lines, and they emit the low-ionization lines throughout their volume. On the initial grid of the single cluster, the
front lies in cells of 2.09 pc that end up about 70% neutral:

| Grid | Cells | Ionized&nbsp;H | Hβ | [O&nbsp;III]/Hβ | [O&nbsp;II]/Hβ | [N&nbsp;II]/Hα | [S&nbsp;II]/Hα | [O&nbsp;I]/Hα |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| no refinement | 3&nbsp;872 | 12.55 | 7&nbsp;440 | 5.77 | 0.88 | 0.114 | 0.392 | 0.137 |
| limit 1.6·10^20 | 29&nbsp;408 | 8.48 | 6&nbsp;671 | 8.47 | 0.608 | 0.0439 | 0.0768 | 0.0093 |
| limit 8·10^19 | 140&nbsp;631 | 8.48 | 6&nbsp;676 | 8.65 | 0.562 | 0.0374 | 0.0651 | 0.0062 |

The initial grid, and the grids refined with a column density limit in cm^−2 and a window of two iterations. Ionized
hydrogen in 10^59 atoms, Hβ in solar luminosities, and ratios of the emitted luminosities of [O III] 5007,
[O II] 3726+3729, [N II] 6584, [S II] 6716+6731, and [O I] 6300.

The refinement lowers the ionized hydrogen by a third, and brings the low-ionization lines down to values typical of
H II regions. At the limit of the reference setup, with cells of 0.131 pc at the front, hydrogen is resolved: halving
the limit changes the ionized hydrogen and Hβ by less than 0.1%, and [O III]/Hβ by 2%. The low-ionization lines are not
resolved, because the layer in which they form is still much thinner than these cells: halving the limit, to cells of
0.065 pc, lowers [O II]/Hβ by 8%, [N II]/Hα and [S II]/Hα by 15%, and [O I]/Hα by a third, with no sign of convergence
yet. Each further halving takes about four times as many cells.

> **Placeholder:** results of the run on the grid refined with half the limit, with four times as many photon packets
> (running), which shows whether the line ratios at this resolution still change with the number of photon packets.

The results depend on the final grid only, not on the rounds that led to it: a run on the grid refined with half the
limit, read back with the `TopologyTreePolicy` and iterated without refinement, reproduces the ionized hydrogen, Hβ,
and all line ratios within 0.1%.

![Ionized hydrogen fraction and cells around the front of the single cluster](figures/results-photoionization.png ':size=100%')

The figure shows the ionized hydrogen fraction and the cells in one quadrant of the plane through the source of the
single cluster, on the three grids of the table. The dashed line is the seeded sphere, which the initial grid resolves
out to twice its radius.

In the star-forming group, the refinement lowers the ionized hydrogen by 3.5%. The refined grids of both examples agree
with those of the reference implementation at the fronts: in the star-forming group nearly cell for cell, and in the
single cluster in the position of the front and the size of its cells.

### Rounds and iterations

A round subdivides each candidate by one level, so that the number of rounds depends on how far the initial grid is
from the final one. In the star-forming group, the seeding has already placed fine cells at most fronts: the first
round adds 8 057 cells to the 96 111 of the initial grid, and four more rounds add 266. In the single cluster, the front
starts four levels above the limit, and the refined band works its way outward through the front, so that the
refinement takes ten or eleven rounds at the limit of the reference setup, and 22 at half that limit. After each round,
the children start from the partly ionized state of their parent, and the ionized hydrogen drops and recovers within a
few iterations.

The averaging window mainly delays the rounds. With two iterations instead of four, the single cluster ends with the
same grid (29 408 against 28 932 cells) and the same results within 0.6% ([O I]/Hα within 2%), and the loop converges
after 41 instead of 60 iterations. The 40 iterations of the reference setup therefore do not suffice for a window of
four.

At half the limit, the refinement settles by itself, but its last seven rounds split only 2 to 19 cells each. These
cells lie at the outer edge of the front, where the ionizing radiation is weak and noisy: their window-averaged `logU`
crossed the bound of −6.5 of the conditions by noise, whereas the median cell at that radius has a `logU` of −8.3. The
gas had converged by iteration 50, but the loop waited for these rounds until iteration 74. The noise test of the
gradient criteria cannot prevent this tail, because it does not apply to conditions, and because the ionization
fractions of the `DiffuseIonizedGasMix` have no noise estimate (see Noise of medium state variables in the Open
questions). The candidate guard ends it: with a `minCandidateFraction` of 10^−3, the refinement counted as settled at
iteration 50 with 44 candidates left unsplit, and the results agree with those of the full refinement within 0.3%.

### Recommended settings

For H II regions, tested on the single cluster:

| Aspect | Setting |
| --- | --- |
| Initial grid | A `ResolvedSpheresTreePolicy` whose spheres cover the expected fronts, so that the refinement starts close to its final grid and needs fewer rounds. Dust and density structure can move a front well inside the dust-free Strömgren radius, to about three quarters of it in the single cluster. |
| Criteria | The three criteria of the reference setup, with a column density limit chosen for the lines of interest. A limit of 1.6·10^20 cm^−2 (counting hydrogen and helium nuclei) suffices for hydrogen, the Balmer lines, and [O III]. The low-ionization lines need a limit below 8·10^19 cm^−2, at about four times as many cells for each halving. |
| Iterations | Six initial iterations, a window of two iterations, and one iteration left out after each round (`numIterationsBeforeRefinement` 6, `numAveragedIterations` 2, `numIterationsAfterRefinement` 1), and a `minCandidateFraction` of 10^−3. A generous `maxPrimaryIterations`, such as 100: the loop ends as soon as the medium state has converged and the refinement has settled, whereas a maximum that is too low cuts the refinement short. |
