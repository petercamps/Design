# Implementation

## Writing API

The `h5` build target described in the [HDF5 input](hdf5-input/04-implementation.md) note is
extended with classes for writing. Every concept splits into a read-only class (suffixed `R`) and
a write-only class (suffixed `W`), except suite, which has no writable form. Writing always
targets a specific bundle directly, never "a suite" as a whole.

**`H5Lib`** gains two functions for writing:

- `H5BundleW createBundle(<filepath>:<suite>/<bundle>)` — creates a bundle for writing, per How
  output files are written in the Features chapter: it creates the HDF5 file and any missing suites
  if needed, and erases an existing bundle of that name first, leaving the rest of the file
  untouched. It attaches the three standard attributes (`producer`, `format`, `created`)
  automatically. It throws a fatal error if the path refers to an existing suite or dataset, if one
  of the parent groups is not a suite, or if the file cannot be opened for writing: because it is
  not an HDF5 file, because it is read-only, or because it is open for reading in this or another
  process (see Concurrency in the Features chapter).
- `setProducer(string producer)` — sets the value of the `producer` attribute. The command line
  handler passes the version string of SKIRT's welcome message, so that the `h5` target does not
  need to depend on the build information itself.

**`H5ObjectW`** is the common base class of the writable classes, as `H5ObjectR` is for reading. It
offers one attribute setter per supported attribute type, mirroring the getters of `H5ObjectR`:
`setBoolAttribute()`, `setIntAttribute()`, `setDoubleAttribute()`, and `setStringAttribute()`, each
taking the name and the value of the attribute, and replacing an existing attribute with that name.
The type is part of the function name, rather than being selected by the overloads of a single
`setAttribute()`, because a string literal would otherwise select the Boolean overload.

**`H5BundleW`** creates the datasets of a bundle. Each of its functions creates a dataset and writes
its contents in a single call, because HDF5 needs the shape and the data type of a dataset when the
dataset is created:

- `createBoolDataset(<name>, values, shape)`, `createIntDataset(<name>, values, shape)`, and
  `createDoubleDataset(<name>, values, shape)` — create a dataset holding the specified values (a
  `vector<char>`, a `vector<int>`, or an `Array`), in row-major order. The shape defaults to a
  one-dimensional dataset holding all values; a shape that does not match the number of values is a
  fatal error.
- `createTextDataset(<name>, text)` — creates a scalar dataset holding the specified `string`; this
  type is output-only.

Each of these functions returns an **`H5DatasetW`**, which offers just the attribute setters of
`H5ObjectW`.

## Object lifetime

The object lifetime rules of the HDF5 input note apply unchanged. They matter in particular for
MPI: when input and output are pointed at the same HDF5 file, every rank must have destroyed all
objects obtained while reading it before the root process opens that same file for writing,
since reading and writing must not overlap in time (see Concurrency in the Features chapter).

## Supported data types

When writing, each value is stored in the data type named by the function that writes it, without
any conversion: a Boolean as an enumeration based on a signed byte with the members `FALSE` (0) and
`TRUE` (1), the representation used by the `h5py` Python package; an integer as a 32-bit signed
integer; a floating point number as a 64-bit floating point number; and a string, whether an
attribute or the contents of a `text` dataset, as a variable-length string in UTF-8 encoding. A
variable-length string cannot be truncated, and `h5py` returns it as a Python `str`. The contents of
a `text` dataset are passed in as a `string`, since the files it holds, those of the Unstructured
text file bundle, are small enough to be held in memory. Datasets are stored contiguously, without
chunking or compression.

Client code is therefore responsible for writing exactly the data type each table in the Data model
chapter specifies.

## Name resolution

The `FilePaths` class's API, as adjusted in the HDF5 input note, is extended as follows:

- `setOutputPath(string value)` — the value of the `-o` option, in the same form as `-i`.
- `setOutputPrefix(string value)` — the ski filename without its extension; unchanged.
- `output(string name)` returns a pair of strings: a plain-file form and an HDF5 form, as
  `input()` does.
- `inputPath()` and `outputPath()` are removed; nothing needs these two queries.
- `outputPrefix()` is unchanged, still returning whatever `setOutputPrefix()` set.

The following table illustrates the combinations `output()` can resolve to, with the same
assumptions as the examples for `input()`, and further assuming `setOutputPrefix("mysim")`, i.e.
a ski file `mysim.ski`.

`output("i_total.fits")` resolves as follows:

| `-o` | Plain-file candidate | HDF5 candidate |
| --- | --- | --- |
| `.` | `mysim_i_total.fits` | — |
| `./data.hdf5` | `mysim_i_total.fits` | `data.hdf5:mysim_i_total.fits` |
| `./data.hdf5:run1` | `mysim_i_total.fits` | `data.hdf5:run1/mysim_i_total.fits` |

Both candidates are populated whenever applicable — the plain one unconditionally, the
HDF5 one only when `<hdf>` is configured — rather than being mutually exclusive. Most callers
use the HDF5 candidate when it is non-empty and the plain one otherwise. In some rare cases,
though, the caller needs the plain file path regardless of HDF5 configuration — `FileLog`,
for example, always writes to a plain file first.

## Column output

**`ColumnOutFile`** replaces direct `TextOutFile` use at every call site that writes a
genuine output text column file today (a Text column file bundle, per the Data model
chapter, when HDF5 applies).

At construction, `ColumnOutFile(const SimulationItem* item, string filename, string description)`
only records its arguments; unlike `TextOutFile`, it opens nothing yet. A new
`setLongDescription(string longDescription)` replaces the pattern every call site uses
today of calling `writeLine()` to write a `#`-prefixed first line by hand.
The constructor's `description` is used for logging, while the `longDescription` goes
on the text file's first line and in the bundle's `description` attribute.

A further `addComment(string comment)` adds another comment line to the header: in the plain file it
follows the column information lines of the columns added before it, and in a bundle it becomes a
further line of the `description` attribute. The instrument statistics tables of `FluxRecorder` need
it for the comment that follows their column information.

`addColumn()` keeps its existing signature unchanged; its `format` and `precision` arguments
only apply to the plain-text branch, since HDF5 stores each column as typed binary data
with no formatting question to answer.

`FilePaths::output(filename)` is resolved when the first row is written, or at `close()` for a table
without rows: by then the description and the full set of columns must be settled, so that changing
the header afterwards is a fatal error. Unlike `ColumnInFile`, there is no trying one candidate and
falling back to the other: per Name resolution, above, `ColumnOutFile` writes a bundle if the HDF5
candidate is non-empty, and the plain file otherwise. The plain file is written row by row through
an internal `TextOutFile`, exactly as a call site using `TextOutFile` directly would write it, so
that the plain output does not change and is not held in memory. For a bundle, the rows are
collected in memory, and the bundle and its datasets are created at `close()`.

A client must call `close()` explicitly after writing the last row. Writing a bundle can fail, for
example when the HDF5 file is open in another program, and only `close()` can report that as a fatal
error. The destructor never throws an exception, as the coding guidelines require: if `close()` was
not called, it completes the file, and logs the message of a fatal error instead of propagating it.
While an exception is propagating, the destructor leaves the file incomplete.

**Memory.** Because a bundle is written at `close()`, the complete table is held in memory in the
HDF5 case, as a 64-bit value for each row and column, while a plain file needs no such buffer. For
most outputs this is negligible, but per-cell probe output for a very large spatial grid is an
exception: ten million cells with ten columns take at least 800 MB, in addition to the memory used
by the simulation itself. Writing the rows to the bundle as they are produced would require chunked,
extendible datasets, whose use the Data model chapter leaves for future consideration. Writing a
plain file first and copying it at the end, as for the incrementally written files below, would
limit the values to the precision of the text file. This is to be revisited if the buffer proves a
problem in practice.

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
- `DustGrainSizeDistributionProbe` — dust grain size distributions.
- `DustAbsorptionPerCellProbe`, `DustEmissivityProbe` — per-cell dust absorption and
  emissivity data.
- `IntegratedSecondaryLineLuminosityProbe` — integrated per-line luminosities.
- `TreeSpatialGridTopologyProbe` — tree topology, as a single column.

**Tree topology.** Today, `TreeSpatialGrid::writeTopology()` writes the topology line by line
through `writeLine()`: after a header comment, the number of children of the root node, followed by
a subdivision flag (0 or 1) for each node in depth-first order. It instead writes these values
through a `ColumnOutFile` with a single dimensionless column, in integer format, so that the plain
file holds the same values as before, preceded by a column information line. Readers skip header
lines, so the new files can still be read by older SKIRT versions, and the `TopologyTreePolicy`
reads both old and new files (see the [HDF5 input](hdf5-input/04-implementation.md) note). In an
HDF5 bundle, the column is a single 1-D dataset. Storing the number of children of the root in the
same column as the subdivision flags is not ideal, but it keeps the format compatible.

The `HistoryProbe` does not move to `ColumnOutFile`, because it writes its files incrementally;
it is covered in a separate section below.

Two groups of `TextOutFile` use stay out of scope, for different reasons:

- `ConvergenceInfoProbe` writes free-form, human-readable text (`convergence.dat`), not a
  column table, and `DustGrainPopulationsProbe` writes a table (`grainpops`) in which one column
  holds the name of a grain material rather than a number. Both become an Unstructured text file
  bundle, covered in a separate section below.
- `SpatialGridPlotFile` and the various spatial grid classes that write to it
  (`CartesianSpatialGrid`, `StructuredSphereSpatialGrid`, `Cylinder2DSpatialGrid`,
  `Cylinder3DSpatialGrid`, `Sphere2DSpatialGrid`) write raw polyline coordinates, not named
  columns — a Spatial grid plot file bundle, covered in a separate section below.

## Incrementally written column output

The `HistoryProbe`, proposed in the [Iteration history](iteration-history/01-introduction.md)
note, writes a separate text column file for each iteration loop. After each iteration, it appends
a row to the file of the current loop and flushes the file, so that the progress of a long run can
be followed while it executes. It keeps writing these files
as plain files through `TextOutFile`, exactly as today's call sites do, regardless of the HDF5
configuration — like `FileLog`, it uses the plain-file candidate of `output()` unconditionally.

At the end of the run, if `output()`'s HDF5 candidate is non-empty, each finished file is copied
into its own Text column file bundle: the file is reopened, its header is parsed for the column
descriptions and units, as `TextInFile` does for input files, and the bundle and its datasets are
created with the complete columns, exactly as `ColumnOutFile` would have created them. The
comment lines of the header other than the column information lines, normally just the free-form
comment on the first line, become the bundle's `description` attribute, one line each. The static
function `ColumnOutFile::copyToBundle()` performs this copy step, with the same arguments as the
constructor of `ColumnOutFile`; it does nothing if the HDF5 candidate is empty, and leaves the
plain file in place.

The copy requires that the history probe is also invoked at the end of the run, in addition to
after each iteration. `Probe`'s dispatch functions, such as `probeRun()`, are not virtual today;
each checks `when()` against its own point and calls `probe()` if it matches. Making `probeRun()`
virtual lets the history probe override it to perform the copy. The
[Checkpointing](checkpointing/01-introduction.md) note proposes the same change for all dispatch
functions, so that the checkpoint probe can fire at every point.

## FITS output

The `FITSInOutFile` class introduced in the HDF5 input note for reading is extended with
`write()` and `writeMap()`, which keep `FITSInOut`'s exact signatures, including the optional
`ObserverInfo` struct. `FilePaths::output(filename)` resolves the output location as for
`read()`; the HDF5 branch creates the bundle and writes `data` plus whichever of `x`/`y`/
`z` apply in one shot each, and sets the bundle's attributes — `description`, and, for
distant-instrument IFUs, `ObserverInfo`'s fields — exactly as the FITS file output bundle
already specifies in the Data model chapter.

Call sites:

- `FluxRecorder` — instrument fluxes (IFU and STM) and their statistics.
- `PlanarCutsForm`, `ParallelProjectionForm`, `AllSkyProjectionForm` — planar cuts and
  projections produced by probes.

## Spatial grid plot output

**`SpatialGridPlotFile`** stays the same class, with the same public API: every grid type
(and `VoronoiMeshSnapshot`, which plots its own tessellation directly) writes plot data
exclusively through its shape-drawing methods — `writeLine`, `writeRectangle`,
`writeCircle`, `writeArc`, `writeCube`, `writeMeridionalHalfCircle`, `writeSphere`,
`writePolyhedron` — never by touching a file directly. None of those call sites need to
change; the whole adaptation is internal to this one class.

Today, every one of those methods writes straight to the protected `_out` stream it
privately inherits from `TextOutFile`, doing its own unit conversion inline —
`SpatialGridPlotFile` never actually uses `TextOutFile`'s own public
`writeLine(string)`/`addColumn()`/`writeRow()` API, just its constructor and protected
members. This needs to become two private primitives, `moveTo`/`lineTo` — each with a 2D and
a 3D overload, mirroring `writeLine`'s own two overloads, and matching the "moveto"/"lineto"
terminology the class's own doc comment already uses — that every one of these public
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

## Unstructured text output

Covers the four cases in the Data model chapter's Unstructured text file bundle —
`convergence.dat`, `grainpops`, `parameters.xml`, and `log.txt`.

All four share the same underlying trick: write the complete text to a real plain file
exactly as today, then, if `output()`'s HDF5 candidate is non-empty, reopen that finished
file, read it into a `string`, and pass it to `H5BundleW::createTextDataset()`, setting
the bundle's `type` attribute (`"free form"`, `"XML"`, or `"log"`, per the Data model
chapter) along the way. None of the writing logic of `ConvergenceInfoProbe`,
`DustGrainPopulationsProbe`, `XmlHierarchyWriter`, or `FileLog` needs to know anything about
HDF5 — only the point where each one is known to be finished changes. This "copy" step could
live in one small shared helper.

`DustGrainPopulationsProbe` keeps writing its table through `TextOutFile`, with the same header
and columns as today, so that the regular output file does not change.
