# Introduction

> Status: draft

## Motivation

SKIRT is increasingly used for large campaigns that post-process cosmological simulations,
generating synthetic observations for thousands of simulated galaxies. Each galaxy is
typically represented by tens to hundreds of thousands of smoothed particles or grid cells,
so both the input and output data volumes scale accordingly.

This scale exposes two limitations of SKIRT's current file handling. On the input side,
text-based files become large and slow to parse. On the output side, a SKIRT run typically
produces many separate files, and HPC file systems handle large numbers of files poorly —
metadata operations such as opening and listing become a bottleneck, and many centers impose
hard quotas on file counts, independent of total data volume.

The solution proposed in this document is to adopt HDF5 as an optional input and output
format. [HDF5](https://www.hdfgroup.org) is a binary, self-describing, hierarchical format
designed for large scientific datasets. It is compact and efficient to read and write; a
single HDF5 file can contain many distinct datasets organized in a nested structure similar
to a filesystem. Datasets and groups can carry attributes, small pieces of metadata such as
units or other descriptors, stored directly alongside the data they describe.

HDF5 files can be viewed and adjusted directly, without writing any code, using
[HDFView](https://www.hdfgroup.org/download-hdfview/), a free graphical browser and editor
distributed by the HDF Group. HDF5 is also easily accessible from Python — widely used in the
community for analysis and pipeline scripting — via the mature `h5py` package, whose
interface closely mirrors NumPy arrays.

A further benefit is that HDF5 is well suited to checkpointing — periodically saving the
complete internal state of a running simulation so that it can be resumed after an
interruption or restarted from a known point. The same properties that make HDF5 attractive
for input and output — a single, self-describing file holding multiple structured datasets —
make it equally suitable for capturing that state.

It is _not_ the intention for SKIRT to directly import data from HDF5 snapshot files. A
pre-processing procedure must extract the relevant information from the snapshot, possibly
split the data into multiple input sets (e.g. star-forming and evolved stellar
populations), derive any missing properties (e.g. dust density or optical properties), and
construct the columns required by each of the SKIRT simulation's source and medium
components. Compared to text column files, the benefits of HDF5 include performance (both
writing and reading) and the option to compress the data if storage size is an issue.

## Compatibility

The features described in this document are additions to SKIRT. With the exceptions noted
below, existing input, output, and command-line behavior remains unchanged, and the new
HDF5-based functionality is layered on top in a backward-compatible way.

Support for the following less-frequently-used file formats is dropped and replaced by
superior HDF5 storage mechanisms:

- The `scol` format, a SKIRT-specific binary alternative to regular text column files.
- The specialized text format used for saving and re-loading spatial grid tree topology.
  This also changes the way a `TreeSpatialGrid` is configured in ski files.

## Impact on PTS

These features will also require updates to PTS, the Python Toolkit for SKIRT — mostly its
functionality for visualizing and testing SKIRT output. They may also enable new PTS
capabilities that take advantage of the additional information now available, such as
checkpoints. Working out these changes, however, is out of scope for this document.

## Overview

The remaining chapters are organized as follows:

- **[Features](02-features.md)** describes the proposed HDF5 input, output, and checkpoint
  features and how to use them — prerequisites, command-line options, and ski file settings.
  It is aimed at SKIRT users.
- **[Data model](03-data-model.md)** specifies how SKIRT data is represented in HDF5: the
  overall structure, the detailed layout that replaces today's text input/output files and
  FITS files, and the checkpoint bundles that capture a running simulation's internal
  state. It is aimed at authors of Python scripts that prepare or consume SKIRT
  input/output. Also, together with the features description in the previous chapter, it
  forms the basis for the implementation in SKIRT.
- **[Implementation](04-implementation.md)** describes how HDF5 support will be implemented
  in SKIRT: the architecture and the code changes and additions involved. It is aimed at
  those carrying out that implementation.
