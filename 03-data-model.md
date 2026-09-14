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

Every bundle written by SKIRT carries three standard attributes, never required when
SKIRT only reads a bundle:

| Attribute | Type | Description |
| --- | --- | --- |
| `producer` | string | SKIRT version and build that wrote the bundle (as in SKIRT's welcome message). |
| `format` | 32-bit integer | Version of this bundle format itself, for future changes; `1` for now. |
| `created` | string | When the bundle was written, as an ISO 8601-with-milliseconds string. |

For example, `producer` might read `SKIRT v9.0 (git 584edd6 built on 03/09/2026 at
10:51:20)`, and `created` might read `2026-09-03T10:53:22.305`. Individual bundle
descriptions below list only attributes beyond these three.

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

None required.

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

None required.

**Datasets**

| Dataset | Dimensions | Type | Attributes / Description |
| --- | --- | --- | --- |
| `is_leaf` | (Nn + N) | boolean | One entry per node, in traversal order; `true` for a leaf. |
| `topology` | (Nn, 3) | 32-bit integer | One (Nx, Ny, Nz) split per `is_leaf = false` entry, same order. |
| one per leaf-cell property | (N) | 64-bit float | `column`, `unit` (as Text column file); one row per leaf. |

`N` is the number of leaf cells; `Nn` is the number of nonleaf (subdivided) nodes.

### Stored table (.stab)

Wraps SKIRT's existing `StoredTable<N>` format: a SKIRT-specific binary lookup table,
heavily used for the library's own built-in resources (e.g. dust optical properties) and
occasionally supplied as input (e.g. SED family templates, polarized Stokes-vector tables).
A stored table defines one or more named, unit-tagged axes, each with its own grid of
points, and one or more named, unit-tagged quantities tabulated over the full grid formed
by all axes combined.

Because the format is designed to be memory-mapped directly rather than read through
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

### FITS file

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

## Output bundles

### Text column file

Wraps the same `TextInFile`/`TextOutFile` convention as the input side (see Text column
file under Input bundles, above), used for SEDs, per-cell and per-position probe output,
and instrument statistics tables. Every output column is written as a 64-bit float and
auto-numbered in the order `addColumn()` is called, so the same per-dataset `column` and
`unit` attributes apply.
Most call sites also write a free-form comment as the very first line, by convention rather
than by any mechanism `TextOutFile` itself enforces (via the public `writeLine()`, before
adding columns) — this becomes the bundle's own `description` attribute.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `description` | string | Free-form comment describing the output. |

**Datasets** — one per column, as on the input side. Shown here for a 2-column SED output
file, where `N` is the number of rows:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `lambda` | (N) | 64-bit float | `column = 1`, `unit = "micron"` |
| `F_nu` | (N) | 64-bit float | `column = 2`, `unit = "Jy"` |

### FITS file

Wraps SKIRT's existing `FITSInOut::write()` and `FITSInOut::writeMap()` helpers (built on
`cfitsio`) for 2-D data frames and 3-D data cubes (a stack of frames along a third axis).
These are used for instrument fluxes (IFUs and STMs) and statistics, and for planar
cuts or projections produced by probes.

To limit storage requirements, data values are stored at 32-bit precision just like in
FITS output files. The three axis coordinate values are the equivalent of information
otherwise stored in the FITS header and/or in FITS table extensions named `GRID_POINTS`.
The quantities represented by data and axes differ for the various use cases, as shown
in the table below. The axis and data datasets each carry attributes defining the
quantity type and unit.

| Output type | x-axis | y-axis | z-axis | data |
| --- | --- | --- | --- | --- |
| Flux IFU | spatial | spatial | spectral | flux |
| Statistics for flux IFU | spatial | spatial | spectral | contribution moments |
| Flux STM | spectral | time lag | - | flux |
| Statistics for flux STM | spectral | time lag | - | contribution moments |
| Planar cut or projection | | | | |
| ... for scalar quantity | spatial | spatial | - | ? |
| ... for velocity | spatial | spatial | (vx, vy, vz) | velocity |
| ... for spectral grid | spatial | spatial | spectral | ? |

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `description` | string | Free-form comment describing the output. |

Only for distant-instrument IFUs:

| Attribute | Type | Description |
| --- | --- | --- |
| `inclination` | 64-bit float, degrees | Viewing inclination (FITS `CROTA1`). |
| `azimuth` | 64-bit float, degrees | Viewing azimuth (FITS `CROTA2`). |
| `roll` | 64-bit float, degrees | Viewing roll angle (FITS `CROTA3`). |
| `redshift` | 64-bit float | Redshift of the source (FITS `REDSHIFT`). |
| `luminosity_distance` | 64-bit float | Luminosity distance to the source (FITS `DISTLUMI`). |
| `angular_diameter_distance` | 64-bit float | Angular-diameter distance to the source (FITS `DISTANGD`). |
| `distance_unit` | string | Unit of the two distance attributes (FITS `DISTUNIT`). |

The remaining FITS header information is represented by the datasets and their attributes,
as listed below, and by the `producer` and `created` attributes carried by the bundle.

**Datasets** — shown here for a 3-D IFU data cube produced by a FrameInstrument:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `x` | (nx) | 64-bit float | `quantity = "length"`, `unit = "pc"` |
| `y` | (ny) | 64-bit float | `quantity = "length"`, `unit = "pc"` |
| `z` | (nz) | 64-bit float | `quantity = "wavelength"`, `unit = "micron"` |
| `data` | (ny, nx) or (nz, ny, nx) | 32-bit float | `quantity = "frequencysurfacebrightness"`, `unit = "MJy/sr"` |

If `z` is missing or has a single value, `data` is 2-D representing a single data frame.
If `z` is present with 2 or more values, `data` is 3-D representing a data cube.

### Spatial grid plot file

Wraps SKIRT's existing `SpatialGridPlotFile` helper (itself a thin wrapper around
`TextOutFile`), used by every spatial grid to plot its own cell geometry: 2-D plane-cut
variants (`_grid_xy`, `_grid_xz`, `_grid_yz`) and a 3-D variant (`_grid_xyz`). Unlike the
other `TextOutFile`-based formats, it writes no header at all — just raw coordinate pairs
or triples, one point per line, with a blank line marking the start of a new, disconnected
polyline (a "moveto" in the class's own terminology; consecutive points with no blank line
between them are connected, a "lineto"). No unit is recorded in the plain-text file itself,
even though the coordinates do have one (the simulation's configured output length unit) —
the bundle adds an explicit `unit` attribute to close that gap.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `unit` | string | Length unit of every `points` coordinate — whatever the simulation's output uses. |

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `points` | (N, 2) or (N, 3) | 64-bit float | One coordinate pair or triple per row. |
| `moveto` | (N) | boolean | `true` marks the start of a new polyline; always `true` for row 0. |

`N` is the total number of points across every polyline in the file.

### Unstructured text file

Covers three otherwise-unrelated plain-text output types — `convergence.dat` (free-form
human-readable text written by `ConvergenceInfoProbe`), `parameters.xml` (a reformatted
copy of the ski file), and `log.txt` (progress, warning, and error messages) — each stored
the same way: as a single string holding the entire file's content, unparsed. A `type`
attribute records which of the three it is.

The log file is the one special case. Because it grows throughout the run and users should
be able to watch progress in real time, it continues to be written incrementally to a
regular file during the simulation, exactly as today (see Log file under Output in
Features, above); only the finished, complete text is added to its bundle at the very end
of the run — the bundle itself is never updated incrementally.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `type` | string | One of `free form`, `XML`, or `log`. |

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `text` | scalar | variable-length string | The entire file's content, unparsed. |

## Checkpoint bundles

### Checkpoint bundle

A checkpoint bundle is a bundle of bundles: its HDF5 group holds, as direct members, some
or all of the four checkpoint bundles described in the following sections (Spatial grid,
Medium state, Radiation field, Recorded fluxes) — never a dataset of its own. Which of the
four are actually present depends on what data the simulation has available at that point
(see Checkpoint probe behavior in the Features chapter).

A checkpoint's bundle name follows the usual `<prefix>_` convention, extended with the
"when" point and the iteration index that together identify it:

```
<prefix>_checkpoint_<when>_<iteration>
```

`<when>` is one of `setup`, `primary`, `secondary`, or `run`, matching the four points
listed under The checkpoint probe in the Features chapter. `<iteration>` is the one-based
primary- or secondary-emission iteration index for a `primary` or `secondary` checkpoint,
and `0` for `setup` and `run`, which each occur only once per simulation. For example, a
simulation iterating over both primary and secondary emission could produce:

```
mysim_checkpoint_setup_0
mysim_checkpoint_primary_1
mysim_checkpoint_primary_2
mysim_checkpoint_secondary_1
mysim_checkpoint_run_0
```

**Attributes**

Besides the standard attributes described earlier, a checkpoint bundle's `when` and
`iteration` attributes duplicate the information already encoded in its name, for easy
programmatic access. The remaining attributes record the ski
file settings that Iterating across simulations, in the Features chapter, allows to change
between runs — the value in effect when this checkpoint was written. `num_packets` is
always present; the other four are present only when the simulation is configured to
iterate over the corresponding emission phase.

| Attribute | Type | Description |
| --- | --- | --- |
| `when` | string | One of `Setup`, `Primary`, `Secondary`, `Run`. |
| `iteration` | 32-bit integer | One-based iteration index; `0` for `Setup` and `Run`. |
| `num_packets` | 64-bit float | Configured photon packet count (ski `numPackets`). |
| `max_primary_iterations` | 32-bit integer | Configured maximum (ski `maxPrimaryIterations`). |
| `primary_packets_multiplier` | 64-bit float | Multiplier (ski `primaryIterationPacketsMultiplier`). |
| `max_secondary_iterations` | 32-bit integer | Configured maximum (ski `maxSecondaryIterations`). |
| `secondary_packets_multiplier` | 64-bit float | Multiplier (ski `secondaryIterationPacketsMultiplier`). |

### Spatial grid

TODO: describe the HDF5 representation of the spatial grid checkpoint bundle.

### Medium state

TODO: describe the HDF5 representation of the medium state checkpoint bundle.

### Radiation field

TODO: describe the HDF5 representation of the radiation field checkpoint bundle.

### Recorded fluxes

TODO: describe the HDF5 representation of the recorded fluxes checkpoint bundle.
