# Implementation

## Responsibilities

- `InstrumentSystem` holds the `recordSpectralVariants` property, the switch.
- `Configuration` makes the switch available to all simulation items; `ConfigurationSetup` retrieves it and checks
  the conditions on the simulation.
- `SourceSystem` asks its first source to provide variants, distributes the photon packets over the sources with the
  launch luminosity of each source, and implements a new interface through which the instruments obtain the number
  of variants, the variant factors of a history, and the descriptions of the variants.
- `Source` declares the functions for the variants, with defaults for sources without variants. `NormalizedSource`
  implements them with its `FileSED`: it normalizes the variants, launches its photon packets from the envelope, and
  calculates the variant factors.
- `FileSED` reads all columns of its file when asked to, provides each variant as an SED of its own, and builds the
  envelope. `ColumnInFile` reports the names of the columns of a file.
- `PhotonPacket` keeps the index of the primary source of its history, also after emulated secondary emission.
- `Instrument` configures the flux recorder of every instrument for the variants if the switch is on: SED and escape
  instruments record the variants, and the other instruments the first variant.
- `FluxRecorder` holds its arrays for one or more variants, reweights the contributions and the statistics, and
  writes the output for each variant.

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
have secondary emission, iterate over primary emission, have a dynamic medium state, or store the radiation field
(`oligochromatic()`, `hasSecondaryEmission()`, `hasPrimaryIterations()`, `hasPrimaryDynamicState()`, and
`hasRadiationField()`). Otherwise it reports an error. The radiation field is excluded because it would hold the
reference luminosity of the photon packets, described next, rather than that of a variant.

## Reference luminosity and variant factors

SKIRT traces each photon packet with a single luminosity. For a photon packet launched by the first source at
wavelength λ<sub>0</sub>, this is the reference luminosity

ℓ<sub>ref</sub> = L<sub>ref,λ</sub>(λ<sub>0</sub>) / (N<sub>s</sub> p(λ<sub>0</sub>)), with
L<sub>ref,λ</sub>(λ) = max<sub>k</sub> L<sub>k,λ</sub>(λ),

the specific luminosity of the brightest variant at the launch wavelength, and p the launch distribution with the
envelope (see Features). The launch luminosity of variant k is then ℓ<sub>k</sub> = r<sub>k</sub> ℓ<sub>ref</sub>,
with the variant factor

r<sub>k</sub>(λ<sub>0</sub>) = L<sub>k,λ</sub>(λ<sub>0</sub>) / L<sub>ref,λ</sub>(λ<sub>0</sub>),

which lies between zero and one. The factor is zero for all variants if no variant emits at λ<sub>0</sub>, in which
case the reference luminosity is zero as well and the photon packet is abandoned right away, as for a photon packet
launched today at a wavelength where the SED of its source is zero. The results do not depend on the choice of the
reference, because only the product r<sub>k</sub> ℓ<sub>ref</sub> enters them; the brightest variant keeps the
factors bounded and the reference positive wherever a variant emits. The photon packets of other sources carry their
own luminosity, with factors one for all variants.

## Sources

### Source system

If the switch is on, `SourceSystem::setupSelfBefore()` calls `enableSpectralVariants()` on its first source, before
the sources are set up, and reports an error if the source returns false.

The source system distributes the photon packets over the sources, and scales the luminosity it passes to the
`launch()` function of a source, with the new function `Source::launchLuminosity()` instead of `luminosity()`. For
a source without variants, both return the same value. For the first source with variants, `launchLuminosity()`
returns the largest luminosity of the variants, so that the first source launches at least as many photon packets as
in the separate simulation of any variant, and the luminosity passed to `launch()` is L<sub>max</sub> /
N<sub>s</sub>.

The instrument target does not depend on the source target, so that the instruments reach the source system through
a new interface in the `utils` target:

```cpp
class SpectralVariantsInterface
{
public:
    virtual int numSpectralVariants() const = 0;
    virtual const double* spectralVariantFactors(size_t historyIndex, int sourceIndex, double lambda0) const = 0;
    virtual string spectralVariantDescription(int k) const = 0;
};
```

`SourceSystem` implements it. `numSpectralVariants()` and `spectralVariantDescription()` forward to the first source.
`spectralVariantFactors()` returns an array of K factors for the history: for source index zero, the factors that the
first source calculates for the launch wavelength, and for any other source, K ones. The factors are constant for a
history, and all photon packets of a history, including its peel-off photon packets, are handled by the thread that
launched it. The source system therefore keeps, for each thread, the history index and the factors of the last
history it was asked about, and recalculates them only for a new history, so that all instruments share a single
calculation per history. An instrument obtains the interface with `interface<SpectralVariantsInterface>(2, 1, true)`,
which searches its parent, the simulation, and their children.

### Sources with a single SED

`Source` gains five virtual functions, which `NormalizedSource` overrides:

```cpp
virtual bool enableSpectralVariants();                                       // default: return false
virtual double launchLuminosity() const;                                     // default: return luminosity()
virtual int numSpectralVariants() const;                                     // default: 0
virtual void spectralVariantFactors(double lambda0, double* factors) const;  // default: nothing to do
virtual string spectralVariantDescription(int k) const;                      // default: empty string
```

`NormalizedSource::enableSpectralVariants()` returns false unless the SED of the source is a `FileSED`, which it then
asks to read the variants, before the SED is set up. During its own setup, after that of its SED, the source:

- obtains the luminosity L<sub>k</sub> of each variant from its normalization, through
  `LuminosityNormalization::luminosityForSED()` applied to the SED of the variant, as it does without variants, and
  their maximum L<sub>max</sub>;
- obtains the integral M of the envelope from its `FileSED`, calculates N = (1 − ξ) M + ξ, and logs N as the factor
  by which to multiply the number of photon packets of a simulation without variants;
- calculates the effective bias fraction ξ′ = ξ / N, with which the launch distribution with the envelope becomes
  the familiar mix p(λ) = (1 − ξ′) Ê(λ) + ξ′ B(λ) of the normalized envelope Ê = E / M and the bias distribution.

With the variants on, `launch()` draws the launch wavelength from the normalized envelope with probability 1 − ξ′,
and from the bias distribution otherwise, and calculates the compensating weight w = s(λ<sub>0</sub>) /
p(λ<sub>0</sub>), with s = L<sub>ref,λ</sub> / L<sub>max</sub>, or zero if s is zero. It always calculates the
weight, also for ξ = 0, because the reference spectrum differs from the envelope. Multiplied by the luminosity
L<sub>max</sub> / N<sub>s</sub> that the source system passes, the photon packet receives the reference luminosity.

`spectralVariantFactors()` calculates the factors r<sub>k</sub> = L<sub>k</sub> S<sub>k</sub>(λ<sub>0</sub>) /
max<sub>j</sub> L<sub>j</sub> S<sub>j</sub>(λ<sub>0</sub>), with S<sub>k</sub> the normalized specific luminosity of
variant k, and sets them to zero where the denominator is zero. The description of variant k is the name of its
column. The functions `luminosity()` and `specificLuminosity()` keep describing the first variant, so that probes and
other users of the source see the first variant, as do the instruments without variants.

### SED file

`FileSED` gains the function `enableSpectralVariants()`, which sets a flag. If the flag is set, the function that
reads the table during setup (`getWavelengthsAndLuminosities()`):

- obtains the names of the columns of the file, and reports an error if the file does not describe its columns or
  has a single specific luminosity column;
- declares the wavelength column and a specific luminosity column for each variant, and reads them all, returning
  the first variant as the table of the `FileSED` itself, as without variants;
- creates, as its children, one SED for each variant, including the first, of the new, non-discoverable class
  `SpectralVariantSED`: a `TabulatedSED` that receives its wavelengths and specific luminosities from the `FileSED`,
  and that is set up and normalized over the wavelength range of the source system like any tabulated SED.

After the variants are set up, `FileSED` builds the envelope at the tabulated wavelengths, as the largest normalized
specific luminosity of the variants at each wavelength, and creates a further `SpectralVariantSED` from it, which
normalizes the envelope and draws the launch wavelengths. It also integrates the envelope over the wavelength range of
the source system, with the same interpolation as a tabulated SED, to obtain M. Between the tabulated wavelengths, the
interpolated envelope can fall slightly below a variant where two variants cross. On a fine grid, this effect is
small, and it affects the noise only, because the weights follow from the actual launch distribution.

`FileSED` offers the number of variants, the SED of variant k, the name of its column, the envelope, and M. Because
the variants share their wavelengths, it can locate the wavelength bin once when the source evaluates all variants at
a launch wavelength. `ColumnInFile` and `TextInFile` gain a function that returns the names of the columns described
by a file, in column order: from the header of a column text file, or from the datasets with a column attribute in a
bundle.

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

If `Configuration::recordSpectralVariants()` is true, `Instrument::setupSelfBefore()` obtains the interface and passes
it to the flux recorder of every instrument, before the recorder finalizes its configuration in `setupSelfAfter()`,
together with the value of the new virtual function `recordsSpectralVariants()`. This function returns false by
default, and true for `SEDInstrument` and `EscapeInstrument`. The instruments have no new property. The conditions on
the simulation are checked once, by `ConfigurationSetup`.

## Flux recorder

`FluxRecorder` gains a configuration function

```cpp
void setSpectralVariants(const SpectralVariantsInterface* variants, bool recordVariants);
```

which makes the recorder hold its arrays for K′ variants: K′ = K if `recordVariants` is true, and K′ = 1 otherwise,
in which case the single variant is the first. Without this call, the recorder holds its arrays for a single variant
with factor one, as today.

**Arrays.** Each array of the recorder, for a flux component, a scattering level, a Stokes vector element, or a
statistics power one to four, holds K′ values per bin, with the variant index varying fastest, so that the K′
additions for a contribution touch consecutive memory. For K′ = 1, the layout is the same as today. The array for the
statistics power zero, the number of contributing histories, holds a single value per bin, shared by all variants.

**Factors.** For each detection, the recorder obtains the factors of the history from the interface, with the history
index, the primary source index, and the launch wavelength of the photon packet. The interface returns them from its
cache for all but the first detection of a history. Without the call to `setSpectralVariants()`, the recorder skips
this step, so that the cost of a simulation without variants is unchanged.

**Recording.** `detect()` calculates each contribution as before, and adds r<sub>k</sub> times the contribution to
the arrays it touches, for each of the K′ variants, with the same lock-free additions.

**Statistics.** The contribution list of each thread, which collects the contributions of a history for the
statistics, also holds the factors of that history. When the list is flushed, the recorder adds, for each bin with
summed contribution w at the reference luminosity, (r<sub>k</sub> w)<sup>n</sup> for n = 1 to 4 to the arrays of
each variant, and one to the array of the power zero.

**Output.** If `recordVariants` is true, `calibrateAndWrite()` writes, for each variant, the output from its values in
the arrays, with `_variant<k>` inserted after the instrument name and the description of the variant in the headers,
and does not write the regular output. For this purpose, the local functions that write the SED files, the statistics
files, and the data cubes take the arrays to write as arguments, and the recorder copies the values of one variant
into temporary arrays of the regular layout before writing them. Otherwise, it writes the regular output, as today.
The summation over MPI processes applies to all arrays, as today.

## Changes to existing classes

| Class | Change |
| --- | --- |
| `InstrumentSystem` | `recordSpectralVariants` property |
| `Configuration`, `ConfigurationSetup` | `recordSpectralVariants()`; check of the conditions on the simulation |
| `SourceSystem` | enables the variants of its first source; distributes the photon packets with `launchLuminosity()`; implements `SpectralVariantsInterface` with a cache per thread |
| `Source` | `enableSpectralVariants()`, `launchLuminosity()`, `numSpectralVariants()`, `spectralVariantFactors()`, and `spectralVariantDescription()`, with defaults for sources without variants |
| `NormalizedSource` | enables the variants of its `FileSED`; normalization of the variants; launch from the envelope with the reference luminosity; variant factors |
| `FileSED` | reads all columns if enabled; one child SED for each variant and one for the envelope; integral of the envelope |
| `ColumnInFile`, `TextInFile` | names of the columns described by a file |
| `PhotonPacket` | `_sourceIndex`, kept through emulated secondary emission and copied to peel-off photon packets; `primarySourceIndex()` |
| `Instrument` | configures the recorder for the variants; virtual `recordsSpectralVariants()` |
| `SEDInstrument`, `EscapeInstrument` | `recordsSpectralVariants()` returns true |
| `FluxRecorder` | arrays for one or more variants, factors per history, reweighted recording and statistics, output for each variant |

New classes: the interface `SpectralVariantsInterface` in the `utils` target, and the non-discoverable
`SpectralVariantSED` next to `FileSED`. No new dependencies between targets are needed.

## Testing

New test cases in the functional test suite:

- **Exact reweighting.** A point source in a dusty sphere, in extinction-only mode, with a variants file holding
  three variants (two black bodies of different temperatures and a power law), and launch wavelengths drawn from the
  bias distribution alone (ξ = 1). The instruments record the flux components, two scattering levels, and the
  statistics, and the SED instrument also records polarization, with a dust mix that uses the scattering type
  `SphericalPolarization`. Separate simulations, each with a two-column file holding one variant, the same bias
  distribution, the same random seed, and a single thread, reproduce the output of each variant of an SED instrument
  and an escape instrument up to rounding errors, in all columns.
- **Envelope.** The same simulation with ξ = 0.5: the logged N agrees with an independent calculation, and the fluxes
  of each variant agree with those of the separate simulations within the noise. With N times as many photon packets
  as the separate simulations, the relative errors of each variant are not larger than theirs.
- **Switch off.** The same simulation with the switch off produces no variant output, and the same output as with a
  two-column file holding the first variant.
- **Instruments without variants.** A frame instrument in the simulation with variants agrees, within the noise, with
  that of a simulation of the first variant alone.
- **Fluorescence.** The exact reweighting case for an X-ray source in a gas with the `XRayAtomicGasMix`, which shows
  that the fluorescent photon packets, recorded as secondary emission, keep their source and are reweighted correctly.
- **Other sources.** Two point sources, of which the first has variants and the second a file with extra columns:
  the variants agree, within the noise, with separate simulations in which only the first source changes.
- **Errors.** The switch with a dust emission simulation, with a stored radiation field, with a first source whose
  SED is not a `FileSED`, with a file with a single specific luminosity column, and with a column text file that does
  not describe its columns each produce an error during setup.

The exact reweighting case is repeated in the category `HDF5`, with the variants file and the output as bundles, and
with several threads and MPI processes, whose results agree statistically with those of a single thread.
