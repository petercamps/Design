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
items. The SMILE schema changes only in the type hierarchy it describes.

## Instrument system

During setup, `InstrumentSystem` partitions its instruments into peel-off instruments and escape
instruments. It groups the peel-off instruments by sight line, as it does now for all instruments, and
offers these groups as `peelOffGroups()` (the current `sightLineGroups()`, with elements of type
`PeelOffInstrument*`), and the escape instruments as `escapeInstruments()`. The `flush()` and `write()`
functions still loop over all instruments, through the `Instrument` interface.

## Escape instrument

`EscapeInstrument` adds the properties `distance`, `binning`, and `outputStyle` to those of `Instrument`.
During setup, after its binning has been set up, it configures its flux recorder:

- the distance or the redshift, in the same way as `DistantInstrument`; the few lines involved are best
  moved to a small helper used by both classes;
- the escape bins, with the number of bins and, for the data cube style, the spatial layout of the bins
  provided by the binning.

The simulation calls its function

```cpp
void detectEscape(const PhotonPacket* pp, double escapeFraction, bool includeTransparent);
```

which returns if the instrument does not record the packet's wavelength, determines the bin of the
packet's direction through the binning, returns if the direction is not recorded, and otherwise passes the
packet, the bin, and the escape fraction to the flux recorder. Like the recorder's `detect()` function, it
is called from many threads at once, so that it must not change any state of its own.

After the simulation, the instrument writes the bin table through a `ColumnOutFile`, with the sky
fractions and bin ranges provided by the binning, next to the output of the flux recorder.

## Escape binnings

The abstract `EscapeBinning` class declares:

```cpp
virtual int numBins() const = 0;
virtual int bin(Direction bfk) const = 0;               // -1 if the direction is not recorded
virtual double skyFraction(int bin) const = 0;
virtual string describeBin(int bin) const = 0;          // for the header of the SED file of a bin
virtual void addBinTableColumns(ColumnOutFile& file) const = 0;   // the ranges defining each bin
virtual std::pair<int, int> layout() const;            // spatial layout for the data cubes: (numBins, 1)
```

- `AllDirectionsEscapeBinning` returns bin 0 for every direction, with a sky fraction of one.
- `InclinationEscapeBinning` calculates the bin from the z component of the direction (or its absolute
  value, if the hemispheres are folded) and, with more than one azimuth bin, from the azimuth. All bins
  cover the same solid angle.
- `HEALPixEscapeBinning` calculates the HEALPix pixel of the direction with the same algorithm as the
  `HEALPixSkyInstrument`. The pixel index calculation of that instrument moves to a small helper class in
  the `utils` target, used by both classes. Its layout for the data cubes is that of the images of the
  `HEALPixSkyInstrument`.

## Flux recorder

`FluxRecorder` gains a configuration function for escape bins:

```cpp
void includeFluxDensityForEscapeBins(int numBins, int numPixelsX, int numPixelsY, bool writeDataCubes);
```

The detector arrays for each recorded quantity then hold one value per bin and wavelength, indexed as
for the surface brightness of a distant instrument, with the bin in the place of the pixel. The existing
`detect()` function serves unchanged, with the bin index as pixel index and the escape fraction as
extinction factor: it adds the unattenuated luminosity of a packet that has not scattered to the transparent
component, and the attenuated luminosity to the direct, scattered, and scattering-level components, and it
collects the contributions of each photon packet history for the statistics. One addition is needed: a
flag that excludes the transparent contribution, for the escape of unscattered photon packets without
forced scattering (see Photon life cycle below).

The thread-safe accumulation, the buffering of the contributions per history, the flush, and the summation
over processes apply unchanged. At the end of the run, `calibrateAndWrite()` receives the sky fraction of
each bin and divides the luminosity of each bin by 4π d<sup>2</sup> f Δλ, or by its cosmological
equivalent, and writes:

- for the SED file style, a `ColumnOutFile` per bin, with the same columns as the SED of a distant
  instrument and the bin description in the header, plus a statistics file per bin if requested;
- for the data cube style, a FITS data cube per flux component, and per statistics sum if requested,
  through `FITSInOutFile`, with the spatial layout given at configuration and the wavelengths along the
  third axis.

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
if (escape) detectEscape(&pp, escapeFraction(&pp), true);
simulateForcedPropagation(&pp);
...
// without forced scattering, when the interaction point lies beyond the boundary
if (!simulateNonForcedPropagation(&pp))
{
    if (escape) detectEscape(&pp, absorptionFactorForRemainingPath(&pp), false);
    break;
}
```

- **Forced scattering.** The escape fraction is e<sup>−τ</sup> with τ the total optical depth of the
  complete path, as used by `simulateForcedPropagation()`; with explicit absorption, the absorption optical
  depth of the complete path is included as well. The call precedes `simulateForcedPropagation()`, so that
  the escaping luminosity is based on the weight before the escape fraction, the albedo, and any path
  length bias are applied. The first step after launch has the full launch luminosity and no scattering
  events, so that the transparent component receives the emitted luminosity of every photon packet.
- **Non-forced scattering.** A photon packet that scatters never escapes without scattering events, so
  that the transparent component cannot be recorded at escape. It is therefore recorded at launch, with an
  escape fraction of zero, which adds the transparent contribution only, and excluded at escape. Without
  explicit absorption, the escape fraction is one; with explicit absorption, it is the attenuation by the
  absorption optical depth of the remaining path.
- **No media.** Every photon packet is recorded at launch, unattenuated.

Secondary emission passes through the same life cycle, so that it is recorded in the same way.

## Changes to existing classes

| Class | Change |
| --- | --- |
| `Instrument` | keeps the common properties, the flux recorder, `recordsWavelength()`, `flush()`, and `write()`; the peel-off interface and `recordPolarization` move to `PeelOffInstrument` |
| `DistantInstrument`, `AllSkyInstrument`, `HEALPixSkyInstrument`, `PerspectiveInstrument` | derive from `PeelOffInstrument`; the HEALPix pixel calculation moves to a helper in `utils` |
| `InstrumentSystem` | separates the peel-off groups from the escape instruments |
| `FluxRecorder` | escape bin mode, calibration per bin, SED file and data cube output, flag to exclude the transparent contribution |
| `MonteCarloSimulation` | serves the escape instruments in the photon life cycle; its peel-off functions take groups of `PeelOffInstrument` |

New classes: `PeelOffInstrument`, `EscapeInstrument`, `EscapeBinning`, `AllDirectionsEscapeBinning`,
`InclinationEscapeBinning`, `HEALPixEscapeBinning`, and the HEALPix helper in `utils`.

## Performance

The escape instruments add, per step of each photon packet, a bin lookup and the work of the flux recorder
for one contribution: a search for the wavelength bin and a few lock-free additions, plus the buffering of
the contribution if statistics are recorded. No path is calculated. Compared with the optical depth
calculation of the step, and with peel-off towards the existing instruments, this should be small, but it
must be measured, in particular with statistics, whose per-history buffers receive more contributions than
for peel-off instruments.

## Testing

New test cases in the functional test suite:

- **No media.** A point source with an anisotropic angular distribution: the inclination and HEALPix bins
  reproduce the angular distribution, and the all-directions spectrum equals the source luminosity divided
  by 4π d<sup>2</sup>.
- **Absorbing sphere.** A source at the center of a uniform, purely absorbing sphere: the spectrum equals
  the source spectrum attenuated by e<sup>−τ</sup>, for forced and non-forced scattering and with explicit
  absorption.
- **Scattering sphere.** The same sphere with scattering: the all-directions spectrum agrees statistically
  with the average of SED instruments distributed uniformly over the sky, for forced and non-forced
  scattering.
- **Axisymmetric model.** A disk: the inclination bins agree statistically with SED instruments at the
  corresponding inclinations, averaged over azimuth.
- **Output.** Both output styles, with and without statistics, as regular files and as HDF5 bundles; the
  same results with one and several threads and processes.
