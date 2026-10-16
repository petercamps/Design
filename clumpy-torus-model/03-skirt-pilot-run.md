# SKIRT pilot run

## Setup

### Geometry

The pilot run uses the geometry file provided by the author of UXClumpy,
`10000_30_f3_gexp5core.hdf5`, with 10 000 clouds including an inner ring of six clouds (see
Reference model). Using this file avoids recreating the cloud distribution. Its clouds are converted
to the column format read by both the `ParticleMedium` and the `ClumpySphericalSpatialGrid`.
The geometry corresponds to `TORsigma` = 42 and `CTKcover` = 0.6 and is
not part of the released table, so that its spectra can be compared with the table spectra only
indirectly, for example through the closest released geometry with the same inner ring,
`TORsigma` = 28 and `CTKcover` = 0.6, which serves as reference geometry for the example spectra
in the Reference model chapter.

### Source

An isotropic point source at the center emits a cutoff power law with photon index Γ = 1.8 and
cutoff energy `Ecut` = 200 keV, in the range from 0.1 to 500 keV. This is the source spectrum
used for the example output spectra in the Reference model chapter, and it is close to typical
values for nearby AGN. The range is capped at 500 keV, above which the cross sections of the SKIRT
X-ray material mix are zero, whereas the UXClumpy source extends to 1517.5 keV (see Next steps).
For this source spectrum, the photons above 500 keV carry only 0.9% of the energy emitted up to
1517.5 keV.

The source spectrum is provided to SKIRT in the tabulated column format read by `FileSED`, as a
text column file bundle named `source_spectrum` in the HDF5 input file of the pilot run, a
capability introduced in SKIRT 10. The bundle lists the energy in keV and the photon luminosity
per unit energy in 1/s/keV, with an arbitrary normalization, since the luminosity is normalized in
the ski file.

**Resolution of the table.** SKIRT interpolates a tabulated spectrum log-log, which reproduces a
power law exactly. The interpolation error is caused only by the exponential cutoff and amounts to
h<sup>2</sup> / 8 · E / `Ecut` for points spaced by h in ln E. Points spaced uniformly in √E
distribute this error evenly over the energy range: 1200 such points keep the maximum relative
error below 10<sup>−6</sup> (8.5 × 10<sup>−7</sup>), whereas logarithmically spaced points would
require about 4800 points. The number of points hardly affects the performance, because SKIRT
samples the wavelength of a photon packet with a single binary search in the cumulative
distribution, so that the cost grows only with the logarithm of the number of points. A test run
with a luminosity probe confirmed that SKIRT reads the bundle and reproduces the cutoff power law
to within 10<sup>−6</sup>.

### Spectral resolution

The instruments record the spectra on an energy grid with two parts:

- from 0.1 to 12 keV, bins with a constant width of 0.5 eV, which amounts to 23 800 bins;
- from 12 to 500 keV, 2000 bins whose width grows by a constant factor of 1.00416 from one bin to
  the next, starting at 0.5 eV, so that the bin width increases linearly with energy, to 2.0 keV
  for the last bin.

In the second part, the resolving power E / ΔE thus decreases smoothly from 24 000 at 12 keV, the
value for the last 0.5 eV bin, to 2800 at 13 keV, 590 at 20 keV, 320 at 50 keV, 270 at 100 keV,
and 250 at 500 keV. Above 12 keV, the spectra contain no fluorescent lines, and hard X-ray
instruments have a much lower resolution than XRISM/Resolve. The upper limit of 500 keV is that of
the source (see Source). The grid thus has 25 800 bins, compared with 1000 for UXClumpy (see
Spectral resolution in the Reference model chapter), and the 0.5 eV part accounts for 92% of them.
The 0.5 eV bins sample the energy resolution of XRISM/Resolve, 4.5 eV FWHM, with about nine bins
per resolution element.

![Spectral resolution of the pilot run](figures/pilot-energy-grid.png)

*Spectral resolution of the instrument wavelength grid of the pilot run, compared with that of the
UXClumpy table spectra: (a) bin width and (b) resolving power as a function of energy. The shaded
band is the XRISM/Resolve band, from 0.3 to 12 keV, and the dotted line marks the Fe Kα line at
6.4 keV.*

A grid that is linear in energy is not linear in wavelength, and the resolving power of the second
part is not constant, so that neither part can be configured with one of the built-in wavelength
grids. Instead, a single `FileBorderWavelengthGrid` lists all 25 801 bin borders in keV. Like the
source spectrum, the borders are provided as a text column file bundle, named `wavelength_grid`,
in the HDF5 input file of the pilot run. A test run with a luminosity probe on this grid confirmed
that SKIRT reads the bundle and constructs the 25 800 bins as intended.

### Sight lines
