# Introduction

> Status: under construction
>
> Depends on: none

## Motivation

Active galactic nuclei (AGN) are surrounded by a parsec-scale obscuring structure of cold gas and
dust, commonly called the torus, which absorbs and reprocesses the primary X-ray emission of the
central engine. X-ray eclipses, sudden changes in the line-of-sight column density on timescales of
hours to days, show that this obscurer is clumpy, with individual clumps subtending 0.1 to 1 degree
and spread over two orders of magnitude in radius.

The UXClumpy model (Buchner et al. 2019) describes this geometry with a large number of clumps whose
radial and angular distributions are constrained by observations. It is the most widely used clumpy
X-ray torus model, available as a table model for the XSPEC spectral fitting package.

The XRISM/Resolve microcalorimeter now delivers an energy resolution of about 4.5 eV, an order of
magnitude better than previous X-ray instruments, and resolves individual line profiles for the
first time. This demands spectral models with matching detail. SKIRT now implements the relevant
X-ray physics: the complete set of K- and L-shell fluorescent lines, intrinsic line profiles, and
bound-electron scattering (Vander Meulen et al. 2023, 2024). None of these are included in the
existing clumpy torus models.

## Goal

The goal is an XSPEC table model equivalent to UXClumpy, but calculated with SKIRT:

- the same radial and angular distributions of the clumps, which are based on observations;
- a uniform background medium between the clumps, since observations do not constrain its
  structure;
- the same parameter space, including the densities, the extent of the clump distribution, and the
  viewing angle;
- a higher spectral resolution, about 0.5 eV, which amounts to tens of thousands of points per
  spectrum;
- more complete physics: bound-electron scattering, more fluorescent lines, and intrinsic line
  shapes.

The table model consists of a set of spectra covering the parameter space, 135 300 of them for
the UXClumpy parameter grid, which must be free of visible Monte Carlo noise. A single SKIRT
simulation covers all sight lines. The other parameters require either a separate simulation for
each combination of values, or a technique that derives the spectra for several parameter values
from a single set of simulations, as the response matrices of the original model do for the
incident spectrum. Each simulation is expected to take on the order of a day at the required
resolution, so that both reducing the number of simulations and reducing the run time of each
simulation are important parts of the project.

## Starting point

The project's first step, a spatial grid that
follows the clump geometry, has been implemented in SKIRT as the `ClumpySphericalSpatialGrid`. This
grid superposes a structured 3D spherical grid with a set of non-overlapping spherical clumps,
loaded from the same file that defines the clumps as particles for a `ParticleMedium`. Each clump is
a single cell, and the structured cells, reduced by the volume of any overlapping clumps, hold the
background medium. Paths through the grid are calculated exactly, without resolving the clumps
through refinement.

This note describes the next steps: reproducing the UXClumpy geometry in SKIRT, validating the
results, scoping and running the production simulations, and building the table model.

## Scope

This is not a design note in the strict sense: it describes a modeling project rather than changes
to the SKIRT code, and no code changes are currently planned. The scoping work may reveal features
that would make the production runs more efficient, in which case these are described as proposals.

Testing and scoping can largely be done on a local workstation. The production runs will be
performed on a high-performance computing facility.

## Overview

- **[Reference model](clumpy-torus-model/02-reference-model.md)** summarizes the UXClumpy model and
  the way it was calculated.
- **[Plan](clumpy-torus-model/03-plan.md)** describes the project steps, starting with those that
  can be done on a local workstation.
- **[Open questions](clumpy-torus-model/04-open-questions.md)** collects the questions that need to
  be answered before the production runs.

## References

- Buchner, J., et al. 2019, A&A, 629, A16:
  [UXClumpy paper](https://ui.adsabs.harvard.edu/abs/2019A%26A...629A..16B/abstract), including
  the angular and radial distributions of the clumps.
- [UXClumpy model description](https://github.com/JohannesBuchner/xars/blob/master/doc/uxclumpy.rst)
  and the [XARS code](https://github.com/JohannesBuchner/xars) used to calculate it.
- Vander Meulen, B., et al. 2023, A&A, 674, A123:
  [X-ray radiative transfer in SKIRT](https://ui.adsabs.harvard.edu/abs/2023A%26A...674A.123V/abstract),
  where Sect. 4.2.4 discusses details and limitations of XARS.
- [XSPEC](https://heasarc.gsfc.nasa.gov/docs/software/xspec/index.html), the X-ray spectral fitting
  package that uses the table model.
