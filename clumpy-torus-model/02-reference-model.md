# Reference model

This chapter summarizes the UXClumpy model as described in its documentation and in the XARS code
repository. Details marked as to be verified still need to be checked against the paper, the code,
or with its author.

## Geometry

The obscurer consists of a population of spherical clouds around a central X-ray source:

- The clouds follow radial and angular distributions constrained by observations of X-ray
  eclipses. The vertical extent of the population is set by the `TORsigma` parameter, the width of
  a Gaussian distribution in elevation, following the infrared CLUMPY model. The total number of
  clouds is the same for all values of `TORsigma`, so that low values yield slightly higher
  covering factors.
- An optional inner ring of Compton-thick clouds, with column densities of log NH = 25 ± 0.5 (in
  cm<sup>−2</sup>), covers a fraction of the sky set by the `CTKcover` parameter. A low covering
  factor corresponds to many small clouds forming a thin ring, and a high covering factor to a few
  large clouds.
- The viewing angle is measured relative to the inner, flat disk portion of the geometry.

To be verified: the exact radial and angular distributions, the number and sizes of the clouds,
whether the clouds can overlap, whether there is a medium between the clouds, and the element
abundances.

## Parameters

The table model has the following parameters. The grid values are those of the table file
`uxclumpy-cutoff.fits` in the current release of the model on
[Zenodo](https://doi.org/10.5281/zenodo.602282), which the documentation refers to as
`uxclumpy.fits`.

| Group | Parameter | Meaning | Grid values |
| --- | --- | --- | --- |
| incident radiation | `PhoIndex` | photon index of the primary power law | 11 values from 1.0 to 3.0, in steps of 0.2 |
| incident radiation | `Ecut` | high-energy cutoff of the primary spectrum, in keV | 5 values: 60, 100, 140, 200, 400 |
| incident radiation | `norm` | photon flux normalization at 1 keV | not tabulated |
| viewing | `NHLOS` | total column density along the line of sight, in 10<sup>22</sup> cm<sup>−2</sup> | 41 values from 0.01 to 10<sup>4</sup>, in steps of 0.15 dex |
| viewing | `Theta_inc` | viewing angle relative to the inner disk, in degrees | 3 values: 0, 60, 90 |
| geometry | `TORsigma` | vertical extent of the cloud population, in degrees | 4 values: 0, 7, 28, 84 |
| geometry | `CTKcover` | covering factor of the inner Compton-thick ring | 5 values: 0, 0.25, 0.3, 0.45, 0.6 |

The table thus contains 41 × 11 × 5 × 4 × 5 × 3 = 135 300 spectra, calculated from 4 × 5 = 20
geometries. In the table file, the line-of-sight column density parameter is called `nH`.

Each spectrum has 1000 energy bins from 0.1 keV to about 1.5 MeV: 800 bins of 10 eV from 0.1 to
8.1 keV, followed by 200 bins whose width grows with energy, to about 1.5 keV near 50 keV. The
table file has a size of about 540 MB.

## Table files

The release contains four table files with the same parameter grid, energy grid, and order of the
spectra:

| File | Contents |
| --- | --- |
| `uxclumpy-cutoff.fits` | the total spectrum along the line of sight: transmitted plus reflected, including the fluorescent lines; this is the main model component |
| `uxclumpy-cutoff-transmit.fits` | the transmitted component: primary photons that escape along the line of sight without interacting |
| `uxclumpy-cutoff-reflect.fits` | the reflected component: Compton-scattered photons and fluorescent lines |
| `uxclumpy-cutoff-omni.fits` | the spectrum of all escaping photons averaged over all directions, called `uxclumpy-omni.fits` in the documentation |

The total spectrum equals the sum of the transmitted and reflected components. For the spectra
checked, the integrated sums agree to within 0.2 percent, and individual bins differ by more than
1 percent only in about 2 percent of the bins, mostly where the values are tiny.

The omni component represents the warm mirror: ionized gas far outside the obscurer that scatters
part of the escaping radiation into the line of sight. This gas sees the obscurer from all
directions, so that it scatters the angle-averaged spectrum, which is dominated by the unobscured
directions and is thus mostly an unobscured power law. The component is added to the main
component with a free scaling constant, typically between 10<sup>−5</sup> and 0.1. Its spectra do
not depend on `NHLOS` or `Theta_inc`: for the spectra checked, they are identical for all values of
these parameters, which the table repeats only to share the grid of the other files.

A combination of `NHLOS` and `Theta_inc` that does not occur in a given geometry yields an empty
spectrum. For the spectra checked at `NHLOS` values of 10 and 2510 (10<sup>23</sup> and 2.5 ×
10<sup>25</sup> cm<sup>−2</sup>), about 15 percent of the spectra are empty.

For SKIRT, none of these files requires separate simulations. The transmitted and reflected
components correspond to the direct and scattered flux components that the instruments record
when `recordComponents` is enabled; in SKIRT, the fluorescent lines are part of the scattered
component. The omni component follows from the spectra of all instruments, averaged over the
sphere with weights for their solid angles, which requires instrument directions that cover the
whole sphere.

The checks above are based on 500 spectra at the lowest `NHLOS` value and 330 spectra at each of
the two other values, for the lowest `PhoIndex` values, read from the four table files.

## Calculation method

UXClumpy was calculated with XARS, a compact Monte Carlo code written in Python, which includes
photoelectric absorption, Compton scattering, and fluorescent line emission. Three features of the
calculation are important for this project:

- **Response matrices.** XARS injects photons at the center in each energy bin separately, and
  records the energy and direction of the escaping photons. The result is a response matrix (a
  Green's function) that maps injected energy to escaping energy. The spectrum for any incident
  spectrum, and thus for any `PhoIndex` and `Ecut`, follows by multiplying the incident spectrum
  with this matrix, without further simulations.
- **Sky binning.** XARS divides the sky as seen from the central source into bins of similar
  line-of-sight column density, further subdivided by inclination angle. The spectra collected in
  these bins provide the `NHLOS` and `Theta_inc` parameters. For a given geometry, a single
  calculation thus covers all viewing parameters. A combination of `NHLOS` and `Theta_inc` that
  does not occur in the geometry yields an empty spectrum.
- **Analytic paths.** The optical depth that a photon travels is drawn at random, but the end point
  is determined analytically, using the LightRayRider library to find the clouds that a ray
  intersects and their order.

As a result, only the geometry parameters `TORsigma` and `CTKcover` require separate
calculations, which explains how a table with many spectra can be produced with a limited number
of simulations.

## Limitations

Sect. 4.2.4 of Vander Meulen et al. (2023) discusses details and limitations of XARS, to be
summarized here. In addition, XARS does not include bound-electron scattering, intrinsic line
shapes, or the complete set of K- and L-shell lines, which SKIRT now does.
