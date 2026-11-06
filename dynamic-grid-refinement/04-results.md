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
