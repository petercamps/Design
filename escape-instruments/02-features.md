# Features

## Recording escaping radiation

### Forced scattering

With forced scattering, the simulation calculates at each step of a photon packet the optical depth
τ<sub>path</sub> of the complete path from the packet's position to the boundary of the model, along its
direction of travel. It then draws an interaction point on that path and multiplies the packet's weight
by (1 − e<sup>−τ<sub>path</sub></sup>) and the scattering albedo. The part L e<sup>−τ<sub>path</sub></sup>
of its luminosity L that is removed in this way leaves the model along the direction of travel, at the
packet's current wavelength, without further interaction.

An escape instrument adds this luminosity to the bin of that direction and wavelength, before the weight
of the packet is changed. Every step contributes: the first step after emission contributes the radiation
that escapes without any interaction, and each step after a scattering event the radiation that escapes
after that event. A path without any optical depth lets the full luminosity escape, after which the
packet is terminated, as before.

### Non-forced scattering

Without forced scattering, a photon packet escapes when the interaction point drawn for it lies beyond
the boundary of the model. The escape instrument then records the full luminosity of the packet, or, if
absorption is treated explicitly, the luminosity attenuated by the absorption optical depth of the
remaining path.

### No media

In a simulation without media, every photon packet escapes unattenuated in its launch direction, and the
escape instrument records it at launch.

### Properties

The recorded luminosity is an unbiased estimate of the radiation that escapes in each bin. The estimate
is exact in the sense that no part of the escaping radiation is approximated or sampled twice: every
photon packet contributes its escaping fractions along its own trajectory. Photon packets that are
terminated because their weight has dropped below the threshold set by the `minWeightReduction` property
no longer contribute, exactly as for peel-off.

## Escape instrument

An `EscapeInstrument` has the following properties:

| Property | Description |
| --- | --- |
| `instrumentName` | the name of the instrument, as for any instrument |
| `wavelengthGrid` | the wavelength grid of the instrument, as for any instrument, or the default grid of the wavelength grid pool |
| `distance` | the distance used for the flux calibration, as for a distant instrument, or zero for a model at a cosmological redshift |
| `binning` | the binning of the escape directions (see below), by default all directions |
| `outputStyle` | `SEDFiles` (default) for one SED column file per bin, or `DataCubes` for one FITS data cube per flux component |
| `recordComponents` | record the flux components separately, as for any instrument |
| `numScatteringLevels` | the number of individually recorded scattering levels, as for any instrument |
| `recordStatistics` | record the information for calculating statistical properties, as for any instrument |

Escape instruments do not record polarization (see Open questions), so that the `recordPolarization`
property moves from `Instrument` to `PeelOffInstrument` (see the Implementation chapter). Like the
`HEALPixSkyInstrument`, the escape instrument is offered from the Regular user level onward.

The following example configures an escape instrument for the angle-averaged spectrum and one for ten
inclination bins, combining both hemispheres:

```xml
<EscapeInstrument instrumentName="omni" distance="10 Mpc" outputStyle="SEDFiles"
                  recordComponents="true" numScatteringLevels="0" recordStatistics="true">
    <binning type="EscapeBinning">
        <AllDirectionsEscapeBinning/>
    </binning>
</EscapeInstrument>
<EscapeInstrument instrumentName="inclinations" distance="10 Mpc" outputStyle="DataCubes"
                  recordComponents="true" numScatteringLevels="0" recordStatistics="true">
    <binning type="EscapeBinning">
        <InclinationEscapeBinning numInclinationBins="10" foldHemispheres="true" numAzimuthBins="1"/>
    </binning>
</EscapeInstrument>
```

## Binnings

All binnings use the model coordinates. The inclination θ of an escape direction is its angle with the
z axis, and its azimuth φ is measured from the x axis in the xy plane, so that an escaping photon packet
in a given bin reaches a distant instrument whose inclination and azimuth fall in that bin.

### All directions

The `AllDirectionsEscapeBinning` has a single bin, which covers the whole sky. The escape instrument then
records the angle-averaged spectrum of the escaping radiation, such as the omni component of a torus
model.

### Inclination and azimuth

The `InclinationEscapeBinning` divides the sky into bins of inclination and, optionally, azimuth:

- `numInclinationBins`: the number of inclination bins, equally spaced in cos θ, so that all bins cover
  the same solid angle (10 by default, at most 100 000);
- `foldHemispheres`: if true, the directions θ and 180° − θ fall in the same bin, so that the bins are
  equally spaced in |cos θ| and cover both hemispheres, as for the inclination bins of XARS; this suits models
  that are symmetric with respect to the xy plane (false by default);
- `numAzimuthBins`: the number of azimuth bins, of equal width in φ (one by default, at most 100 000); the
  wizard asks for it only for a three-dimensional model.

The inclination bins run from the +z axis to the −z axis, or, if the hemispheres are folded, from the z
axis to the xy plane. The azimuth bins start at φ = 0. With n<sub>φ</sub> azimuth bins, the index of the bin
with inclination index i<sub>θ</sub> and azimuth index i<sub>φ</sub> is i<sub>θ</sub> n<sub>φ</sub> +
i<sub>φ</sub>.

### HEALPix pixels

The `HEALPixEscapeBinning` divides the sky into the 12 N<sub>side</sub><sup>2</sup> pixels of equal area
of a HEALPix tessellation, with N<sub>side</sub> = 2<sup>`order`</sup> (`order` 3 by default, at most 10),
using the same tessellation as the `HEALPixSkyInstrument`, aligned with the model coordinates. The bins
are numbered in the ring scheme, starting at the +z axis. With the data cube output style, the pixels are
arranged in the same two-dimensional layout as the images of the `HEALPixSkyInstrument`, so that the
frames of the cube can be viewed as maps of the sky.

## Calibration

For a bin that covers a fraction f of the sky, the escape instrument reports the flux density

F<sub>λ</sub> = L<sub>bin</sub> / (4π d<sup>2</sup> f Δλ),

with L<sub>bin</sub> the luminosity recorded in the bin within a wavelength bin of width Δλ, and d the
distance of the instrument. This is the average flux density received by observers at distance d in the
directions of the bin. For the all-directions binning, f = 1, which is the calibration of an SED
instrument for a source that radiates isotropically. As for distant instruments, the cosmology of the
simulation determines whether the distance is interpreted as a rest-frame distance or through the
redshift and the corresponding angular-diameter and luminosity distances, and the output style of the
units system determines whether flux densities are given per unit wavelength, per unit frequency, or
multiplied by the wavelength.

## Flux components and statistics

With `recordComponents` enabled, an escape instrument records the same flux components as an SED
instrument:

- the transparent flux: the luminosity emitted by the sources into the bin, without any attenuation;
- the direct primary and secondary flux: the radiation that escapes without any interaction;
- the scattered primary and secondary flux: the radiation that escapes after one or more scattering
  events, and, with `numScatteringLevels` > 0, the primary flux after each number of scattering events.

As for other instruments, fluorescent emission by an X-ray gas is recorded as secondary radiation.

With `recordStatistics` enabled, an escape instrument records, for each bin and wavelength, the sums of
the contributions of each photon packet history to the powers zero to four, from which the relative error
and the variance of the variance follow as for an SED instrument. A history contributes the escaping
fractions of all its steps, including those after scattering events. Contributions of zero are not
recorded, so that the number of contributions counts the histories that actually contribute to a bin; in
particular, the recording at launch without forced scattering, which adds only the transparent flux (see
the Implementation chapter), does not count as a contribution.

## Output

### SED files

With the `SEDFiles` output style, the escape instrument writes the spectrum of each bin as a separate SED
column file, `prefix_instr_bin<i>_sed.dat`, and, with `recordStatistics`, a statistics file
`prefix_instr_bin<i>_sedstats.dat`, with the bin index zero-padded to the same number of digits for all
bins. The files have the same columns as those of an SED instrument, and their first header line describes
the bin, for example "SED of escape bin 3: inclination 45.57 deg to 53.13 deg in both hemispheres, sky
fraction 0.1, distance 1 Mpc", with the angles in the output units of the simulation. This style suits a
modest number of bins, up to a few hundred.

### Data cubes

With the `DataCubes` output style, the escape instrument writes the spectra of all bins as FITS data cubes,
one per flux component, `prefix_instr_total.fits`, `prefix_instr_transparent.fits`, and so on, with the
same component names as a frame instrument, and, with `recordStatistics`, the cubes
`prefix_instr_stats0.fits` to `prefix_instr_stats4.fits`. The third axis holds the wavelengths. The
spatial axes hold the bins:

- for the all-directions and inclination binnings, in a single row, with the bin index as dimensionless
  value along the x axis (0 to n − 1) and a single pixel along the y axis, also for bins of inclination and
  azimuth;
- for HEALPix pixels, in the two-dimensional layout of the images of the `HEALPixSkyInstrument`, with the
  pixel-in-ring index along the x axis and the ring index along the y axis, and the same axis values
  (azimuth 0° to 360°, inclination 0° to 180°), but without the offset of that instrument's crosshair
  direction. Frame pixels that correspond to no HEALPix pixel, in the shorter rings near the poles, are
  zero.

The values are the calibrated flux densities of the bins, not surface brightnesses, and the cubes carry no
observer information. This style suits large numbers of bins, such as fine HEALPix tessellations.

### Bin table

With either output style, the escape instrument writes a column text file `prefix_instr_bins.dat` that
lists, for each bin, its index and the fraction of the sky it covers, followed by the values that define
the bin: for the inclination binning, the minimum and maximum inclination and, with more than one azimuth
bin, the minimum and maximum azimuth; for the HEALPix binning, the ring index, the pixel-in-ring index, and
the inclination and azimuth of the pixel center. The all-directions binning adds no values. Together with
the binning, these values determine the pixel of each bin in the data cubes.

### HDF5 bundles

All files are written through the existing writer classes, so that they become bundles in an HDF5 file
when the output path names one (see the [HDF5 output](hdf5-output/01-introduction.md) design note).

## Cost, memory, and noise

- **Cost.** At each step, an escape instrument determines the bin of the direction and the wavelength, and
  adds the contribution to the recorder. The optical depth of the path is known already, so that no path
  calculation is needed.
- **Memory.** The recorder holds one value per bin, wavelength, and recorded quantity (flux components and
  statistics). With 100 bins, a grid of 25 800 wavelength bins, and 12 recorded quantities (seven flux
  components and five statistics), this amounts to about 250 MB. HEALPix binnings must therefore remain
  coarse when combined with a fine wavelength grid.
- **Noise.** For wide bins, the noise is low, because every step of every photon packet contributes to some
  bin. The noise increases as the bins become smaller, and for individual lines of sight, peel-off remains
  more efficient. Because no phase function is involved, the heavy tail of peel-off contributions caused by
  a sharply peaked phase function, such as that of Rayleigh scattering at hard X-ray energies, does not
  occur. The spread of the escaping fractions e<sup>−τ</sup> themselves remains: in a bin of heavily
  obscured directions, rare paths with a low optical depth may still dominate.
