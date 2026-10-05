# Open questions

## Design decisions

**Number of regions.** The proposed default of 10 × 10 regions has been tested most extensively,
including with exact absorption. With 30 × 30 regions, the slow drift of the absorbed secondary
luminosity that remains after the deficit has converged was smaller, and the proposed criteria would
have stopped the loop after 20 instead of 24–26 iterations. In return, 40–50 of the 900 regions fell
back to the global factor in later iterations, without visible effect, and the tallies need 6.5 MB
per thread. A default of 20 × 20 is a plausible compromise, but it has not been tested.

**Convergence thresholds.** The proposed defaults stop ID39321 when the absorbed secondary luminosity
is within about 5% of its value after 40 iterations. The comparison of 100 and 900 regions suggests
that the dust emission spectrum is much less sensitive than the absorbed secondary luminosity, but
the spectrum at the iteration where the loop would stop has not been compared with the final one.
The experiments recorded the spectrum only at the end of each run. A few runs of the same model that
stop after different numbers of iterations would settle the default of `maxSecondaryChange`.

**A criterion on the spectrum.** A criterion could compare the emitted spectrum between iterations,
which is the quantity users care about. This requires peel-off during the iterations, which SKIRT
does not perform, or a bolometric proxy such as the luminosity escaping from each region, which the
escape tally of the rebalance provides. The proposal keeps the criteria global and simple.

**Gas emission.** The rebalance and the deficit criterion are disabled when the simulation also has
gas emission, because the dust then also absorbs radiation from gas, which the transfer matrix does
not describe and which the deficit would count as a shortfall. The tallies could be extended by
treating the dust absorption of gas emission like primary absorption, using the emission cell to
tell the two apart, since gas-emitted packets have emission cell -1. Whether this is worth the
effort depends on whether models with both strong dust self-absorption and gas emission arise.

**Absorption by other media.** If other media absorb part of the dust emission, the converged
deficit is not zero but equals the fraction of the dust emission absorbed by those media, relative
to the absorbed primary luminosity. In practice, gas and electrons absorb little at infrared
wavelengths, so this is ignored. If needed, a tally of the luminosity absorbed by other media could
correct the deficit.

**Rebalance in the merged loop.** In the merged loop, the dynamic medium state is updated with the
measured radiation field, before the rebalance rescales the secondary field. Rebalancing before the
update would let the medium state see the improved field one iteration earlier, but would make the
medium state, and its convergence test, depend on an extrapolated rather than a measured field. The
proposal makes this order a fixed rule rather than an implementation detail, because it keeps the
dust acceleration independent of the medium state iteration, including any future acceleration of
photoionization (see below). The rebalance has not been tested in the merged loop.

**Averaging the tallies.** The transfer fractions could be averaged over several iterations to
reduce their noise, using an exponentially weighted or running average. With 31.6 million packets,
the remaining noise in the deficit was about ±0.5% per iteration, without any sign of instability,
so the proposal does not average. Averaging would add memory for an extra transfer matrix and
complicates the partition, which changes from one iteration to the next.

**Region coordinates.** The regions are formed from the ratio of secondary to primary absorption
and the absorption per unit dust mass. Alternatives are the entries of the cell library, which
group cells by temperature and spectral shape, or a fixed spatial partition. Regions with an equal
number of cells were clearly slower in the experiments than regions with equal luminosity. The
library entries are too many (thousands) for a dense solve, and a spatial partition does not follow
the structure of the problem, so the proposal keeps the luminosity-based coordinates.

**Thread-local tallies.** The tally objects are associated with threads through the existing
`ThreadLocalMember<T>` template, because the parallel execution engine does not expose a thread
index. Adding a thread index to `Parallel` would allow a plain vector of tally objects indexed by
thread, which avoids the lookup in `local()`, but changes the interface of a central class for one
client.

**The DustAbsorptionPerCellProbe.** This probe writes the spectral absorbed luminosity of each cell
on the radiation field wavelength grid, calculated from the binned field. Its integral over
wavelength differs from the exact absorbed luminosity by the error that exact absorption removes. It
could rescale the spectrum of each cell to the exact total, or add a column with the exact total.
The proposal only documents the difference.

**Default cell library.** For strongly self-absorbing models, the `TemperatureWavelengthCellLibrary`
makes iterations about 25 times faster without a measurable effect on the spectrum. It could become
the default when the simulation iterates over secondary emission. This would change many functional
test results for a speed gain that matters only for large models, so the proposal only adds it to the
recommended settings.

**Final emission without the library.** An option could calculate the final secondary emission with
a different library, such as `AllCellsLibrary`, than the iterations. The experiments did not show a
library effect that would justify it, so the option is not proposed.

## Relation to photoionization

The photoionization iterations of the `DiffuseIonizedGasMix` also converge slowly. Accelerating
them is outside the scope of this note, but the dust proposal should not stand in the way, and some
of its ideas may carry over.

### Compatibility

The dust proposal can be implemented first, without the risk of a later incompatibility, because
the two problems barely interact:

- **Different loops and data.** The rebalance acts on the secondary radiation field in the
  secondary and merged loops. Photoionization iterates the primary dynamic medium state in the
  primary and merged loops. An accelerator for photoionization would act on the medium state, and
  would not touch the absorbed luminosity arrays, the emission cell, or the rebalancer.
- **Separate convergence tests.** The dust criteria are evaluated by the simulation, and the
  photoionization criteria by the material mix. The loop combines them, as today, and the iteration
  history keeps their series apart through their keys.
- **Weak physical coupling.** Dust emission does not ionize gas, and the ionization state does not
  change the dust opacity.

The points of contact, and how the proposal handles them, are:

- **The merged loop** is the only place where both iterate in the same iteration. The fixed order
  described above, with the medium state updated from the measured field before the rebalance,
  keeps them independent.
- **The merged loop correction** changes the results of photoionization models with merged
  iterations, because the secondary radiation field now persists between iterations. The effect on
  the ionization state is expected to be small, but functional test results change, which is why
  the correction is a separate step in the test plan.
- **Gas emission disables the rebalance.** Photoionization models usually include gas emission, so
  the dust rebalance does not run in models that combine both. This is a limitation, not an
  incompatibility: extending the tallies to dust absorption of gas emission (see Gas emission above)
  is purely additive.
- **The per-path storage loop.** Moving the loop that stores the radiation field into the medium
  system makes it the natural place for any future per-path tally, such as the number of ionizing
  photons absorbed per cell. The dust tallies must therefore plug into that loop without assuming
  that they are the only ones.

### Possible techniques

The regional rebalance works so well for dust because dust self-absorption is linear: the dust
opacity does not depend on the solution, so the transfer fractions measured in one iteration remain
valid at the fixed point. Photoionization is nonlinear, because the opacity depends on the neutral
fraction that is being solved for. What helps depends on the cause of the slow convergence:

- **Diffuse ionizing radiation.** If recombination radiation that ionizes nearby gas is iterated
  explicitly, rather than treated with the on-the-spot approximation, that part of the problem is a
  linear secondary emission loop like dust self-absorption, and the rebalance carries over almost
  unchanged.
- **Advancing ionization fronts.** Starting from a mostly neutral state, the radiation of each
  iteration is blocked at the first opaque layer, so a front advances only a limited depth per
  iteration. A rebalance with fixed transfer fractions does not address this. A photon count
  rebalance, enforcing that recombinations balance the absorbed ionizing photons, is the analogue of
  the global rebalance: it corrects the level but not the position of the front. A nonlinear
  version of the regional rebalance, in which the region-level system includes the ionization
  balance and is solved with Newton's method, exists in the neutron transport literature, but is a
  much larger project. A better initial state, fully ionized or estimated from the Strömgren radius
  of each source, may gain more than any accelerator, because the radiation then reaches all cells
  in the first iteration.
- **Anderson acceleration.** This is a generic accelerator for fixed-point iterations: it combines
  the last few iterates so as to minimize the residual, and is related to Ng acceleration, used in
  accelerated line transfer codes. It handles nonlinear problems, and the state per cell
  (ionization fractions and temperature) is small, so keeping two or three previous iterates is
  affordable; this would require a per-cell history type in the iteration history, next to the cell
  windows. It needs safeguards: clipping fractions to their valid range, damping, a regularized
  least-squares solve, and a restart when the residual grows. Its weakness is Monte Carlo noise,
  because the differences between iterates become dominated by noise near convergence. It works
  best when the error decays geometrically with a single dominant factor, and much less well for a
  moving front. For dust, it is not needed on top of the rebalance.
- **Binning bias.** The photoionization rates are calculated from the binned radiation field, like
  the dust absorption today, so they may carry a similar discretization error, in particular near
  ionization edges. If so, an exact per-path tally of the photoionization rates, analogous to exact
  dust absorption, would remove it, and the per-path storage loop would make it easy to add.

Before choosing a technique, the convergence behavior of representative photoionization runs should
be measured with the history probe: an error that decays geometrically points to Anderson
acceleration or extrapolation, a front that moves steadily outward points to a better initial state,
an oscillation points to damping, and a plateau points to Monte Carlo noise, to be addressed with more
photon packets or criteria averaged over several iterations.
