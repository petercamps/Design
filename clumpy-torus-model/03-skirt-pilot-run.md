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
cutoff energy `Ecut` = 200 keV, in the range from  0.1 to 1520 keV. This is the source spectrum
used for the example output spectra in the Reference model chapter, and it is close to typical
values for nearby AGN.

The source spectrum is provided to SKIRT in the tabulated column format read by `FileSED`.

Resolution?

### Spectral resolution

The instruments record the spectra on an energy grid with two parts:

- from 0.1 to 12 keV, bins with a constant width of 0.5 eV, which amounts to 23 800 bins;
- from 12 to 1520 keV, logarithmically distributed bins with the same resolving power as the last
  0.5 eV bin at 12 keV, E / ΔE = 24 000, which amounts to about 116 200 bins.

The grid thus has about 140 000 bins, compared with 1000 for UXClumpy (see Spectral resolution in
the Reference model chapter), and the logarithmic part accounts for 83% of them. The 0.5 eV bins
sample the energy resolution of XRISM/Resolve, 4.5 eV FWHM, with about nine bins per resolution
element.

A grid that is linear in energy is not linear in wavelength, so that the first part cannot be
configured as a `LinBorderWavelengthGrid`. Instead, a `FileBorderWavelengthGrid` can list its bin
borders in keV, while a `ResolutionBorderWavelengthGrid` with a resolution of 24 000 provides the
second part, and a `CompositeWavelengthGrid` joins the two. Alternatively, a single
`FileBorderWavelengthGrid` lists all bin borders.

### Sight lines
