# Introduction

> Status: under construction

## Motivation

In a simulation with dust emission, the dust absorbs part of its own emission. SKIRT handles this
self-absorption by iterating over secondary emission: each iteration launches dust emission based
on the radiation field of the previous iteration, and records a new secondary radiation field. This
is a Λ-iteration. Its error shrinks per iteration by a factor close to the fraction of the dust
emission that is reabsorbed by the dust. For a galaxy that reabsorbs 95% of its dust emission,
each iteration removes only 5% of the remaining error.

Such galaxies exist. The compact, dusty galaxies at redshift 4 in the COLIBRE simulations reach
optical depths above 1000 in the V band. Runs of 158 of these galaxies needed 25 to 60 or more
secondary emission iterations, up to 37 times the cost of the primary emission, and 17 of them had
not converged after 60 iterations. For the most extreme galaxy tested, plain iteration would need
many hundreds of iterations to bring the escaping dust luminosity within 1% of its converged value.

Two further problems surfaced while investigating this:

- **The convergence criterion is misleading.** The `maxFractionOfPrevious` criterion compares the
  change of the dust-absorbed secondary luminosity with the secondary luminosity itself. When the
  secondary luminosity is many times the primary luminosity, a change of 1% per iteration can hide
  an error of 15% or more in the escaping luminosity.
- **The absorbed luminosity is biased by the radiation field wavelength grid.** SKIRT calculates the
  luminosity absorbed by the dust in each cell from the binned radiation field, using the dust
  opacity at the characteristic wavelength of each bin, whereas photon packets are attenuated with
  the opacity at their exact wavelength. With a grid of 50 bins, the dust absorbs
  0.4–0.6% more energy than the packets lose, in every pass. In a strongly self-absorbing galaxy,
  this small excess is amplified roughly by the inverse of the escaping fraction, and the converged
  dust emission comes out 13% too bright, with the dust 3 K too warm.

## Proposal

This design note proposes three changes:

- **Exact absorption.** The luminosity absorbed by the dust in each cell is accumulated along the
  path of each photon packet, using the dust absorption opacity at the packet's exact wavelength.
  The binned radiation field is still used for everything that needs a spectrum, such as the shape
  of the dust emission spectrum, but no longer for the amount of absorbed energy. Energy is then
  conserved exactly, up to Monte Carlo noise, for any radiation field wavelength grid.
- **Regional rebalance.** After each secondary emission iteration, the cells are grouped into some
  100 regions, a small region-to-region transfer matrix measured during the iteration is solved for
  the self-consistent luminosity of each region, and the secondary radiation field is rescaled per
  region accordingly. This is coarse-mesh rebalance, a classic accelerator for Monte Carlo neutron
  transport, adapted to SKIRT. It works on luminosities only, so it is independent of the emission
  model, including stochastic heating and cell libraries.
- **New convergence criteria.** The secondary emission loop converges when the escaping dust
  luminosity balances the absorbed primary luminosity, and the absorbed secondary luminosity has
  become stationary, both evaluated over a few iterations to suppress Monte Carlo noise. The
  criterion based on the change relative to the previous iteration is removed.

## Background

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

## Scope and assumptions

- The note assumes that the changes proposed in the [SKIRT 10](skirt-10/01-introduction.md)
  design note have been implemented, including the incompatible ski file changes it allows. The
  proposal makes some incompatible changes of its own, listed in the Features chapter.
- The note also assumes that the central iteration history described in the
  [Convergence history](convergence-history/01-introduction.md) design note has been implemented,
  including the history probe. All convergence data of the secondary emission loop are kept in that
  history, and the log messages and the history probe are formed from the same series.
- Exact absorption applies to all simulations with dust emission, iterated or not. The regional
  rebalance and the new convergence criteria apply to simulations that iterate over secondary
  emission, with or without including primary emission in these iterations.
- Exact absorption changes only the luminosity absorbed by dust. The extinction of photon packets by
  all media, the radiation field used by gas, electrons, and other media, gas emission, and the
  dynamic medium state are not affected.
- The rebalance applies when dust emission is the only form of secondary emission (see the
  Features chapter).

## Overview

- **[Features](dust-self-absorption/02-features.md)** describes the configuration, the behavior
  of the iteration loop, the logging and probing, and the incompatibilities, as seen by the user.
- **[Implementation](dust-self-absorption/03-implementation.md)** describes the absorption tally,
  the rebalance, the convergence test, and the changes to existing classes.
- **[Open questions](dust-self-absorption/04-open-questions.md)** collects the design decisions that
  need input, and the departures from the experimental implementation.
