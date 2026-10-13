# Implementation

## Build option

The SKIRT build procedure offers a CMake option called `BUILD_HDF5`. When enabled, the
build procedure locates and links the official C HDF5 library, which should already be
installed on the user's system. When disabled (the default), SKIRT builds without any
dependency on HDF5.

A new `h5` build target implements a thin C++ wrapper around the C HDF5 library,
offering just the features used by SKIRT. All code elsewhere in SKIRT depends only on
this target's C++ API, never directly on the HDF5 library. In case `BUILD_HDF5` is
disabled, the wrapper implements stub functions that throw a fatal error as soon as the
caller attempts to open an HDF5 file. Note that SKIRT does not use the C++ wrapper layer
that HDF5 also offers; this would only complicate matters in view of the limited
functionality actually needed.

The target is named `h5` rather than `hdf5`, because a CMake target called `hdf5` collides
with the target exported by the HDF5 library itself.

Within the target, a single internal namespace, `H5Core`, includes the HDF5 headers, makes
all calls into the library, and contains the conditional compilation for `BUILD_HDF5`,
including the stubs. The classes of the C++ API are built on top of it, and are the same
with and without HDF5 support.

## Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to the `h5`
build target. The library cannot be assumed to be thread-safe on its own, since that
depends on how the installed HDF5 happens to have been built, which is outside SKIRT's
control.

## C++ API

The C++ API offered by the `h5` target is a small set of classes representing the
concepts defined in the Data model chapter — suite, bundle, dataset — plus the factory
methods that construct them. Each object acquires its underlying HDF5 resource on
construction and releases it in its destructor. Objects can be moved but not copied.

None of the objects represents an open file: every factory method resolves a full
`<filepath>:<suite>/...` path directly, and file-handle management is this target's own
internal concern. The class names are suffixed with `R` for read-only so that future
extensions can differentiate between reading and writing.

**`H5Lib`** is not instantiated; it offers only static functions, the API's sole entry points
that name a file — nothing else in this API ever names a file path directly. Every
function of the other classes that takes a `<name>` argument, rather than a full
`<filepath>:<suite>/...` path, resolves that name relative to the object the function is
called on.

- `bool available()` — true if this build was compiled with `BUILD_HDF5`.
- `H5SuiteR openSuite(<filepath>:<suite>)` — opens a suite for reading.
- `H5BundleR openBundle(<filepath>:<suite>/<bundle>)` — opens a bundle for reading
  directly, without going through its suite first.
- `bool hasBundle(<filepath>:<suite>/<bundle>)` — true if the HDF5 file exists and holds a
  bundle at that location. This allows a caller to report a missing input file in its own
  words; see Name resolution, below.

A path is split at the first occurrence of the `.hdf5` filename extension that is followed by a
colon, so that the name of a suite or bundle can end with that extension as well. A path
without such an occurrence must end with the extension, and refers to the root group of the file.

**`H5ObjectR`** is the common base class of the three classes below. It owns the underlying HDF5
resource and offers the functions for reading attributes, so that suites, bundles, and datasets
all have them:

- `bool hasAttribute(<name>)`
- `bool getBoolAttribute(<name>)`, `int getIntAttribute(<name>)`,
  `double getDoubleAttribute(<name>)`, `string getStringAttribute(<name>)` — one function for
  each supported attribute type, returning the value.

**`H5SuiteR`**

- `vector<string> getBundleNames()` — the names of every bundle directly in this suite.
- `H5BundleR openBundle(<name>)` — opens one of this suite's bundles for reading.

**`H5BundleR`**

- `vector<string> getDatasetNames()` — the names of every dataset in this bundle. A bundle with
  a dataset for each column, axis, or quantity has no fixed list of dataset names.
- `bool hasDataset(<name>)`
- `H5DatasetR getDataset(<name>)` — opens one of this bundle's datasets for reading.

**`H5DatasetR`**

- `vector<size_t> shape()` — the size of the dataset along each of its dimensions.
- `vector<char> readBool()`, `vector<int> readInt()`, `Array readDouble()` — one function for
  each supported dataset content type, returning the dataset's full contents at whatever shape
  it was created with, as a flat sequence in which the last index varies most rapidly. Booleans
  are returned as characters holding 1 or 0.

The getters are separate functions named for the type they return, rather than overloads of a
single function, because functions that differ only in their return type cannot be overloaded,
and the SKIRT coding style avoids output arguments.

The `hasAttribute()` function is needed because in some places attribute
absence is meaningful and distinct from an empty value.

## Object lifetime

Any object this API hands out remains fully valid after the object it came from has been
destroyed. For example, an `H5DatasetR` does not depend on its `H5BundleR` still being alive.
The HDF5 C library reference-counts its own resources internally, so a file
stays genuinely open, at the library level, for as long as any handle into it remains open,
and only actually closes once the last handle does. It is thus allowed to let an `H5BundleR`
go out of scope as soon as the single `H5DatasetR` actually needed has been obtained from it.

This is not, however, a license to lose track of things: every object this API hands out must
still eventually be destroyed by the client, and until that has happened for everything ever
opened against a given file, that file remains genuinely open. This target places that
responsibility on the client rather than tracking open descendants itself.

## Supported data types

Attributes and dataset contents share three data types: boolean, 32-bit integer, and 64-bit
float. Attributes additionally support `string`, passed as a `std::string` in UTF-8.

Reading returns the data type named by the function the caller uses, and leans on HDF5's own
conversion machinery for however the value is actually stored on disk, as described under
Bundles in the Data model chapter. A stored value that cannot be converted without loss of
information causes a fatal error.

## Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, suite, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the `h5` target described in the previous sections. To
handle this, the `FilePaths` class's API is adjusted as follows.

- `setInputPath(string value)` — the value of the `-i` option: a plain directory path, or a
  directory plus an HDF5 file and suite. The function verifies that the directory exists, and
  that the HDF5 file exists if one is given.
- `input(string name)` returns a pair of strings: a plain-file form and an HDF5 form.
- `isAbsolutePath(string path)` is a static function that tells whether a path that may refer to
  an HDF5 file is absolute. The generic test for an absolute path cannot be used for this,
  because it considers any path that contains a colon to be absolute.

An HDF5 file is recognized by its `.hdf5` filename extension followed by a colon, or, in the
value of the `-i` option, by that extension at the end of the value.

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

The caller is expected to try the plain-file candidate first, falling back
to the HDF5 one only if that file does not exist. If both candidates are given and the input
file is found at neither location, the caller reports this in a single fatal error that
mentions both the plain file and the bundle. It uses `H5Lib::hasBundle()` to establish that
the bundle is missing too, rather than leaving the error to `H5Lib::openBundle()`, which
knows about the bundle only.

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

A `name` argument can itself embed a `<hdf>:<suite>/...` reference. When it does, `input()`
resolves the HDF5 candidate directly from that embedded reference, regardless of what `-i` itself
specifies — even if `-i` names no HDF5 file at all, or a different one; only `-i`'s directory
component still applies, exactly as for any other input file.

Any ski file property naming an input file may use this mechanism:

`input("other.hdf5:campaign/ID4521_stars")` resolves as follows:

| `-i` | Plain-file candidate | HDF5 candidate |
| --- | --- | --- |
| *(not given)* | — | `other.hdf5:campaign/ID4521_stars` |
| `.` | — | `other.hdf5:campaign/ID4521_stars` |
| `./data.hdf5:run1` | — | `other.hdf5:campaign/ID4521_stars` |

## Column input

**`ColumnInFile`** replaces direct `TextInFile` use at every call site that reads a genuine
input text column file today (a Text column file bundle, per the Data model chapter,
when HDF5 applies). A resource file never goes through this class — resource lookup never
involves HDF5 at all, so it keeps using `TextInFile` directly.

At construction, `ColumnInFile(const SimulationItem* item, string filename, string description)`
only records its arguments; unlike `TextInFile`, it does not open anything yet.
The `description` argument is only used for logging and in error messages. `useColumns()` and
`addColumn()` are called next, same as today. Once all `addColumn()` calls have happened,
triggered by the first `readRow()`/`readAllRows()`/`readAllColumns()` call,
the filename is resolved by calling `FilePaths::input(filename)`.
If the plain-file candidate exists, it is opened through an internally held `TextInFile`.
If not, the HDF5 candidate is opened as a bundle. The datasets with a `column` attribute are the
columns of the table; the function reads just the datasets for the columns that are needed, each
as a whole, and converts each column to internal units in a single pass.
`close()` and the destructor release whichever of the two
was actually opened. `readNonLeaf()` has no counterpart here; AMR input needs a different
mechanism, covered in its own subsection below.

**`InputColumns`** is a helper class that holds the column bookkeeping, which used to be part of
`TextInFile`: the columns present in the file, the columns expected by the program, the mapping
between them requested through `useColumns()`, the verification of the units, and the conversion
to internal units. `TextInFile` and `ColumnInFile` both delegate to it, so that matching datasets
with column names happens in exactly the same way as for a plain text column file, and a change
in these rules applies to both formats.

Call sites that construct a `TextInFile` for a genuine input file all move to
`ColumnInFile`:

- `Snapshot` (and snapshot subclasses: `CellSnapshot`,
  `ParticleSnapshot`, `CylindricalCellSnapshot`, `SphericalCellSnapshot`,
  `VoronoiMeshSnapshot`) — imports per-particle or per-cell properties. `Snapshot` creates its
  input file through a virtual function, so that a subclass can substitute another class.
- `VoronoiMeshSnapshot` — also reads site positions directly, separately from the shared
  `Snapshot` mechanism above.
- `FileWavelengthGrid`, `FileBorderWavelengthGrid` — wavelength grids.
- `FileSED`, `FileLineSED`, `FileGaussianLinesSED` — spectral energy distributions.
- `FileBand` — transmission curves.
- `FileWavelengthDistribution` — wavelength probability distributions.
- `FileGrainSizeDistribution` — grain size distributions.
- `FileMesh` — mesh border points.
- `FileTimeGrid` — time grid points.
- `TetraMeshSpatialGrid` — tetrahedral vertices.
- `ClumpySphericalSpatialGrid` — clump centers and radii.
- `MultiGaussianExpansionGeometry` — expansion parameters.
- `MeanFileDustMix` — optical dust properties.
- `NonLTELineGasMix` — initial level populations.
- `AtPositionsForm` — probe sample positions.
- `TopologyTreePolicy` — tree topology recorded by the `TreeSpatialGridTopologyProbe`.
- `SiteListTreePolicy`, `ResolvedSpheresTreePolicy` — site positions, and the centers and radii
  of spheres, for the tree policies proposed in the
  [Tree-based spatial grids](tree-based-spatial-grids/01-introduction.md) note.

**Tree topology.** The topology file written by the `TreeSpatialGridTopologyProbe` is not a table
in the usual sense. After a header comment, it lists a single integer per line: the number of
children of the root node, followed by a subdivision flag (0 or 1) for each node in depth-first
order. The `TopologyTreePolicy` proposed in the
[Tree-based spatial grids](tree-based-spatial-grids/01-introduction.md) note nevertheless reads it
through `ColumnInFile`, as a text column file with a single dimensionless column. Existing topology
files carry no column information in their header, in which case `TextInFile` assumes the column
declared by the program, so these files continue to work unchanged. In an HDF5 bundle, the column
is a single 1-D dataset. Storing the number of children of the root in the same column as the
subdivision flags is not ideal, but it keeps the format compatible.

`AdaptiveMeshSnapshot` needs more than this mechanism offers: since its topology and properties
are read from a single interleaved stream, it uses the `AdaptiveMeshInFile` subclass for the
whole file, covered under AMR input, below.

`GasLineEmission`, `XRayAtomicGasMix`, `XRayIonicGasMix`, and `ElectronScatteringHelper` construct
`TextInFile` with `resource = true`; none of those call sites move, since built-in resources
never use `ColumnInFile`.

**Stored columns format.** Because the binary `scol` format is removed (see the SKIRT 10
[Incompatibilities](skirt-10/04-incompatibilities.md) chapter), `TextInFile` reads column text
files only, and `ColumnInFile` reports a fatal error for a filename with the `.scol` extension,
whether it refers to a plain file or to a bundle.

## AMR input

**`AdaptiveMeshInFile`** replaces the `TextInFile` used today to parse the interleaved topology-and-property
stream `readAndClose()` triggers via `new Node(extent, infile(), cells)`. It keeps
`TextInFile::readNonLeaf(int& nx, int& ny, int& nz)`'s exact signature and behavior — peek
the next line or entry; if it is a nonleaf specification, consume it and return true;
otherwise leave the position untouched for the following `readRow()` call and return false —
so `Node`'s recursive constructor needs no change beyond the type of the pointer it holds.

`AdaptiveMeshInFile` derives from `ColumnInFile`, which provides `useColumns()`, `addColumn()`,
and `readRow()` for the leaf-cell properties, and adds `readNonLeaf()`.
On the plain-text branch, that function forwards to the `TextInFile` held by its base class,
unchanged from today. On the HDF5 branch, when the bundle is opened, the class reads the bundle's
whole `is_leaf` and `topology` datasets into memory, and verifies that the nodes form a single,
complete tree. It then walks `is_leaf` with an internal position.
`readNonLeaf()` consumes the next `topology` row and returns true if the current entry is
`false` (nonleaf), or returns false without consuming anything otherwise; `readRow()` reads
the next row from each declared property dataset, using a separate counter over the `N` leaf
entries only, exactly as the Data model chapter's AMR text file bundle specifies. A mesh can be
read without any property columns, when just its structure is needed.

Call sites: `AdaptiveMeshSnapshot`, which creates an `AdaptiveMeshInFile` through the virtual
function of `Snapshot` mentioned above.

## Stored table input

No separate wrapper class is needed here. `StoredTable::open()` with `resource = false` resolves
the name via `FilePaths::input()`, as elsewhere in this chapter, and branches between the plain
`.stab` file and the HDF5 bundle itself, so that its call sites do not change. As for a plain
file, the `.stab` filename extension is appended to the name of the bundle if it is missing.

The data in a stored table input bundle cannot be memory mapped (as for a regular `.stab` file)
for several reasons:

- The quantity values are ordered differently (interleaved in `.stab`, per dataset in bundle).
- The bundle datasets may be compressed and/or chunked.
- If the same HDF5 file is used for input and output (see the HDF5 output note), memory mapping
  a portion of the file is unsafe because writing to the file might relocate its contents.

For a bundle, `StoredTable::open()` therefore copies the grid points of the axes and the values of
the requested quantity into memory owned by the `StoredTable` instance, which releases that
memory when it is destroyed. The values are reordered from the layout of the dataset, in which
the last index varies most rapidly, to the layout of a stored table file, in which the first
index does, so that the interpolation code is the same for both sources.

The function locates the dataset for each axis and for the requested quantity by its name, and
verifies the `axis` attribute of each axis and the `unit` attribute of each dataset against the
specifications passed by the caller, as it does for the names and units in a plain file. This is
why the names of the axes and quantities in a bundle must all differ, as stated in the Data model
chapter.

Call sites that open a genuine input stored table (`resource = false`):

- `FileIndexedSEDFamily` — an indexed SED family template.
- `FileSSPSEDFamily` — SSP SED family templates, with or without an ionization-parameter axis.
- `FilePolarizedPointSource` — the four Stokes-parameter (I/Q/U/V) tables of a polarized
  point source.
- `MultiGrainDustMix` — the absorption efficiencies for the emission of spheroidal grains, when
  the grain composition takes these tables from files provided by the user.

Every other `StoredTable::open()` call site uses `resource = true` (the default) and stays
untouched.

## FITS input

**`FITSInOutFile`** replaces `FITSInOut` wherever a call site reads a genuine FITS
file (a FITS bundle, per the Data model chapter, when HDF5 applies), as static functions
wrapping the existing, cfitsio-backed `FITSInOut` static functions. Unlike
`ColumnInFile`, no deferred-open or buffering is needed: every `FITSInOut`
call already carries, or produces, the complete array in one shot, so the shape an HDF5
dataset needs at creation is always already known.

`read()` keeps `FITSInOut::read()`'s exact signature. It resolves `FilePaths::input(filename)`
and either delegates straight to the existing cfitsio-backed implementation for the plain-file
candidate, or opens the HDF5 candidate as a bundle and reads its `data` dataset — its shape
supplies `nx`/`ny`/`nz` — for the FITS input bundle already specified in the Data model
chapter. Other datasets and attributes in the bundle are not read, so that a FITS file bundle
written as output is accepted as well.

`FITSInOut::read()` supports `[EXTNAME]`-style addressing for picking a non-primary data unit
out of one FITS file. This has no equivalent in the HDF5 bundle scheme; instead each FITS
data unit should be placed in its own individual HDF5 bundle.

Call sites:

- `ReadFitsGeometry` — a 2-D image.
- `ReadFits3DGeometry` — a 3-D data cube.
