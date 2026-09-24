# HDF5 input

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

## Features

### File types

SKIRT currently supports several input file types, each suited to a different kind of
source data:

- **Text column file** — a plain text table with one row per item (wavelength, particle,
  cell, …) and one column per property (luminosity, position, density, …).
  It is widely used for importing particle- or cell-based spatial distributions, and for
  specifying spectral energy distributions, mesh definitions, material properties and more.
- **AMR text** — describes a hierarchical, adaptively refined Cartesian grid together with
  its per-cell data. It is used for importing adaptive-mesh-refinement simulation output
  (`AdaptiveMeshSnapshot`).
- **Stored tables (.stab)** — a SKIRT-specific binary multi-axis table format heavily used
  for storing SKIRT's built-in resources. On the input side, it is sometimes used for defining
  multi-axis tables such as SED family templates (`FileSSPSEDFamily`) or
  polarized Stokes-vector tables (`FilePolarizedPointSource`).
- **FITS** — a binary image format widely used in astronomy. It is used to import two- or
  three-dimensional geometries (`ReadFitsGeometry` and `ReadFits3DGeometry`).

### Command-line syntax

The SKIRT command-line option that specifies the input location, `-i`, now accepts up to
three components combined into a single argument:

```
-i <dir>/<hdf>:<suite>
```

- `<dir>` — the input directory, exactly as before.
- `<hdf>` — optional: the name of an HDF5 file located inside `<dir>`, including the `.hdf5`
  extension.
- `<suite>` — optional, and only meaningful together with `<hdf>`: a path within the HDF5
  file, used as described below.

Only `<dir>` is required.

### How input files are located

If only `<dir>` is given, SKIRT behaves exactly as before: every input file it needs is
expected to be a plain file inside `<dir>`. (The ski file itself is not an input file in this
sense — its full path is given directly as a separate command-line argument, independent of
`-i`.)

If `<hdf>` is also given, SKIRT looks for each input file in two steps: first as a plain file
in `<dir>`, exactly as before; if that file is not found, SKIRT looks inside `<hdf>` instead,
for a bundle whose name matches the input file name.

If `<suite>` is given as well, it is prefixed to the input file name before that lookup, so
SKIRT looks for `<suite>/<bundle>` rather than `<bundle>` alone. This makes it possible to
store the input for several simulations inside a single HDF5 file, each under its own suite,
and select the right one per run through the `-i` option. On the other hand, multiple SKIRT
simulations may simply share the same input file, for example a common custom spectrum.

### Examples

Assuming a ski file `mysim.ski`, an input directory `in`, and an HDF5 file `data.hdf5`:

- `skirt -i in mysim.ski` — unchanged behavior: all input files are read from plain files in
  `in`.
- `skirt -i in/data.hdf5 mysim.ski` — input files are sought as plain files in `in` first,
  then as bundles in `in/data.hdf5`.
- `skirt -i in/data.hdf5:run01 mysim.ski` — as above, but bundles are looked up under the
  `run01` group, i.e. as `run01/<bundle>`.
- `skirt -i in/data.hdf5:campaign7/galaxy042 mysim.ski` — the suite can itself be a
  multi-level path, here selecting the input for one galaxy out of many stored in the same
  file.
- `skirt -i ./data.hdf5 mysim.ski` — edge case: the HDF5 file sits directly in the current
  directory (`<dir>` is `.`).

### Concurrency

Any number of independent SKIRT processes may open and read the same HDF5 file at the same
time, with no special mode or coordination needed.

### PTS

To help migrating existing input files, and to serve as a Python coding example, PTS will
be extended with functions and commands to convert the file types discussed above to
their corresponding HDF5 bundle form.

## Data model

### Bundles

The complete contents of a SKIRT input file are often more naturally represented as
several related datasets than as a single one. This document calls such a SKIRT-defined
named object a **bundle**. In the file, a bundle is always an HDF5 group containing one
or more datasets with names fixed by the SKRT data model.

Upon input, the HDF5 library performs reasonable data type conversions (e.g. between
32-bit and 64-bit number representations) and decompresses chunked or compressed data as
needed. This is fully transparent to SKIRT, so everything is fine as long as there is no
data loss (e.g. because a number doesn't fit in SKIRT's representation).

### Text column file bundle

Wraps SKIRT's existing `TextInFile` convention: a table with one row per item and one
column per property, each column individually named, unit-tagged, and always stored as a
64-bit float — the type `TextInFile` already uses throughout. The bundle name is the
plain input filename that would otherwise have been used. Each column becomes its own 1-D
dataset, named after the column, rather than one shared 2-D array: this matches the
existing header convention (`# column N: description (unit)`) more directly than a flat
table would, and lets a Python reader address a column by name (e.g.
`bundle["mass_density"]`) instead of by position. All of a bundle's datasets must have
the same length. No separate row or column count is needed, since an HDF5 dataset already
carries its own length as part of its shape.

An HDF5 group has no equivalent to a text file's inherent left-to-right column sequence,
so each dataset also carries its own 0-based `column` attribute to record its position
explicitly.

> SKIRT's text column files carry 1-based column numbers, while the corresponding HDF5 bundle
> carries 0-based column numbers, aligning with the programmatic standards in both Python and C++.

**Attributes**

None required.

**Datasets** — one per column; names, units, and count come from the source data, not a
fixed schema. Shown here for a 4-column particle-import file,
where `N` is the number of rows:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `x` | (N) | 64-bit float | `column = 0`, `unit = "pc"` |
| `y` | (N) | 64-bit float | `column = 1`, `unit = "pc"` |
| `z` | (N) | 64-bit float | `column = 2`, `unit = "pc"` |
| `mass` | (N) | 64-bit float | `column = 3`, `unit = "Msun"` |

### AMR text file bundle

The Adaptive Mesh Refinement (AMR) import file lists the nodes of a hierarchical tree
in Morton order: a depth-first, preorder traversal that visits a nonleaf node's children
row-major (x fastest, then y, then z), recursively at every level. In the text version
of the file, nonleaf nodes are represented by `!`-marked lines specifying the subdivision
counts. Leaf nodes carry their properties in the same column format as described in
the previous section. Nonleaf and leaf nodes are interleaved as a single stream in that
order. The file contains no cell positions or sizes; instead the domain box size
is configured in the ski file.

In the HDF5 bundle, the same information is represented by the two structural datasets
`is_leaf` and `topology`, plus a dataset for each property column. The `is_leaf` dataset
has one boolean entry per node (leaf and nonleaf) in the same order as the text lines.
The `topology` dataset lists the subdivision counts for each nonleaf node,
and each of the property datasets has a value for each leaf node.

**Attributes**

None required.

**Datasets**

| Dataset | Dimensions | Type | Attributes / Description |
| --- | --- | --- | --- |
| `is_leaf` | (Nn + N) | boolean | One entry per node, in traversal order; `true` for a leaf. |
| `topology` | (Nn, 3) | 32-bit integer | One (Nx, Ny, Nz) split per `is_leaf = false` entry, same order. |
| one per leaf-cell property | (N) | 64-bit float | `column`, `unit` (as Text column file); one row per leaf. |

`N` is the number of leaf cells; `Nn` is the number of nonleaf (subdivided) nodes.

### Stored table (.stab) bundle

Wraps SKIRT's existing `StoredTable<N>` format: a SKIRT-specific binary lookup table,
heavily used for SKIRT's own built-in resources (e.g. dust optical properties) and
occasionally supplied as input (e.g. SED family templates, polarized Stokes-vector tables).
A stored table defines one or more named, unit-tagged axes, each with its own grid of
points, and one or more named, unit-tagged quantities tabulated over the full grid formed
by all axes combined.

> SKIRT's built-in resources continue to be supplied in .stab format because using
> HDF5 would force a non-optional dependency on the HDF5 library.

Because the `.stab` format is designed to be memory-mapped directly rather than read through
regular file I/O, it also carries its own version tag, endianness tag, and end-of-file tag
— bookkeeping an HDF5 bundle does not need, since the container format already handles all
of that. Each axis becomes its own 1-D dataset, and each quantity becomes its own N-D
dataset shaped by the axis lengths, rather than reproducing the file's own interleaved,
quantity-fastest storage order chosen for memory-mapping locality — a Python reader gets a
plain, natural NumPy array per quantity instead.

An HDF5 group does not preserve the order in which its datasets were created, and matching
dimensions up by length is not reliable either, since two axes can happen to share the same
number of grid points. Each axis dataset therefore also carries a 0-based `axis` attribute,
giving its position among a quantity dataset's dimensions; a dataset with no `axis`
attribute is a quantity, not an axis.

**Attributes**

None required.

**Datasets** — shown here for a 3-axis, 1-quantity SED template file:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `lambda` | (1221) | 64-bit float | `axis = 0`, `unit = "m"`, `log = true` |
| `Z` | (6) | 64-bit float | `axis = 1`, `unit = "1"`, `log = true` |
| `t` | (67) | 64-bit float | `axis = 2`, `unit = "yr"`, `log = true` |
| `Llambda` | (1221, 6, 67) | 64-bit float | `unit = "W/m"`, `log = true` |

Dimension `i` of a quantity dataset corresponds to the axis dataset with `axis = i` — here,
dimension 0 to `lambda`, dimension 1 to `Z`, and dimension 2 to `t`. A table with several
quantities gets one dataset per quantity, all sharing the same axis datasets; `log` records
whether that axis or quantity interpolates logarithmically.

### FITS file bundle

Wraps SKIRT's existing `FITSInOut::read()` helper function (built on `cfitsio`) for
reading 2-D images and 3-D data cubes. The function recovers only the pixel/voxel
array and its dimensions from the file. None of the metadata a FITS file might otherwise
carry (such as pixel scale) is read; this information is configured in the ski file.

**Attributes**

None required.

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `image` | (ny, nx) or (nz, ny, nx) | 64-bit float | Pixel or voxel values. |

`image` is 2-D for `ReadFitsGeometry` or 3-D for `ReadFits3DGeometry`.

## Implementation

### Build option

The SKIRT build procedure offers a CMake option called `BUILD_HDF5`. When enabled, the
build procedure locates and links the official C HDF5 library, which should already be
installed on the user's system. When disabled (the default), SKIRT builds without any
dependency on HDF5.

A new `hdf5` build target implements a thin C++ wrapper around the C HDF5 library,
offering just the features used by SKIRT. All code elsewhere in SKIRT depends only on
this target's C++ API, never directly on the HDF5 library. In case `BUILD_HDF5` is
disabled, the wrapper implements stub functions that throw a fatal error as soon as the
caller attempts to open an HDF5 file. Note that SKIRT does not use the C++ wrapper layer
that HDF5 also offers; this would only complicate matters in view of the limited
functionality actually needed.

### Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to the `hdf5`
build target. The library cannot be assumed to be thread-safe on its own, since that
depends on how the installed HDF5 happens to have been built, which is outside SKIRT's
control.

### C++ API

The C++ API offered by the `hdf5` target is a small set of classes representing the
concepts defined in the Data model section — suite, bundle, dataset — plus the factory
methods that construct them. Each object acquires its underlying HDF5 resource on
construction and releases it in its destructor. Objects can be moved but not copied.

None of the objects represents an open file: every factory method resolves a full
`<filepath>:<suite>/...` path directly, and file-handle management is this target's own
internal concern. The class names are suffixed with `R` for read-only so that future
extensions can differentiate between reading and writing.

**`H5Lib`** is not instantiated; it offers only static factory methods, the API's sole
file-opening entry points — nothing else in this API ever names a file path directly. Every
method below that takes a `<name>` argument, rather than a full `<filepath>:<suite>/...`
path, resolves that name relative to the object the method is called on.

- `bool available()` — true if this build was compiled with `BUILD_HDF5`.
- `H5SuiteR openSuite(<filepath>:<suite>)` — opens a suite for reading.
- `H5BundleR openBundle(<filepath>:<suite>/<bundle>)` — opens a bundle for reading
  directly, without going through its suite first.

**`H5SuiteR`**

- `vector<string> getBundleNames()` — the names of every bundle directly in this suite,
  excluding checkpoints.
- `H5BundleR openBundle(<name>)` — opens one of this suite's bundles for reading.

**`H5BundleR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — one overload per supported attribute type (boolean, 32-bit
  integer, 64-bit float, string).
- `bool hasDataset(<name>)`
- `H5DatasetR getDataset(<name>)` — opens one of this bundle's datasets for reading.

**`H5DatasetR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — as `H5BundleR::getAttribute()`, above.
- `read()` — one overload per supported dataset content type (boolean, 32-bit integer,
  64-bit float), returning the dataset's full contents at whatever shape it was created
  with.

### Object lifetime

Any object this API hands out remains fully valid after the object it came from has been
destroyed. For example, an `H5Dataset` does not depend on its `H5Bundle` still being alive.
The HDF5 C library reference-counts its own resources internally, so a file
stays genuinely open, at the library level, for as long as any handle into it remains open,
and only actually closes once the last handle does. It is thus allowed to let an `H5Bundle`
go out of scope as soon as the single `H5Dataset` actually needed has been obtained from it.

This is not, however, a license to lose track of things: every object this API hands out must
still eventually be destroyed by the client, and until that has happened for everything ever
opened against a given file, that file remains genuinely open. This target places that
responsibility on the client rather than tracking open descendants itself.

### Supported data types

Attributes (on `H5Bundle` and `H5Dataset`) and dataset contents share three data types:
boolean, 32-bit integer, and 64-bit float. Attributes additionally support `string`,
passed as a `std::string` in UTF-8.

Reading returns the data type the caller requests and leans on HDF5's own conversion
machinery for however the value is actually stored on disk, as described under Bundles in
the Data model section.

### Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, suite, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the HDF5 target described in the previous section. To
handle this, the `FilePaths` class's API is adjusted as follows.

- `setInputPath(string value)` — the value of the `-i` option: a plain directory path, or a
  directory plus an HDF5 file and suite.
- `input(string name)` returns a pair of strings: a plain-file form and an HDF5 form.

Given the value of a ski file's `filename` attribute, or a programmatically fabricated
filename or bundle name, the `input` function combines that name with the setup value
taken from the command line to construct fully resolved locations. The first string of
the returned pair is the absolute canonical path for a plain file corresponding to
`name`, and the second is an HDF5 file path plus a resolved bundle path corresponding to
`name`; either string is empty if there is no such correspondence for that side. The
function does no I/O of its own, so a non-empty string is never a guarantee that the file
or bundle it names exists.

These functions throw a fatal error if the passed argument string implies HDF5 is needed
but `H5Lib::available()` (above) returns false.

The following tables illustrate various combinations, assuming every file sits in the
current directory and ignoring the canonical-path detail in the returned strings —
resolving absolute and relative paths is a well-understood problem, not what these
examples are about.

`input("import.txt")` resolves as follows:

| `-i` | Plain-file candidate | HDF5 candidate |
| --- | --- | --- |
| `.` | `import.txt` | — |
| `./data.hdf5` | `import.txt` | `data.hdf5:import.txt` |
| `./data.hdf5:run1` | `import.txt` | `data.hdf5:run1/import.txt` |
| `./data.hdf5:campaign7/run1` | `import.txt` | `data.hdf5:campaign7/run1/import.txt` |

A `name` argument can itself embed a `<hdf>:<suite>/...` reference. When it does, `input()` resolves the
HDF5 candidate directly from that embedded reference, regardless of what `-i` itself
specifies — even if `-i` names no HDF5 file at all, or a different one; only `-i`'s
directory component still applies, exactly as for any other input file.
`CheckpointTreePolicy::filename` requires this, since it must be able to point at a
checkpoint in a file unrelated to whatever `-i` is reading from, but any ski file property
naming an input file may use the same mechanism:

`input("other.hdf5:campaign/mysim_checkpoint_primary_2")` resolves as follows:

| `-i` | Plain-file candidate | HDF5 candidate |
| --- | --- | --- |
| *(not given)* | — | `other.hdf5:campaign/mysim_checkpoint_primary_2` |
| `.` | — | `other.hdf5:campaign/mysim_checkpoint_primary_2` |
| `./data.hdf5:run1` | — | `other.hdf5:campaign/mysim_checkpoint_primary_2` |

For both forms, the caller is expected to try the plain-file candidate first, falling back
to the HDF5 one only if that file does not exist.


### Column input

**`ColumnInFile`** replaces direct `TextInFile` use at every call site that reads a genuine
input text column file today (a Text column file bundle, per the Data model chapter,
when HDF5 applies). A resource file never goes through this class — resource lookup never
involves HDF5 at all, so it keeps using `TextInFile` directly.

At construction, `ColumnInFile(const SimulationItem* item, string filename, string description)`
only records its arguments; unlike `TextInFile`, it does not open anything yet.
The `description` argument is only used for logging. `useColumns()` and `addColumn()` are
called next, same as today. Once all `addColumn()` calls have happened,
triggered by the first `readRow()`/`readAllRows()`/`readAllColumns()` call,
the filename is resolved by calling `FilePaths::input(filename)`.
If the plain-file candidate exists, it is opened through an internally held `TextInFile`.
If not, the HDF5 candidate is opened as a bundle and each declared column is read from its own named dataset.
Matching datasets with column names happens in the same way as for a plain text column file.
`close()` and the destructor release whichever of the two
was actually opened. `readNonLeaf()` has no counterpart here; AMR input needs a different
mechanism, covered in its own subsection below.

Call sites that construct a `TextInFile` for a genuine input file all move to
`ColumnInFile`:

- `Snapshot` (and snapshot subclasses: `CellSnapshot`,
  `ParticleSnapshot`, `CylindricalCellSnapshot`, `SphericalCellSnapshot`,
  `VoronoiMeshSnapshot`) — imports per-particle or per-cell properties.
- `VoronoiMeshSnapshot` — also reads site positions directly, separately from the shared
  `Snapshot` mechanism above.
- `FileWavelengthGrid`, `FileBorderWavelengthGrid` — wavelength grids.
- `FileSED`, `FileLineSED`, `FileGaussianLinesSED` — spectral energy distributions.
- `FileBand` — transmission curves.
- `FileWavelengthDistribution` — wavelength probability distributions.
- `FileGrainSizeDistribution` — grain size distributions.
- `FileMesh` — mesh border points.
- `TetraMeshSpatialGrid` — tetrahedral vertices.
- `ClumpySphericalSpatialGrid` — clump centers and radii.
- `MultiGaussianExpansionGeometry` — expansion parameters.
- `MeanFileDustMix` — optical dust properties.
- `NonLTELineGasMix` — initial level populations.
- `AtPositionsForm` — probe sample positions.

`AdaptiveMeshSnapshot` does not use this mechanism at all: since its topology and properties
are read from a single interleaved stream, it needs its own `AdaptiveMeshInFile` for the
whole file, covered under AMR input, below.

`GasLineEmission` and `XRayAtomicGasMix` construct `TextInFile` with `resource = true` at
several call sites each; none of those move, since built-in resources never use
`ColumnInFile`.

### AMR input

**`AdaptiveMeshInFile`** replaces the `TextInFile` used today to parse the interleaved topology-and-property
stream `readAndClose()` triggers via `new Node(extent, infile(), cells)`. It keeps
`TextInFile::readNonLeaf(int& nx, int& ny, int& nz)`'s exact signature and behavior — peek
the next line or entry; if it is a nonleaf specification, consume it and return true;
otherwise leave the position untouched for the following `readRow()` call and return false —
so `Node`'s recursive constructor needs no change beyond the type of the pointer it holds.
`useColumns()`/`addColumn()`/`readRow()`, for leaf-cell properties, keep their `ColumnInFile`
signatures.

On the plain-text branch, `AdaptiveMeshInFile` is a thin wrapper around one internally held
`TextInFile`, unchanged from today. On the HDF5 branch, at first use it reads the bundle's
whole `is_leaf` and `topology` datasets into memory and then walks `is_leaf` with an internal position counter.
`readNonLeaf()` consumes the next `topology` row and returns true if the current entry is
`false` (nonleaf), or returns false without consuming anything otherwise; `readRow()` reads
the next row from each declared property dataset, using a separate counter over the `N` leaf
entries only, exactly as the Data model chapter's AMR text file bundle specifies.

Call sites: `AdaptiveMeshSnapshot`.

### Stored table input

A wrapper is needed here too, to branch between the plain `.stab` file and the HDF5 bundle,
resolved via `FilePaths::input()` as elsewhere in this chapter.

The data in a stored table input bundle cannot be memory mapped (as for a regular `.stab` file)
for several reasons:

- The quantity values are ordered differently (interleaved in `.stab`, per dataset in bundle).
- The bundle datasets may be compressed and/or chunked.
- (In a possible future) If the same HDF file is used for input and output, memory mapping
  a portion of the file is unsafe because writing to the file might relocate its contents.

The StoredTable constructor for a bundle must therefore copy the data into newly allocated
memory, and the destructor must deallocate that memory.

Call sites that open a genuine input stored table (`resource = false`) move to this wrapper:

- `FileIndexedSEDFamily` — an indexed SED family template.
- `FileSSPSEDFamily` — SSP SED family templates, with or without an ionization-parameter axis.
- `FilePolarizedPointSource` — the four Stokes-parameter (I/Q/U/V) tables of a polarized
  point source.

Every other `StoredTable::open()` call site uses `resource = true` (the default) and stays
untouched.

### FITS input

**`FITSInOutFile`** replaces `FITSInOut` wherever a call site reads or writes a genuine FITS
file (a FITS bundle, per the Data model chapter, when HDF5 applies), as static functions
wrapping the existing, cfitsio-backed `FITSInOut` static functions. Unlike
`ColumnInFile`/`ColumnOutFile`, no deferred-open or buffering is needed: every `FITSInOut`
call already carries, or produces, the complete array in one shot, so the shape an HDF5
dataset needs at creation is always already known.

`read()` keeps `FITSInOut::read()`'s exact signature. It resolves `FilePaths::input(filename)`
and either delegates straight to the existing cfitsio-backed implementation for the plain-file
candidate, or opens the HDF5 candidate as a bundle and reads its `image` dataset — its shape
supplies `nx`/`ny`/`nz` — for the FITS input bundle already specified in the Data model
chapter.

`FITSInOut::read()` supports `[EXTNAME]`-style addressing for picking a non-primary data unit
out of one FITS file. This has no equivalent in the HDF5 bundle scheme; instead each FITS
data unit should be placed in its own individual HDF5 bundle.

Call sites:

- `ReadFitsGeometry` — a 2-D image.
- `ReadFits3DGeometry` — a 3-D data cube.


