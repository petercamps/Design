# Features

## The checkpoint probe

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

## Checkpoint contents

Each checkpoint is stored as a single bundle. In addition to defining its "when" point
(including the iteration index), a checkpoint may contain the following parts, each capturing
one aspect of the simulation's runtime state.

**Spatial grid.** Captures the grid's structure. For grid types whose structure is built up
rather than determined directly by the ski file or an input file — tree grids, whose
subdivision decisions depend on random sampling of an input density field, foremost among
them — reconstruction can be expensive or even impossible. Therefore, a checkpoint stores
the precise topology of such grids, rather than requiring it to be rebuilt on resume.

This part also includes enough information to visualize or sample grid-discretized
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
and secondary emission are stored separately. In a simulation with dust emission, this part also
holds the luminosity absorbed by the dust in each cell, accumulated along the photon paths as
proposed in the [Dust self-absorption](dust-self-absorption/01-introduction.md) design note. The
dust emission is normalized to these values, so they must be restored for the secondary emission
of a resumed run to be correct.

**Recorded fluxes.** Holds each instrument's accumulated detections — SEDs,
data cubes, and similar — built up one photon packet at a time over the course of the run.
The data stored here is not yet calibrated to instrument properties such as distance
or pixel scale (such calibration happens just before the instrument output is written).
If applicable, the accumulated statistics (higher moments of the detected fluxes)
are stored as well.

**Iteration history.** Holds the series kept in the central iteration history proposed in the
[Iteration history](iteration-history/01-introduction.md) note: the values of each scalar series
over its most recent iterations, including the aggregates of the medium state, and the running sums
of each cell window. Convergence criteria compare the current iteration with earlier ones, and
dynamic grid refinement averages per-cell fields over several iterations. Restoring the history
lets both continue after resuming exactly as they would have in an uninterrupted run.

The [Data model](checkpointing/03-data-model.md) chapter provides details on how these are stored in
HDF5.

## Checkpoint probe behavior

The checkpoint probe outputs only data that is available in the simulation. For example,
a no-medium simulation does not have a spatial grid, and an extinction-only simulation may
not have a radiation field.

As indicated above, the checkpoint probe is invoked for every "when" point that actually
occurs in a simulation. This always includes the Setup and Run checkpoints, and may
include iteration checkpoints. Conceptually, the probe outputs all available parts at
each checkpoint.

In practice, for datasets that remain unchanged, the probe simply links to the dataset
stored during a previous checkpoint. This makes every checkpoint's output
self-consistent while avoiding data duplication. For example, if the spatial grid
never changes during a simulation, it is stored only once. Similarly, some or all of the
medium state properties may be constant and are thus stored only once.

## Unsupported features

**History probe output.** The `HistoryProbe` proposed in the Iteration history note writes a
separate file for each iteration loop, with a row for each iteration it performs. After resuming,
the file of the loop being resumed into therefore starts at the first iteration of the resumed run,
and the files of loops that ended before the checkpoint are not written again. The rows for earlier
iterations are in the output of the original run. If the resumed run writes to the same output
location, the file of the resumed loop replaces the one written by the original run.

**Spatial grid types.** The linear cell list representation is initially supported only
for grids with cuboidal, axis-aligned cells. This includes hierarchical octree and
binary-tree grids, AMR-based grids, and Cartesian grids. Consequently, other grid types
lack an easy visualization mechanism.

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
checkpointed, they will reset to zero when resuming. This could be resolved by adding
extra datasets to the checkpoint, but this is left for future consideration.

## Resuming from a checkpoint

A new command-line option, `-c`, specifies the checkpoint data used to resume a simulation:

```
-c <dir>/<hdf>:<suite>
```

This follows the same `<dir>/<hdf>:<suite>` format as `-i` and `-o` (see Command-line syntax in
the [HDF5 input](hdf5-input/02-features.md) note), except that `<hdf>` is required rather than
optional, since a checkpoint exists only inside an HDF5 file, never as a collection of plain files.

By default, SKIRT resumes from the most recent checkpoint found under the given suite. To
resume from an earlier one instead, extend the suite with that checkpoint's own name, i.e.
`<suite>/<checkpoint>` rather than `<suite>` alone (the [Data model](checkpointing/03-data-model.md)
chapter explains how individual checkpoints are named).

Resuming does not read the ski file from the checkpoint data. The ski file governing the
resumed run is always given explicitly on the command line, exactly as for a fresh run, and
it **must** be the exact same file used to produce the checkpoint — SKIRT cannot reliably detect a
mismatch, so a different or modified ski file will silently produce incorrect results.

Assuming a ski file `mysim.ski` and three HDF5 files `in/data.hdf5`, `out/data.hdf5`, and
`chk/data.hdf5`, where the last one holds a checkpoint written by a previous run:

- `skirt -i in/data.hdf5 -o out/data.hdf5 -c chk/data.hdf5 mysim.ski` — input, output, and
  checkpoint each use their own file, entirely independent of one another.
- `skirt -i in/data.hdf5 -o in/data.hdf5 -c in/data.hdf5 mysim.ski` — all three share a
  single file: the same file that served as input and output for the previous run, and
  therefore already holds its checkpoints, continues to serve that role for the resumed run.

Either way, resuming loads the spatial grid, medium state, radiation field, recorded fluxes,
and iteration history stored in the selected checkpoint, and continues the simulation from the
corresponding "when" point onward instead of redoing any of the work that checkpoint already
reflects. For example, resuming from a checkpoint written after a primary-emission iteration
continues with the next primary-emission iteration.

This has one limitation, following directly from restricting checkpoints to well-defined,
between-phases points (see The checkpoint probe, above): a single long stretch between two
"when" points cannot itself be interrupted and resumed. A non-iterating, extinction-only
simulation is a good example: its only checkpoints are Setup and Run, with the
entire, possibly very long, photon packet launch running uninterrupted in between, so
resuming after an interruption means repeating that launch in full. The next section
describes a workaround.

## Iterating across simulations

Resuming from a checkpoint, above, requires the resumed run's ski file to be identical to
the one that produced the checkpoint — but SKIRT does not fully verify this. This section
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
