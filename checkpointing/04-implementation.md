# Implementation

## Checkpoint API

A checkpoint is an ordinary bundle, so it is read and written through the `H5BundleR` and
`H5BundleW` classes of the `hdf5` build target described in the
[HDF5 input](hdf5-input/04-implementation.md) and [HDF5 output](hdf5-output/04-implementation.md)
notes. The checkpoint probe creates a checkpoint with `H5Lib::createBundle()`, sets the checkpoint
attributes (`when`, `iteration`, the ski parameters from Iterating across simulations,
`grid_type` and `tree_type`, and `history_loop` and `history_iteration`) with `setAttribute()`,
and creates a dataset for each stored array with `createDataset()`, named with the prefixes
listed in the Data model chapter. Resuming opens the checkpoint with `H5Lib::openBundle()`, and
checks which parts are present with `hasDataset()`.

The only addition is a function for reusing unchanged data:

**`H5BundleW`**

- `linkDataset(<name>, <suite>/<bundle>)` — hard-links this bundle's dataset `<name>` to the
  same-named, already-written dataset of the bundle at `<suite>/<bundle>`, instead of writing a
  new copy of it — what Checkpoint probe behavior in Features relies on to avoid re-storing
  unchanged data across checkpoints. A hard link cannot cross files, so the other bundle must be
  in the same HDF5 file.

To find the checkpoints in a suite, for example to resume from the most recent one, a reader lists
the bundles with `H5SuiteR::getBundleNames()` and recognizes the checkpoints by their `when`
attribute. The most recent checkpoint is the one with the latest `created` attribute.

Wherever a command-line option or ski file setting requires HDF5 — `-c` always does, and so does
a `TopologyTreePolicy::filename` that names a checkpoint — setup-time validation checks
`H5Lib::available()` first and reports a clear, immediate error if HDF5 support is missing,
rather than letting execution reach a stub's fatal error later.

## Name resolution

The `FilePaths` class's API, as adjusted in the HDF5 input and HDF5 output notes, is extended as
follows:

- `setCheckpointPath(string value)` — the value of the `-c` option: always a directory plus
  an HDF5 file and suite, since a checkpoint has no plain-file form.
- `checkpoint(string name)` returns a single string, since a checkpoint has no plain-file
  form.

Assuming `setOutputPrefix("mysim")`, i.e. a ski file `mysim.ski`,
`checkpoint("checkpoint_primary_2")` resolves as follows:

| `-c` | Result |
| --- | --- |
| `./data.hdf5` | `data.hdf5:mysim_checkpoint_primary_2` |
| `./data.hdf5:run1` | `data.hdf5:run1/mysim_checkpoint_primary_2` |
| `./data.hdf5:campaign7/run1` | `data.hdf5:campaign7/run1/mysim_checkpoint_primary_2` |

`-c` always requires `<hdf>` (Resuming from a checkpoint in Features), so there is no
plain-only row here.

## The checkpoint probe

The Data model chapter already specifies, part by part, exactly what the checkpoint
probe stores; this section only covers what implementing it requires that the Data model and
Features chapters do not already say.

### Firing at every when point

Unlike every other probe, the checkpoint probe must fire at all four "when" points (Setup,
Primary, Secondary, Run), not just the one `when()` selects — see The checkpoint probe in
Features. `Probe`'s own dispatch functions, `probeSetup()`, `probeRun()`, `probePrimary(int
iter)`, and `probeSecondary(int iter)`, are not virtual today; each simply checks `when()`
against its own point and calls `probe()` if it matches, doing nothing otherwise, so a
subclass cannot make itself run at more than one point through the existing mechanism at
all. These four functions need to become virtual, and the checkpoint probe needs to override
all four directly, unconditionally producing its output from each one rather than relying on
`when()`/`probe()` at all.

### Extra public getters needed per part

Each of the five parts of a checkpoint needs data that is not fully accessible through today's
public API. The gaps differ considerably in size.

**Spatial grid.** `TreeSpatialGrid` exposes nothing about its own node tree publicly — only
per-*cell* queries (`cellBox()`, `volume()`, ...) exist; `_nodev`, `root()`,
`nodeForCellIndex()`, and `cellIndexForNode()` are all private. New getters are needed for:

- `int numNodes() const` — `Nn`, the total node count (leaves and nonleaves).
- `vector<bool> nodeIsLeaf() const` — `grid_is_leaf`, one entry per node, in id order.
- `vector<int> nodeParentIds() const` — `grid_parent_id`, one entry per node, in id order.
- `string treeType() const` — `"BinTree"` or `"OctTree"`, for the `tree_type` attribute.

The `grid_min`/`grid_max` linear cell list needs no new getter: `cellBox(m)`, already public on every
cuboidal grid type, gives both corners directly. `VoronoiMeshSpatialGrid` and
`TetraMeshSpatialGrid` likewise need a new getter for their site or vertex positions, unless
one already exists somewhere in the snapshot classes backing them — not checked here.

**Medium state.** Already in good shape: `MediumSystem` — the class a probe would actually
use, per Probe's "public interfaces only" rule — already exposes `numCells()`, `numMedia()`,
`mix(m, h)`, and every per-cell/per-component value (`volume()`, `bulkVelocity()`,
`magneticField()`, `numberDensity()`, `metallicity()`, `temperature()`, `custom()`) as
public functions, and `MaterialMix::specificStateVariableInfo()` — which lists exactly which
specific variables (including custom ones, with their description and quantity) a component
requests — is public too. No new getters appear to be needed here at all.

**Radiation field.** `MediumSystem::meanIntensity(m)` is public, but it returns the
normalized mean intensity, not the raw, undivided accumulator the checkpoint needs
(see Radiation field in the Data model chapter for why the raw form is required). The
underlying raw accumulator, `MediumSystem::radiationField(m, ell)`, is private, and does not
appear to distinguish primary from secondary contributions the way the `rf_primary` and
`rf_secondary` datasets must. A new getter is needed for:

- The raw, per-wavelength-bin, unnormalized accumulator for a cell, separately for primary
  and secondary contributions — signature to be worked out once the underlying storage
  (`meanIntensity()` presumably sums over both) is examined more closely.

The wavelength grid itself (`rf_wavelength`/`rf_width`) does not need a new getter:
`Configuration::radiationFieldWLG()` is already public and should be sufficient.

**Recorded fluxes.** The largest gap of the five. `FluxRecorder`'s public API is entirely
configuration (`setWavelengthGrid()`, `setUserFlags()`, `includeLightCurve()`, ...) and
operation (`detect()`, `flush()`, `calibrateAndWrite()`); nothing lets a caller read back
either the raw accumulated values or what was configured. New getters are needed for:

- Which output types are enabled (SED / IFU / LC / STM) and, for each, which components are
  being tracked (single `total`, or the full six-way breakdown; scattering-level count;
  polarization; statistics) — none of this configuration state can currently be queried back.
- The raw, uncalibrated per-bin values for each enabled output type and component — exactly
  the values `calibrateAndWrite()` currently calibrates and writes in one step, with no
  intermediate access point.
- The configured wavelength and/or time grid — though the owning `Instrument` may already
  expose this itself, since it is the one that configured the recorder in the first place,
  making a new getter on `FluxRecorder` unnecessary.

**Iteration history.** Mostly in place: `IterationHistory` already lists its contents through
`allScalarSeries()` and `allCellWindows()` for the history probe, and exposes its current
`loop()` and `iteration()`. Each scalar series exposes its key, description, depth, quantity,
aggregation rule, and its values through `value(lag)`; each cell window its key, description,
number of cells, and number of accumulated iterations. New getters are needed for:

- `Lifetime lifetime() const`, on both `ScalarSeries` and `CellWindow`.
- `double sum(int m) const` on `CellWindow` — the raw running sum, since `mean()` divides by the
  number of accumulated iterations and returns NaN if there are none.

In addition, a small helper translates between an item pointer and its path in the simulation item
hierarchy (see Iteration history in the Data model chapter), by walking the `children()` of each
item from the root. This helper belongs with `SimulationItem`, since it applies to any item.

## Resuming from a checkpoint

Resuming means each affected class's setup needs a way to load its own state from the
checkpoint named by `-c` instead of computing it the normal way. That needs one new, shared
piece of infrastructure: something that resolves `-c`'s value once, via
`H5Lib::openBundle()`, and hands out the resulting `H5BundleR` to whichever
`setupSelfBefore()`/`setupSelfAfter()` functions need it, plus a simple "is this a resume at
all" query. Since the parts of a checkpoint are optional, each of the cases below
falls back to its normal, non-resume setup whenever its own datasets happen to be
absent (for example, resuming from a `Setup` checkpoint before any packet has been traced
leaves Radiation field absent). Datasets the checkpoint probe only linked to, rather than
storing fresh, are read like any other dataset, since HDF5 follows hard links transparently.

### Spatial grid

`AdaptiveMeshSpatialGrid` needs no changes at all: its structure is already fully
deterministic from the ski file and its own input, identical on every run, resumed or not.

`VoronoiMeshSpatialGrid` and `TetraMeshSpatialGrid` need their site- or vertex-placement
setup to check for a resume first and, if one applies, load positions from the checkpoint's
`grid_x`/`grid_y`/`grid_z` datasets — the same substitution their `File` policy already performs for the
separate "reusing grid topology" ski feature (see Features), just triggered automatically by
`-c` rather than by an explicit policy choice.

Tree grids need the most care. On resume, `grid_is_leaf` and `grid_parent_id` let the tree be
rebuilt directly, without needing to reproduce any particular construction order — but if a tree
can be subdivided further during later iterations, as proposed in the
[Dynamic grid refinement](dynamic-grid-refinement/01-introduction.md) note, resuming from an
earlier checkpoint and continuing the run means live construction must still be able to extend
the tree beyond what was recorded. The reconstruction therefore has to match each live node to its counterpart in
the recorded topology by following the same parent/child-slot path from the root — not by
assuming live construction visits nodes in the same order the checkpoint was originally
written in — so that once a live node falls past what was recorded, it correctly answers no
(nothing recorded here, fall through to whatever normal construction or refinement would
otherwise decide) rather than either inventing structure that was never recorded or refusing
to grow at all. This is the same matching problem the `TopologyTreePolicy`, for the "reusing
grid topology" feature, already has to solve, and resuming can reuse that same mechanism —
triggered automatically by `-c` there too.

### Medium state

`MediumSystem`'s setup currently calls `MediumState`'s `setXxx()` functions with values
sampled from the input model. On resume, it needs to call the same functions instead with
values read back from the checkpoint's per-dataset arrays (`medium_volume`,
`medium_bulk_velocity`, `medium_magnetic_field`, `medium_number_density_<h>`, and so on) — no new
public API on `MediumState` itself, since this is `MediumSystem`'s own setup writing into a member it already owns; only
`MediumSystem`'s internal setup logic needs the resume branch.

### Radiation field

Normally, `MediumSystem`'s primary and secondary accumulators start at zero and are built up
packet by packet over the course of the run. On resume, that same storage needs to be
pre-loaded from the checkpoint's `rf_primary`/`rf_secondary` datasets before the run continues, so
that further packets accumulate on top of the resumed values rather than starting over — the
whole reason those datasets are stored raw and undivided (see Radiation field in the Data
model chapter). Like Medium state, this is internal to `MediumSystem`'s own setup, not a new
public API.

### Recorded fluxes

Unlike the other three, this one needs new public API, not just an internal setup branch:
each `Instrument`'s `FluxRecorder` starts empty and fills up only through `detect()` calls
during the run, and nothing lets a caller load a previously accumulated value back in. Every
raw value the checkpoint probe's new getters (above) read out needs a matching setter here,
so that resuming can restore a `FluxRecorder` to exactly the state the checkpoint captured
before the run resumes accumulating on top of it.

### Iteration history

Series are created on demand, so the history of a resumed run starts empty, apart from the
aggregate series that clients declare during setup. Resuming recreates every stored series, with
its key, depth, lifetime, description, quantity, aggregation rule, and values, and every stored
cell window, with its running sums and number of accumulated iterations, and sets the current loop
and iteration index. This needs a new function on `IterationHistory`, taking the opened checkpoint
bundle, that translates each stored item path back to the live item and reports a fatal error if
the path does not lead to an item of the stored type.

Because declaring a series is idempotent, the clients need no changes: when a client declares a
series in the next iteration, it receives the restored series, with the same lifetime and
aggregation rule, and compares with the restored values exactly as it would have in an
uninterrupted run. This includes the aggregates declared during setup, which are overwritten with
the stored values.

The restore must happen after the steps that would otherwise disturb the restored state:

- after the medium system records the initial values of the aggregates at the end of setup;
- after `beginLoop()` for the loop being resumed into, which clears the series with loop lifetime.

`MonteCarloSimulation` therefore restores the history right after calling `beginLoop()` for the
loop it resumes into, or at the end of setup when resuming from a `Setup` or `Run` checkpoint. The
next `beginIteration()` then advances all series exactly as in an uninterrupted run.

### The checkpoint probe's own setup

Resuming affects the checkpoint probe's own setup too, independently of the parts
above. On a fresh run, `probeSetup()` always writes a `Setup` checkpoint; on resume, it must
not — `Setup` was already checkpointed in the original run (every resumable checkpoint
implies it), and writing another one would target the exact same bundle name
(`<prefix>_checkpoint_setup_0`), which `H5Lib::createBundle()` would erase and overwrite
outright. So on resume, `probeSetup()`'s override needs to skip writing entirely at that
point.

It still has work to do there, though: whatever bookkeeping the probe uses, for each dataset,
to decide "store fresh" versus "link to what I already wrote" needs to be
seeded from the checkpoint being resumed from, rather than starting empty the way it would
for a fresh run — otherwise the first post-resume checkpoint would have nothing to link to
and would re-store everything, including data (such as the spatial grid) that never changes.
Seeding is simple, thanks to `hasDataset()` and HDF5's transparent hard-link following:
for each dataset present in the resumed-from checkpoint, record that checkpoint's own
name as where the probe should link to next; for any dataset absent there (for example, those
of the Radiation field when resuming from `Setup` before any packet has been traced), leave it
unrecorded, exactly as for a fresh run — linking, when the data is unchanged, transitively
reaches whichever earlier checkpoint actually stored it fresh, so the probe never needs to
know more than the one checkpoint it is resuming from. Because hard links cannot cross files,
this applies only if the resumed run writes its checkpoints to the file it resumes from; otherwise
the first checkpoint after resuming stores all of its datasets fresh.

One thing this section does not need to solve: `probePrimary(int iter)`/`probeSecondary(int
iter)` simply use whatever iteration index they are called with, so getting `<iteration>`
right after resuming is not the checkpoint probe's own concern — see Iteration continuity,
below, for what that actually takes.

### Iteration continuity

This belongs to `MonteCarloSimulation`, which drives the primary- and secondary-emission
loops, not to the checkpoint probe. Two separate things need to happen.

The easy one: the next iteration's number and which phase to resume into both come directly
from the checkpoint's own `when`/`iteration` attributes — resuming from
`mysim_checkpoint_primary_2` means continuing at primary iteration 3, or moving on to
secondary emission (or `Run`) if primary has already reached `max_primary_iterations`.

The other one: whether there even *should* be a primary iteration 3 depends on convergence. The
decision for iteration 2 was already made when the checkpoint was written, and is recorded in the
"loop converged" series of the iteration history. If that loop converged, resuming moves on to the
next phase. Otherwise, the loop continues with iteration 3, and because the iteration history is
restored (see Iteration history, above), every convergence criterion compares the new iteration
with the same earlier values as in an uninterrupted run. This holds for the primary, secondary, and
merged loops alike, and for dynamic grid refinement, whose cell windows are restored as well.
