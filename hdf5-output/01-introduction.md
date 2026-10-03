# Introduction

> Status: under construction
>
> Depends on: [HDF5 input](hdf5-input/01-introduction.md)

## Motivation

SKIRT is increasingly used for large campaigns that post-process cosmological simulations,
generating synthetic observations for thousands of simulated galaxies. A SKIRT run typically
produces many separate output files, and HPC file systems handle large numbers of files poorly —
metadata operations such as opening and listing become a bottleneck, and many centers impose
hard quotas on file counts, independent of total data volume.

The [HDF5 input](hdf5-input/01-introduction.md) note proposes HDF5 as an optional input format.
This note extends that proposal to output: SKIRT optionally writes all of its output files as
bundles in a single HDF5 file. The input and output of a simulation, or the output of several
simulations, can then be collected in one file. HDF5 is easily accessible from Python — widely
used in the community for analysis and pipeline scripting — via the mature `h5py` package, and
HDF5 files can be inspected without writing any code using
[HDFView](https://www.hdfgroup.org/download-hdfview/).

The concepts, the command-line syntax, the bundles, and the `hdf5` build target introduced in the
HDF5 input note are assumed here and extended where needed.

## Compatibility

The features described in this note are additions to SKIRT. Existing output and command-line
behavior remains unchanged, and the new HDF5-based functionality is layered on top in a
backward-compatible way.

## Impact on PTS

These features will also require updates to PTS, the Python Toolkit for SKIRT — mostly its
functionality for visualizing and testing SKIRT output. Working out these changes, however, is out
of scope for this note.

## Overview

- **[Features](hdf5-output/02-features.md)** describes the output file types, the command-line
  syntax, and the way output files are written. It is aimed at SKIRT users.
- **[Data model](hdf5-output/03-data-model.md)** specifies the HDF5 bundle that corresponds to each
  output file type. It is aimed at authors of Python scripts that consume SKIRT output.
- **[Implementation](hdf5-output/04-implementation.md)** describes the extensions to the `hdf5`
  build target and the name resolution, and the changes to the output call sites.
