# Introduction

> Status: implemented
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md),
> [Iteration history](iteration-history/01-introduction.md)

This note speaks of dust self-heating rather than dust self-absorption, the term used so far. The
photon packets emitted by dust are attenuated by the dust they cross in every simulation, so that
self-absorption is always taken into account in the observed emission. What the iterations add is
the heating of the dust by the energy it absorbs from its own emission.

## Motivation

In a simulation with dust emission, the dust absorbs part of its own emission, and is heated by
it. SKIRT calculates this self-heating by iterating over secondary emission: each iteration
launches dust emission based on the radiation field of the previous iteration, and records a new
secondary radiation field. This is a Λ-iteration. Its error shrinks per iteration by a factor
close to the fraction of the dust emission that is reabsorbed by the dust. For a galaxy that
reabsorbs 95% of its dust emission, each iteration removes only 5% of the remaining error.

Such galaxies exist. The compact, dusty galaxies at redshift 4 in the COLIBRE simulations reach
optical depths above 1000 in the V band. Runs of 158 of these galaxies needed 25 to 60 or more
secondary emission iterations, up to 37 times the cost of the primary emission, and 17 of them had
not converged after 60 iterations. For the most extreme galaxy tested, plain iteration would need
many hundreds of iterations to bring the escaping dust luminosity within 1% of its converged value.

Two further problems surfaced while investigating this:

- **The convergence criterion is misleading.** The `maxFractionOfPrevious` criterion monitors the
  change of the dust-absorbed secondary luminosity between consecutive iterations. When the secondary
  luminosity is many times the escaping primary luminosity, a change of 1% per iteration can hide
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
  transport (see References), adapted to SKIRT. It works on luminosities only, so it is independent of the emission
  model, including stochastic heating and cell libraries.
- **New convergence criteria.** The secondary emission loop converges when the escaping dust
  luminosity balances the absorbed primary luminosity, and the absorbed secondary luminosity has
  become stationary, both evaluated over a few iterations to suppress Monte Carlo noise. The
  criterion based on the change relative to the previous iteration is removed.

## Scope and assumptions

- The note assumes that the changes proposed in the [SKIRT 10](skirt-10/01-introduction.md)
  design note have been implemented, including the incompatible ski file changes it allows. The
  proposal makes some incompatible changes of its own, listed in the Features chapter.
- The note also assumes that the central iteration history described in the
  [Iteration history](iteration-history/01-introduction.md) design note has been implemented,
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

- **[Features](dust-self-heating/02-features.md)** describes the configuration, the behavior
  of the iteration loop, the logging and probing, the incompatibilities, and the recommended
  settings, as seen by the user.
- **[Implementation](dust-self-heating/03-implementation.md)** describes the absorption tally,
  the rebalance, the convergence test, and the changes to existing classes.
- **[Results](dust-self-heating/04-results.md)** summarizes some measurements made with the
  implementation for the three COLIBRE galaxies that triggered this proposal.
- **[Open questions](dust-self-heating/05-open-questions.md)** collects the design decisions that
  may need further consideration and the relation to photoionization and to non-LTE line transfer.

## References

Coarse-mesh rebalance in neutron transport:

- Asaoka, T., et al. 1976, Nucl. Sci. Eng., 59, 326:
  [Application of coarse-mesh rebalance acceleration to Monte Carlo eigenvalue
  problems](https://ui.adsabs.harvard.edu/abs/1976NSE....59..326A/abstract), an early application
  of the method to Monte Carlo transport.
- Cefus, G. R., & Larsen, E. W. 1990, Nucl. Sci. Eng., 105, 31:
  [Stability analysis of coarse-mesh
  rebalance](https://ui.adsabs.harvard.edu/abs/1990NSE...105...31C/abstract).
- Yamamoto, A. 2005, Nucl. Sci. Eng., 151, 274:
  [Generalized coarse-mesh rebalance method for acceleration of neutron transport
  calculations](https://ui.adsabs.harvard.edu/abs/2005NSE...151..274Y/abstract), which unifies
  coarse-mesh rebalance and coarse-mesh finite difference acceleration in a single scheme.
- Lee, M. J., et al. 2014, Ann. Nucl. Energy, 65, 101:
  [Coarse mesh finite difference formulation for accelerated Monte Carlo eigenvalue
  calculation](https://ui.adsabs.harvard.edu/abs/2014AnNuE..65..101L/abstract), a more recent
  variant for Monte Carlo transport, in which the coarse-mesh solution adjusts the source
  distribution through the weights of the particles.
- Wolters, E. R., Larsen, E. W., & Martin, W. R. 2013, Nucl. Sci. Eng., 174, 286:
  [Hybrid Monte Carlo-CMFD methods for accelerating fission source
  convergence](https://ui.adsabs.harvard.edu/abs/2013NSE...174..286W/abstract), which improves the
  efficiency of this method by changing how the Monte Carlo tallies enter the coarse-mesh
  equations.
