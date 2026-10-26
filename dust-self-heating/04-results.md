# Results

## The COLIBRE galaxies

The proposal was implemented in the SKIRT 10 beta in October 2026. This section summarizes the
measurements made with that implementation, which complement the prototype results in the
Introduction. All of them ran on an Intel laptop with 20 threads, unless stated otherwise.

### Cost of exact absorption

A timing test compared builds just before and after the implementation, three alternating runs each,
using a timing simulation of the galaxy ID39321 with seven dust components, stochastic heating, the
`TemperatureWavelengthCellLibrary`, and a single secondary emission iteration. Exact absorption
makes the primary emission 22% slower (109 instead of 89 s) and a secondary emission iteration 14%
slower (86 instead of 75 s), against 16% and 13% for the prototype. The final secondary emission,
which stores no radiation field, is not slower. Simulations that store the radiation field without
dust emission, and the setup, are 1–3% slower, which is probably a general difference between the
two builds rather than a cost of the restructured storage loop. The first iteration of a loop is not
rebalanced, so this test does not include the tallies of the rebalance; in the full runs below, a
rebalanced iteration takes 1–4% longer than the first one, which bounds their cost.

### Convergence for the COLIBRE galaxies

The three galaxies at redshift 4 that Andrea Gebek simulated with SKIRT 9 were run again with his
input files, spatial grid, and 31.6 million photon packets, with the same settings for all three: at
most 60 secondary iterations, the default convergence criteria and rebalance, and the
`TemperatureWavelengthCellLibrary` with 100 × 50 entries instead of the `AllCellsLibrary` of his
runs:

| Galaxy | Iterations | Run time | Time per iteration | Absorbed secondary / primary | Reabsorbed fraction | Deficit below 1% from iteration |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| ID33341 | 17 | 28 min | 86 s | 6.1 | 86.0% | 8 |
| ID5642 | 21 | 30 min | 77 s | 9.4 | 90.4% | 8 |
| ID39321 | 26 | 31 min | 65 s | 34.3 | 97.1% | 11 (for good from 15) |

The rebalance behaves as in the experiments. The first rebalance multiplies the absorbed secondary
luminosity by a factor of 2.4 to 4.3, the deficit then falls below 1% within 8 to 15 iterations, and
all rebalances used between 55 and 100 regions without any fallback to the global factor. As
expected, the secondary change is the last criterion to be met: after the deficit has become small,
the absorbed secondary luminosity still grows by 9 to 21% while the radiation trapped in the core
builds up, and all three runs end when its mean change falls below 0.5% per iteration, while it
still grows by 0.4 to 0.5% per iteration. Whether a stricter `maxSecondaryChange` would change the
observed spectrum noticeably remains to be tested (see Convergence thresholds in the Open questions
chapter).

A dusty sphere with a central source, which reabsorbs 84% of its dust emission, serves as a
functional test. With the rebalance, it converges in 7 iterations and reproduces a fully converged
plain iteration (28 iterations with five times more photon packets) within 0.03% in dust luminosity
and 0.16 K in temperature; merged iterations give the same result within 0.3%. The tally closure is
one within 0.003% (within 0.2% for a moving medium, consistent with its Doppler shifts).

Models with little self-heating need at least four iterations with the default
`numConvergenceIterations`, where the removed `maxFractionOfPrevious` criterion often stopped after
two. Excluding the first iteration from the secondary change (see Criteria) keeps this at four
rather than five.

### Comparison with the SKIRT 9 runs

Compared with Andrea Gebek's runs, which stopped after 60 plain iterations:

- The absorbed primary luminosity is 2.6%, 2.4%, and 6.9% higher for ID33341, ID5642, and ID39321.
  This is mostly the optical depth underflow described in the Implementation chapter, which lost
  part of the energy absorbed in the most opaque cells, less the small decrease from exact
  absorption.
- The spectrum below 5 µm agrees within 0.2% for all three galaxies, since the transport of the
  photon packets did not change.
- The spectrum between 5 and 1000 µm is 1.1%, 3.3%, and 19.5% brighter. For ID39321, his runs were
  still far from convergence after 60 iterations, with an absorbed secondary luminosity of 18
  instead of 34 times the absorbed primary luminosity, and they also lacked the 7% lost to the
  underflow. The other two galaxies self-heat less, so that 60 plain iterations came much closer.

### Background

The proposal is based on a series of experiments with quick and dirty code on top of SKIRT 9, using
two of the COLIBRE galaxies. The most extreme of these, called ID39321 in what follows, reabsorbs
about 96.5% of its dust emission in the converged state, and its secondary radiation field holds
about 28 times the absorbed primary luminosity. Its main results are:

| Configuration | Iterations for a 1% luminosity deficit | Converged dust emission |
| --- | --- | --- |
| plain iteration, 50 bins (current SKIRT) | many hundreds (13% deficit after 100) | 13% too bright |
| plain iteration, 200 bins | many hundreds | correct within 0.5% |
| regional rebalance, 50 bins, current absorption | stalls at a deficit of 2–4% | 13% too bright |
| regional rebalance, 200 bins, current absorption | about 11–15 | correct within 0.5% |
| regional rebalance, 50 bins, exact absorption | about 11–15 | correct within noise |
| regional rebalance, 200 bins, exact absorption | about 11–15 | reference |

With exact absorption, the simulations with 50 and 200 bins agree to 0.2% in dust luminosity, to
0.1 K in mass-weighted dust temperature, and within the noise of about 1% in each band between
3.6 and 500 µm. The bias of the current code therefore stems entirely from the absorbed luminosity,
not from the coarser spectral shape of the radiation field. The cost of exact absorption is 13% per
iteration and 16% for the primary emission, at the same grid. Compared with the 200-bin grid that
the current code needs for an unbiased result, exact absorption with 50 bins is 7% faster per
iteration and needs half the memory.

The experiments used the `TemperatureWavelengthCellLibrary` with 100 × 50 entries, which makes
an iteration about 25 times faster than calculating the emission spectrum of each cell, without a
measurable effect on the emitted spectrum. The library and the rebalance are complementary: the
library makes each iteration cheap, and the rebalance reduces the number of iterations.

The experiments, and the SKIRT 9 runs by Andrea Gebek on which they built, were affected by an
underflow of the extinction factor in extremely opaque cells, which came to light during the
implementation (see Optical depth underflow in the Implementation chapter). For ID39321, the dust
absorbed 6.6% of the source luminosity too little, so that the absolute dust luminosities of these
runs are about 7% too low. The comparisons between the configurations above are not affected,
because all of them shared the loss.
