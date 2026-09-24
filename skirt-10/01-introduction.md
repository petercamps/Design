# Introduction

> Status: draft

## Motivation

SKIRT 9 was publicly released on May 10, 2019. In the seven years since, both its physics
capabilities and its supporting infrastructure have grown substantially. On the physics
side, major additions include a full Lyman-alpha photon cycle, polarized secondary
emission by aligned dust grains, gas emission with self-consistent non-LTE line transfer,
X-ray photo-absorption, fluorescence and Compton scattering, 21 cm hydrogen spin-flip
line emission, treatment of diffuse ionized gas, and new SED templates for stellar
populations (BPASS) and star formation regions (TODDLERS). On the infrastructure side,
the probe system was redesigned around composable probe forms, the dynamic medium state
execution flow was substantially revised, and new spatial grid types — cylindrical,
spherical, and tetrahedral — were introduced.

This ongoing evolution, together with a paper in preparation describing SKIRT's current
capabilities, makes this a natural point to declare a new major version — timing the
bump to coincide with the paper gives its announcement some extra weight. There are more
practical reasons too. Users increasingly need a stable, citable label for the code
version they used in a given study — "SKIRT 9.2.1" rather than a git commit hash — and a
new major version is a natural point to put that labeling on a proper footing. It is
also, as with any major version bump, the point at which removing little-used features
or introducing other incompatible changes is expected, rather than something to keep
postponing.

## Repositories and versioning

One complication stands in the way: the current repositories are named after the major
version they hold — `SKIRT9`, `PTS9`, and so on — which does not sit well with declaring
a new one. This document proposes moving to SKIRT 10, renaming the repositories to
generic, version-independent names, and putting the version label inside each repository
instead of in its name. Development then needs a workflow that supports multiple major
versions in parallel — maintaining SKIRT 9 for existing users while developing SKIRT
10 — within that same, generically named set of repositories.

## Overview

The remaining chapters are organized as follows:

- **[Organization and workflow](skirt-10/02-organization-and-workflow.md)** describes the
  proposed repository restructuring and the workflow for developing SKIRT 9 and SKIRT 10
  in parallel.
- **[System requirements](skirt-10/03-system-requirements.md)** lists what is needed to
  build and run SKIRT 10, including the new optional HDF5 dependency.
- **[Incompatibilities](skirt-10/04-incompatibilities.md)** lists the features dropped
  and the ski file changes required when upgrading from SKIRT 9.
- **[Tree-based spatial grids](skirt-10/05-tree-based-spatial-grids.md)** proposes a
  restructuring of the hierarchical tree classes, described separately because of its
  scope.
- **[HDF5 input](skirt-10/06-hdf5-input.md)** proposes optional HDF5 support for
  simulation input.
