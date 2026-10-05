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

## Relation to non-LTE line transfer

The `NonLTELineGasMix` calculates the level populations of selected atoms and molecules from the
radiation field, and iterates over secondary emission to make the populations, the line emission,
and the radiation field consistent. It is often combined with dust, and the two interact: the dust
emission, including its self-absorption, sets the infrared and submillimeter radiation field that
excites many of the supported transitions. The dust proposal should not stand in the way of
accelerating the line iterations later.

### Current state

The level populations of each cell are updated after each secondary emission segment, or at the end
of each merged iteration, by solving the statistical equilibrium equations. These use the mean
intensity of each transition, obtained by integrating the binned radiation field over the line
profile. The loop has converged for this mix when the fraction of cells whose populations still
change by more than `maxChangeInLevelPopulations` is at most `maxFractionNotConvergedCells`, and the
populations summed over the domain change by at most `maxChangeInGlobalLevelPopulations`. The latter
relies on the aggregate cells of the medium state, which become aggregate series in the iteration
history.

The mix offers no acceleration of the iterations. Its only lever is the initial state:
`initialLevelPopsCase` selects LTE populations, the collisionally excited solution, or custom
populations read from a file, for example written by a `CustomStateProbe` in a previous run. Like
dust self-absorption, the transfer in optically thick lines is a Λ-iteration: a line photon is
reabsorbed close to where it was emitted, so that each iteration propagates the excitation only a
short distance, and the convergence slows down as the line optical depth increases.

### Compatibility

The dust proposal can be implemented first. The points of contact, and how the proposal handles
them, are:

- **Gas emission disables the rebalance.** The mix emits lines, so the rebalance and the deficit
  criterion are disabled in every simulation that includes it. The dust emission then converges by
  the secondary change criterion, and exact absorption still applies. As for photoionization, this
  is a limitation, not an incompatibility: extending the tallies to the dust absorption of gas
  emission (see Gas emission above) is purely additive.
- **Rescaling the secondary field.** The rebalance rescales the stable secondary radiation field of
  each cell, which holds the radiation of all secondary sources. Without gas emission, this is the
  dust emission only. If the rebalance is extended to models with line emission, the rescaling must
  not scale the line radiation along with the dust emission. This requires either separate
  secondary tables for dust and gas emission, or a rebalance factor that applies to the dust
  contribution only. Neither is part of this proposal, and neither is excluded by it.
- **One-way coupling.** The dust emission excites the lines, but the lines carry a negligible
  fraction of the luminosity absorbed by the dust, and their opacity hardly affects the dust heating
  in most models. Converging the dust faster therefore helps the line iteration, because the level
  populations see a settled continuum sooner, without destabilizing it.
- **Order in the loop.** In both loops, the dynamic medium state, and thus the level populations,
  is updated from the measured radiation field before `endRebalance()` rescales the secondary
  field. The populations therefore see the rebalanced dust emission one iteration later, and their
  iteration does not depend on the dust acceleration.
- **The merged loop correction** changes the dust emission of each merged iteration, and thus the
  continuum that excites the lines. It changes the results of non-LTE models with merged iterations
  and substantial dust self-absorption, including functional tests.
- **Exact absorption** changes only the luminosity absorbed by dust. The mean intensities used for
  the level populations are still obtained from the binned radiation field.
- **The emission cell.** The tallies recognize dust emission by an emission cell other than -1,
  because only `DustSecondarySource` sets it. An accelerator for line transfer (see below) may need
  the emission cell of line photon packets as well. The tallies would then need another way to tell
  dust emission apart, such as a flag for the type of the emitting source. Keeping this in mind
  prevents the emission cell from becoming tied to dust.
- **Separate convergence tests.** The dust criteria are evaluated by the simulation, and the level
  population criteria by the material mix. The loop combines them, and the iteration history keeps
  their series apart through their keys.
- **The per-path storage loop** is the natural place for any future per-path tally for lines, as
  for photoionization.

### Possible techniques

The regional rebalance does not carry over directly: the line opacity depends on the level
populations being solved for, including negative opacities in case of population inversion, so
that transfer fractions measured in one iteration do not remain valid at the fixed point. Other
techniques are established in line transfer codes:

- **Ng acceleration** extrapolates the level populations of each cell from the last three iterates.
  It is a special case of the Anderson acceleration discussed for photoionization, with the same
  requirements: a per-cell history type in the iteration history, holding a few iterates of the
  populations of each cell, and safeguards against negative populations. It also shares the same
  weakness, Monte Carlo noise.
- **Separating the local contribution.** Accelerated Λ iteration treats the radiation that a cell
  emits and reabsorbs itself implicitly, in the statistical equilibrium of that cell, so that the
  iteration only has to propagate the exchange between cells. In SKIRT, this would require a
  per-cell, per-transition tally of the line radiation that a cell absorbs from its own emission,
  the line analogue of the diagonal of the rebalance transfer matrix, which needs the emission cell
  of line photon packets.
- **Exact line intensities.** The mean intensity of each transition is obtained from the binned
  radiation field, which therefore must resolve every line profile; the `errorForGaussianIntegral`
  property exists because this is not always the case. A per-path tally that adds the luminosity
  times path length, weighted by the line profile at the photon packet's wavelength in the frame of
  the cell, would remove this discretization error, in the same way as exact absorption does for
  dust, and would relax the requirements on the radiation field wavelength grid. The per-path
  storage loop makes it easy to add.
- **A better initial state.** The existing options for the initial populations already provide a
  restart from a previous solution. Because of the one-way coupling, a staged approach that first
  converges the dust emission and only then starts updating the level populations may also save
  iterations.

As for photoionization, the convergence behavior of representative non-LTE runs should be measured
with the history probe before choosing a technique.
