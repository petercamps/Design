# Implementation

> Status: draft

## HDF5 target

### Build option

Whether HDF5 support is available is decided at compile-time, but 
client code links into the HDF5 target's API unconditionally, everywhere in SKIRT, rather
than being scattered with `#ifdef BUILD_HDF5` guards.

Concretely, this follows the same pattern already used for SKIRT's MPI support (see
`SKIRT/mpi/CMakeLists.txt` and `ProcessManager.cpp`): a new `hdf5` CMake target, always built as a
static library regardless of `BUILD_HDF5`, that contains every line of code in SKIRT that
touches the HDF5 C API directly. No other target ever includes an HDF5 header or links
against the HDF5 library — code elsewhere in SKIRT depends only on this target's own C++
API, never on HDF5 itself. `BUILD_HDF5` controls only how that one target's own sources are
compiled: when enabled, `find_package(HDF5 REQUIRED)` locates the library, the target links
against it, and a `BUILD_HDF5` preprocessor symbol — defined only for this target's own
sources, exactly as `mpi` does for `BUILD_WITH_MPI` — selects the real implementation inside
each `#ifdef` block; when disabled, the target still builds, taking the `#else` branch of
each of those blocks instead, and links against nothing HDF5-related at all, so a
`BUILD_HDF5`-disabled SKIRT binary has no HDF5 footprint whatsoever, exactly as
Prerequisites in Features requires.

The stub implementation throws a fatal error the moment any real work
is attempted. As an exception, the `H5Lib` class (see below) offers a static `available()`
query that simply reports whether this build was compiled with `BUILD_HDF5`.
Wherever a ski file setting or command-line option would require
HDF5 — anywhere `-i`/`-o`/`-c` names an HDF5 file, or a property such as
`CheckpointTreePolicy::filename` targets one — setup-time validation checks
`H5Lib::available()` first and reports a clear, immediate error if HDF5 support is missing,
rather than letting execution reach a stub's fatal error later.

### Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to this target: the
library cannot be assumed to be thread-safe on its own, since that depends on how the
installed HDF5 happens to have been built, which is outside SKIRT's control.

### C++ API

The C++ API offered by the HDF5 target is a small set of classes representing the concepts
defined in the Data model chapter — suite, bundle, checkpoint, dataset — plus the factory
methods that construct them. Each object acquires its underlying HDF5 resource on
construction and releases it in its destructor. Objects can be moved but not copied.

None of the objects represents an open file: every factory method
resolves a full `<filepath>:<suite>/...` path directly, and file-handle management is this
target's own internal concern. Every concept splits into a read-only class (suffixed `R`)
and a write-only class (suffixed `W`), except suite, which has no writable form.
Writing always targets a specific bundle or checkpoint directly, never
"a suite" as a whole.

**`H5Lib`** is not instantiated; it offers only static factory methods, the API's sole
file-opening entry points — nothing else in this API ever names a file path directly. Every
method below that takes a `<name>` argument, rather than a full `<filepath>:<suite>/...`
path, resolves that name relative to the object the method is called on.

- `bool available()` — true if this build was compiled with `BUILD_HDF5`.
- `H5SuiteR openSuite(<filepath>:<suite>)` — opens a suite for reading.
- `H5BundleR openBundle(<filepath>:<suite>/<bundle>)` — opens a bundle for reading directly,
  without going through its suite first.
- `H5CheckpointR openCheckpoint(<filepath>:<suite>/<checkpoint>)` — opens a checkpoint for
  reading directly, without going through its suite first.
- `H5BundleW createBundle(<filepath>:<suite>/<bundle>)` — creates a bundle for writing,
  erasing any existing bundle of that name first, per How output files are written in
  Features.
- `H5CheckpointW createCheckpoint(<filepath>:<suite>/<checkpoint>)` — creates a checkpoint
  for writing, erasing any existing checkpoint of that name first.

**Reading.**

**`H5SuiteR`**

- `vector<string> getBundleNames()` — the names of every bundle directly in this suite,
  excluding checkpoints.
- `vector<string> getCheckpointNames()` — the names of every checkpoint directly in this
  suite.
- `H5BundleR openBundle(<name>)` — opens one of this suite's bundles for reading.
- `H5CheckpointR openCheckpoint(<name>)` — opens one of this suite's checkpoints for reading.

**`H5BundleR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — one overload per supported attribute type (boolean, 32-bit
  integer, 64-bit float, string).
- `bool hasDataset(<name>)`
- `H5DatasetR getDataset(<name>)` — opens one of this bundle's datasets for reading.

**`H5CheckpointR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — as `H5BundleR::getAttribute()`, above.
- `bool hasBundle<Name>()` — `<Name>` is one of the fixed sub-bundle names (Spatial grid,
  Medium state, Radiation field, Recorded fluxes), never an arbitrary one.
- `H5BundleR getBundle<Name>()` — opens one of the checkpoint's fixed-name sub-bundles for
  reading.

**`H5DatasetR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — as `H5BundleR::getAttribute()`, above.
- `read()` — one overload per supported dataset content type (boolean, 32-bit integer,
  64-bit float), returning the dataset's full contents at whatever shape it was created
  with.

**Writing.**

**`H5BundleW`**

- `setAttribute(<name>, value)` — one overload per supported attribute type, as for
  `H5BundleR::getAttribute()`, above.
- `H5DatasetW createDataset(<name>)` — creates one of this bundle's datasets for writing,
  with the shape and on-disk type the Data model chapter specifies for it.

**`H5CheckpointW`**

- `setAttribute(<name>, value)` — as `H5BundleW::setAttribute()`, above.
- `H5BundleW createBundle<Name>()` — creates one of the checkpoint's fixed-name sub-bundles
  for writing.
- `linkBundle<Name>ToCheckpoint(<suite>/<checkpoint>)` — hard-links this checkpoint's `<Name>`
  sub-bundle to the same-named, already-written sub-bundle of the checkpoint at
  `<suite>/<checkpoint>`, instead of writing a new copy of it — what Checkpoint probe
  behavior in Features relies on to avoid re-storing unchanged data across checkpoints.

**`H5DatasetW`**

- `setAttribute(<name>, value)` — as `H5BundleW::setAttribute()`, above.
- `write()` — as `H5DatasetR::read()`, above, plus one further overload for `text`, since
  that type is output-only.

A bundle created for writing attaches the three standard attributes (`producer`, `format`,
`created`) automatically; one opened for reading exposes whatever attributes it actually has.
A checkpoint handles the checkpoint-specific attributes (`when`, `iteration`, the ski
parameters from Iterating across simulations) and hands out the up to four sub-bundles
(Spatial grid, Medium state, Radiation field, Recorded fluxes) by name.
The `hasAttribute()` function is needed
because in some places attribute absence is meaningful and distinct from an empty value.

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
responsibility on the client rather than tracking open descendants itself. The concrete case
is MPI: when input and output are pointed at the same HDF5 file, every rank must have
destroyed all objects obtained while reading it before the root process opens that same file
for writing, since reading and writing must not overlap in time (see Concurrency under
Output in Features).

### Supported data types

Attributes (on `H5Bundle`, `H5Checkpoint`, and `H5Dataset`) and dataset contents share three
data types: boolean, 32-bit integer, and 64-bit float. Attributes additionally support
`string`, passed as a `std::string` in UTF-8 and stored as a fixed 128-byte field —
comfortably larger than any attribute value in the Data model chapter, including the free-form
`description` attributes. Datasets additionally support `text`, for output only, passed in as a
`std::istream` in UTF-8 and stored as a variable-length string on disk — Unstructured text
file's `text` dataset is the only case that needs it, and unlike an attribute, its content
can genuinely be arbitrarily long.

No conversion happens when writing attributes or data, so client code is responsible for writing
exactly the data type each table in the Data model chapter specifies. Reading returns
the data type the caller requests and leans on HDF5's own conversion machinery for however the
value is actually stored on disk, as described under Bundles in the Data model chapter.

## Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, suite, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the HDF5 target described in the previous section. To
handle this, the `FilePaths` class's API is adjusted as follows.

**Setup**

- `setInputPath(string value)` — the value of the `-i` option: a plain directory path, or a
  directory plus an HDF5 file and suite.
- `setOutputPath(string value)` — the value of the `-o` option, in the same form.
- `setCheckpointPath(string value)` — the value of the `-c` option: always a directory plus
  an HDF5 file and suite, since a checkpoint has no plain-file form.
- `setOutputPrefix(string value)` — the ski filename without its extension; unchanged.

**Trivial queries**

- `inputPath()` and `outputPath()` are removed; nothing needs these two queries.
- `outputPrefix()` is unchanged, still returning whatever `setOutputPrefix()` set.

**Name-constructing queries**

- `input(string name)` and `output(string name)` each return a pair of strings: a plain-file
  form and an HDF5 form.
- `checkpoint(string name)` returns a single string, since a checkpoint has no plain-file
  form.

Given the value of a ski file's `filename` attribute, or a programmatically fabricated
filename or bundle name, these three functions combine that name with the setup values taken
from the command line to construct fully resolved locations. For `input()`/`output()`, the
first string of the pair is the absolute canonical path for a plain file corresponding to
`name`, and the second is an HDF5 file path plus a resolved bundle path corresponding to
`name`; either string is empty if there is no such correspondence for that side. Neither
function does any I/O of its own, so a non-empty string is never a guarantee that the file or
bundle it names already exists.

The setup and name-constructing functions throw a fatal error if the passed argument string
implies HDF5 is needed but `H5Lib::available()` (above) returns false.

The following tables illustrate the various combinations these three functions can resolve
to, assuming every file sits in the current directory and ignoring the canonical-path detail
in the returned strings — resolving absolute and relative paths is a well-understood problem,
not what these examples are about. The output and checkpoint tables further assume
`setOutputPrefix("mysim")`, i.e. a ski file `mysim.ski`.

**Input**

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

**Output**

`output("i_total.fits")` resolves as follows:

| `-o` | Plain-file candidate | HDF5 candidate |
| --- | --- | --- |
| `.` | `mysim_i_total.fits` | — |
| `./data.hdf5` | `mysim_i_total.fits` | `data.hdf5:mysim_i_total.fits` |
| `./data.hdf5:run1` | `mysim_i_total.fits` | `data.hdf5:run1/mysim_i_total.fits` |

Both candidates are populated whenever applicable — the plain one unconditionally, the
HDF5 one only when HDF5 is configured — rather than being mutually exclusive. Most callers
use the HDF5 candidate when it is non-empty and the plain one otherwise.
In some rare cases, the caller needs the plain file path regardless of HDF5 configuration.
For example, `FileLog` always writes to a plain file first, and thus needs this path.

**Checkpoint**

`checkpoint("checkpoint_primary_2")` resolves as follows:

| `-c` | Result |
| --- | --- |
| `./data.hdf5` | `data.hdf5:mysim_checkpoint_primary_2` |
| `./data.hdf5:run1` | `data.hdf5:run1/mysim_checkpoint_primary_2` |
| `./data.hdf5:campaign7/run1` | `data.hdf5:campaign7/run1/mysim_checkpoint_primary_2` |

`-c` always requires `<hdf>` (Resuming from a checkpoint in Features), so there is no
plain-only row here.

## Input

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
- If the same HDF file is used for input and output, memory mapping a portion of the file
  is unsafe because writing to the file might relocate its contents.

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

## Output

### Column output

**`ColumnOutFile`** replaces direct `TextOutFile` use at every call site that writes a
genuine output text column file today (a Text column file bundle, per the Data model
chapter, when HDF5 applies).

At construction, `ColumnOutFile(const SimulationItem* item, string filename, string description)`
only records its arguments; unlike `TextOutFile`, it opens nothing yet. A new
`setLongDescription(string longDescription)` replaces the pattern every call site uses
today of calling `writeLine()` to write a `#`-prefixed first line by hand.
The constructor's `description` is used for logging, while the `longDescription` goes
on the text file's first line and in the bundle's `description` attribute.

`addColumn()` keeps its existing signature unchanged; its `format` and `precision` arguments
only apply to the plain-text branch, since HDF5 stores each column as typed binary data
with no formatting question to answer.

`FilePaths::output(filename)` is only resolved once writing is actually finished, at
`close()`, since the description and the full set of columns must be settled first. Unlike
`ColumnInFile`, there is no trying one candidate and falling back to the other: per Name
resolution, above, `output()`'s two candidates are already mutually exclusive, so
`ColumnOutFile` writes the plain file if that candidate is non-empty, or creates the bundle
and its datasets if the HDF5 candidate is.

Call sites that construct a `TextOutFile` for a genuine column output file all move to
`ColumnOutFile`:

- `LinearCutForm`, `PerCellForm`, `MeridionalCutForm`, `AtPositionsForm` — the shared forms
  `ProbeFormBridge` uses to write a probe's sampled quantities; `OpacityProbe`,
  `RadiationFieldProbe`, `SecondaryLineLuminosityProbe`, `ImportedSourceLuminosityProbe`,
  `CustomStateProbe`, and other probes supply their own column definitions to whichever form
  they use, without constructing a file themselves.
- `FluxRecorder` — SED, light curve, and instrument statistics tables.
- `LaunchedPacketsProbe` — photon packets launched by primary sources.
- `LuminosityProbe` — primary source luminosities.
- `InstrumentTimeGridProbe`, `InstrumentWavelengthGridProbe` — instrument time and
  wavelength grids.
- `SpatialGridSourceDensityProbe` — gridded primary source densities.
- `SpatialCellPropertiesProbe` — per-cell spatial grid properties.
- `OpticalMaterialPropertiesProbe` — per-medium optical properties.
- `DustGrainPopulationsProbe`, `DustGrainSizeDistributionProbe` — dust grain population and
  size-distribution data.
- `DustAbsorptionPerCellProbe`, `DustEmissivityProbe` — per-cell dust absorption and
  emissivity data.
- `IntegratedSecondaryLineLuminosityProbe` — integrated per-line luminosities.

Three groups of `TextOutFile` use stay out of scope, for different reasons:

- `ConvergenceInfoProbe` writes free-form, human-readable text (`convergence.dat`), not a
  column table — an Unstructured text file bundle, not covered here.
- `SpatialGridPlotFile` and the various spatial grid classes that write to it
  (`CartesianSpatialGrid`, `StructuredSphereSpatialGrid`, `Cylinder2DSpatialGrid`,
  `Cylinder3DSpatialGrid`, `Sphere2DSpatialGrid`) write raw polyline coordinates, not named
  columns — a Spatial grid plot file bundle, covered in a separate section below.
- `TreeSpatialGrid::writeTopology()` and `TreeSpatialGridTopologyProbe` are removed outright,
  per Compatibility in Features, replaced by the spatial grid checkpoint bundle's topology.

### FITS output

`FITSInOutFile::write()` and `::writeMap()` keep `FITSInOut`'s exact signatures, including
the optional `ObserverInfo` struct. `FilePaths::output(filename)` resolves the same way as
for `read()`; the HDF5 branch creates the bundle and writes `data` plus whichever of `x`/`y`/
`z` apply in one shot each, and sets the bundle's attributes — `description`, and, for
distant-instrument IFUs, `ObserverInfo`'s fields — exactly as the FITS file output bundle
already specifies in the Data model chapter.

Call sites:

- `FluxRecorder` — instrument fluxes (IFU and STM) and their statistics.
- `PlanarCutsForm`, `ParallelProjectionForm`, `AllSkyProjectionForm` — planar cuts and
  projections produced by probes.

### Spatial grid plot output

**`SpatialGridPlotFile`** stays the same class, with the same public API: every grid type
(and `VoronoiMeshSnapshot`, which plots its own tessellation directly) writes plot data
exclusively through its eleven shape-drawing methods — `writeLine`, `writeRectangle`,
`writeCircle`, `writeArc`, `writeCube`, `writeMeridionalHalfCircle`, `writeSphere`,
`writePolyhedron` — never by touching a file directly. None of those call sites need to
change; the whole adaptation is internal to this one class.

Today, every one of those methods writes straight to the protected `_out` stream it
privately inherits from `TextOutFile`, doing its own unit conversion inline —
`SpatialGridPlotFile` never actually uses `TextOutFile`'s own public
`writeLine(string)`/`addColumn()`/`writeRow()` API, just its constructor and protected
members. This needs to become two private primitives, `moveTo`/`lineTo` — each with a 2D and
a 3D overload, mirroring `writeLine`'s own two overloads, and matching the "moveto"/"lineto"
terminology the class's own doc comment already uses — that every one of the eleven public
methods is rewritten to call instead of writing to `_out` directly. `moveTo` starts a new
polyline and `lineTo` continues the current one, matching the `points`/`moveto` datasets the
Spatial grid plot file bundle already specifies in the Data model chapter.

Because deciding between the plain file and the HDF5 bundle needs to wait until the total
point count is known, exactly as for Column output, `SpatialGridPlotFile` can no longer
construct its `TextOutFile` base immediately the way it does today; it needs to hold one
internally instead, built lazily — the same shift `ColumnOutFile` already made. On the
plain-text branch, `moveTo`/`lineTo` can still write immediately, since a text file needs no
upfront point count; only the HDF5 branch actually needs to buffer until `close()`.

Two call patterns exercise this file type, neither needing to change: most grid types
override `SpatialGrid::write_xy()`/`write_xz()`/`write_yz()`/`write_xyz()`, each receiving an
already-constructed `SpatialGridPlotFile*` from `SpatialGrid`'s own driving code, which
constructs one file per view; `VoronoiMeshSnapshot` and `TetraMeshSpatialGrid` instead each
construct their own four files directly, in one combined `writeGridPlotFiles()` method, since
their geometry doesn't naturally split by view.

### Unstructured text output

Covers the three cases in the Data model chapter's Unstructured text file bundle —
`convergence.dat`, `parameters.xml`, and `log.txt`.

All three share the same underlying trick: write the complete text to a real plain file
exactly as today, then, if `output()`'s HDF5 candidate is non-empty, reopen that finished
file as an `std::ifstream` and hand it to `H5DatasetW::write()`'s `text` overload, setting
the bundle's `type` attribute (`"free form"`, `"XML"`, or `"log"`, per the Data model
chapter) along the way. Neither `ConvergenceInfoProbe`'s, `XmlHierarchyWriter`'s, nor
`FileLog`'s own writing logic needs to know anything about HDF5 — only the point where each
one is known to be finished changes. This "copy" step could live in one small shared helper.

## Notes to revisit

- `CheckpointTreePolicy` must match each live node to its counterpart in the recorded
  topology — parsed into a real tree at setup via its `parent_id` links — by following the
  same parent/child-slot path from the root, rather than assuming live construction visits
  nodes in the same order the checkpoint was originally written in; once a live node falls
  past what was recorded, it correctly answers no from there down.
