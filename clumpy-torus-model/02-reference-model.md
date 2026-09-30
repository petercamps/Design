# Reference model

This chapter summarizes the UXClumpy model as described in its documentation and in the XARS code
repository. Details marked as to be verified still need to be checked against the paper, the code,
or with its author.

## Geometry

The geometry is recreated from the published distributions and parameters, rather than from the cloud
catalogs used for UXClumpy. This section collects the information needed to do so, from Sects. 2,
3, and Appendix C of Buchner et al. (2019), complemented by the table export scripts in the XARS
repository where the paper is ambiguous. The cloud generator itself is not part of the repository.

### Target model versus computed model

The paper first constructs a clumpy model that reproduces the observed column density distribution
of AGN and the observed cloud eclipse rates (Sects. 3.1 and 3.2, Table 2). The spectra of the
released table model, however, were calculated with a less demanding set of values (Sect. 3.3).
The XARS export scripts confirm the computed values: the geometry files are named after 10 000
clouds, and the cloud angular size is derived from the number of clouds as
θ<sub>cloud</sub> = (N<sub>tot</sub> / 10<sup>5</sup>)<sup>−1/2</sup> degrees.

| Quantity | Target model (Sects. 3.1–3.2, Table 2) | Computed model (Sect. 3.3, XARS scripts) |
| --- | --- | --- |
| total number of clouds N<sub>tot</sub> | 10<sup>5</sup> | 10<sup>4</sup> |
| characteristic cloud angular size θ<sub>cloud</sub> | 1° | √10 · 1° ≈ 3.16° |
| cloud column density N<sub>H</sub><sup>cloud</sup> | 10<sup>23 ± 1.0</sup> cm<sup>−2</sup> (Sect. 3.1) | 10<sup>23.5 ± 1.0</sup> cm<sup>−2</sup> |
| angular width σ | 6° to 90° (Tables 1 and 2) | 0, 7°, 28°, 84° (0 means no cloud population) |
| inner Compton-thick ring | not included | covering factor C = 0, 0.25, 0.3, 0.45, or 0.6 |

Reducing the number of clouds by a factor of ten while increasing their solid angle by the same
factor keeps the number of clouds along a line of sight unchanged, and the paper reports that the
spectrum is not affected by the number of clouds beyond 10<sup>4</sup> (Sect. 2.2). Table 2 of
the paper mixes both sets: it lists 10<sup>5</sup> clouds of 1°, but the column density
distribution of the computed model. Recreating the released table model requires the values in
the right-hand column.

### Cloud population

The clouds are spheres of uniform density, placed around a central point source, with azimuthal
symmetry. Their distributions follow the formalism of the infrared CLUMPY model (Nenkova et al.
2008).

**Angular distribution.** The mean number of clouds encountered along a radial line of sight at
elevation β above the equatorial plane is

    N(β) = N0 · exp(−(β / σ)^m),    with m = 2,

where N0 is the mean number of clouds along a line of sight in the equatorial plane, and σ is the
angular width, the `TORsigma` parameter. The cloud centers thus have a density per unit solid
angle proportional to exp(−(β / σ)²), which is a Gaussian in β with standard deviation σ / √2.
The XARS geometry files store this standard deviation, s, rather than σ: the files for the
released table use s = 5°, 20°, and 60°, and the export scripts convert these to
σ = √2 · s, rounded to 7°, 28°, and 84°. At σ = 0 there is no cloud population, and only the
inner ring (if any) remains. N0 is not a free parameter: it follows from the number and sizes of
the clouds, and ranges from about 2 to 9 in the target model.

**Radial distribution.** The clouds are distributed uniformly over two orders of magnitude in
radius, from R<sub>in</sub> to R<sub>out</sub> = Y · R<sub>in</sub> with Y = 100. In the CLUMPY
formalism, the radial distribution describes the number of clouds per unit length along a radial
line of sight, proportional to r<sup>−q</sup>, and the paper uses q = 0. Because the angular
diameters of the clouds are drawn independently of their distance (see Cloud sizes below), the mean
cross section of the clouds at radius r is proportional to r². A constant number per unit length
then means that the number of clouds per unit radius is constant: the radii of the cloud centers
are distributed uniformly between R<sub>in</sub> and R<sub>out</sub>. The model is scale-free, so that
the length unit can be chosen freely, for example R<sub>out</sub> = 1 as in the XARS
visualizations.

**Cloud sizes.** The angular diameters θ of the clouds, as seen from the center, follow an
exponential distribution with mean θ<sub>cloud</sub>, which is √10 · 1° for the computed model.
A cloud at distance d from the center has diameter D = d · sin θ.

**Cloud column densities.** The column density of each cloud through its center, N<sub>H</sub> =
n<sub>H</sub> · D, follows a log-normal distribution: log N<sub>H</sub> (in cm<sup>−2</sup>) is
normally distributed with mean 23.5 and standard deviation 1.0 for the computed model. The
uniform hydrogen number density of the cloud is n<sub>H</sub> = N<sub>H</sub> / D, as in the
XARS geometry class.

**Placement.** The clouds are placed at random according to these distributions, and a cloud that
overlaps a previously placed cloud is drawn again. The resulting clouds therefore do not overlap,
as the `ClumpySphericalSpatialGrid` requires.

**Medium between the clouds.** The XARS geometry consists of the clouds only, so that UXClumpy has
no medium between the clouds.

### Inner Compton-thick ring

The released table model adds an optional inner ring of Compton-thick clouds close to the central
source (Sect. 3.3, Fig. 5). A number of identical spheres, just touching each other, form a ring in
the equatorial plane, and a second row of spheres fills the gaps between them, forming a thick
donut. The covering factor C of the ring, the `CTKcover` parameter, determines the number and
radius of the spheres, from sixteen small spheres to three large ones. The XARS scripts associate
the covering factors with the number of touching spheres in the first row as follows:

| `CTKcover` | 0 | 0.25 | 0.3 | 0.45 | 0.6 |
| --- | --- | --- | --- | --- | --- |
| touching spheres | none | 8 | 6 | 4 | 3 |

Fig. 5 of the paper shows the configuration with six touching spheres and a second row of
gap-filling spheres, which covers 30% of the sky. The ring clouds have column densities of
log N<sub>H</sub> = 25 ± 0.5 according to the model documentation, while the name of the geometry
files for the ring alone suggests a value of 25.5.

### Recipe

With these ingredients, a geometry for given values of σ and C is recreated as follows:

1. Choose the length unit so that R<sub>out</sub> = 1 and R<sub>in</sub> = 0.01.
2. For each of the 10 000 clouds, draw a radius d uniformly between R<sub>in</sub> and
   R<sub>out</sub>, an elevation β from the density exp(−(β / σ)²) per unit solid angle, and an
   azimuth uniformly between 0 and 2π. Draw an angular diameter θ from the exponential distribution
   with mean √10 · 1°, set the diameter to D = d · sin θ, draw log N<sub>H</sub> from the normal
   distribution with mean 23.5 and standard deviation 1.0, and set the density to
   n<sub>H</sub> = N<sub>H</sub> / D. Draw the cloud again if it overlaps an existing cloud.
3. Add the inner ring for the requested covering factor C.
4. Write the cloud centers, radii, and densities to the text column file read by both the
   `ParticleMedium` and the `ClumpySphericalSpatialGrid`.

The resulting geometry can be checked against the published distributions: the cumulative
distribution of the line-of-sight column density as seen from the center (Fig. C.1 of the paper,
which shows the computed model with 10 000 clouds), the value of N0, and the volume filling factor
of 2% to 5%.

### Open points

The following details are not specified in the paper, and should be confirmed with the author:

- whether the radii of the cloud centers are indeed uniform in r, rather than uniform in log r;
- whether the angular diameters of the clouds are indeed drawn independently of their distance,
  which the derivation of the radial distribution assumes;
- how the elevations are drawn, in particular whether the solid-angle factor cos β is taken into
  account, which matters for the widest distribution, σ = 84°;
- whether the exponential size distribution is truncated, and whether clouds must lie entirely
  between R<sub>in</sub> and R<sub>out</sub>;
- the radius of the inner ring relative to R<sub>in</sub>, the exact arrangement of the second row
  of gap-filling spheres, and the column density of the ring clouds (25 or 25.5 in log
  N<sub>H</sub>, with a spread of 0.5 dex);
- the meaning of the "gexp5core" suffix in the names of the geometry files, which may indicate
  details not described in the paper.

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
photoelectric absorption, Compton scattering (with the Klein-Nishina cross section for free
electrons), and fluorescent line emission: Fe Kα and Kβ, and Kα of C, O, Ne, Mg, Si, Ar, Ca, Cr,
and Ni. It assumes solar abundances (Anders & Grevesse 1989) and the photoionization cross sections
of Verner et al. (1996). Three features of the calculation are important for this project:

- **Response matrices.** XARS injects photons at the center in each energy bin separately, and
  records the energy and direction of the escaping photons. The result is a response matrix (a
  Green's function) that maps injected energy to escaping energy. The spectrum for any incident
  spectrum, and thus for any `PhoIndex` and `Ecut`, follows by multiplying the incident spectrum
  with this matrix, without further simulations.
- **Sky binning.** XARS divides the sky as seen from the central source into bins of similar
  line-of-sight column density, further subdivided by inclination angle. The spectra collected in
  these bins provide the `NHLOS` and `Theta_inc` parameters. For a given geometry, a single
  calculation thus covers all viewing parameters. The inclination bins are 0° to 30° (face-on),
  30° to 60° (intermediate), and 60° to 90° (edge-on), presumably represented by the `Theta_inc`
  grid values 0, 60, and 90. A combination of `NHLOS` and `Theta_inc` that does not occur in the
  geometry yields an empty spectrum.
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
