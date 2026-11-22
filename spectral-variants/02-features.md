# Features

## Example

This chapter illustrates the features with a fictitious grid of models: a table model for the X-ray spectra of an
active galactic nucleus obscured by a clumpy torus of neutral gas.

- The table has 10 torus geometries and, for each geometry, 40 incident spectra: cutoff power laws with photon flux
  N(E) ∝ E<sup>−Γ</sup> exp(−E/E<sub>c</sub>), with 8 photon indices Γ from 1.4 to 2.8 in steps of 0.2 and 5 cutoff
  energies E<sub>c</sub> of 50, 100, 200, 300, and 500 keV.
- The central source is a point source that emits between 0.2 and 500 keV, normalized to a luminosity of
  10<sup>43</sup> erg/s between 2 and 10 keV. The gas emulates X-ray fluorescence as a scattering channel.
- 100 SED instruments record the spectra between 0.2 and 200 keV, on a grid of 20 000 wavelength bins, along lines
  of sight that sample the torus, with flux components and statistics.

Without variants, the table requires 400 simulations. With variants, it requires 10, one for each geometry.

## Launching photon packets

### Without variants

A source launches its photon packets at wavelengths drawn from the probability density

p(λ) = (1 − ξ) S(λ) + ξ B(λ),

where S is the normalized SED of the source, B its wavelength bias distribution, and ξ its wavelength bias fraction. A
photon packet launched at wavelength λ<sub>0</sub> carries the luminosity L<sub>λ</sub>(λ<sub>0</sub>) /
(N<sub>s</sub> p(λ<sub>0</sub>)), with L<sub>λ</sub> the specific luminosity of the source and N<sub>s</sub> the
number of photon packets launched from it, so that the result does not depend on the choice of p, apart from the
noise. The launch wavelength is taken in the rest frame of the source, before any Doppler shift by the source
velocity.

The two terms of p have different roles:

- **The spectrum term, (1 − ξ) S, is a safety net.** The photon packets follow the luminosity of the source, so that
  no photon packet carries more than 1/(1 − ξ) times the average luminosity per photon packet. This term needs no
  design.
- **The bias term, ξ B, provides what the model needs.** It adds photon packets where the model needs them but the
  spectrum provides few, for example at energies from which a faint but important part of the output stems after
  scattering. What the model needs depends on the geometry, the medium, and the instruments, and the bias distribution
  is designed for it, for example from the statistics of an earlier simulation.

### With variants: the envelope

The first source has K spectral variants, with specific luminosities L<sub>k,λ</sub>, for k = 0 to K − 1, each
normalized with the normalization of the source. The normalized spectrum of variant k, with unit integral over the
wavelength range of the source, is S<sub>k</sub>. All variants share the photon packets of a single simulation. A
separate simulation of variant k would launch its photon packets from

p<sub>k</sub>(λ) = (1 − ξ) S<sub>k</sub>(λ) + ξ B(λ).

The bias term is the same for all variants; only the safety net differs. Launching from the distribution of any one of
the variants would undersample the variants that are much brighter elsewhere. In the example, with ξ = 0.5 and the
default bias distribution, which is uniform in log λ, launching from the variant with photon index 1.8 and cutoff 200
keV would give the variant with photon index 2.8 and cutoff 50 keV, at 0.2 keV, only about a fifth of the photon
packets of its own simulation.

The first source therefore replaces the spectrum in the safety net by the upper envelope of the spectra of all its
variants,

E(λ) = max<sub>k</sub> S<sub>k</sub>(λ),

and launches its photon packets from

p(λ) = [(1 − ξ) E(λ) + ξ B(λ)] / N, with N = (1 − ξ) M + ξ and M = ∫ E(λ) dλ.

Equivalently, p is the upper envelope of the launch distributions p<sub>k</sub> of all variants, normalized. Because
the envelope is at least as large as the spectrum of each variant, M is at least one, and equal to one only if all
variants have the same shape.

| | Without variants | With variants | Role |
| --- | --- | --- | --- |
| Spectrum term | (1 − ξ) S | (1 − ξ) E | safety net, which follows the luminosity |
| Bias term | ξ B | ξ B | what the model needs |
| Normalization | 1 | N | |

Because N p ≥ p<sub>k</sub> for every variant, a simulation with N n photon packets launches, at every wavelength and
for every variant, at least as many photon packets as the variant's own simulation with n photon packets, each
representing no more luminosity of that variant (see Reweighting). The noise of each variant is then at most about
that of its own simulation. SKIRT calculates the envelope and N during setup, and logs N as the factor by which to
multiply the number of photon packets of a simulation without variants to obtain this guarantee.

In the example, M = 2.35, so that N = 1.68 for ξ = 0.5, and N = 1.14 for ξ = 0.9. A single simulation with N n photon
packets thus replaces 40 simulations with n photon packets each.

If ξ = 1, the photon packets are launched from the bias distribution alone, and N = 1. As without variants, the bias
distribution must then be positive wherever a variant emits.

## Reweighting

### Variant luminosities

For variant k, a photon packet launched by the first source at wavelength λ<sub>0</sub> represents the luminosity

ℓ<sub>k</sub> = L<sub>k,λ</sub>(λ<sub>0</sub>) / (N<sub>s</sub> p(λ<sub>0</sub>)).

Under the conditions listed in the Introduction, everything that happens to the photon packet after its launch is
proportional to its launch luminosity, with factors that do not depend on the source spectrum: the attenuation along
its paths, the probabilities and wavelength shifts of its interactions, including emulated emission such as
fluorescence, the biasing factors of forced scattering and peel-off, and the termination criterion, which is
relative to the launch weight. Every contribution of the history to an instrument, at whatever wavelength, is
therefore proportional to ℓ<sub>k</sub>, with a factor that is the same for all variants. SKIRT traces each photon
packet once, and an instrument that records the variants records each contribution of the history for each variant,
scaled to the launch luminosity of that variant. The contributions to two variants k and k′ are in the ratio
ℓ<sub>k</sub> / ℓ<sub>k′</sub> = L<sub>k,λ</sub>(λ<sub>0</sub>) / L<sub>k′,λ</sub>(λ<sub>0</sub>), which is constant
for a history, from launch to termination, including all its peel-off photon packets. A variant that does not emit
at the launch wavelength receives no contribution from the history.

### Accuracy

The fluxes of a variant are unbiased estimates of the fluxes of a separate simulation with the variant as the SED of
the source, and with the same bias distribution and bias fraction. If ξ = 1, the launch wavelengths and all later
random draws do not depend on the spectra, so that the separate simulation with the same random seed would produce
the fluxes of the variant exactly, up to rounding. If ξ < 1, the two agree within the noise.

### Other sources

In a model with several sources, only the first source has variants. The other sources can be of any type, including
imported sources with an SED for each particle or cell. They keep their own spectrum and launch their photon packets
as without variants, so that the contributions of their photon packet histories enter each variant unchanged. They
thus form a fixed background to the variants. For example, a model of a galaxy can vary the spectrum of a central
AGN, configured as the first source, while the stars, imported from a hydrodynamical simulation, keep theirs.

When distributing the photon packets over the sources, SKIRT uses for the first source the largest luminosity of its
variants, so that the first source launches at least as many photon packets as in the separate simulation of any of
its variants.

## Configuration

### Variants file

The first source reads its SED from a file with `FileSED`, which expects a column text file, or a column file bundle
in an HDF5 file, with the wavelengths in the first column. In a variants file, every further column holds the specific
luminosity of a variant, at the same wavelengths.

- With the variants off, `FileSED` reads only the first two columns, as it always does, so that the first variant
  serves as the SED of the source, and a variants file serves as an ordinary SED file, for this or any other source.
- Each column has its own unit and may give the specific luminosity per unit of wavelength, frequency, or energy, in
  arbitrary scaling: each variant is normalized with the normalization of the source.
- The name of a column describes the variant in the output. The file must therefore describe its columns: a column
  text file in its header, and a bundle through the names of its datasets, as it always does.
- The number of variants K follows from the number of columns. Variant k, counting from zero, is column k + 2 in the
  header of a column text file, which numbers its columns from one, and the dataset with column index k + 1 in a
  bundle, which numbers them from zero.

In the example, the variants file holds the 40 cutoff power laws, for example in a column text file with the header

```
# column 1: wavelength (keV)
# column 2: photon index 1.4, cutoff 50 keV (erg/s/keV)
# column 3: photon index 1.4, cutoff 100 keV (erg/s/keV)
...
# column 41: photon index 2.8, cutoff 500 keV (erg/s/keV)
```

or as a bundle with the same 41 columns. The source is configured as without variants:

```xml
<PointSource positionX="0 pc" positionY="0 pc" positionZ="0 pc" velocityX="0 km/s" velocityY="0 km/s"
             velocityZ="0 km/s" sourceWeight="1" wavelengthBias="0.5">
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
iterations over primary emission. When it is set, the first source reads all columns of its file as variants and
launches its photon packets from the envelope, and every SED instrument and escape instrument records the variants
instead of its regular output. The instruments have no property of their own for the variants.

```xml
<instrumentSystem type="InstrumentSystem">
    <InstrumentSystem recordSpectralVariants="true">
        <instruments type="Instrument">
            <SEDInstrument instrumentName="los017" distance="10 Mpc" inclination="72.5 deg" azimuth="131.8 deg"
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
- the simulation stores the radiation field, which would not describe any of the variants;
- the first source is not a source with a single SED, or its SED is not a `FileSED`;
- the file of that `FileSED` does not describe its columns, or has a single specific luminosity column.

Because the launch distribution covers every wavelength where a variant emits, the variants need not satisfy any
condition among themselves. The extra columns in the SED files of other sources are ignored, as when the variants are
off.

## Recorded quantities and statistics

An instrument that records the variants records, for each variant, the quantities configured by its own properties:

- the total flux, or, with `recordComponents` enabled, the transparent, direct primary, scattered primary,
  transparent secondary, direct secondary, and scattered secondary flux, where the secondary components hold the
  emulated secondary emission, such as X-ray fluorescence, which is reweighted with the history that produced it;
- with `recordComponents` enabled and `numScatteringLevels` positive, the primary flux scattered once, twice, and so
  on, up to that number of times;
- for an SED instrument with `recordPolarization` enabled, the Stokes vector elements Q, U, and V of the total flux
  and, with `recordComponents` enabled, of each flux component. Escape instruments do not record polarization.

The instrument does not record its regular output, which would describe the first source with the envelope, a launch
distribution without physical meaning.

All of these quantities scale with the launch luminosity of the variant. The number of scattering events of a photon
packet and its normalized Stokes vector follow from its history, that is, from the geometry, the interactions, and the
random numbers, and not from the source spectrum. A contribution to a scattering level or a Stokes vector element is
therefore the luminosity of the detected photon packet, which scales with ℓ<sub>k</sub>, multiplied by a factor that
is the same for all variants. The output of a variant thus has the same columns as the regular output of the
instrument without variants.

With `recordStatistics` enabled, the instrument records, for each variant, the sums of the powers one to four of the
contribution of each history to the total flux in each bin. If w<sub>k</sub> is the summed contribution of a history
to a bin for variant k, these are the sums of w<sub>k</sub><sup>n</sup> for n = 1 to 4. The power zero, the number of
contributing histories, is the same for all variants. From these sums, the relative error and the variance of the
variance of each variant follow as for the regular output of an instrument.

**Instruments without variants.** Frame, full, all-sky, HEALPix, perspective, and time instruments do not record the
variants, because the memory of their data cubes would be multiplied by the number of variants. These instruments
see the first source with its first variant, as when the variants are off. Their photon packets are launched from
the envelope nevertheless, which changes their noise but not their expected values.

## Output

For each variant k, the instrument writes the files it would write without variants, with `_variant<k>` after the
instrument name and the variant index zero-padded to the same number of digits for all variants, counting from zero:
`prefix_instr_variant07_sed.dat` and `prefix_instr_variant07_sedstats.dat` for an SED instrument, and
`prefix_instr_variant07_bin03_sed.dat` or `prefix_instr_variant07_total.fits` for an escape instrument. It does not
write the files of its regular output. The files of a variant have the same columns as those of the regular output,
and the statistics files of each variant repeat the power zero. The header of each file adds a line that describes
the variant with the name of its column in the variants file, for example "variant 7: photon index 1.6, cutoff 200
keV". The bin table of an escape instrument is written once.

The output grows with the number of variants. For a large number, writing the output as bundles in an HDF5 file (see
the [HDF5 output](hdf5-output/01-introduction.md) design note) keeps it compact and in a single file.

## Designing the bias distribution

**The bias distribution.** The bias distribution and the bias fraction are designed as for a single simulation with
a representative variant, regardless of the other variants. In the example, a first simulation with the variant with
photon index 1.8 and cutoff 200 keV yields, from its statistics, a target launch density that brings the relative
error below a threshold in all relevant bins. The bias distribution is the part of that target that the spectrum term,
(1 − ξ) times the spectrum of the representative variant, does not provide. This design remains valid with variants:
the envelope contains the spectrum of the representative variant, so that the simulation, with N times as many photon
packets, still launches at least the photon packets of the target, and where another variant is brighter, the
envelope adds the photon packets of that variant's own simulation.

**The bias fraction.** A bias fraction close to one makes N close to one, but leaves a thin safety net: a photon
packet can represent up to N/(1 − ξ) times the average luminosity per photon packet of a variant. A smaller bias
fraction follows the spectra more closely, at the price of a larger N.

**Noise per variant.** The relative error of variant k in a bin depends on the number of histories that contribute to
the bin and on how much their launch luminosities ℓ<sub>k</sub> vary. If these histories start within a narrow range
of launch wavelengths, ℓ<sub>k</sub> is nearly constant over them, and the relative error is about the same for all
variants. In X-ray simulations, photon packets only lose energy, apart from thermal broadening, and most of the
radiation in a bin stems from launch energies just above it: the transmitted continuum from the energy of the bin
itself, and a fluorescent line mostly from energies up to about twice its absorption edge. The Compton hump, in
contrast, builds up from launch energies up to several times the energy of the bin, so that variants whose shape
differs strongly from that of the launch distribution have more noise there.

**Correlated noise.** All variants are calculated from the same histories, so that their noise is correlated. A noise
feature, such as a spike caused by a single history with a large weight, then appears in all variants at once, where
it can look like a physical feature. The relative errors in the statistics output of each variant, or a second
simulation with another random seed, tell the two apart. On the other hand, the difference between two neighbouring
variants stems from the same histories as the variants themselves, so that most of their noise cancels in it. What
remains depends only on how much ℓ<sub>k</sub> − ℓ<sub>k′</sub> varies among the histories that contribute to a bin,
and is much smaller than the noise in the difference of two independent simulations. The effect of a change of the
source spectrum is therefore determined more accurately than the spectra themselves, which helps a table model
interpolate between neighbouring spectral parameters.

## Cost and memory

### Memory

An instrument that records the variants holds, for each variant, the arrays it would hold without variants: one for
the total flux and one for each recorded flux component, scattering level, and Stokes vector element, and one for each
statistics power one to four. Its memory thus grows by a factor K. Because all SED and escape instruments record the
variants, this applies to each of them, and each MPI process holds all these arrays. Each scattering level adds one
array per variant, and polarization three, or 21 with the flux components. The instruments of a simulation with many
variants should therefore record only what is needed, and the simulation should use fewer processes with more threads
each.

In the example, with 20 000 wavelength bins, an array of an SED instrument takes 160 kB. With the total flux, six flux
components, and four statistics powers, a variant takes about 1.8 MB per instrument, and the 40 variants of the 100
instruments take about 7 GB in each MPI process, or about 3 GB without the flux components.

### Time

Each detection adds its contribution, for every variant, to each array that it touches, and, at the end of the
history, four statistics powers for every variant, so that the recording work grows about linearly with the number of
variants. The number of additions per photon packet is the number of contributions, which is the number of
instruments times the number of detections per instrument, one for the emission and one for each scattering event,
multiplied by the number of variants and by the number of arrays touched per contribution. The relative overhead
depends on how this compares with the time to trace a photon packet, which is large in a medium with many cells or
clumps along each path. In addition, the simulation launches N times as many photon packets as a simulation without
variants.

In the example, suppose that each instrument registers about four contributions per photon packet, one for the
emission and about three for scattering events, and that tracing a photon packet takes about 3 ms on one core. With
100 instruments, a photon packet then makes about 400 contributions, each of which needs about 240 lock-free additions
for 40 variants with flux components and statistics, or about 10<sup>5</sup> additions per photon packet. At a few
nanoseconds per addition, this takes 0.2 to 0.5 ms, an overhead of roughly 5% to 15%. The cost of an addition remains
to be measured.

### Output and total cost

Each instrument writes its output once for each variant, so that the output of a simulation grows linearly with the
number of variants. In return, a simulation with K variants and N n photon packets replaces K simulations with n
photon packets each, so that the computation time for a grid of models over the spectrum of the first source drops by
about a factor K/N, apart from the recording overhead.

In the example, the output of a simulation with 40 variants amounts to about 17 GB of text files, or about 8 GB of
HDF5 bundles. If each separate simulation needs 5 × 10<sup>8</sup> photon packets, the 400 simulations of the table
take about 170 000 core hours. With variants and ξ = 0.5, the 10 simulations take about 7 000 core hours, or about
8 000 core hours with the recording overhead.
