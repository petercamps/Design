# Implementation

## Responsibilities

- `Instrument` keeps what all instruments share: the name, the wavelength grid, the flags for flux
  components, scattering levels, and statistics, and the flux recorder with its flush and write functions.
- `PeelOffInstrument`, a new abstract class, holds the peel-off interface that `Instrument` offers now,
  and becomes the base class of all existing instruments.
- `EscapeInstrument`, a new instrument class, owns an escape binning, configures its flux recorder for
  the bins, and records the escaping luminosity passed to it by the simulation.
- `EscapeBinning`, a new abstract class with three subclasses, maps escape directions to bins, and provides
  the sky fraction and a description of each bin.
- `FluxRecorder` gains a mode for escape bins, with a calibration per bin and two output styles.
- `InstrumentSystem` separates the peel-off instruments, grouped by sight line as now, from the escape
  instruments.
- `MonteCarloSimulation` passes the escaping luminosity of each step to the escape instruments.

## Class hierarchy

```
Instrument (abstract)
├── PeelOffInstrument (abstract, new)
│   ├── DistantInstrument (abstract)
│   │   ├── FrameInstrument
│   │   └── ApertureInstrument (abstract)
│   │       ├── SEDInstrument
│   │       └── TimeInstrument (abstract)
│   │           ├── LightCurveInstrument
│   │           └── SpectralTimeMapInstrument
│   ├── AllSkyInstrument
│   ├── HEALPixSkyInstrument
│   └── PerspectiveInstrument
└── EscapeInstrument (new)
```

The `Instrument` class currently declares the peel-off interface as pure virtual functions:
`hasSameSightLine()`, `bfkobs()`, `bfky()`, and `locate()`, together with the `Detection` structure and
the `detect()` function that forwards a detected peel-off photon packet to the flux recorder. These move to
`PeelOffInstrument`, as does the `recordPolarization` property, because escape instruments do not record
polarization. The existing instruments derive from `PeelOffInstrument` instead of `Instrument`, without
further changes.

Moving a property to an intermediate base class does not change the ski file format, because a ski file
names the concrete type of each instrument, and the instrument system still holds a list of `Instrument`
items. The SMILE schema changes only in the type hierarchy it describes, and in one detail of the property
order: SMILE lists the properties of an intermediate class at the place where its base class lists those
of its subclasses, so that `recordPolarization` now follows the properties of the concrete instrument and
precedes `recordComponents`, whereas it used to sit between `numScatteringLevels` and `recordStatistics`.
This changes the order of the attributes in the ski files written by SKIRT and MakeUp, and of the questions
in the wizard, but not the parsing of ski files.

## Instrument system

During setup, `InstrumentSystem` partitions its instruments into peel-off instruments and escape
instruments. It groups the peel-off instruments by sight line, as it does now for all instruments, and
offers these groups as `peelOffGroups()` (the current `sightLineGroups()`, with elements of type
`PeelOffInstrument*`), and the escape instruments as `escapeInstruments()`. The `flush()` and `write()`
functions still loop over all instruments, through the `Instrument` interface.

## Escape instrument

`EscapeInstrument` adds the properties `distance`, `binning`, and `outputStyle` to those of `Instrument`.
During setup, it configures its flux recorder:

- the distance or the redshift, in `setupSelfBefore()`, through the protected function
  `Instrument::configureRecorderDistance()`, which holds the few lines that `DistantInstrument` used for
  this purpose and now serves both classes;
- the escape bins, in `setupSelfAfter()`, once the binning has been set up and before the base class
  finalizes the configuration of the recorder: the sky fraction and description of each bin and, for the
  data cube style, the frame layout and the pixel of each bin provided by the binning.

The simulation calls its function

```cpp
void detectEscape(const PhotonPacket* pp, double escapeFraction, bool includeTransparent);
```

which returns if the instrument does not record the packet's wavelength, determines the bin of the
packet's direction through the binning, returns if the direction is not recorded, and otherwise passes the
packet, the bin, and the escape fraction to the flux recorder. Like the recorder's `detect()` function, it
is called from many threads at once, so that it must not change any state of its own.

After the simulation, the instrument writes the bin table through a `ColumnOutFile`, with the sky
fractions and the values defining the bins provided by the binning, next to the output of the flux
recorder. For this purpose, `write()` becomes a virtual function of `Instrument`, which `EscapeInstrument`
extends.

## Escape binnings

The abstract `EscapeBinning` class declares:

```cpp
virtual int numBins() const = 0;
virtual int bin(Direction bfk) const = 0;                    // -1 if the direction is not recorded
virtual double skyFraction(int bin) const = 0;
virtual string describeBin(int bin) const = 0;               // for the header of the SED file of a bin
virtual void addBinTableColumns(ColumnOutFile& file) const;  // columns defining the bins (default: none)
virtual vector<double> binTableValues(int bin) const;        // values for these columns (default: none)
virtual FrameLayout frameLayout() const;                     // frame for the data cubes (default: one row)
virtual int framePixel(int bin) const;                       // pixel of a bin in that frame (default: bin)
```

The `FrameLayout` structure holds the number of pixels along each axis of the frame, the increment between
pixels and the center of the frame along each axis, and the quantity of these axis values (none for the
bin index).

- `AllDirectionsEscapeBinning` returns bin 0 for every direction, with a sky fraction of one.
- `InclinationEscapeBinning` calculates the bin from the z component of the direction (or its absolute
  value, if the hemispheres are folded) and, with more than one azimuth bin, from the azimuth. All bins
  cover the same solid angle. The bin edges are calculated such that the outer edge of the last
  inclination bin is exactly 180° (90° if folded).
- `HEALPixEscapeBinning` calculates the HEALPix pixel of the direction with the same algorithm as the
  `HEALPixSkyInstrument`. The pixel location calculation of that instrument moves to the helper class
  `HEALPixTessellation` in the `utils` target, used by both classes, which also converts between a pixel
  location (the pixel-in-ring and ring indices) and the pixel index in the ring scheme, and provides the
  pixel centers. The frame of the data cubes is that of the images of the `HEALPixSkyInstrument`.

## Flux recorder

`FluxRecorder` gains two configuration functions for escape bins:

```cpp
void includeFluxDensityForEscapeBins(const vector<double>& skyFractions, const vector<string>& descriptions);
void includeDataCubesForEscapeBins(int numPixelsX, int numPixelsY, const vector<int>& pixelIndices,
                                   double incrementX, double incrementY, double centerX, double centerY,
                                   string quantityXY = string());
```

The first enables the escape bins, with the sky fraction and the description of each bin, which also set
the number of bins; by default, the recorder then writes an SED file per bin. The second selects the data
cube style, with the frame in which the bins are arranged, the pixel of each bin in that frame, and the
axis metadata.

The escape bins have detector arrays of their own, with one value per bin and wavelength for each recorded
quantity, next to the arrays for the SED and the data cubes of a peel-off instrument. The existing
`detect()` function serves with one addition, with the bin index as pixel index and the escape fraction as
extinction factor: it adds the unattenuated luminosity of a packet that has not scattered to the transparent
component, and the attenuated luminosity to the direct, scattered, and scattering-level components, and it
collects the contributions of each photon packet history for the statistics. The addition is an
`includeTransparent` argument (true by default), which excludes the transparent contribution, for the
escape of unscattered photon packets without forced scattering (see Photon life cycle below). For escape
bins, contributions of zero are left out of the statistics. The polarization flag moves from
`setUserFlags()` to a separate function `setRecordPolarization()`, called by `PeelOffInstrument` only.

The thread-safe accumulation, the buffering of the contributions per history, the flush, and the summation
over processes apply unchanged. At the end of the run, `calibrateAndWrite()` divides the luminosity of
each bin by 4π d<sup>2</sup> f Δλ, with the sky fraction f given at configuration and the rest-frame or
luminosity distance d, and writes:

- for the SED file style, a `ColumnOutFile` per bin, through the same local function as the SED of a
  distant instrument, with the bin description in the header, plus a statistics file per bin if
  requested;
- for the data cube style, a FITS data cube per flux component, and per statistics sum if requested,
  through `FITSInOutFile` and the same local function as the data cubes of a peel-off instrument, with the
  bins arranged in the frame given at configuration and the wavelengths along the third axis.

Both writer classes produce bundles when the output path names an HDF5 file.

## Photon life cycle

In `MonteCarloSimulation::performLifeCycle()`, the escape instruments are served only when the `peel` flag
is set, like the peel-off instruments, and only if the instrument system holds any. A new function
`detectEscape(pp, escapeFraction, includeTransparent)` loops over the escape instruments and calls their
function of the same name.

```cpp
// after launching the photon packet
if (escape)
{
    if (!_config->hasMedium()) detectEscape(&pp, 1., true);            // escapes unattenuated
    else if (!_config->forceScattering()) detectEscape(&pp, 0., true);  // transparent flux only
}
...
// with forced scattering, after calculating the optical depths of the complete path
if (escape) detectEscape(&pp, exp(-pp.totalExtinctionOpticalDepth()), true);
if (store) mediumSystem()->storeRadiationField(primary, &pp);
simulateForcedPropagation(&pp);
...
// without forced scattering, when the interaction point lies beyond the boundary
if (!simulateNonForcedPropagation(&pp))
{
    if (escape)
    {
        double escapeFraction = _config->explicitAbsorption() ? exp(-pp.interactionOpticalDepth()) : 1.;
        detectEscape(&pp, escapeFraction, false);
    }
    break;
}
```

- **Forced scattering.** The escape fraction is e<sup>−τ</sup> with τ the total optical depth of the
  complete path, as used by `simulateForcedPropagation()`. The new function
  `SpatialGridPath::totalExtinctionOpticalDepth()` returns it: the sum of the scattering and absorption
  optical depths if they are stored separately, as with explicit absorption, and the extinction optical
  depth otherwise. The call precedes `simulateForcedPropagation()`, so that the escaping luminosity is
  based on the weight before the escape fraction, the albedo, and any path length bias are applied. The
  first step after launch has the full launch luminosity and no scattering events, so that the transparent
  component receives the emitted luminosity of every photon packet.
- **Non-forced scattering.** A photon packet that scatters never escapes without scattering events, so
  that the transparent component cannot be recorded at escape. It is therefore recorded at launch, with an
  escape fraction of zero, which adds the transparent contribution only, and excluded at escape. Without
  explicit absorption, the escape fraction is one; with explicit absorption, it is the attenuation by the
  absorption optical depth of the remaining path. For this purpose,
  `MediumSystem::setInteractionPointUsingScatteringAndAbsorption()` now stores the absorption optical depth
  of the complete remaining path in the photon packet when it finds no interaction point on the path; the
  arithmetic that determines an interaction point is unchanged.
- **No media.** Every photon packet is recorded at launch, unattenuated.

Secondary emission passes through the same life cycle, so that it is recorded in the same way.

## Changes to existing classes

| Class | Change |
| --- | --- |
| `Instrument` | keeps the common properties, the flux recorder, `recordsWavelength()`, `flush()`, and `write()`, which becomes virtual; gains `configureRecorderDistance()`; the peel-off interface and `recordPolarization` move to `PeelOffInstrument` |
| `DistantInstrument`, `AllSkyInstrument`, `HEALPixSkyInstrument`, `PerspectiveInstrument` | derive from `PeelOffInstrument`; `DistantInstrument` uses `configureRecorderDistance()`; the HEALPix pixel calculation moves to `HEALPixTessellation` |
| `InstrumentSystem` | separates the peel-off groups (`peelOffGroups()`, formerly `sightLineGroups()`) from the escape instruments (`escapeInstruments()`) |
| `FluxRecorder` | escape bin mode with detector arrays of its own, calibration per bin, SED file and data cube output, `includeTransparent` argument of `detect()`, `setRecordPolarization()` |
| `MonteCarloSimulation` | serves the escape instruments in the photon life cycle; its peel-off functions take groups of `PeelOffInstrument` |
| `MediumSystem` | stores the absorption optical depth of the remaining path when a non-forced photon packet with explicit absorption escapes |
| `SpatialGridPath` | `totalExtinctionOpticalDepth()` |
| `SimulationItemRegistry` | registers the new classes |

New classes: `PeelOffInstrument`, `EscapeInstrument`, `EscapeBinning`, `AllDirectionsEscapeBinning`,
`InclinationEscapeBinning`, and `HEALPixEscapeBinning` in the `instrument` target, and `HEALPixTessellation`
in the `utils` target. No new dependencies between targets are needed.

## Testing

New test cases in the functional test suite:

- **No media.** A point source with an anisotropic (Netzer) angular distribution, with SED instruments at
  several inclinations: the inclination bins reproduce the analytic bin averages of the angular
  distribution, the SED instruments divided by the angular distribution equal the all-directions spectrum,
  and the sums of all bins weighted with their sky fractions equal the all-directions spectrum, for the
  inclination, folded inclination and azimuth, and HEALPix binnings.
- **Absorbing sphere.** A source at the center of a uniform, purely absorbing sphere: the escaping
  spectrum equals that of a peel-off SED instrument and the source spectrum attenuated by e<sup>−τ</sup>,
  for forced and non-forced scattering, each with and without explicit absorption (four cases); for
  non-forced scattering without explicit absorption, whether a packet escapes is a random draw, so that
  the agreement is statistical.
- **Scattering sphere.** The same sphere with scattering: the all-directions total, direct, and scattered
  fluxes agree statistically with the average of SED instruments spread over the sky, and the inclination
  and azimuth bins and HEALPix pixels are consistent with isotropy, for forced and non-forced scattering
  (two cases).
- **Axisymmetric model.** A dusty disk: the folded inclination bins agree statistically with the average of
  SED instruments over each bin.
- **Dust emission.** A dusty shell with dust emission: primary and secondary components, and the
  all-directions spectrum against an SED instrument.
- **Output style and redshift.** The no-media case with energy output, whose files are the exact reversal
  of the wavelength output; and an isotropic source at redshift 1, with escape instruments at zero
  distance and at a rest-frame distance, and with a wavelength grid of its own, each next to an SED
  instrument with the same settings.

The dust emission case is repeated in the category `HDF5`, with its output written as bundles, which
reproduce the regular files. Both output styles, with and without statistics, are covered, and the
results with several threads and with several MPI processes were found consistent with those of a single
thread. The restructuring of the instruments leaves the output of all existing test cases unchanged.
