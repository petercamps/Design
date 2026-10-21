# Next steps

The project proceeds in steps. The first steps can be done on a local workstation, with a reduced
number of photon packets and a coarser wavelength grid where needed. They establish the model
setup and provide the measurements needed to plan the production runs, which are performed on a
high-performance computing facility.

## 1. Reproduce the geometry

- Obtain the clump catalogs used for UXClumpy, or the scripts that generate them, from the XARS
  repository or from its author, for each combination of `TORsigma` and `CTKcover`.
- Convert each catalog to the text column format read by both the `ParticleMedium` (with the
  `UniformSmoothingKernel`) and the `ClumpySphericalSpatialGrid`: the center position and radius of
  each clump, followed by its density or mass. The spatial domain must extend to a radius of at
  least 1.1 R<sub>out</sub>, because the UXClumpy placement procedure only requires the cloud
  centers to lie within R<sub>out</sub>, so that some clouds extend beyond it (see Reference model).
- Verify that the clumps are fully inside the spatial domain and do not overlap. The grid removes
  offending clumps, while the medium keeps their mass, which would distort the densities. If the
  catalogs contain overlapping clumps, a strategy is needed, such as shrinking, merging, or
  removing clumps while conserving the covering factor. The UXClumpy placement procedure rejects
  overlapping clouds with a small margin, so that its catalogs should already satisfy the
  requirement of the grid.
- Configure the uniform background medium between the clumps, with a polar mesh of the structured
  grid that matches any angular boundaries of the background geometry.
- Compare the column density maps as seen from the center, calculated by SKIRT (for example with a
  probe or a set of instruments), with the maps published for UXClumpy.

## 2. Set up the physics

- Configure the X-ray gas mix with the abundances used by UXClumpy, and the source with the
  UXClumpy incident spectrum at the center.
- Run a first simulation with a few viewing directions and a modest resolution, to verify the
  setup and measure the run time and memory use.
- Record the direct and scattered flux components separately. The direct component is the
  incident spectrum attenuated along the line of sight, which may allow a separate, noise-free
  treatment (see Open questions below).

## 3. Validate against UXClumpy

- Run SKIRT with its physics reduced to match XARS as closely as possible: free-electron Compton
  scattering, the XARS set of fluorescent lines, and no intrinsic line shapes.
- Compare the resulting spectra with the UXClumpy table for a set of parameter values, and explain
  any differences.
- Repeat with the full SKIRT physics, to document the effect of bound-electron scattering, the
  additional lines, and the line shapes.

## 4. Scope the production runs

The number of simulations, their cost, and the size of the resulting table follow from a few
decisions, which are listed under Open questions below:

- how the incident spectrum parameters `PhoIndex` and `Ecut` are covered;
- how the viewing parameters `NHLOS` and `Theta_inc` are covered, and with how many viewing
  directions;
- the energy range and resolution of the spectra;
- the number of photon packets needed to suppress Monte Carlo noise at 0.5 eV resolution.

The scoping runs measure how the run time scales with the number of viewing directions, the number
of wavelength bins, and the number of photon packets. From these measurements, the total cost of
the production runs can be estimated. Two routes to reduce the total cost are pursued in parallel:

- **Fewer simulations.** If the incident spectrum parameters can be derived from a single set of
  simulations per geometry, as with the response matrices of XARS, only the 20 geometries of the
  UXClumpy grid require separate simulations, and the total cost is a few tens of days of
  computing.
- **Shorter simulations.** If each combination of geometry and incident spectrum requires its own
  simulation, the UXClumpy grid amounts to 4 × 5 × 11 × 5 = 1100 simulations. At about a day each,
  this is feasible on a high-performance computing facility only if the run time of each
  simulation can be reduced substantially.

The two routes can also be combined.

## 5. Production runs

- Prepare the ski files for all parameter combinations, with scripts that generate them from a
  single template.
- Run the simulations on a high-performance computing facility, in parallel over the parameter
  combinations, possibly combined with the hybrid parallelization of SKIRT within each simulation.
- Verify each run for completeness and noise level.

## 6. Build and test the table model

- Assemble the spectra into a table model in the format expected by XSPEC, including the
  equivalent of the angle-averaged omni component if needed.
- None of the four UXClumpy table files requires separate simulations. The transmitted and
  reflected components correspond to the direct and scattered flux components that the instruments
  record when `recordComponents` is enabled; in SKIRT, the fluorescent lines are part of the
  scattered component. The omni component follows from the spectra of all instruments, averaged
  over the sphere with weights for their solid angles, which requires instrument directions that
  cover the whole sphere.
- Test the table model in XSPEC, in particular the interpolation between parameter values and the
  loading time, given the size of the table.
- Compare fits of the new model and UXClumpy on archival X-ray spectra.

## 7. Apply the model

- Fit the model to XRISM/Resolve data of a nearby, obscured AGN.
- Explore self-consistent infrared and X-ray models, using the multi-wavelength capabilities of
  SKIRT.

## Open questions

**Incident spectrum parameters.** XARS covers `PhoIndex` and `Ecut` without extra simulations,
through response matrices. SKIRT emits photon packets from a given source spectrum, and its
instruments do not record the energy at which a packet was emitted. The options are:

- one simulation for each combination of incident spectrum parameters, which multiplies the number
  of simulations by the number of such combinations and is probably not feasible;
- a set of simulations, each with a source that emits only in one injection energy band, forming a
  basis from which the spectrum for any incident spectrum is obtained as a weighted sum. The
  number of bands must be limited, so the incident spectrum is represented only approximately
  within each band;
- a hybrid approach, in which the direct component is obtained exactly by applying the
  line-of-sight transmission to any incident spectrum, and only the scattered component, including
  the fluorescent lines, is obtained from a basis of injection bands. The scattered continuum is
  smooth, and the line strengths depend on the incident spectrum integrated above the absorption
  edges, so a modest number of bands may suffice.

Each option needs to be tested for accuracy and cost. Recording the injection energy band of each
detected photon packet in a single simulation would make the basis approach much cheaper, but would
require a change to SKIRT.

**Run time.** Independently of the number of simulations, the run time of each simulation should be
reduced as much as possible. Possible levers include the number of photon packets and the
wavelength bias of the source, the number of instruments and wavelength bins, the treatment of
the direct component, the configuration of the physics, and the parallelization. The scoping runs
must measure how the run time depends on each of these, and how much each can be reduced without
compromising the accuracy or the noise level of the spectra.

With forced scattering, the `minWeightReduction` property of the photon packet options is a
further lever. A photon packet is terminated once its weight has been reduced by this factor
(10<sup>4</sup> by default, 10<sup>3</sup> at least), so that a lower value shortens the life
cycles of packets that scatter many times in the Compton-thick clumps. These packets contribute to
the reflected component and the Compton hump, so that the effect on those parts of the spectra
must be checked. Path length stretching, which helps in regions of high optical depth, is not
available, because the photon packet wavelength changes in Compton scattering.

**Viewing directions.** XARS records escaping photons in sky bins, so that a single simulation
yields spectra for all directions. SKIRT records the flux for a limited set of instruments through
peel-off, with a cost per interaction that grows with the number of lines of sight (instruments with
the same line of sight share a single peel-off photon packet). The spectra for bins of similar
line-of-sight column density can be obtained by grouping the spectra of many instruments, but the
number of directions needed to sample the column density distribution, and the resulting cost, must
be determined. The line-of-sight column density of each direction can be calculated from the
geometry.

**Energy range and resolution.** A resolution of 0.5 eV over the XRISM/Resolve band, roughly 0.3 to
12 keV, amounts to more than 20 000 bins. The Compton hump near 20 keV and data from hard X-ray
instruments require a wider range, possibly at a coarser resolution. The source must in any case
emit up to a few hundred keV, because Compton scattering moves photons to lower energies.

**High-energy limits in SKIRT.** The UXClumpy source spectra extend to 1517.5 keV (see Reference
model), but SKIRT currently limits the energy range in two ways. Wavelengths must be at least
1 pm, which corresponds to 1239.8 keV, so that higher energies cannot be configured at all.
Furthermore, the `XRayAtomicGasMix` sets all cross sections to zero above 500 keV, so that photons
with higher energies cross the medium without interacting. The pilot run is therefore capped at
500 keV (see SKIRT pilot run), which omits only 0.9% of the energy of its source spectrum. For the
hardest source spectra of the UXClumpy grid, with Γ = 1 and `Ecut` = 400 keV, however, the photons
above 500 keV carry 27% of the energy, and those above 1239.8 keV still 2%. After scattering to
lower energies, these photons contribute to the reflected component and the Compton hump. Production
runs that cover the full parameter range of UXClumpy therefore require extending SKIRT: lowering
the 1 pm limit, and extending the X-ray material mix to about 1.5 MeV, where Compton scattering on
free electrons, with the Klein-Nishina cross section, is by far the dominant process.
Alternatively, the energy range and the incident spectrum parameters of the new model could be
restricted.

**Monte Carlo noise.** The spectra must be free of visible noise at 0.5 eV resolution, in each sky
bin. The number of photon packets needed, and techniques that reduce the noise, such as a separate
treatment of the direct component or a suitable wavelength bias of the source, must be determined
in the scoping runs.

**Rayleigh peel-off noise.** In the pilot run, rare Rayleigh scattering events, whose peel-off picks
up the narrow forward peak of the phase function, cause most of the bins between about 7 and
100 keV in which the VOV exceeds its threshold (see Reliability in the SKIRT pilot run chapter).
Possible remedies:

- **More photon packets.** This works, but slowly, because the VOV of such a heavy-tailed
  distribution of contributions decreases more slowly than 1 / N.
- **Free-electron scattering.** Replacing bound-electron by free-electron scattering removes the tail
  and changes the fluxes by less than 1% above 10 keV, but it also removes Rayleigh scattering and
  the binding effects on Compton scattering, which matter at lower energies (3.6% at 7 to 10 keV in
  the diagnostic runs). Bound electrons at low energies combined with free electrons at high energies
  would require a change to SKIRT.
- **Delta-scaling.** The narrow forward peak could be removed from the Rayleigh phase function,
  together with the corresponding fraction of the cross section, so that nearly forward Rayleigh
  scattering is treated as no scattering at all. Because such scattering hardly changes the
  direction and does not change the energy, the spectra should remain nearly unchanged, while the
  large peel-off contributions disappear. This also requires a change to SKIRT.
- **Relaxed criterion.** Because R and the fluxes converge as expected, the VOV threshold could be
  relaxed for the scattered light above 7 keV, accepting occasional deviations of a few percent in
  single bins.

**Table size.** With the parameter grid of UXClumpy, the table contains 135 300 spectra. A
resolution of 0.5 eV from 0.3 to 12 keV amounts to about 23 400 energy bins, compared with 1000 in
UXClumpy, so that a table in single precision takes about 13 GB, compared with about 540 MB for
UXClumpy. A wider energy range, or a finer parameter grid, increases the size further. Whether
XSPEC can handle such a table efficiently, or whether it must be split, for example by geometry
parameters or energy range, must be tested.

**Background medium.** The density of the uniform background medium is not constrained by
observations. It could be fixed at a low value, set to zero, or become an additional parameter,
which would multiply the number of simulations. Whether UXClumpy includes such a medium is to be
verified.

**Clump realizations.** UXClumpy uses a particular random realization of the clump distribution
for each geometry. Using the same realizations allows a direct comparison. Using several
realizations would show how much the spectra depend on the particular realization, at additional
cost.

**Omni component.** The angle-averaged omni component follows from the same simulations, as the
solid-angle weighted average of the spectra of all instruments (see step 6 above).
This requires instrument directions that cover the whole sphere, which constrains the choice of
viewing directions. Whether the new model includes this component, and whether it keeps the
redundant `NHLOS` and `Theta_inc` parameters for compatibility with UXClumpy, is to be decided.
