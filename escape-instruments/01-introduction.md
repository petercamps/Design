# Introduction

> Status: draft
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md),
> [Wavelength grid pool](wavelength-grid-pool/01-introduction.md),
> [HDF5 output](hdf5-output/01-introduction.md)

## Motivation

SKIRT records radiation with instruments that rely on peel-off. At each emission and scattering event,
the simulation sends a peel-off photon packet towards each distinct line of sight, and the instruments
on that line of sight record it after attenuation along the path. Peel-off yields results with little
noise for a given direction, but it cannot provide quantities defined over a range of directions. The
spectrum of all radiation that escapes from the model, averaged over all directions, or the spectrum
averaged over a region of the sky, can only be approximated by averaging the results of many
instruments, at a cost that grows with the number of lines of sight.

Such quantities are needed, for example, for:

- **X-ray torus models.** XSPEC table models of clumpy AGN tori, such as UXClumpy, include an angle-averaged
  spectrum (the omni component), which represents radiation scattered into the line of sight by a warm
  mirror (see the [Clumpy torus model](clumpy-torus-model/01-introduction.md) note).
- **Energy budgets.** The luminosity that escapes from a model, as a function of wavelength, complements
  the absorbed luminosity in energy balance checks.
- **Anisotropic escape.** Maps of the escaping radiation over the sky show the angular distribution of
  the radiation that escapes from disks, outflows, or clumpy media.

With forced scattering, SKIRT already calculates, at each step of a photon packet, the fraction of its
luminosity that escapes along its current direction without interacting. The
simulation removes this fraction from the packet's weight and discards it. Recording it instead, binned
by direction and wavelength, yields the escaping radiation exactly, at a negligible extra cost.

## Proposal

This design note proposes three related changes:

- **Escape instruments.** A new kind of instrument records, at each step of each photon packet, the
  luminosity that escapes from the model, in bins of escape direction and wavelength, with the same flux
  components and statistics as an SED instrument. The binning of the directions is configurable: all
  directions together, bins of inclination and azimuth, or HEALPix pixels.
- **Peel-off instruments.** The existing instruments, which all rely on peel-off, are grouped under a
  new abstract class `PeelOffInstrument` between `Instrument` and its current subclasses. The
  `Instrument` base class then contains only what all instruments share, and the simulation drives the
  two kinds of instruments separately.
- **Output options.** An escape instrument writes the spectrum of each bin as a separate SED column file,
  like an SED instrument, or the spectra of all bins as FITS data cubes, one per flux component, with
  the bins along the spatial axes and the wavelengths along the third axis.

## Comparison with peel-off

| | Peel-off instrument | Escape instrument |
| --- | --- | --- |
| Directions | a single line of sight per instrument | bins that together cover part or all of the sky |
| Contribution | the peel-off packet at each event, weighted with the phase function and attenuated towards the observer | the fraction of the packet's luminosity that escapes along its own direction, at each step |
| Cost per event | one path calculation for each distinct line of sight | no extra path calculation; one bin lookup per step |
| Noise | low for a single direction; heavy tails if the phase function is sharply peaked, such as Rayleigh scattering at hard X-ray energies | low for wide bins, because every step contributes; increases as the bins become smaller; no phase function involved |
| Not suited for | averages over a range of directions | individual lines of sight |

The two kinds of instruments complement each other and can be combined in a single simulation.

## Scope and assumptions

- The note assumes that the changes proposed in the [SKIRT 10](skirt-10/01-introduction.md) design note
  have been implemented. Escape instruments obtain their wavelength grid in the same way as other
  instruments, from their own configuration or the
  [Wavelength grid pool](wavelength-grid-pool/01-introduction.md), and write their output as regular files
  or as bundles in an HDF5 file, as described in the [HDF5 output](hdf5-output/01-introduction.md) design
  note.
- Escape instruments record during the peel-off segments of a simulation only, like the existing
  instruments, for primary and secondary emission alike.
- They support forced and non-forced scattering, and simulations without media.
- **Polarization and arrival times are not supported** in this first version (see Open questions).
- The new abstract class does not change the ski file format: the instrument system still holds a list
  of `Instrument` items, and the existing instrument types keep their names and properties.

## Impact on PTS

The SED column files of an escape instrument have the same layout as those of an SED instrument, and its
data cubes are FITS files like those of a frame instrument, so that the existing PTS functions can read
and plot them. Convenience functions for loading the spectra of all bins of an escape instrument at once,
and for plotting HEALPix maps, are not required, but would be useful.

## Overview

- **[Features](escape-instruments/02-features.md)** describes how escape instruments record radiation,
  their configuration and binnings, the calibration, the flux components and statistics, and the output.
- **[Implementation](escape-instruments/03-implementation.md)** describes the new class hierarchy of the
  instruments, the escape instrument and binning classes, the extension of the flux recorder, and the
  changes to the photon life cycle.
- **[Open questions](escape-instruments/04-open-questions.md)** collects the design decisions that remain
  to be taken.
