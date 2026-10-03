# Data model

## Bundles

The complete contents of a SKIRT input file are often more naturally represented as
several related datasets than as a single one. This note calls such a SKIRT-defined
named object a **bundle**. In the file, a bundle is always an HDF5 group containing one
or more datasets with names fixed by the SKIRT data model.

Upon input, the HDF5 library performs reasonable data type conversions (e.g. between
32-bit and 64-bit number representations) and decompresses chunked or compressed data as
needed. This is fully transparent to SKIRT, so everything is fine as long as there is no
data loss (e.g. because a number doesn't fit in SKIRT's representation).

## Text column file bundle

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

## AMR text file bundle

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

## Stored table (.stab) bundle

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

## FITS file bundle

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
