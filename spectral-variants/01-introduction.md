# Introduction

> Status: under construction
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md),
> [Escape instruments](escape-instruments/01-introduction.md),
> [HDF5 output](hdf5-output/01-introduction.md)

## Motivation

A grid of models may vary the spectrum of the primary sources while keeping everything else fixed. Each
spectrum then requires a separate simulation, although the simulations differ only in the weights with which their
photon packets are launched.

Within a simulation, the source spectrum enters only through the launch weights. The history of a photon packet
launched at wavelength λ<sub>0</sub>, that is, its paths, interactions, wavelength shifts, and the peel-off photon
packets it sends to the instruments, depends on λ<sub>0</sub> and on random numbers, but not on the source spectrum at
other wavelengths. If the response of the model is linear in the luminosity of the sources, the same history in an
identical model with a different source spectrum carries the same contributions, multiplied by the ratio of the two
specific luminosities at λ<sub>0</sub>. An instrument that knows the launch wavelength behind a contribution can
therefore also calculate the corresponding contribution for many other source spectra, without launching additional
photon packets.

The source spectra must be chosen before the simulation. Because an extra variant costs only memory
and recording time, far less than a new simulation, it pays to choose the set of variants generously.

## Proposal

This design note proposes:

- **Spectral variants in the SED file of the first source.** If the first source of the simulation is a normalized
  source with an SED read from a file, every column of that file after the wavelengths can hold a spectral variant,
  normalized in the same way as the SED of the source. Without variants, the source uses the first of these columns
  as its SED, as it does today.
- **Launching from the envelope.** With the variants on, the source launches its photon packets from the upper
  envelope of the spectra of all its variants, combined with its wavelength bias distribution as usual. A simulation
  with N times as many photon packets as a simulation without variants then samples each variant at least as well as
  the variant's own simulation. SKIRT calculates the envelope and logs N, which depends only on the shapes of the
  variants and on the wavelength bias fraction.
- **A single switch.** One property of the instrument system turns the variants on. Every SED instrument and escape
  instrument then records, instead of its regular output, the flux for each variant: every contribution of a photon
  packet history launched by the first source, scaled to the specific luminosity of the variant at the launch
  wavelength of the history.
- **Output per variant.** Each of these instruments writes its complete output for each variant, in the same format
  as its regular output, with the variant index in the file names.

## Scope and assumptions

The reweighting is exact under the following conditions, which SKIRT checks during setup:

- **Primary emission only.** The simulation has no secondary emission by dust or gas. Secondary emission
  in general depends nonlinearly on the source spectrum, and its photon packets have no primary launch
  wavelength. Emission emulated as a scattering channel, such as X-ray fluorescence, belongs to
  the history of the primary photon packet and is reweighted with it.
- **A fixed medium.** The simulation has no iterations over primary emission and no dynamic medium state, so that the
  medium does not depend on the radiation field. The response is then linear in the luminosity at each launch
  wavelength.
- **Only the spectrum varies.** A variant replaces the SED of the first source, and, through the normalization, its
  luminosity. The position, geometry, angular distribution, polarization profile, and velocity of the source are the
  same for all variants, and the wavelength range of the source system applies to all variants.
- **A single source with variants.** The variants belong to the first source of the source system, which must be a
  source with a single SED (`NormalizedSource` and its subclasses, such as a point source) read from a file
  (`FileSED`), in a panchromatic simulation. Additional sources, which may be of any type, keep their own spectrum
  in every variant.
- **All SED and escape instruments.** When the variants are on, every SED instrument and escape instrument records
  them, instead of its regular output, and no other instrument does. Frame, full, all-sky, HEALPix, perspective, and
  time instruments do not, because the memory of their data cubes would be multiplied by the number of variants; they
  see the first source with its first variant. For each variant, an instrument records everything it would record
  without variants, as configured by its properties.

## Impact on PTS

The output files of a variant have the same format as the regular output of the instrument, so that the existing PTS
functions load them. PTS gains a function that loads all variants of an instrument at once, as an array over
wavelength and variant, together with the variant descriptions, which are the column names of the variants file,
and the `plot_seds` command can select variants. The variants file itself is an ordinary column text file or column
file bundle, which a script writes from the parameters of the variants.

## Overview

- **[Features](spectral-variants/02-features.md)** introduces an example, and describes the launch distribution with
  the envelope, the reweighting, the variants file and the switch, the recorded quantities and statistics, the
  output, the design of the bias distribution, and the cost.
- **[Implementation](spectral-variants/03-implementation.md)** describes the changes to the instrument system, the
  sources and their SED file, the photon packet, the instruments, and the flux recorder, and the tests.
- **[Open questions](spectral-variants/04-open-questions.md)** lists alternatives and possible extensions.
