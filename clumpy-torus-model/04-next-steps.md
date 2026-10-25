# Next steps

The pilot run has established the setup of the SKIRT simulations, the wavelength bias of the source, and
the reliability that can be reached with a given number of photon packets (see SKIRT pilot run). The
remaining work is listed below, starting with the issues that determine whether and how the production
runs can be done.

## Pre-production considerations

**Review.** Have the previous chapters reviewed by Bert Vander Meulen, and possibly by other candidate
co-authors.

**Geometries.** Find out whether the geometry files of the released UXClumpy geometries can be obtained
from the author of UXClumpy, who provided the geometry file used in the pilot run, and decide whether to
use these geometries or freshly regenerated ones.

- *Released geometries.* The spectra can be compared directly with the UXClumpy table, for the same clouds and
  sky fractions, and users can switch between the two models without a change of geometry. On the other hand,
  the parameter grid is that of UXClumpy, with a single random realization per geometry, whose particular
  structures cause irregular dependencies on the geometry parameters (see Example spectra in the Reference
  model chapter), and obtaining the files depends on the cooperation of the author.
- *Regenerated geometries.* These allow a different or finer parameter grid, and several realizations per
  geometry, which show how much the spectra depend on the particular realization. On the other hand, the
  recipe must be reproduced faithfully, which is uncertain for some details (see Open points in the Reference
  model chapter), and verified, for example through the column density distributions. Moreover, the comparison
  with UXClumpy becomes statistical rather than direct.

**Energetic end of the spectral range.** SKIRT currently does not allow wavelengths short of 1 pm (1239.8 keV),
and the `XRayAtomicGasMix` sets all cross sections to zero above 500 keV, whereas the UXClumpy source extends
to 1517.5 keV (see Central source in the Reference model chapter). The photons above 500 keV carry 0.9% of the
energy of the source spectrum of the pilot run, but 27% for the hardest source spectra of the UXClumpy grid,
with Γ = 1 and `Ecut` = 400 keV. The options are:

- Extend SKIRT: lower the 1 pm limit and extend the X-ray material mix to about 1.5 MeV, where Compton
  scattering on free electrons, with the Klein-Nishina cross section, is by far the dominant process. A
  simpler variant would treat the photons above 500 keV with free-electron scattering only.
- Keep the source capped at 500 keV, and restrict the new model accordingly: limit the incident spectrum
  parameters to spectra with little energy above 500 keV, or limit the energy range of the table to energies
  where the cap does not affect the reflected component (in the pilot run, the reflected component falls off
  above about 200 keV).

In either case, find out how far the source range must extend beyond the energy range of the detectors to
include the photons that Compton scattering moves into that range. After a single scattering, photons at
energy E stem from energies up to E / (1 − 2E / 511 keV), for E below 255.5 keV: up to 114 keV for the
upper limit of NuSTAR at 79 keV, and up to 824 keV for that of Swift/BAT at 195 keV. Multiple scattering
extends the range further, so that test runs with different upper limits of the source are needed.

**Omni spectrum.** Find a way to obtain the angle-averaged spectrum of all escaping photons (see Table files
in the Reference model chapter). There are two options:

- *Additional sight lines* that cover the whole sphere, so that the omni spectrum follows as the
  solid-angle weighted average of the spectra of all sight lines. This requires no change to SKIRT, but the
  cost grows with the number of sight lines, and the accuracy depends on how well they sample the angular
  variation of the reflected component.
- *An escape recorder*: a new SKIRT feature that records, at each step of a photon packet with forced
  scattering, the part of its luminosity that escapes along its direction of travel, a fraction e<sup>−τ</sup>
  of its weight that SKIRT already calculates. This yields the angle-integrated escaping spectrum exactly,
  with low noise and at negligible extra cost. Binned by escape direction, it would also yield spectra for
  sky bins as in XARS, which allow a like-for-like comparison with UXClumpy.

**Bound-electron scattering.** Decide which treatment of bound-electron scattering to use. The pilot run uses
the `Good` treatment (`scatterBoundElectrons`), with smooth Rayleigh scattering and exact bound-Compton
scattering. The `Exact` treatment differs only in Rayleigh scattering, for which it also uses the anomalous
scattering factors that describe the deviation from the smooth form-factor approximation near absorption edges.
The effect on the spectra is expected to be limited to energies near the absorption edges, and remains to be
quantified. Using `Exact` instead of `Good` has no performance cost: for the pilot configuration, both take the
same time per photon packet. Both treatments include the forward peak of the Rayleigh phase function that causes
the VOV tail (see Rayleigh peel-off noise below).

**Signal-to-noise level.** Decide whether R < 0.1 suffices, or whether the spectra must be essentially free
of noise. In the pilot run, R < 0.1 in 99% of the relevant bins for 90% of the spectra requires about
1.4 × 10<sup>8</sup> photon packets (see Reliability in the SKIRT pilot run chapter). Because R<sup>2</sup>
scales as 1 / N, R < 0.01 requires 100 times more packets, about 1.4 × 10<sup>10</sup>. An intermediate
criterion follows from the energy resolution of XRISM/Resolve, 4.5 eV FWHM, which spans about nine 0.5 eV
bins: R < 0.03 per bin amounts to about 0.01 per resolution element, and requires about 1.5 × 10<sup>9</sup>
packets. These estimates consider R only; the VOV above 7 keV requires more packets (see Rayleigh peel-off
noise below).

**Rayleigh peel-off noise.** Rare Rayleigh scattering events, whose peel-off picks up the narrow forward peak
of the phase function, cause most of the bins between about 7 and 100 keV in which the VOV exceeds its
threshold (see Reliability in the SKIRT pilot run chapter). Apart from launching more photon packets, a
possible remedy is delta-scaling of the Rayleigh phase function, removing its forward peak together with the
corresponding fraction of the cross section, which requires a change to SKIRT.

**Direct component.** The direct component is the incident spectrum attenuated along the line of sight,
which can be calculated exactly for any sight line and incident spectrum, removing its noise and leaving
only the scattered component to be simulated.

**Run time on the production platform.** Run a few representative simulations on the computing platform that
will be used for the production runs, to measure the run time. The pilot run with 5 × 10<sup>8</sup> packets
took 34.8 hours, or 0.25 ms per packet, on a laptop with an Intel i7-13700H processor using 20 threads; a
workstation with 16 threads ran 2.5 times faster per packet. Each simulation covers all viewing directions of
one combination of geometry and incident spectrum, so that the UXClumpy grid, with 4 × 5 = 20 geometries and
11 × 5 = 55 incident spectra, requires 1100 simulations. Evaluate whether this is feasible.

**SKIRT performance.** Measure where SKIRT spends time when simulating these models, and implement performance
improvements where possible.

**Comparison with released geometries.** Run a few simulations with geometries identical or close to
released ones, such as those with `TORsigma` = 28 and 84 and `CTKcover` = 0.6 used in the comparison of the
pilot run, both with the full SKIRT physics and with physics reduced to match XARS (free-electron
scattering, the XARS set of fluorescent lines, and no intrinsic line shapes), and preferably with spectra
for sky bins ((see Omni spectrum above), to allow a like-for-like comparison with UXClumpy.

## Production tasks

- **Production workflow.** Generate the ski files for all parameter combinations from a single template,
  run the simulations in parallel over the parameter combinations, and verify each run for completeness
  and noise level.
- **Table model.** Assemble the spectra into table files in the format expected by XSPEC: the transmitted
  and reflected components correspond to the direct and scattered flux components recorded by the
  instruments, with the fluorescent lines in the scattered component. With 0.5 eV bins from 0.3 to 12 keV,
  a table with the parameter grid of UXClumpy takes about 13 GB in single precision, compared with about
  540 MB for UXClumpy. Whether XSPEC handles such a table efficiently, in particular the interpolation
  between parameter values and the loading time, or whether it must be split, must be tested.
- **Validation and application.** Compare fits of the new model and UXClumpy to archival X-ray spectra,
  fit the model to XRISM/Resolve data of a nearby, obscured AGN, and explore self-consistent infrared and
  X-ray models, using the multi-wavelength capabilities of SKIRT.
