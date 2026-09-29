# Plan

The project proceeds in steps. The first steps can be done on a local workstation, with a reduced
number of photon packets and a coarser wavelength grid where needed. They establish the model
setup and provide the measurements needed to plan the production runs, which are performed on a
high-performance computing facility.

## 1. Reproduce the geometry

- Obtain the clump catalogs used for UXClumpy, or the scripts that generate them, from the XARS
  repository or from its author, for each combination of `TORsigma` and `CTKcover`.
- Convert each catalog to the text column format read by both the `ParticleMedium` (with the
  `UniformSmoothingKernel`) and the `ClumpySphericalSpatialGrid`: the center position and radius of
  each clump, followed by its density or mass.
- Verify that the clumps are fully inside the spatial domain and do not overlap. The grid removes
  offending clumps, while the medium keeps their mass, which would distort the densities. If the
  catalogs contain overlapping clumps, a strategy is needed, such as shrinking, merging, or
  removing clumps while conserving the covering factor.
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
  treatment (see Open questions).

## 3. Validate against UXClumpy

- Run SKIRT with its physics reduced to match XARS as closely as possible: free-electron Compton
  scattering, the XARS set of fluorescent lines, and no intrinsic line shapes.
- Compare the resulting spectra with the UXClumpy table for a set of parameter values, and explain
  any differences.
- Repeat with the full SKIRT physics, to document the effect of bound-electron scattering, the
  additional lines, and the line shapes.

## 4. Scope the production runs

The number of simulations, their cost, and the size of the resulting table follow from a few
decisions, which are listed in the Open questions chapter:

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
- Test the table model in XSPEC, in particular the interpolation between parameter values and the
  loading time, given the size of the table.
- Compare fits of the new model and UXClumpy on archival X-ray spectra.

## 7. Apply the model

- Fit the model to XRISM/Resolve data of a nearby, obscured AGN.
- Explore self-consistent infrared and X-ray models, using the multi-wavelength capabilities of
  SKIRT.
