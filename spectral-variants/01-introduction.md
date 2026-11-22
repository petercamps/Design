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

Within a simulation, the source spectrum enters only through the launch weights. A photon packet launched at
wavelength λ<sub>0</sub> follows a history of paths, interactions, wavelength shifts, and peel-off events that depends
on λ<sub>0</sub> and on random numbers, but not on the source spectrum at other wavelengths. If the response of the
model is linear in the luminosity of the sources, the same history launched from a source with a different spectrum
would carry all its contributions multiplied by the ratio of the two specific luminosities at λ<sub>0</sub>. An
instrument that knows the launch wavelength behind each contribution can therefore record the spectra for many source
spectra at once, from the same photon packets.

## Proposal

This design note proposes:

- **Spectral variants in the SED file of the first source.** If the first source of the simulation is a normalized
  source with an SED read from a file, it may provide spectral variants as extra columns in that file. After the
  wavelengths and the specific luminosity of the source's own SED, the file offers one column for each variant,
  normalized in the same way as the source's own
  SED. The source still launches its photon packets according to its own SED and wavelength bias; the variants only
  determine the factors with which the contributions are reweighted.
- **A single switch.** One property of the instrument system turns the variants on.
  Every SED instrument and escape instrument then records, next to its regular output, the flux for
  each variant: every contribution of a photon packet history launched by the first source, multiplied by the ratio
  of the variant's specific luminosity to the source's specific luminosity at the launch wavelength of the history.
- **Output per variant.** Each of these instruments writes its complete output once more for each variant, in the
  same format as its regular output, with the variant index in the file names.

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
  same for all variants.
- **Covered launch wavelengths.** Wherever a variant emits, the source's own SED emits as well. A photon packet
  launched at a wavelength where the source's specific luminosity is zero carries no luminosity, so that it cannot be
  reweighted. The wavelength range of the source system applies to the variants as well.
- **A single source with variants.** The variants belong to the first source of the source system, which must be a
  source with a single SED (`NormalizedSource` and its subclasses, such as a point source) read from a file
  (`FileSED`), in a panchromatic simulation. Additional sources, which may be of any type, keep their own spectrum
  in every variant.
- **All SED and escape instruments.** When the variants are on, every SED instrument and escape instrument records
  them, and no other instrument does. Frame, full, all-sky, HEALPix, perspective, and time instruments do not,
  because the memory of their data cubes would be multiplied by the number of variants. For each variant, an
  instrument records everything it records in its regular output, as configured by its properties.

## Impact on PTS

The output files of a variant have the same format as the regular output of the instrument, so that the existing PTS
functions load them. PTS gains a function that loads all variants of an instrument at once, as an array over
wavelength and variant, together with the variant descriptions, which are the column names of the variants file,
and the `plot_seds` command can select variants. The variants file itself is an ordinary column text file or column
file bundle, which a script writes from the parameters of the variants.

## Overview

- **[Features](spectral-variants/02-features.md)** describes the reweighting, the variants file and the switch, the
  recorded quantities and statistics, the output, the choice of the launch distribution, and the cost.
- **[Implementation](spectral-variants/03-implementation.md)** describes the changes to the instrument system, the
  sources and their SED file, the photon packet, the instruments, and the flux recorder, and the tests.
- **[Open questions](spectral-variants/04-open-questions.md)** lists alternatives and possible extensions.
