# Implementation

## Responsibilities

- `InstrumentSystem` holds the `recordSpectralVariants` property, the switch.
- `Configuration` makes the switch available to all simulation items; `ConfigurationSetup` retrieves it and checks
  the conditions on the simulation.
- `SourceSystem` asks its first source to provide variants, and implements a new interface through which the
  instruments obtain the number of variants, the variant factors for a source index and a launch wavelength, and the
  descriptions of the variants.
- `Source` declares the functions for the variants, with defaults for sources without variants. `NormalizedSource`
  implements them with its `FileSED`: it normalizes the variants and calculates the variant factors.
- `FileSED` reads the variant columns when asked to, checks them, and provides each variant as an SED of its own.
  `ColumnInFile` reports the names of the columns of a file.
- `PhotonPacket` keeps the index of the primary source of its history, also after emulated secondary emission.
- `SEDInstrument` and `EscapeInstrument` configure their flux recorder for the variants if the switch is on.
- `FluxRecorder` holds the variant arrays, reweights the contributions and the statistics, and writes the output for
  each variant.

## Switch

`InstrumentSystem` declares the new property before its list of instruments:

```cpp
PROPERTY_BOOL(recordSpectralVariants, "record the flux for each spectral variant of the first source")
ATTRIBUTE_DEFAULT_VALUE(recordSpectralVariants, "false")
ATTRIBUTE_RELEVANT_IF(recordSpectralVariants, "Panchromatic&!Emission&!IteratePrimary")
ATTRIBUTE_DISPLAYED_IF(recordSpectralVariants, "Level3")
```

The simulation mode and the option to iterate over primary emission are declared before the instrument system, so
that the names they insert restrict the property in the Q&A. `Configuration` gains the function
`recordSpectralVariants()`, which `ConfigurationSetup` retrieves from the instrument system. If it is true,
`ConfigurationSetup` also verifies the conditions on the simulation, which a ski file edited by hand may violate and
which include the dynamic medium state that the Q&A does not know about: the simulation must not be oligochromatic,
have secondary emission, iterate over primary emission, or have a dynamic medium state (`oligochromatic()`,
`hasSecondaryEmission()`, `hasPrimaryIterations()`, and `hasPrimaryDynamicState()`). Otherwise it reports an error.

## Sources

### Source system

If the switch is on, `SourceSystem::setupSelfBefore()` calls `enableSpectralVariants()` on its first source, before
the sources are set up, and reports an error if the source returns false.

The instrument target does not depend on the source target, so that the instruments reach the source system through
a new interface in the `utils` target:

```cpp
class SpectralVariantsInterface
{
public:
    virtual int numSpectralVariants() const = 0;
    virtual void spectralVariantFactors(int sourceIndex, double lambda0, double* factors) const = 0;
    virtual string spectralVariantDescription(int k) const = 0;
};
```

`SourceSystem` implements it. `numSpectralVariants()` and `spectralVariantDescription()` forward to the first source.
`spectralVariantFactors()` forwards to the first source for source index zero, and fills K ones for any other source.
An instrument obtains the interface with `interface<SpectralVariantsInterface>(2, 1, true)`, which searches its
parent, the simulation, and their children.

### Sources with a single SED

`Source` gains four virtual functions, which `NormalizedSource` overrides:

```cpp
virtual bool enableSpectralVariants();                                       // default: return false
virtual int numSpectralVariants() const;                                     // default: 0
virtual void spectralVariantFactors(double lambda0, double* factors) const;  // default: all factors one
virtual string spectralVariantDescription(int k) const;                      // default: empty string
```

`NormalizedSource::enableSpectralVariants()` returns false unless the SED of the source is a `FileSED`, which it then
asks to read the variants, before the SED is set up. During its own setup, after that of its SED, the source obtains
the luminosity L<sub>k</sub> of each variant from its normalization, through
`LuminosityNormalization::luminosityForSED()` applied to the SED of the variant, as it does for its own SED. It
calculates the factors as r<sub>k</sub> = L<sub>k</sub> S<sub>k</sub>(λ<sub>0</sub>) / (L S(λ<sub>0</sub>)), with S
and S<sub>k</sub> the normalized specific luminosities of its own SED and of variant k, and L and L<sub>k</sub> the
corresponding luminosities; a factor is zero where the denominator is zero, which happens only for photon packets
that carry no luminosity. Because all variants share the wavelengths of the source's own SED, the wavelength bin of
λ<sub>0</sub> needs to be located only once for all of them. The description of variant k is the name of its column.

### SED file

`FileSED` gains the function `enableSpectralVariants()`, which sets a flag. If the flag is set, the function that
reads the table during setup (`getWavelengthsAndLuminosities()`):

- obtains the names of the columns of the file, and reports an error if the file does not describe its columns or
  has no more than two;
- declares the wavelength column, the specific luminosity column of the source's own SED, and a further specific
  luminosity column for each variant, and reads them all, keeping the first two as before;
- reports an error if a variant has a positive specific luminosity at a tabulated wavelength where the second column
  has none;
- creates, as its children, one SED for each variant, of the new, non-discoverable class `SpectralVariantSED`: a
  `TabulatedSED` that receives its wavelengths and specific luminosities from the `FileSED`, and that is set up and
  normalized over the wavelength range of the source system like any tabulated SED.

`FileSED` offers the number of variants, the SED of variant k, and the name of its column. `ColumnInFile` and
`TextInFile` gain a function that returns the names of the columns described by a file, in column order: from the
header of a column text file, or from the datasets with a column attribute in a bundle.

## Photon packet

The photon packet already keeps its launch wavelength in the rest frame of the source (`_lambda0`, returned by
`sourceRestFrameWavelength()`), through all interactions and in its peel-off photon packets. The index of its source,
however, is held in the component index, which emulated secondary emission replaces by the index of the medium
component, for example after an X-ray fluorescence event. The photon packet therefore gains a member

```cpp
int _sourceIndex{-1};  // index of the primary source that launched the history, or -1
```

set by `setPrimaryOrigin()`, reset to −1 by `setSecondaryOrigin()` for genuine secondary emission, kept by
`setEmulatedSecondaryOrigin()`, which now sets the component index without calling `setSecondaryOrigin()`, and copied
by `launchEmissionPeelOff()` and `launchScatteringPeelOff()`. It is returned by `primarySourceIndex()`. With a single
source with variants, a flag would suffice, but the index leaves room for several such sources.

## Instruments

`SEDInstrument` and `EscapeInstrument` have no new property. If `Configuration::recordSpectralVariants()` is true, each
of them obtains the interface in `setupSelfBefore()` and passes it to its flux recorder, before the recorder finalizes
its configuration. A protected function of `Instrument` holds these few lines for both classes. The conditions on the
simulation are checked once, by `ConfigurationSetup`.

## Flux recorder

`FluxRecorder` gains a configuration function

```cpp
void setSpectralVariants(const SpectralVariantsInterface* variants);
```

which enables the recording of the variants, with K = `variants->numSpectralVariants()`. It applies to the SED of a
distant instrument and to the escape bins, the two modes of the recording instruments.

**Arrays.** For each recorded flux component and statistics power, the recorder allocates a variant array of K times
the size of the corresponding regular array, with the variant index varying fastest, so that the K additions for a
contribution touch consecutive memory.

**Factors.** The factors are constant for a history. Each thread keeps the history index and the K factors of the
history it is processing, and obtains new factors from the interface, for the packet's primary source index and launch
wavelength, when a detection belongs to another history.

**Recording.** `detect()` records each contribution in the regular arrays as before, and, for each component it
touches, adds r<sub>k</sub> times the contribution to the variant arrays, for k = 0 to K − 1, with the same lock-free
additions. Photon packets without a primary source index, which cannot occur under the conditions checked during
setup, are not recorded in the variant arrays.

**Statistics.** The contribution list of each thread, which collects the contributions of a history for the
statistics, also holds the factors of that history. When the list is flushed, the recorder adds, for each bin, the
regular powers of the summed contribution w as before, and r<sub>k</sub><sup>n</sup> w<sup>n</sup> for n = 1 to 4 to
the variant arrays.

**Output.** `calibrateAndWrite()` writes the regular output, and then, for each variant, the same output from the
variant arrays, with `_variant<k>` inserted after the instrument name and the description of the variant in the
headers. For this purpose, the local functions that write the SED files, the statistics files, and the data cubes take
the arrays to write as arguments, and the recorder copies the values of one variant from the variant arrays into
temporary arrays of the regular layout before writing them. The summation over MPI processes applies to the variant
arrays as to the regular arrays.

## Changes to existing classes

| Class | Change |
| --- | --- |
| `InstrumentSystem` | `recordSpectralVariants` property |
| `Configuration`, `ConfigurationSetup` | `recordSpectralVariants()`; check of the conditions on the simulation |
| `SourceSystem` | enables the variants of its first source; implements `SpectralVariantsInterface` |
| `Source` | `enableSpectralVariants()`, `numSpectralVariants()`, `spectralVariantFactors()`, and `spectralVariantDescription()`, with defaults for sources without variants |
| `NormalizedSource` | enables the variants of its `FileSED`; normalization of the variants; variant factors |
| `FileSED` | reads and checks the variant columns if enabled; one child SED for each variant |
| `ColumnInFile`, `TextInFile` | names of the columns described by a file |
| `PhotonPacket` | `_sourceIndex`, kept through emulated secondary emission and copied to peel-off photon packets; `primarySourceIndex()` |
| `Instrument` | protected function that configures the recorder for the variants |
| `SEDInstrument`, `EscapeInstrument` | call that function if the switch is on |
| `FluxRecorder` | variant arrays, factors per history, reweighted recording and statistics, output for each variant |

New classes: the interface `SpectralVariantsInterface` in the `utils` target, and the non-discoverable
`SpectralVariantSED` next to `FileSED`. No new dependencies between targets are needed.

## Testing

New test cases in the functional test suite:

- **Exact reweighting.** A point source in a dusty sphere, in extinction-only mode, with a variants file holding a
  tabulated black body as its own SED and two variants (a black body of another temperature and a power law), and
  launch wavelengths drawn from the bias distribution alone (ξ = 1). Separate simulations, each with a two-column
  file holding one variant as the source's own SED, with the same random seed and a single thread, reproduce the
  variant output of an SED instrument and an escape instrument up to rounding errors, including the statistics.
- **Switch off.** The same simulation with the switch off produces no variant output, and the same regular output as
  with the switch on and as with a two-column file.
- **Fluorescence.** The same comparison for an X-ray source in a gas with the `XRayAtomicGasMix`, which shows that the
  fluorescent photon packets, recorded as secondary emission, keep their source and are reweighted correctly.
- **Statistical agreement.** The dusty sphere with ξ = 0.5: the variant fluxes agree with those of the separate
  simulations within the noise.
- **Other sources.** Two point sources, of which the first has variants and the second a file with extra columns:
  the variants reproduce separate simulations in which only the first source changes.
- **Errors.** The switch with a dust emission simulation, with a first source whose SED is not a `FileSED`, with a
  two-column file, with a column text file that does not describe its columns, and with a variant that emits where
  the source's own SED does not each produce an error during setup.

The exact reweighting case is repeated in the category `HDF5`, with the variants file and the output as bundles, and
with several threads and MPI processes, whose results agree statistically with those of a single thread.
