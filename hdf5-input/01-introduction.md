# Introduction

> Status: draft
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md)

## Motivation

SKIRT is increasingly used for large campaigns that post-process thousands of simulated
galaxies obtained from cosmological simulations. Each galaxy is typically represented by
tens to hundreds of thousands of smoothed particles or grid cells. Passing this
information along as text-based files becomes slow and unwieldy.

The solution proposed here is to adopt HDF5 as an optional input format.
[HDF5](https://www.hdfgroup.org) is a binary, self-describing, hierarchical format
designed for large scientific datasets. It is compact and efficient to read and write;
and a single HDF5 file can contain many distinct datasets organized in a nested structure
similar to a filesystem. Datasets and groups can carry attributes, small pieces of
metadata such as units or other descriptors, stored directly alongside the data they
describe.

HDF5 files can be viewed directly, without writing any code, using
[HDFView](https://www.hdfgroup.org/download-hdfview/), a free graphical browser
distributed by the HDF Group. HDFView offers limited editing capabilities, such as
updating a given value "in place" or deleting a complete dataset, but generally
modifications to an HDF5 file should be performed programmatically. Importantly, HDF5 is
easily accessible from Python via the mature `h5py` package, whose interface closely
mirrors NumPy arrays.

It is _not_ the intention for SKIRT to directly import data from HDF5 snapshot files. A
pre-processing procedure must extract the relevant information from the snapshot,
possibly split the data into multiple input sets (e.g. star-forming and evolved stellar
populations), derive any missing properties (e.g. dust density or optical properties),
and construct the columns required by each of the SKIRT simulation's source and medium
components. Compared to text column files, the benefits of HDF5 include performance (both
writing and reading) and the option to compress the data if storage size is an issue.

## HDF5 concepts

An HDF5 file is organized like a small filesystem. It has a root group, which can contain
further groups as well as datasets — the actual blocks of stored data. A dataset is
addressed by a path within the file, built from the names of the groups leading up to it,
similar to a file path.

A dataset is a rectangular, multidimensional array of same-typed data elements, stored
together with the metadata needed to interpret it. Its shape is described by a separate
object, created together with the dataset, called a **dataspace**. Every dataset also has
a **datatype**, describing the layout of one data element. HDF5 predefines the usual
atomic types — signed and unsigned integers of various widths, IEEE floating-point
numbers, and both fixed- and variable-length strings — covering everything SKIRT needs.

Besides datasets, HDF5 also offers **attributes**: small, named pieces of metadata
attached to a group or a dataset. An attribute has its own datatype and dataspace, just
like a dataset, but is deliberately limited. For example, it cannot be compressed and it
cannot itself carry further attributes. These restrictions keep attributes cheap enough
to use liberally, for example to record a dataset's units or a human-readable
description, without the overhead of a full dataset for each one.

By default, a dataset is stored **contiguously**, as one unbroken block, which is
simplest and fastest for data that is written once and read as a whole. Storing it
**chunked** instead — as equally sized, independently stored pieces — costs a little
overhead, but is a prerequisite for two other useful capabilities: making a dataset
extendible, since a new chunk only needs to be allocated once it is actually written, and
applying a compression filter, since HDF5 can only compress a dataset one chunk at a
time. The HDF5 library ships with several industry-standard compression methods built in.

The [HDF5 Data Model and File
Structure](https://support.hdfgroup.org/documentation/hdf5/latest/_h5_d_m__u_g.html)
reference documents all of this in full.

## Overview

- **[Features](hdf5-input/02-features.md)** describes the supported file types, the command-line
  syntax, and the way input files are located.
- **[Data model](hdf5-input/03-data-model.md)** specifies the HDF5 bundle that corresponds to each
  input file type.
- **[Implementation](hdf5-input/04-implementation.md)** describes the `hdf5` build target, its C++
  API, the name resolution, and the changes to the input call sites.
