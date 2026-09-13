# Data model

> Status: draft

## HDF5 concepts

The Features chapter introduced the basics of HDF5's file organization: a root group that
can contain further groups and datasets, addressed by a path much like a file path. This
section goes into more detail on the concepts SKIRT's data model actually relies on: how
objects are linked together, how a dataset's shape and type are described, small pieces of
attached metadata, and how data gets compressed on disk. The [HDF5 Data Model and File
Structure](https://support.hdfgroup.org/documentation/hdf5/latest/_h5_d_m__u_g.html)
reference documents all of this in full; this section only summarizes the parts most
relevant here.

### Groups and links

A group is HDF5's container object, conceptually a directory, and every file has exactly
one root group. Membership in a group is not intrinsic to the object it contains, but
implemented through a separate **link** that the group owns and that points to a named
object elsewhere in the file. HDF5 offers three kinds of links: **hard links**, the normal
case, which keep an object alive as long as at least one hard link to it still exists;
**soft links**, which store a path as a string and can therefore dangle if that path is
later removed; and **external links**, which point to an object in a different HDF5 file —
already introduced under Concurrency in the Output section, where they combine several
simulations' output files into one navigable hierarchy without copying any data.

### Datasets and dataspaces

A dataset is a rectangular, multidimensional array of same-typed data elements, stored
together with the metadata needed to interpret it. Its shape is described by a separate
object, created together with the dataset, called a **dataspace**. A dataspace records both
the current size of each dimension and its maximum size, which may be declared unlimited —
a capability the data model can draw on wherever a dataset needs to grow after it is first
created, rather than being written once in full.

### Datatypes

Every dataset and attribute also has a **datatype**, describing the layout of one data
element. HDF5 predefines the usual atomic types — signed and unsigned integers of various
widths, IEEE floating-point numbers, and both fixed- and variable-length strings — covering
almost everything SKIRT needs. Beyond these, HDF5 also supports **compound**
datatypes, similar to a C struct, letting several differently typed fields (for example, the
center coordinates and radius of a spherical clump) be stored together as a single element
per row, rather than as several parallel datasets. A frequently reused datatype can be
**committed** to the file under its own name, so that several datasets can share and refer
to one common definition instead of each repeating it.

### Attributes

Besides datasets, HDF5 also offers **attributes**: small, named pieces of metadata attached
to a group, a dataset, or a committed datatype. An attribute has its own datatype and
dataspace, just like a dataset, but is deliberately limited — it must be read or written in
one piece rather than partially, it cannot be compressed or resized after creation, and it
cannot itself carry further attributes. These restrictions keep attributes cheap enough to
use liberally, for example to record a dataset's units or a human-readable description,
without the overhead of a full dataset for each one.

### Chunking and compression

Most of HDF5's more advanced behavior, including compression, is configured through
**property lists**: named collections of settings supplied when an object is created or
accessed, rather than properties of the stored data itself.

By default, a dataset is stored **contiguously**, as one unbroken block, which is simplest
and fastest for data that is written once and read as a whole. Storing it **chunked**
instead — as equally sized, independently stored pieces — costs a little overhead, but is a
prerequisite for two other useful capabilities: making a dataset extendible, since a new
chunk only needs to be allocated once it is actually written, and applying a compression
filter, since HDF5 can only compress a dataset one chunk at a time.

The HDF5 library ships with several industry-standard compression methods built in, and
additional ones can be registered if needed.

### Bundles

Many of the things SKIRT itself treats as one self-contained named object — an input
file's worth of data, an output file, one component of a checkpoint — need more than a
single dataset to represent: several related quantities, each possibly of a different
type or shape, that all belong together. Rather than special-casing which of these map to
a literal HDF5 dataset and which need a group, SKIRT represents all of them the same way,
as a **bundle**: always an HDF5 group, containing one or more datasets whose names are
fixed by SKIRT itself.

Every bundle written by SKIRT also carries a `created` attribute: a string recording when
it was written, in the same ISO 8601-with-milliseconds format SKIRT already uses
elsewhere (e.g. `2026-09-03T10:53:22.305`). The `created` attribute is present whenever
SKIRT itself writes the bundle, but is never required of one it only reads.

The following sections define, for each input, output, and checkpoint case introduced
in the Features chapter, exactly what its bundle contains. Each is documented with two
tables: **Attributes**, small metadata attached directly to the bundle's group, and
**Datasets**, the arrays it actually contains — their names, dimensions, and datatypes.

## Input bundles

### Text column file

Wraps SKIRT's existing `TextInFile` convention: a table with one row per item and one
column per property, each column individually named, unit-tagged, and always stored as a
64-bit float — the type `TextInFile` already uses throughout. The bundle name is the plain
input filename that would otherwise have been used. Each column becomes its own 1-D
dataset, named after the column, rather than one shared 2-D array: this matches
the existing header convention (`# column N: description (unit)`) more directly than a flat
table would, and lets a Python reader address a column by name (e.g.
`bundle["mass_density"]`) instead of by position. All of a bundle's datasets must have the
same length. No separate row or column count is needed,
since an HDF5 dataset already carries its own length as part of its shape.

An HDF5 group has no equivalent to a text
file's inherent left-to-right column sequence, so each dataset also carries its own 1-based
`column` attribute to record its position explicitly.

**Attributes**

None required — `created` (see Bundles, above) is never read on the input side.

**Datasets** — one per column; names, units, and count come from the source data, not a
fixed schema. Shown here for a 4-column particle-import file,
where `N` is the number of rows:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `x` | (N) | 64-bit float | `column = 1`, `unit = "pc"` |
| `y` | (N) | 64-bit float | `column = 2`, `unit = "pc"` |
| `z` | (N) | 64-bit float | `column = 3`, `unit = "pc"` |
| `mass` | (N) | 64-bit float | `column = 4`, `unit = "Msun"` |

### AMR text file

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

None required — `created` (see Bundles, above) is never read on the input side.

**Datasets**

| Dataset | Dimensions | Type | Attributes / Description |
| --- | --- | --- | --- |
| `is_leaf` | (Nn + N) | boolean | One entry per node, in traversal order; `true` for a leaf. |
| `topology` | (Nn, 3) | 32-bit integer | One (Nx, Ny, Nz) split per `is_leaf = false` entry, same order. |
| one per leaf-cell property | (N) | 64-bit float | `column`, `unit` (as Text column file); one row per leaf. |

`N` is the number of leaf cells; `Nn` is the number of nonleaf (subdivided) nodes.

### Stored table (.stab)

TODO: describe the HDF5 representation of stored table (.stab) data.

### FITS file

Wraps SKIRT's existing `FITSInOut::read()` helper function (built on `cfitsio`) for
reading 2-D images and 3-D data cubes. The function recovers only the pixel/voxel
array and its dimensions from the file. None of the metadata a FITS file might otherwise
carry (such as pixel scale) is read; this information is configured in the ski file.

**Attributes**

None required — `created` (see Bundles, above) is never read on the input side.

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `image` | (ny, nx) or (nz, ny, nx) | 64-bit float | Pixel or voxel values. |

`image` is 2-D for `ReadFitsGeometry` or 3-D for `ReadFits3DGeometry`.

## Output bundles

Each output file type described in the Features chapter needs an HDF5 representation.

### Text column file

TODO: describe the HDF5 representation of text column file data written by SKIRT.

### FITS file

TODO: describe the HDF5 representation of FITS data written by SKIRT.

### Spatial grid plot file

TODO: describe the HDF5 representation of spatial grid plot data.

### Unstructured text file

TODO: describe the HDF5 representation of unstructured text output such as
`convergence.dat`.

### XML file

TODO: describe the HDF5 representation of the `parameters.xml` output.

### Log file

TODO: describe the HDF5 representation of the `log.txt` output.

## Checkpoint bundles

Each bundle listed under Checkpoint bundles in the Features chapter needs an HDF5
representation.

### Spatial grid

TODO: describe the HDF5 representation of the spatial grid checkpoint bundle.

### Medium state

TODO: describe the HDF5 representation of the medium state checkpoint bundle.

### Radiation field

TODO: describe the HDF5 representation of the radiation field checkpoint bundle.

### Recorded fluxes

TODO: describe the HDF5 representation of the recorded fluxes checkpoint bundle.
