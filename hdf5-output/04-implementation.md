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
since reading and writing must not overlap in time (see Concurrency in the Features chapter). The
I/O cutover, below, guarantees this.

## I/O cutover

The concurrency rules of HDF5 (see Concurrency in the Features chapter) matter when the input and
the output of a simulation share an HDF5 file: a file cannot be opened for writing while any process
has it open for reading, and there can be only one writer. Within a single process, the object
lifetime rules and the mutex in the `h5` target suffice, because each reader releases its HDF5
objects as soon as it has loaded its data, and each writer closes its bundle after writing it.
Across the processes of a multiprocessing run, however, a process could still be reading an input
bundle while the root process writes an output bundle into the same file.

The *I/O cutover* is the moment in a simulation that separates all bundle input from all bundle
output: before it, the processes only read bundles; after it, only the root process writes bundles.
It falls at the end of setup, in `MonteCarloSimulation::setupSimulation()`: after
`SimulationItem::setup()` and the existing `wait("setup")` barrier, which synchronizes all
processes, and before `probeSystem()->probeSetup()`. No further barrier is needed.

The cutover exists for all simulations, because:

- every class that reads a user input file does so during setup: directly in `setupSelfBefore()` or
  `setupSelfAfter()`, in a function that its base class calls during setup (such as the `get...()`
  functions of the tabulated SEDs, bands, and dust mixes), or while creating the input file of an
  importer or a mesh;
- all bundle output comes from probes, which run after setup, from instruments, which write their
  output at the end of the run, and from the command line handler, which stores the log file and the
  parameter file at the end of the simulation.

The one exception today is `AtPositionsForm`, which reads its positions file when the probe runs,
i.e. after setup. It must instead read the positions during setup and keep them in memory. Planned
features follow the same pattern: resuming from a checkpoint reads the checkpoint during setup,
while checkpoints and iteration history files are written later.

To make this rule explicit and enforce it, each simulation records on which side of the cutover it
is, in its `FilePaths` instance. This state cannot live in the `h5` target, because with the `-s`
option several simulations run in parallel in the same process, each at its own stage. Before the
cutover, opening a bundle for writing is a fatal error; after it, opening a bundle or suite for
reading, or testing whether a bundle exists, is a fatal error. Regular files are not affected, so
that, for example, the log file can still be read when it is stored at the end. When a process runs
a single simulation, which is always the case in a multiprocessing run because the `-s` option
cannot be combined with multiprocessing, each process also verifies at the cutover that it has no
open HDF5 objects, so that a reader that keeps a bundle open is reported right away rather than
causing a conflict later. This check is skipped when several simulations run in parallel in the
process, because the other simulations may legitimately hold HDF5 objects at that moment, for
example while they read their input; conflicts within the process are then handled by the wait for
readers described below. The checks on opening bundles apply in every run, including runs with a
single process, and the functional tests run one simulation per process, so that they catch any
violation.

**Parallel simulations.** With the `-s` option, several simulations run in parallel threads of the
same process (which then cannot use multiprocessing), each at its own stage, so that the cutover of
one simulation says nothing about the others. Within one process, HDF5 keeps a single shared object
for each open file, and the mutex in the `h5` target serializes all calls. As a result, reading a
file that another simulation holds open for writing is safe, and so is writing a file that another
simulation writes as well. The one combination that fails is opening a file for writing while
another simulation holds it open for reading: HDF5 cannot turn the shared read-only object into a
writable one, so that `createBundle()` reports a fatal error, depending on the timing of the
simulations. This happens when one simulation reads its input from an HDF5 file that another
simulation in the same run writes its output to, for example with `-i in/data.hdf5 -o in/data.hdf5`
for a set of ski files.

To avoid this, the `h5` target counts, for each file, the objects that hold it open for reading in
the process, and opening a file for writing waits until no object holds that file open for reading,
rather than failing. Readers hold a file only while they load a bundle, so that the wait is short. A
thread that holds a file open for reading while it opens the same file for writing would wait
forever; the cutover excludes this within a simulation, and a time limit on the wait turns any
remaining case into a fatal error rather than a hang.

**Early check of the output file.** A failure to write a bundle, because the HDF5 output file is not
an HDF5 file, is read-only, or is held open by another program, would otherwise surface only when
the first bundle is written, which for many simulations is at the end of the run. The cutover is the
earliest moment at which the root process can test the output file without any risk of a conflict
with readers among the processes of the simulation, because these have finished reading and closed
their files; with the wait for readers, it also waits for parallel simulations that are still
reading the file. Right after the cutover, the root process therefore opens the HDF5 output file for
writing, creating it if it does not exist, and closes it again; if this fails, the simulation ends
with a fatal error before the run. The file is not kept open until the end of the run, because that
would prevent other programs from reading output that has already been written, and a crash would
leave the file marked as open for writing. The check is a snapshot: the file can still become
unwritable later, for example when another program opens it during the run, so that each write still
reports its own failure. A check before setup would need a barrier of its own, and would create the
HDF5 file even if setup fails.

Together with the fact that only the root process writes bundles, the cutover guarantees that the
processes of a multiprocessing run never violate the HDF5 concurrency rules among themselves, and
the wait for readers does the same for simulations that run in parallel in one process. This matters
in particular on the parallel file systems of compute clusters, where HDF5 file locking is often
disabled, so that a violation would not be reported as an error but could corrupt the file. The
guarantee does not cover other programs that open the HDF5 file while a simulation runs, nor
independent SKIRT processes; the Concurrency section of the Features chapter describes how to avoid
such conflicts.

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

The `FITSInOutFile` class introduced in the HDF5 input note for reading is extended with `write()`
and `writeMap()`, which keep the exact signatures of the functions of `FITSInOut` that write in the
context of a simulation item, including the optional `ObserverInfo` struct. These functions are
removed from `FITSInOut`, whose remaining basic functions become private, with `FITSInOutFile` as a
friend; `ObserverInfo` stays in `FITSInOut`. `FilePaths::output(filename)` resolves the output
location as for `read()`; the HDF5 branch creates the bundle and writes `data` plus whichever of
`x`/`y`/`z` apply in one shot each, and sets the bundle's attributes — `description`, and, for
distant-instrument IFUs, `ObserverInfo`'s fields — as the FITS file output bundle specifies in the
Data model chapter. For the regularly spaced x- and y-axes passed to `write()`, the grid points are
computed as they follow from the CRPIX, CRVAL, and CDELT cards of a FITS file, i.e. the pixel
centers. A data cube with a single frame is stored as a 2-D frame next to a `z` dataset with a
single grid point.

The datasets carry a `unit` attribute but no `quantity` attribute. The write functions receive only
unit strings, and the names of the wavelength- and flux-dependent quantities, which depend on the
output style, exist only inside the `Units` class; passing them would change every signature. Adding
the attribute was postponed on 2026-10-17.

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

Every one of those methods used to write straight to the protected `_out` stream it privately
inherited from `TextOutFile`, doing its own unit conversion inline — `SpatialGridPlotFile` never
actually used `TextOutFile`'s own public `writeLine(string)`/`addColumn()`/`writeRow()` API, just
its constructor and protected members. Each method now converts its coordinates to output units
exactly as before, and passes the results to two private primitives, `moveTo`/`lineTo` — each with a
2D and a 3D overload, mirroring `writeLine`'s own two overloads, and matching the "moveto"/"lineto"
terminology the class's own doc comment already uses. `moveTo` starts a new polyline and `lineTo`
continues the current one, matching the `points`/`moveto` datasets the Spatial grid plot file bundle
specifies in the Data model chapter. Because the primitives take coordinates that are already in
output units, every value is computed with the same arithmetic as before, so that the plain file
remains byte-identical.

`SpatialGridPlotFile` no longer derives from `TextOutFile`. Its constructor resolves the output
location right away, because the file has no header and the choice does not depend on the contents:
for a plain file, it opens the file through a private subclass of `TextOutFile` that exposes the
output stream, and the primitives write each point immediately, ending a polyline with an empty
line; for a bundle, the primitives collect the points in memory, and the bundle is written at
`close()`. As for `ColumnOutFile`, writing a bundle can fail, so the class gains an explicit
`close()`, which the code that constructs a plot file calls when the plot is complete, and a
destructor that does not throw: if `close()` was not called, it completes the file and logs the
message of a fatal error instead of propagating it. A plot without any points gets two or three
columns based on its name, as described in the Data model chapter.

Two call patterns exercise this file type, neither needing to change apart from the added calls to
`close()`: most grid types override
`SpatialGrid::write_xy()`/`write_xz()`/`write_yz()`/`write_xyz()`, each receiving an
already-constructed `SpatialGridPlotFile*` from `SpatialGrid`'s own driving code, which constructs
one file per view; `VoronoiMeshSnapshot` and `TetraMeshSpatialGrid` instead each construct their own
four files directly, in one combined `writeGridPlotFiles()` method, since their geometry doesn't
naturally split by view.

## Unstructured text output

Covers the four cases in the Data model chapter's Unstructured text file bundle —
`convergence.dat`, `grainpops`, `parameters.xml`, and `log.txt`.

The two probe outputs and the two files written for every simulation are handled differently,
because the latter remain regular files.

**Probe outputs.** `TextOutFile` takes an optional text type in its constructor. Without it, the
class writes a regular file as before, which `ColumnOutFile`, `ColumnOutFile::copyToBundle()`, and
the spatial grid plot files rely on. With a text type and a non-empty HDF5 candidate from
`output()`, it opens no file but collects the lines in memory, and `close()` writes the bundle with
the `text` dataset and the `type` attribute. `ConvergenceInfoProbe` and `DustGrainPopulationsProbe`
pass the type `free form`, so that their outputs become bundles only, without a regular file next to
them; their writing logic does not change, apart from an explicit call to `close()`. As for the
other output classes, writing a bundle can fail, and only `close()` can report that as a fatal
error; the destructor does not throw, and logs the message of a fatal error if the client did not
call `close()`.

**Log file and parameter file.** The static function `TextOutFile::copyToBundle()` — the shared
"copy" helper — reads a finished regular file into a `string` and passes it to
`H5BundleW::createTextDataset()`, setting the `type` attribute (`"XML"` or `"log"`). The command
line handler calls it for the parameter file and then for the log file at the end of each
simulation, after the last message has been logged, and also after a fatal error during the run, so
that the stored log includes the error; a failure to store the files after such an error is reported
on the console only, so that the original error remains the one that is reported. The files are not
stored when the simulation could not even be constructed. Only the root process stores them, so that
the per-process log files of the verbose multiprocessing mode remain regular files only. The
function logs no message: PTS recognizes a finished simulation by the message reporting the end of
the simulation at the end of the log file, so nothing may be logged after it, and the stored log is
identical to the regular one. The command line handler reports the storage on the console instead.

The iteration history files of the `HistoryProbe` use `ColumnOutFile::copyToBundle()` instead, as
described in the previous section.

`DustGrainPopulationsProbe` keeps writing its table through `TextOutFile`, with the same header
and columns as today, so that the regular output file does not change.
