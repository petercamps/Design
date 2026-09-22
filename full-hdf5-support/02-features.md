# Features

## Prerequisites

SKIRT uses the official C HDF5 library — not the C++ wrapper layer that HDF5 also
provides. HDF5 support is optional: the SKIRT build procedure offers a CMake option called
`BUILD_HDF5`, following the same pattern already used for SKIRT's MPI support. When enabled,
this option locates and links the HDF5 library and compiles in the corresponding
input/output/checkpoint code; when disabled (the default), SKIRT builds exactly as it does
today, with no dependency on HDF5.

Before enabling `BUILD_HDF5`, the HDF5 C library must be installed on the build machine.

### Obtaining and installing HDF5

The library, its source code, and prebuilt binaries for Windows, macOS, and Linux are
distributed by [The HDF Group](https://github.com/HDFGroup/hdf5/releases), the organization
that develops and maintains HDF5. Full installation documentation is available from the
[HDF5 documentation](https://support.hdfgroup.org/documentation/) site; the summary below
should be enough for a first-time installation.

**macOS.** The simplest route is [Homebrew](https://brew.sh):

```bash
brew install hdf5
```

**Ubuntu and other Debian-based Linux.** Install the development package, which includes the
headers needed to compile against the library:

```bash
sudo apt install libhdf5-dev
```

**Other Unix-like systems (Fedora, RHEL, and derivatives).** Use the equivalent `dnf`/`yum`
package:

```bash
sudo dnf install hdf5-devel
```

For other Unix flavors, check the distribution's package manager first; if no package is
available, HDF5 can be built from source with CMake, following the instructions linked above.

**Windows.** The most convenient option for a CMake-based project such as SKIRT is
[vcpkg](https://vcpkg.io):

```bash
vcpkg install hdf5
```

Prebuilt Windows installers are also available from the releases page linked above.

### HPC systems

On shared HPC clusters, HDF5 is usually already installed as an environment module rather
than needing to be built by hand. A typical example (exact module names and versions vary by
site — check `module avail hdf5` on the cluster in question):

```bash
module load hdf5
```

## Input

This section describes how SKIRT locates its input files once HDF5 support is enabled.

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

The [Data model](full-hdf5-support/03-data-model.md) chapter explains how each of these maps to HDF5
storage mechanisms and how to generate those from Python.

**Incompatibility notes**:

- Support for the `scol` format, a less-frequently-used SKIRT-specific binary alternative
  to regular text column files, is dropped because the HDF5 alternative accomplishes
  the same goals in an industry-standard way.
- Support for saving and re-loading spatial grid tree topology to and from text files is
  dropped and replaced by a more general HDF5 storage mechanism.

### HDF5 concepts

An HDF5 file is organized like a small filesystem. It has a root group, which can contain
further groups as well as datasets — the actual blocks of stored data. A dataset is addressed
by a path within the file, built from the names of the groups leading up to it, similar to a
file path.

Many of the things SKIRT itself treats as one self-contained named object — an input file's
worth of data, an output file, one component of a checkpoint — are more naturally
represented as several related datasets than as a single one. This document calls such a
SKIRT-defined named object a **bundle**: in the file, a bundle is always an HDF5 group, even
when it happens to need only a single dataset, containing one or more datasets with names
fixed by SKIRT itself. Whenever this chapter refers to `<suite>/<bundle>` addressing a
location in the HDF5 file, it is a bundle's group being addressed rather than a
literal HDF5 dataset. The [Data model](full-hdf5-support/03-data-model.md) chapter defines the exact bundle
for each input, output, and checkpoint case.

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
and select the right one per run through the `-i` option.

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
time, with no special mode or coordination needed. The one requirement is that no process
still has the file open for writing. A write-open imposes this restriction as long as it
stays open — not just while a write call is actually in progress — so in practice the writer
needs to have closed the file, not merely paused, before readers open it.

## Output

This section describes how SKIRT writes its output files once HDF5 support is enabled.

### File types

SKIRT currently writes the following kinds of output files:

- **Text column files** — employed for all single-axis tables: SEDs, per-cell and
  per-position probe output, and instrument statistics tables (`FluxRecorder`).
  Uses the same header-comment convention as the input side (`# column N: ... (unit)`).
- **FITS files** — employed for instrument frames and data cubes,
  and for planar cuts or projections produced by probes.
- **Spatial grid plot files** — a polyline format where
  each line holds 2 or 3 coordinates meaning "draw a line to this point", and a blank line
  starts a new, disconnected segment. Every spatial grid writes one of these to plot its own
  cell geometry.
- **Unstructured text files** - the `convergence.dat` plain text file intended for human
  consumption, written by `ConvergenceInfoProbe`.
- **XML file** - the `parameters.xml` file, a reformatted version of the ski file
  governing the simulation.
- **Log file** - The `log.txt` plain text file with progress, warning and error messages.
  Because this file grows as the simulation runs, and users should be able to view the
  simulation's progress in real-time, the log file is always written as a regular file
  and then stored in the HDF file after the simulation has ended.

The [Data model](full-hdf5-support/03-data-model.md) chapter explains how each of these maps to bundles in
HDF5, and how to read them from Python.

**Incompatibility note**:

- Support for saving and re-loading spatial grid tree topology to and from text files is
  dropped and replaced by a more general HDF5 storage mechanism.

### Command-line syntax

The command-line option that specifies the output location, `-o`, follows the same
`<dir>/<hdf>:<suite>` format as `-i`, with the same meaning for each component (see
Command-line syntax under Input, above).

### How output files are written

If only `<dir>` is given, SKIRT behaves exactly as before: every output file is written as a
plain file inside `<dir>`.

If `<hdf>` is also given, SKIRT writes into that HDF5 file instead. The file as a whole is
never replaced: if it already exists, SKIRT opens it and adds to it. Each individual output
is written as a new bundle, or replaces an existing one if a bundle with the same name is
already present; every other bundle in the file is left untouched. The bundle name matches
the plain output file that would otherwise have been produced, prefixed with `<suite>` if
given, exactly as on the input side.

This makes it possible to collect the output of several simulations in a single HDF5 file,
each under its own suite, or to store both the input and the output of a single simulation
together in one file.

### Examples

Assuming a ski file `mysim.ski`, an input directory `in`, an output directory `out`, and an
HDF5 file `data.hdf5`:

- `skirt -i in -o out/data.hdf5 mysim.ski` — plain-file input, HDF5 output: input files are
  read from `in` as before, while every output file is written as a bundle inside
  `out/data.hdf5` instead of as a plain file in `out`.
- `skirt -i in/data.hdf5 -o out mysim.ski` — the reverse: input files are sought as bundles
  in `in/data.hdf5`, while every output file is written as a plain file in `out`, as before.
- `skirt -i in/data.hdf5 -o in/data.hdf5 mysim.ski` — input and output share the same HDF5
  file: input bundles are read from it, and output bundles are added into that same file
  alongside them.

`-i` and `-o` are independent: each may target a plain directory or an HDF5 file regardless
of what the other one uses.

### Concurrency

HDF5 does not support safe, uncoordinated writes to the same file from more than one
independent process — only a single writer is allowed at a time. Therefore, simulations that
run concurrently should each write to their own HDF5 file. SKIRT output files can be combined
afterward, either by merging them, or by building a small "umbrella" HDF5 file that uses
[external links](https://support.hdfgroup.org/documentation/hdf5/latest/group___h5_l.html)
to present the separate files as a single navigable hierarchy, without copying any data.

Using the same HDF5 file for both `-i` and `-o` in a given simulation, as in the last
example above, does not violate this concurrency rule: a single process reading from and
writing to a file it already has open is explicitly allowed.

When a simulation runs in MPI multi-processing mode, SKIRT ensures that there are no
concurrency conflicts between these processes.

## Checkpointing

### Introduction

A checkpoint is a snapshot of a SKIRT simulation's complete internal state, written
periodically while the simulation runs. Resuming from a checkpoint continues the simulation
from that point onward, rather than restarting it from the beginning.

Beyond resuming after an interruption — a crash, or a cluster job hitting its time limit —
the checkpoint file is an ordinary, self-describing HDF5 file, which makes it useful in its
own right:

- **Inspection and visualization in Python.** Its contents — the radiation field, per-cell
  medium state, and so on — can be explored directly with `h5py`, independently of whether
  the run is ever actually resumed.
- **Iterating across simulations.** The saved state can seed a new run configured with more
  photon packets or additional iterations, to improve signal-to-noise or convergence, rather
  than starting that more expensive run from scratch. This makes it practical to chain
  several SKIRT runs together, with convergence judged externally — from the intermediate
  results — rather than automatically inside SKIRT itself.
- **Refining the spatial grid.** A simulation's spatial grid can likewise be refined based on
  the results of a previous, coarser run. This is a more advanced case: carrying medium
  state across a change in spatial resolution will need some form of remapping.

### The checkpoint probe

Checkpointing is implemented as a specialty probe, configured in the ski file exactly like
any other probe. Because a checkpoint's contents are too complex to express as plain text or
FITS output, this probe is only available when SKIRT is built with `BUILD_HDF5` enabled.

Like every probe, the checkpoint probe runs at the fixed "when" points already defined by
SKIRT's probe mechanism:

| When | Runs |
| --- | --- |
| `Setup` | Once, after every item has been configured, before any photon packet is shot. |
| `Primary` | After each primary-emission iteration, once its medium state and radiation field are updated. |
| `Secondary` | After each secondary-emission iteration, once its medium state and radiation field are updated. |
| `Run` | Once, after every photon packet has been emitted and detected — the very end of the run. |

Restricting checkpoints to these four points, rather than an arbitrary moment, keeps the
implementation manageable: the simulation is always in a well-defined, between-phases state
when a checkpoint runs, so there is no need to save fine-grained progress state such as how
many photon packets have already been sent. Concurrency has also already settled down by
then — like every probe, the checkpoint probe is always called from the main thread, never
from a worker thread mid-loop.

Unlike other probes, at most one checkpoint probe is allowed per simulation. That single
probe fires at all four points listed above — each firing is referred to as a checkpoint
from here on — and decides autonomously what to store at each one, as described below.
Consequently, the checkpoint probe exposes no configuration properties beyond the
`probeName` property every probe inherits from the `Probe` base class.

### Checkpoint bundles

In addition to defining its "when" point (including the iteration index), a checkpoint
may save the following bundles, each capturing one aspect of the simulation's
runtime state.

**Spatial grid.** Captures the grid's structure. For grid types whose structure is built up
rather than determined directly by the ski file or an input file — tree grids, whose
subdivision decisions depend on random sampling of an input density field, foremost among
them — reconstruction can be expensive or even impossible. Therefore, a checkpoint stores
the precise topology of such grids, rather than requiring it to be rebuilt on resume.

This bundle also includes enough information to visualize or sample grid-discretized
quantities without needing to fully reconstruct the grid. Grids with cuboidal, axis-aligned
cells — tree, AMR, and Cartesian grids — store a linear list of corner coordinates for each
cell, regardless of the structural relationships between cells.

**Medium state.** Represents the discretization of the simulation's medium properties
on the spatial grid. This includes per-cell quantities such as cell volume and bulk velocity,
and per-cell, per-medium-component quantities such as density (mandatory), temperature, or
custom quantities defined by the configured material mix.

Even if the medium state does not vary during the simulation, it is still necessary to store
it in a checkpoint. The medium property values are often determined during setup by randomly
sampling the input model's spatial distribution. Repeating the process would never produce
the exact same result. Furthermore, the checkpoint data can be used for visualization.

**Radiation field.** Holds the per-cell, per-wavelength radiation field accumulated from
every photon packet traced so far. If applicable, the results accumulated from primary
and secondary emission are stored separately.

**Recorded fluxes.** Holds each instrument's accumulated detections — SEDs,
data cubes, and similar — built up one photon packet at a time over the course of the run.
The data stored here is not yet calibrated to instrument properties such as distance
or pixel scale (such calibration happens just before the instrument output is written).
If applicable, the accumulated statistics (higher moments of the detected fluxes)
are stored as well.

The [Data model](full-hdf5-support/03-data-model.md) chapter provides details on how these are stored in HDF5.

### Checkpoint probe behavior

The checkpoint probe outputs only data that is available in the simulation. For example,
a no-medium simulation does not have a spatial grid, and an extinction-only simulation may
not have a radiation field.

As indicated above, the checkpoint probe is invoked for every "when" point that actually
occurs in a simulation. This always includes the Setup and Run checkpoints, and may
include iteration checkpoints. Conceptually, the probe outputs all available bundles at
each checkpoint.

In practice, for portions of the data that remain unchanged, the probe simply links to the
data stored during a previous checkpoint. This makes every checkpoint's output
self-consistent while avoiding data duplication. For example, the spatial grid bundle
never changes during a simulation, so it is stored only once. Similarly, some or all of the
medium state properties may be constant and are thus stored only once.

### Unsupported features

**Convergence history.** Convergence criteria determine when to end iterations over primary
and/or secondary emission. Some criteria are implemented globally for all medium components
of a given type (dust self-absorption); others are implemented by individual material mixes
(e.g. `DiffuseIonizedGasMix`) or dynamic state recipes (e.g. `DustDestructionRecipe`).
Often such recipes retain some history from one or more earlier iterations to help determine
whether convergence has been reached. The checkpoint probe does _not_ store this information.
After resuming a simulation, one or more extra iterations may be required to build up this
history (again). In practice, this should not be a concern.

**Spatial grid types.** The linear cell list representation is initially supported only
for grids with cuboidal, axis-aligned cells. This includes hierarchical octree and
binary-tree grids, AMR-based grids, and Cartesian grids. Consequently, other grid types
cannot be used as input to a follow-up grid-refinement simulation and lack an easy
visualization mechanism.

**Geometry decorators.** The `ClumpyGeometryDecorator` with a default `seed` value of
0 is not supported because it will position the clumps differently when resuming. This is
easily resolved by configuring the decorator with a nonzero seed.

Some decorators (`ClipGeometryDecorator` and `RedistributeGeometryDecorator` subclasses)
renormalize the advertised total mass by randomly sampling densities. When used as a source,
the luminosity normalization will vary slightly after resuming. When used as a medium,
there is no discrepancy because the medium state will be reloaded from the checkpoint data
without querying the geometry again.

**Launched packets probe.** The `LaunchedPacketsProbe` keeps track of the number of photon
packets launched during primary and secondary emission. Because the counters are not
checkpointed, they will reset to zero when resuming. This could be resolved by adding an
extra checkpoint bundle, but this is left for future consideration.

## Using checkpoints

### Resuming from a checkpoint

A new command-line option, `-c`, specifies the checkpoint data used to resume a simulation:

```
-c <dir>/<hdf>:<suite>
```

This follows the same `<dir>/<hdf>:<suite>` format as `-i` and `-o` (see Command-line
syntax under Input, above), except that `<hdf>` is required rather than optional, since a
checkpoint exists only inside an HDF5 file, never as a collection of plain files.

By default, SKIRT resumes from the most recent checkpoint found under the given suite. To
resume from an earlier one instead, extend the suite with that checkpoint's own name, i.e.
`<suite>/<checkpoint>` rather than `<suite>` alone (the [Data model](full-hdf5-support/03-data-model.md) chapter
explains how individual checkpoints are named).

Resuming does not read the ski file from the checkpoint data. The ski file governing the
resumed run is always given explicitly on the command line, exactly as for a fresh run, and
it **must** be the exact same file used to produce the checkpoint — SKIRT cannot detect a
mismatch, so a different or modified ski file will silently produce incorrect results.

Assuming a ski file `mysim.ski` and three HDF5 files `in/data.hdf5`, `out/data.hdf5`, and
`chk/data.hdf5`, where the last one holds a checkpoint written by a previous run:

- `skirt -i in/data.hdf5 -o out/data.hdf5 -c chk/data.hdf5 mysim.ski` — input, output, and
  checkpoint each use their own file, entirely independent of one another.
- `skirt -i in/data.hdf5 -o in/data.hdf5 -c in/data.hdf5 mysim.ski` — all three share a
  single file: the same file that served as input and output for the previous run, and
  therefore already holds its checkpoints, continues to serve that role for the resumed run.

Either way, resuming loads the spatial grid, medium state, radiation field, and recorded
fluxes stored in the selected checkpoint, and continues the simulation from the corresponding
"when" point onward instead of redoing any of the work that checkpoint already reflects.
For example, resuming from a checkpoint written after a primary-emission iteration continues
with the next primary-emission iteration.

This has one limitation, following directly from restricting checkpoints to well-defined,
between-phases points (see The checkpoint probe, above): a single long stretch between two
"when" points cannot itself be interrupted and resumed. A non-iterating, extinction-only
simulation is a good example: its only checkpoints are Setup and Run, with the
entire, possibly very long, photon packet launch running uninterrupted in between, so
resuming after an interruption means repeating that launch in full. The next section
describes a workaround.

### Iterating across simulations

Resuming from a checkpoint, above, requires the resumed run's ski file to be identical to
the one that produced the checkpoint — but SKIRT never actually verifies this. This section
deliberately makes constructive use of that gap to let a handful of ski file parameters be
changed between runs, so that a simulation can be pushed further
without repeating work already reflected in a checkpoint. Which checkpoint to resume from,
and which parameters are supported, depends on how the simulation iterates.

**Case 1 — non-iterating, extinction-only simulations.** This is the workaround
promised above: although such a simulation cannot be resumed after being interrupted
mid-launch, a run that completed normally can still be resumed from its Run checkpoint —
its only other checkpoint besides Setup — purely to add more packets:

| Parameter | ski file property | Allowed change |
| --- | --- | --- |
| Photon packets | `numPackets` | Increase only |

**Case 2 — iterating over primary emission only.** Resuming from a Primary checkpoint to run
more iterations supports two parameters:

| Parameter | ski file property | Allowed change |
| --- | --- | --- |
| Iterations | `maxPrimaryIterations` | Increase only |
| Photon packet multiplier | `primaryIterationPacketsMultiplier` | Any |

The new multiplier value applies only to the iterations run after resuming, not
retroactively to the ones already reflected in the checkpoint.

**Case 3 — iterating over secondary emission, on its own or merged with primary emission.**
Resuming from a Secondary checkpoint to run more iterations supports:

| Parameter | ski file property | Allowed change |
| --- | --- | --- |
| Iterations | `maxSecondaryIterations` | Increase only |
| Photon packet multiplier | `secondaryIterationPacketsMultiplier` | Any |
| Photon packet multiplier<br>(merged case only) | `primaryIterationPacketsMultiplier` | Any |

As in case 2, a new multiplier value applies only to the iterations run after resuming.

For cases 2 and 3 to be effective, any convergence criteria specified in the original ski
file must be configured strictly, not liberally. An iteration loop exits as soon as its
convergence criteria are satisfied, regardless of how high the maximum number of iterations
is set — and a liberal, easily satisfied criterion is met almost immediately, ending the
loop well short of that maximum.

The parameter changes discussed above are safe given how a SKIRT simulation's
checkpoint resume operation works: a higher
packet count (case 1) simply launches the additional packets needed to reach the new total;
a higher iteration limit (cases 2 and 3) simply lets the already-running convergence loop
continue further than originally planned; and a packet multiplier scales only the packets
launched by the new iterations run after resuming. Either way, the checkpoint's existing
contribution is built upon, never redone.

Changing any other ski file parameter is not supported and will likely cause undefined
behavior, silently or loudly. Specifically, there is currently no support for increasing
the number of photon packets in a non-iterating simulation that includes primary and
secondary emission; this is left for future consideration.

The mechanism described above can be used to judge convergence of simulation results
from the outside, across several resumed runs. After each run, the regular SKIRT output can
be inspected before deciding whether to continue:

- **Instrument output** — the calibrated SEDs, data cubes, and similar output
  written by each instrument — to see whether the physical result itself has stopped
  changing meaningfully from one run to the next. For cases 2 and 3, this requires that
  the previous simulation has run to completion so that instrument output has been written,
  even if the follow-up simulation resumes from the most recent iteration checkpoint.
- **Recorded flux statistics** — the raw, uncalibrated flux accumulators and their higher
  moments held in the checkpoint — to judge whether shot noise has dropped enough.
- **Medium state** — for simulations with a dynamic medium state, to check whether
  quantities such as level populations have stabilized.

This turns convergence into an externally driven loop; the following pseudo-code shows this
for case 1, extending the photon packet count of a non-iterating simulation:

```
packets = 1e6
skirt -i in.hdf5 -o run.hdf5 mysim.ski

loop:
    inspect run.hdf5
    if results have converged:
        stop
    packets = packets * 3
    edit mysim.ski: set numPackets to packets
    skirt -i in.hdf5 -o run.hdf5 -c run.hdf5 mysim.ski
```

### Restructuring tree policies

Checkpoint-based resuming, and the grid-reuse mechanism described in Reusing grid topology
below, require a tree grid to be reconstructible from a saved topology no matter how it was
originally configured to subdivide. This section proposes a restructuring of SKIRT's
hierarchical tree policies — the classes that determine how nodes get subdivided — that
makes this possible; the rest of this chapter assumes it is in place.

Today, a tree-based spatial grid is either a `PolicyTreeSpatialGrid`, configured with a
single construction policy, or a `FileTreeSpatialGrid`, which loads a previously saved
topology instead of building one — two separate, mutually exclusive classes. Resuming a
`PolicyTreeSpatialGrid` simulation would therefore need to silently substitute a different
grid class behind the user's back, rather than there being one mechanism that works
uniformly regardless of how the grid was originally configured. The proposal merges these
back into a single, concrete `TreeSpatialGrid` class, configured with a _list_ of
construction policies rather than a single one. `treeType`, `minLevel`, and `maxLevel` —
currently split between `PolicyTreeSpatialGrid` and the individual policy — become
properties of `TreeSpatialGrid` itself, shared by every policy in the list.

During construction, a node is subdivided as soon as any one policy in the list asks for
it. This makes the policies freely combinable: a grid can use dust density, electron
density, gas density, and the positions of an imported medium's particles all at once,
simply by listing one policy of each kind, rather than being restricted to a single
criterion. It also makes refining an existing grid with an extra criterion trivial — just
add one more policy to the list.

The following policies would be available. The current, single `DensityTreePolicy` splits
into one policy per material type, since its three material types already have
independent, separately configurable criteria; the existing `SiteListTreePolicy` carries
over unchanged; and two new policies take over what `FileTreeSpatialGrid` and
`NestedDensityTreePolicy` do today, as described below the table.

| Policy | Properties |
| --- | --- |
| `DustDensityTreePolicy` | `maxDustFraction`, `maxDustOpticalDepth`, `wavelength`, `maxDustDensityDispersion` |
| `ElectronDensityTreePolicy` | `maxElectronFraction` |
| `GasDensityTreePolicy` | `maxGasFraction` |
| `SiteListTreePolicy` | `numExtraLevels` (unchanged from today) |
| `BoxTreePolicy` | a bounding box, plus a nested `policy` |
| `CheckpointTreePolicy` | `filename` (an HDF5 checkpoint; see Reusing grid topology, below) |

**`CheckpointTreePolicy`** replaces `FileTreeSpatialGrid`: it loads a previously recorded
topology from an HDF5 checkpoint instead of computing one, but now as one policy among
others rather than a separate spatial grid class. Resuming a tree grid relies on exactly
this: whatever policies the ski file configures, resuming substitutes a `CheckpointTreePolicy`
pointing at the `-c` checkpoint's own spatial grid bundle in their place, replaying the
saved topology instead of resampling it. The same mechanism is also what makes refining a
previously saved grid with an extra criterion possible: list the checkpoint policy alongside
a fresh density policy, and the combined grid subdivides at least everywhere the checkpoint
did, plus wherever the new criterion additionally asks for it.

**`BoxTreePolicy`** avoids one policy inheriting from another. Today,
`NestedDensityTreePolicy` specifies a higher resolution in a region of interest by
inheriting from `DensityTreePolicy` for the outer criteria and holding a second
`DensityTreePolicy` instance for the inner region — a pattern that would need a new
subclass for every kind of criterion one might want to nest. `BoxTreePolicy` holds an
arbitrary policy as an ordinary sub-item, together with a bounding box — a cuboid aligned
with the coordinate axes — and defers to that
policy only for nodes intersecting the box. Listed alongside an ordinary, ungated policy for
the rest of the domain, it reproduces the same result for the typical case, since the
box-gated policy's criteria are normally the stricter of the two and therefore dominate
wherever they overlap. The same mechanism now works for any kind of policy, and for
more than one nested box, without a dedicated class for each combination.

### Reusing grid topology

There are situations where multiple simulations should use the exact same spatial grid, or
where the time needed to build a grid should be avoided — for example, an octree whose
structure is determined by sampling the input density distribution. A typical case is a
study of several similar input models with variations in material properties, where the
underlying spatial distribution of the medium, and thus the ideal grid, stays the same
across runs.

This is not an issue for spatial grid types whose structure is fully determined by the ski
file: building such a grid is fast, and for a given ski file, always produces the exact same
result. It is an issue for the SKIRT spatial grid types whose structure instead depends on
sampling the input density distribution:

- **Tree grids** (`PolicyTreeSpatialGrid` configured with the default `DensityTreePolicy`,
  or with `NestedDensityTreePolicy`) sample density at many randomly selected positions to
  decide where to subdivide.
- **`VoronoiMeshSpatialGrid`** and **`TetraMeshSpatialGrid`**, unless configured with their
  `File`, `ImportedSites`, or `ImportedMesh` policy, place their sites or vertices by random
  sampling — either from a synthetic distribution (`Uniform`, `CentralPeak`) or, more
  commonly, importance-sampled from the actual input density (`DustDensity`, the default
  for both grids, `ElectronDensity`, or `GasDensity`) — before tessellating them.

Not only does the random sampling take time, but the resulting grid will differ subtly
between SKIRT runs because the employed pseudo-random sequence is unique for each run
(except in single-threaded execution mode).

Other spatial grid types are unaffected: their structure is either fully parametric (e.g.
`CartesianSpatialGrid` and the `Cylinder`/`Sphere` grids) or taken wholesale from an
imported mesh (`AdaptiveMeshSpatialGrid`), with no sampling involved either way.

The spatial grid checkpoint bundle already captures the resolved topology of these
variable grid types (see Checkpoint bundles, above), so a follow-up simulation can load it
instead of rebuilding the grid from scratch. Because this is not a resume, the source is not
given through the `-c` command-line option; instead, each variable grid type gains a ski
file option to load its topology from a previous checkpoint. The following treats each case
in turn.

**Tree grids.** The new mechanism replaces the existing one: the
`TreeSpatialGridTopologyProbe` is removed, since the spatial grid checkpoint bundle already
captures the same topology as a side effect during checkpointing. The `filename` property
of `CheckpointTreePolicy` names an HDF5 file — resolved relative to the simulation's input
directory, like any other input file — followed by a mandatory `:<checkpoint>` component. This
component consists of an optional suite and a mandatory name identifying which checkpoint
to load from. `CheckpointTreePolicy` then picks out the relevant bundle within that
checkpoint on its own. The saved topology remains scale-free, so the simulation loading it
still specifies the domain extent itself.

**`VoronoiMeshSpatialGrid`.** The `File` policy is extended: the
`filename` property can still name a plain text file of site positions, or now instead an
HDF5 file, resolved relative to the input directory, with
the same mandatory `:<checkpoint>` component described above. Either way, `File` loads
previously recorded site positions rather than sampling new ones; the tessellation itself
still runs as before.

**`TetraMeshSpatialGrid`.** The same extension applies to the `File` policy: the
`filename` property can name either a plain text file or an HDF5 file. This loads previously
recorded vertex positions rather than sampling new ones, while the Delaunay
tetrahedralization still runs as before.

### Refining the spatial grid

Today's hierarchical tree subdivision criteria focus mostly on the medium's density distribution.
Resolving gradients in the radiation field itself is at least as important, and arguably
more so, since it is the quantity that ultimately determines the accuracy of every
simulation result, while density is only a proxy for it.

Because the radiation field is one of the bundles captured in a checkpoint (see Checkpoint
bundles, above), it can be used for exactly this purpose across two simulations. A first,
deliberately cheap simulation computes an approximate radiation field — with fewer photon
packets, a coarser spatial grid, or a wavelength grid restricted to the spectral range of
interest — and writes it to a checkpoint. A follow-up simulation then reads that checkpoint
back to refine the spatial grid further, wherever the approximate field indicates that it is
not yet properly resolved.

Because this follow-up run is a fresh simulation rather than a resume, there is no need
to remap medium state or radiation-field accumulators onto the changed grid — every
quantity is simply computed fresh on the refined grid, exactly as in any ordinary run.
The only requirement is that the original simulation's spatial grid checkpoint includes the
linear cell list representation (see Spatial grid types under Unsupported features,
above), so that the radiation field can be sampled without fully reconstructing the
original grid.

This is a direct application of the restructured tree policies proposed above (see
Restructuring tree policies). The follow-up simulation's grid could combine a
`CheckpointTreePolicy` — which reuses the first simulation's grid as a starting point —
with a new policy based on the radiation field. One possible such policy is sketched
below, extending the table given there; the exact set of properties, and how it should
relate to primary versus secondary emission, are left for future consideration.

| Policy | Properties |
| --- | --- |
| `RadiationFieldTreePolicy` | `maxFieldFraction`, `maxFieldDispersion`, `minWavelength`, `maxWavelength` |

Like `CheckpointTreePolicy`, it takes a `filename` property identifying the source
checkpoint, using the same HDF5-file-plus-checkpoint syntax. `maxFieldFraction` mirrors
`maxDustFraction`: it limits the fraction of the total radiation field energy contained in
each cell, forcing subdivision in cells that dominate the energy budget.
`maxFieldDispersion` mirrors `maxDustDensityDispersion`: it limits how much the field is
allowed to vary within a single cell, sampled from the checkpoint rather than computed on
the fly. `minWavelength` and `maxWavelength` restrict both criteria to a spectral range of
interest, defaulting to the full range covered by the source checkpoint.
