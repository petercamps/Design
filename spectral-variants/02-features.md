# Features

## Reweighting

### Launch luminosity

A source s launches its photon packets at wavelengths drawn from a mix of its own spectrum and its wavelength bias
distribution, with probability density p<sub>s</sub>(λ) = (1 − ξ) S<sub>s</sub>(λ) + ξ B<sub>s</sub>(λ), where
S<sub>s</sub> is the normalized SED of the source, B<sub>s</sub> the bias distribution, and ξ the wavelength bias
fraction. A photon packet launched at wavelength λ<sub>0</sub> carries the luminosity

L = L<sub>s,λ</sub>(λ<sub>0</sub>) / (N<sub>s</sub> p<sub>s</sub>(λ<sub>0</sub>)),

with L<sub>s,λ</sub> the specific luminosity of the source and N<sub>s</sub> the number of photon packets launched
from it. The launch wavelength is taken in the rest frame of the source, before any Doppler shift by the source
velocity.

### Variant factors

A variant k of the first source s, with specific luminosity L<sub>k,s,λ</sub>, would have given the same launch the
luminosity L<sub>k,s,λ</sub>(λ<sub>0</sub>) / (N<sub>s</sub> p<sub>s</sub>(λ<sub>0</sub>)) = r<sub>k</sub> L, with
the variant factor

r<sub>k</sub>(λ<sub>0</sub>) = L<sub>k,s,λ</sub>(λ<sub>0</sub>) / L<sub>s,λ</sub>(λ<sub>0</sub>).

Under the conditions listed in the Introduction, everything that happens to the photon packet after its launch is
proportional to its launch luminosity, with factors that do not depend on the source spectrum: the attenuation along
its paths, the probabilities and wavelength shifts of its interactions, including emulated emission such as
fluorescence, the biasing factors of forced scattering and peel-off, and the termination criterion, which is
relative to the launch weight. Every contribution of the history to an instrument, at whatever wavelength, therefore
scales with r<sub>k</sub>. An instrument that records the variants adds, for each variant, the contribution
multiplied by r<sub>k</sub>(λ<sub>0</sub>). The factor is constant for a history, from launch to termination,
including all its peel-off photon packets.

The variant fluxes are unbiased estimates of the fluxes of a simulation with the variant as the source SED. If the
launch wavelengths are drawn from the bias distribution alone (ξ = 1), the launch wavelengths and all later random
draws do not depend on the SED, so that a simulation of a single source with the variant as its SED and the same
random seed would produce the variant fluxes exactly, up to rounding.

### Other sources

In a model with several sources, only the first source has variants. The other sources can be of any type, including
imported sources with an SED for each particle or cell, and keep their own spectrum in every variant, so that the
contributions of their photon packet histories enter each variant unchanged (r<sub>k</sub> = 1). They thus form a
fixed background to the variants. For example, a model of a galaxy can vary the spectrum of a central AGN, configured
as the first source, while the stars, imported from a hydrodynamical simulation, keep theirs.

## Configuration

### Variants file

The first source reads its SED from a file with `FileSED`, which expects a column text file, or a column file
bundle in an HDF5 file, with the wavelengths in the first column and the specific luminosity in the second. A
variants file has an extra column for each variant, with its specific luminosity at the same wavelengths. `FileSED`
reads only the first two columns of a file, so that a variants file serves as an ordinary SED file when the variants
are off, for this or any other source.

- Each variant column has its own unit and, like the second column, may give the specific luminosity per unit of
  wavelength, frequency, or energy, in arbitrary scaling: each variant is normalized with the normalization of the
  source, in the same way as the source's own SED.
- The name of a variant column describes the variant in the output. The file must therefore describe its columns: a
  column text file in its header, and a bundle through the names of its datasets, as it always does.
- The number of variants K follows from the number of columns. Variant k, counting from zero, is column k + 3 in the
  header of a column text file, which numbers its columns from one, and the dataset with column index k + 2 in a
  bundle, which numbers them from zero.

For the clumpy torus model, a variants file holds the launch spectrum of the source and the 55 cutoff power laws of
UXClumpy, for example in a column text file with the header

```
# column 1: wavelength (keV)
# column 2: launch spectrum (erg/s/keV)
# column 3: PhoIndex 1.0, Ecut 60 keV (erg/s/keV)
# column 4: PhoIndex 1.0, Ecut 100 keV (erg/s/keV)
...
# column 57: PhoIndex 3.0, Ecut 400 keV (erg/s/keV)
```

or as a bundle with the same 57 columns. The source is configured as without variants:

```xml
<PointSource positionX="0 pc" positionY="0 pc" positionZ="0 pc" velocityX="0 km/s" velocityY="0 km/s"
             velocityZ="0 km/s" sourceWeight="1" wavelengthBias="0.9">
    ...
    <sed type="SED">
        <FileSED filename="variants.hdf5:cutoff_power_laws"/>
    </sed>
    <normalization type="LuminosityNormalization">
        <IntegratedLuminosityNormalization wavelengthRange="Custom" minWavelength="10 keV" maxWavelength="2 keV"
                                           integratedLuminosity="1e43 erg/s"/>
    </normalization>
    ...
</PointSource>
```

### The switch

`InstrumentSystem` gets one property, before its list of instruments:

| Property | Description |
| --- | --- |
| `recordSpectralVariants` | record the flux for each spectral variant of the first source in all SED and escape instruments (false by default) |

The property is offered at the Expert user level, in panchromatic simulations without secondary emission and without
iterations over primary emission. When it is set, the first source reads its variants from its file, and every SED
instrument and escape instrument records them, next to its regular output for the spectra of the sources themselves.
The instruments have no property of their own for the variants.

```xml
<instrumentSystem type="InstrumentSystem">
    <InstrumentSystem recordSpectralVariants="true">
        <instruments type="Instrument">
            <SEDInstrument instrumentName="i60_n27" distance="10 Mpc" inclination="60.3 deg" azimuth="218.4 deg"
                           roll="0 deg" recordComponents="true" numScatteringLevels="0" recordPolarization="false"
                           recordStatistics="true"/>
            ...
        </instruments>
    </InstrumentSystem>
</instrumentSystem>
```

### Checks

During setup, SKIRT reports an error if the variants are on and:

- the simulation does not satisfy the conditions of the Introduction: it is oligochromatic, has secondary emission,
  iterates over primary emission, or has a dynamic medium state;
- the first source is not a source with a single SED, or its SED is not a `FileSED`;
- the file of that `FileSED` does not describe its columns, or has no more than two;
- a variant has a positive specific luminosity at a tabulated wavelength where the source's own SED has none. Because
  all columns share the wavelengths and the interpolation, this check at the tabulated wavelengths also covers the
  wavelengths in between.

The extra columns in the SED files of other sources are ignored, as when the variants are off.

## Recorded quantities and statistics

For each variant, the instrument records the same quantities as for its regular output, as configured by its own
properties:

- the total flux, or, with `recordComponents` enabled, the transparent, direct primary, scattered primary,
  transparent secondary, direct secondary, and scattered secondary flux, where the secondary components hold the
  emulated secondary emission, such as X-ray fluorescence, which is reweighted with the history that produced it;
- with `recordComponents` enabled and `numScatteringLevels` positive, the primary flux scattered once, twice, and so
  on, up to that number of times;
- for an SED instrument with `recordPolarization` enabled, the Stokes vector elements Q, U, and V of the total flux
  and, with `recordComponents` enabled, of each flux component. Escape instruments do not record polarization.

All of these quantities are reweighted with the same factor r<sub>k</sub>. The number of scattering events of a
photon packet and its normalized Stokes vector follow from its history, that is, from the geometry, the interactions,
and the random numbers, and not from the source spectrum. A contribution to a scattering level or a Stokes vector
element is therefore the luminosity of the detected photon packet, which scales with r<sub>k</sub>, multiplied by a
factor that is the same for all variants. The variant arrays thus mirror the regular arrays, so that the output of a
variant has the same columns as the regular output.

With `recordStatistics` enabled, the instrument records, for each variant, the sums of the powers one to four of the
reweighted contribution of each history to the total flux in each bin, as it does for the regular output. If w is
the summed contribution of a history to a bin in the regular output, its contribution to variant k is r<sub>k</sub> w,
and its powers are r<sub>k</sub><sup>n</sup> w<sup>n</sup>. The power zero, the number of contributing histories, is
the same as for the regular output. From these sums, the relative error and the variance of the variance of each
variant follow as for the regular output.

## Output

For each variant k, the instrument writes the files of its regular output once more, with `_variant<k>` after the
instrument name and the variant index zero-padded to the same number of digits for all variants, counting from zero:
`prefix_instr_variant07_sed.dat` and `prefix_instr_variant07_sedstats.dat` for an SED instrument, and
`prefix_instr_variant07_bin03_sed.dat` or `prefix_instr_variant07_total.fits` for an escape instrument. The files have
the same columns as those of the regular output, and the variant statistics files repeat the power zero. The header of
each file adds a line that describes the variant with the name of its column in the variants file, for example
"variant 7: PhoIndex 1.0, Ecut 400 keV". The bin table of an escape instrument is written once.

The output grows with the number of variants. For a large number, writing the output as bundles in an HDF5 file (see
the [HDF5 output](hdf5-output/01-introduction.md) design note) keeps it compact and in a single file.

## Choosing the launch distribution

**Noise per variant.** The relative error of variant k in a bin depends on how much r<sub>k</sub> varies among the
histories that contribute to the bin. If these histories start within a narrow range of launch wavelengths,
r<sub>k</sub> is nearly constant over them, and the relative error of the variant is close to that of the regular
output. In X-ray simulations, photon packets only lose energy, apart from thermal broadening, and most of the
radiation in a bin stems from launch energies just above it: the transmitted continuum from the energy of the bin
itself, and a fluorescent line mostly from energies up to about twice its absorption edge. The Compton hump, in
contrast, builds up from launch energies up to several times the energy of the bin, so that variants whose shape
differs strongly from the source's own SED have more noise there.

**The source's own SED and the bias.** The source's own SED, the second column of the variants file, with weight
1 − ξ, and the bias distribution together determine the launch wavelengths, and must suit all variants. A practical
choice is a bias distribution designed for the whole set, for example from the requirements of the hardest and
softest variants, with ξ close to one. The source's own SED then mainly sets the regular output and the launch
wavelengths that can be reweighted.

**Correlated noise.** All variants are calculated from the same histories, so that their noise is correlated. The
differences between variants are less noisy than the variants themselves, which helps a table model interpolate
between neighbouring spectral parameters.

## Cost and memory

### Memory

For each variant, the recorder holds a copy of each regular array, one for the total flux and one for each recorded
flux component, scattering level, and Stokes vector element, and one array for each statistics power one to four,
each of the size of the corresponding array of the regular output. Because all SED and escape instruments record the
variants, this applies to each of them, and each MPI process holds all these arrays. The memory for the variants thus
grows with the product of the number of variants, the number of instruments, the number of recorded arrays, and the
size of an array. Each scattering level adds one array per variant, and polarization three, or 21 with the flux
components. The instruments of a simulation with many variants should therefore record only what is needed, and the
simulation should use fewer processes with more threads each.

For example, the clumpy torus model has a grid of 25 371 wavelength bins, so that an array of an SED instrument takes
about 200 kB. With the total flux, six flux components, and four statistics powers, a variant takes about 2 MB per
SED instrument. With 55 variants and 176 instruments, the variants take about 20 GB in each MPI process, or about
10 GB without the flux components. Each further array, such as a scattering level, adds about 2 GB.

### Time

Each detection adds its contribution for every variant to each array that it touches in the regular output, and, at
the end of the history, four statistics powers for every variant, so that the recording work grows about linearly
with the number of variants. The number of additions per photon packet is the number of contributions, which is the
number of instruments times the number of detections per instrument, one for the emission and one for each
scattering event, multiplied by the number of variants and by the number of arrays touched per contribution. The
relative overhead depends on how this compares with the time to trace a photon packet without variants, which is
large in a medium with many cells or clumps along each path.

For example, in the first pilot run of the clumpy torus model, each instrument registered 4.3 contributions per photon
packet on average, according to the sums of the statistics of power zero: one for the emission and one for each of
about 3.3 scattering events. With 176 instruments, a photon packet then makes about 750 contributions, each of which
needs about 300 lock-free additions for 55 variants with flux components and statistics, or about 2 × 10<sup>5</sup>
additions per photon packet. At a few nanoseconds per addition, this takes 0.5 to 1 ms, compared with about 3 ms per
photon packet on one core of the HPC without variants, an overhead of roughly 15% to 30%. The cost of an addition
remains to be measured. Even if a simulation with variants took twice as long as one without, it would replace 55
simulations.

### Output and total cost

Each instrument writes its regular output once more for each variant, so that the output of a simulation grows
linearly with the number of variants. In return, a simulation with K variants replaces K simulations, so that the
computation time for a grid of models over the spectrum of the first source drops by about a factor K, apart from the
recording overhead.

For example, for the clumpy torus model, the output of a simulation with 55 variants amounts to about 100 GB of text
files, or about 30 GB of HDF5 bundles. The 20 simulations of the table, with 5 × 10<sup>9</sup> photon packets each,
would take about 100 000 core hours, plus the recording overhead, instead of 5.5 million.
