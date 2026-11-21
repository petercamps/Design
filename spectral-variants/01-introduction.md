# Introduction

> Status: under construction
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md),
> [Escape instruments](escape-instruments/01-introduction.md),
> [HDF5 output](hdf5-output/01-introduction.md)

## Motivation

A grid of models often varies the spectrum of the primary sources while keeping everything else fixed. Each
spectrum then requires a separate simulation, although the simulations differ only in the weights with which their
photon packets are launched.

The [Clumpy torus model](clumpy-torus-model/01-introduction.md) note is an example. The UXClumpy table model has 20
geometries and 55 incident spectra: cutoff power laws with 11 photon indices from 1.0 to 3.0 and 5 cutoff energies
from 60 to 400 keV. Recreating it with SKIRT requires 1100 simulations, one for each combination. The timing tests
for the second pilot run on the HPC predict about 5000 core hours for a simulation with 5 × 10<sup>9</sup> photon
packets, so that the complete table would take about 5.5 million core hours.

Within a simulation, the source spectrum enters only through the launch weights. A photon packet launched at
wavelength λ<sub>0</sub> follows a history of paths, interactions, wavelength shifts, and peel-off events that depends
on λ<sub>0</sub> and on random numbers, but not on the source spectrum at other wavelengths. If the response of the
model is linear in the luminosity of the sources, the same history launched from a source with a different spectrum
would carry all its contributions multiplied by the ratio of the two specific luminosities at λ<sub>0</sub>. An
instrument that knows the launch wavelength behind each contribution can therefore record the spectra for many source
spectra at once, from the same photon packets. For the clumpy torus model, this would reduce the 1100 simulations to
20, one for each geometry.

XARS, the code that calculated UXClumpy, follows a related route: it injects photons in each energy bin separately and
stores a response matrix from injected to escaping energy, from which the spectrum for any incident spectrum follows.
At the energy resolution of the SKIRT model, 0.5 eV, such a matrix becomes impractically large, and binning the launch
wavelengths introduces an approximation (see Open questions). Recording the spectra for a given list of source
spectra avoids both.

## Proposal

This design note proposes:

- **Spectral variants in the SED file of the first source.** The first source of the simulation, which has a single
  SED read from a file, provides its spectral variants as extra columns in that file: after the wavelengths and the
  specific luminosity of its own SED, one column for each variant, normalized in the same way as the source's own
  SED. The source still launches its photon packets according to its own SED and wavelength bias; the variants only
  determine the factors with which the contributions are reweighted.
- **A single switch.** One property of the instrument system turns the variants on. The first source then reads them
  from its file, and every SED instrument and escape instrument records, next to its regular output, the flux for
  each variant: every contribution of a photon packet history launched by the first source, multiplied by the ratio
  of the variant's specific luminosity to the source's specific luminosity at the launch wavelength of the history.
- **Output per variant.** Each of these instruments writes its complete output once more for each variant, in the
  same format as its regular output, with the variant index in the file names.

With the variants in the file of a single source and one switch, the ski file of a simulation with variants differs
from one without in a single property. Several sources with variants, or variants given as other kinds of SED, could
be added later (see Open questions).

## Scope and assumptions

The reweighting is exact under the following conditions, which SKIRT checks during setup:

- **Primary emission only.** The simulation has no secondary emission by dust or gas, so that its mode is `NoMedium`,
  `ExtinctionOnly`, or `ResonanceExtinction`. Secondary emission follows from the radiation absorbed by the medium,
  which in general depends nonlinearly on the source spectrum, and its photon packets have no primary launch
  wavelength. Emission that a material mix emulates as a scattering channel, such as X-ray fluorescence, belongs to
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
  (`FileSED`), in a panchromatic simulation. Other sources, of any type, keep their own spectrum in every variant.
  Variants of imported sources, with an SED for each particle or cell, variants given by other kinds of SED, and
  oligochromatic simulations are not supported (see Open questions).
- **All SED and escape instruments.** When the variants are on, every SED instrument and escape instrument records
  them, and no other instrument does. Frame, full, all-sky, HEALPix, perspective, and time instruments do not,
  because the memory of their data cubes would be multiplied by the number of variants. Scattering levels and
  polarization are not recorded for the variants.

## Impact on PTS

The output files of a variant have the same format as the regular output of the instrument, so that the existing PTS
functions load them. PTS gains a function that loads all variants of an instrument at once, as an array over
wavelength and variant, together with the variant descriptions, which are the column names of the variants file,
and the `plot_seds` command can select variants. The variants file itself is an ordinary column text file or column
file bundle, which a script writes from the parameters of the variants.

## Overview

- **[Features](spectral-variants/02-features.md)** describes the reweighting, the variants file and the switch, the
  flux components and statistics, the output, the choice of the launch distribution, and the cost.
- **[Implementation](spectral-variants/03-implementation.md)** describes the changes to the instrument system, the
  sources and their SED file, the photon packet, the instruments, and the flux recorder, and the tests.
- **[Open questions](spectral-variants/04-open-questions.md)** lists alternatives and possible extensions.
