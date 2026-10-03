# Implementation

## Build option

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

## Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to the `hdf5`
build target. The library cannot be assumed to be thread-safe on its own, since that
depends on how the installed HDF5 happens to have been built, which is outside SKIRT's
control.

## C++ API

The C++ API offered by the `hdf5` target is a small set of classes representing the
concepts defined in the Data model chapter — suite, bundle, dataset — plus the factory
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

## Object lifetime

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

## Supported data types

Attributes (on `H5Bundle` and `H5Dataset`) and dataset contents share three data types:
boolean, 32-bit integer, and 64-bit float. Attributes additionally support `string`,
passed as a `std::string` in UTF-8.

Reading returns the data type the caller requests and leans on HDF5's own conversion
machinery for however the value is actually stored on disk, as described under Bundles in
the Data model chapter.

## Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, suite, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the HDF5 target described in the previous sections. To
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

The caller is expected to try the plain-file candidate first, falling back
to the HDF5 one only if that file does not exist.

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

## AMR input

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

## Stored table input

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

## FITS input

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
