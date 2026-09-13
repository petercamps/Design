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
fixed by SKIRT itself. The following two sections define, for each input, output, and
checkpoint case introduced in the Features chapter, exactly what its bundle contains.

## Input/output bundles

Each input and output file type described in the Features chapter needs an HDF5
representation. A type used on both the input and the output side gets a single
subsection below.

### Text column file

TODO: describe the HDF5 representation of text column file data (input and output).

### FITS file

TODO: describe the HDF5 representation of FITS data (input and output).

### AMR text file

TODO: describe the HDF5 representation of AMR text data (input).

### Stored table (.stab)

TODO: describe the HDF5 representation of stored table (.stab) data (input).

### Spatial grid plot file

TODO: describe the HDF5 representation of spatial grid plot data (output).

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
