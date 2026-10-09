# Open questions

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

**Monte Carlo noise.** The spectra must be free of visible noise at 0.5 eV resolution, in each sky
bin. The number of photon packets needed, and techniques that reduce the noise, such as a separate
treatment of the direct component or a suitable wavelength bias of the source, must be determined
in the scoping runs.

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
solid-angle weighted average of the spectra of all instruments (see the Reference model chapter).
This requires instrument directions that cover the whole sphere, which constrains the choice of
viewing directions. Whether the new model includes this component, and whether it keeps the
redundant `NHLOS` and `Theta_inc` parameters for compatibility with UXClumpy, is to be decided.
