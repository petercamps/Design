# Data model

## Links

Checkpoints rely on one HDF5 concept beyond those introduced in the
[HDF5 input](hdf5-input/01-introduction.md) note. Membership in a group is not intrinsic to the
object it contains, but implemented through a separate **link** that the group owns and that
points to a named object elsewhere in the file. A **hard link**, the normal case, keeps an object
alive as long as at least one hard link to it still exists. Several groups can therefore hold
hard links to the same dataset, which is how a checkpoint refers to an unchanged dataset stored by
an earlier checkpoint without copying it.

## Checkpoint bundle

A checkpoint is a single bundle: an HDF5 group holding all of the datasets that capture the
simulation's state at that point, plus the attributes described below. The datasets fall into five
parts, described in the following sections: Spatial grid, Medium state, Radiation field, Recorded
fluxes, and Iteration history. Which parts are actually present depends on what data the simulation has
available at that point (see Checkpoint probe behavior in the Features chapter). A reader can tell
whether a part is present by checking for a dataset that the part always contains, such as
`medium_volume` or `rf_wavelength`, or, for the spatial grid, which may have no datasets at all,
for the `grid_type` attribute.

The name of each dataset starts with a prefix identifying its part — `grid_`, `medium_`, `rf_`,
`flux_`, or `history_` — so that names from different parts cannot clash, even though the names of the recorded
flux datasets include user-chosen instrument names.

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

Besides the standard bundle attributes described in the
[HDF5 output](hdf5-output/03-data-model.md) data model, a checkpoint bundle's `when` and
`iteration` attributes duplicate the information already encoded in its name, for easy
programmatic access. The `when` attribute also identifies the bundle as a checkpoint, since a
checkpoint is otherwise an ordinary bundle in its suite. The remaining attributes record the ski
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
| `grid_type` | string | The grid's SKIRT class name, converted to underscore style (see Spatial grid). |

`grid_type` is present whenever the checkpoint includes the Spatial grid part. For tree grids, it
also determines the number of children per node.

## Spatial grid

Captures the grid's structure, as introduced under Checkpoint contents in the Features
chapter. A simulation with a medium has exactly one spatial grid, so this part is present in
every such simulation, identified by the checkpoint's `grid_type` attribute — the grid's SKIRT class name
converted to underscore style, e.g. `OctTreeSpatialGrid` becomes `oct_tree_spatial_grid`. Beyond that
attribute, the part's contents depend on the grid type: tree grids need their topology
stored, grids with cuboidal, axis-aligned cells additionally (or instead) get a linear cell
list enabling visualization or sampling without reconstructing the grid (tree, AMR, and
Cartesian grids), and Voronoi/Tetra grids get their site or vertex positions; other grid
types carry no datasets at all.

**`BinTreeSpatialGrid` and `OctTreeSpatialGrid`.** *Topology*: stored as the flat node array
that these grids use for path segment generation (see the
[Tree-based spatial grids](tree-based-spatial-grids/03-implementation.md) note), with one row per
node; row `i` describes the node with id `i`, the root always being id `0`. The node ids are the
indices in the array: after construction, nodes are numbered level by level, and dynamic grid
refinement appends the children of a subdivided node to the end of the array. In both cases the
children of a node are consecutive, and they always come after their parent. `grid_first_child`
gives the id of a node's first child, or `-1` for a leaf; the number of children (2 or 8) follows
from the grid type. `grid_cell_index` gives the cell index of a leaf, or `-1` for a nonleaf node.
The cell indices are stored explicitly because they cannot be derived from the node order: when
dynamic grid refinement subdivides a cell, the first child keeps its parent's cell index, and the
other children receive new indices at the end of the cell list. The node extents and neighbor
links are not stored, because they follow from the domain extent and the topology.

With dynamic grid refinement, `grid_initial_level` gives, for each cell, the level of the cell
at the end of the initial construction, or of its ancestor at that time. The `maxExtraLevels`
limit compares a cell's current level with this initial level. *Linear cell list*: yes, as
described below.

**`AdaptiveMeshSpatialGrid` and `CartesianSpatialGrid`.** *Topology*: none. An AMR grid's
structure comes wholesale, and deterministically, from its own text/HDF5 input bundle; a
Cartesian grid's structure is fully determined by the ski file — neither needs to be
stored again here. *Linear cell list*: yes, as described below, for visualization and
sampling.

**`VoronoiMeshSpatialGrid` and `TetraMeshSpatialGrid`.** *Topology*: none — their sites or
vertices are not organized hierarchically, so there is no topology to speak of. *Linear
cell list*: none either, since neither grid type is cuboidal (see Unsupported features in
Features). What needs preserving instead is simply the site or vertex positions themselves,
to avoid resampling them; this part therefore consists of the datasets `grid_x`, `grid_y`, and
`grid_z`, with the same layout and attributes as the columns of a Text column file bundle (see the
[HDF5 input](hdf5-input/03-data-model.md) data model).

**`Cylinder`/`Sphere` grids.** *Topology*: none, since these grids are fully parametric.
*Linear cell list*: none either, since neither grid family is cuboidal (see Spatial grid
types under Unsupported features in Features). This part has no datasets at all, just the
checkpoint's `grid_type` attribute.

**Datasets** — shown here for an octree with `Nn` nodes (leaves and nonleaves combined) and
`M` leaf cells:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `grid_first_child` | (Nn) | 32-bit integer | `description = "id of the first child node; -1 for a leaf"`. Tree grids only. |
| `grid_cell_index` | (Nn) | 32-bit integer | `description = "cell index of a leaf node; -1 for a nonleaf node"`. Tree grids only. |
| `grid_initial_level` | (M) | 32-bit integer | `description = "level of the cell after initial construction"`. Tree grids with dynamic refinement only. |
| `grid_min` | (M, 3) | 64-bit float | `description = "minimum corner of the cell's bounding box"`, `quantity = "length"`, `unit = "m"` |
| `grid_max` | (M, 3) | 64-bit float | `description = "maximum corner of the cell's bounding box"`, `quantity = "length"`, `unit = "m"` |

`grid_min` and `grid_max` are present for tree, AMR, and Cartesian grids; `grid_first_child` and
`grid_cell_index` for tree grids only, and `grid_initial_level` only for tree grids configured for
dynamic refinement. The row index into `grid_min`/`grid_max` and `grid_initial_level` is the cell
index `m` used throughout the Medium state and Radiation field parts; for tree grids, it matches
`grid_cell_index`.

## Medium state

Wraps the `MediumState` data member of `MediumSystem`, which holds the complete per-cell
medium state. For each spatial cell, this includes a set of **common** variables shared by
all medium components, and, for each medium component, a set of **specific** variables —
some always present, some requested only by certain material mixes. A material mix can
also request any number of **custom** variables, each with its own human-readable
description and physical quantity type.

Every dataset carries an `M`-length first dimension, one entry per spatial cell. `medium_volume`
is always present; `medium_bulk_velocity` and `medium_magnetic_field` are present only if requested by at
least one medium component. `medium_number_density_<h>` is always present for every medium
component `h` (0-based); `medium_metallicity_<h>` and `medium_temperature_<h>` are present only if
requested by that component's material mix; `medium_custom_<h>_<k>` is present once for each
custom variable `k` (0-based, in declaration order) that component requests. This
index-suffix naming follows the same convention SKIRT's `MetallicityProbe`,
`TemperatureProbe`, and `CustomStateProbe` already use for their own per-component output
today (`<h>_Z`, `<h>_T`, `<h>_customstate`).

**Datasets** — the names and the number of per-component and custom datasets come from
the simulation's configuration, not a fixed schema. Shown here for a two-component medium —
component 0 a plain dust mix, component 1 an ionized gas mix requesting two custom
variables:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `medium_volume` | (M) | 64-bit float | `description = "volume"`, `quantity = "volume"`, `unit = "m3"` |
| `medium_bulk_velocity` | (M, 3) | 64-bit float | `description = "bulk velocity"`, `quantity = "velocity"`, `unit = "m/s"` |
| `medium_magnetic_field` | (M, 3) | 64-bit float | `description = "magnetic field"`, `quantity = "magneticfield"`, `unit = "T"` |
| `medium_number_density_0` | (M) | 64-bit float | `component = 0`, `description = "number density"`, `quantity = "numbervolumedensity"`, `unit = "1/m3"` |
| `medium_number_density_1` | (M) | 64-bit float | `component = 1`, `description = "number density"`, `quantity = "numbervolumedensity"`, `unit = "1/m3"` |
| `medium_metallicity_1` | (M) | 64-bit float | `component = 1`, `description = "metallicity"`, `quantity = ""`, `unit = ""` |
| `medium_temperature_1` | (M) | 64-bit float | `component = 1`, `description = "temperature"`, `quantity = "temperature"`, `unit = "K"` |
| `medium_custom_1_0` | (M) | 64-bit float | `component = 1`, `description = "helium abundance"`, `quantity = ""`, `unit = ""` |
| `medium_custom_1_1` | (M) | 64-bit float | `component = 1`, `description = "hydrogen neutral fraction"`, `quantity = ""`, `unit = ""` |

Every dataset carries a `description` attribute with a short human-readable label, a
`quantity` attribute giving SKIRT's physical-quantity-type identifier, and a `unit`
attribute reflecting the default SI unit for the quantity type, because that's what SKIRT
uses internally. For a dimensionless quantity, both `quantity` and `unit` are empty
strings. The per-component datasets also carry a `component` attribute repeating the
zero-based index.

## Radiation field

Wraps the `MediumSystem` data members that hold the radiation field accumulated from
primary and, if applicable, secondary sources, at each spatial cell and each bin of
the configured radiation-field wavelength grid. This part is present only if the
simulation has a radiation field; `rf_secondary` is present only if the simulation
also has a secondary radiation field.

The stored values are the raw, unnormalized quantity SKIRT accumulates per photon packet,
i.e. `L·Δs`, the packet's luminosity times its path length through the cell, summed over
every packet contributing to the bin. To reproduce the mean intensity `J_λ` reported by
`RadiationFieldProbe`, this quantity must still be divided by 4π times the cell volume
(the `medium_volume` dataset) and the bin's effective width:

```
J_λ = (L·Δs) / (4π · V · Δλ)
```

Keeping the raw, undivided form lets a resumed run add further packets on top of what a
checkpoint already reflects.

**Datasets** — shown here for a medium with `M` cells and `W` radiation-field wavelength bins and both a
primary and a secondary radiation field:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `rf_wavelength` | (W) | 64-bit float | `description = "characteristic wavelength of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `rf_width` | (W) | 64-bit float | `description = "effective width of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `rf_primary` | (M, W) | 64-bit float | `description = "radiation field accumulated from primary sources"`, `unit = "W m"` |
| `rf_secondary` | (M, W) | 64-bit float | `description = "radiation field accumulated from secondary sources"`, `unit = "W m"` |

`rf_primary` and `rf_secondary` carry no `quantity` attribute, since this raw, unnormalized form
has no established SKIRT quantity-type identifier — it is never otherwise exposed outside
`MediumSystem`. Their `unit` attribute is set directly to `W m` (power times length), the
dimension of `L·Δs` in the formula above.

## Recorded fluxes

Wraps `FluxRecorder`, the helper class each `Instrument` instance uses to accumulate the
effect of every detected photon packet. A simulation can have several instruments, each
with its own `FluxRecorder`, and each recorder can produce up to four output types — an
SED, an IFU data cube, a light curve (LC), or a spectral-time map (STM) — depending on
which `Instrument` subclass is configured (for example, `SEDInstrument` produces an SED,
`LightCurveInstrument` produces an LC). The recorded
values are the raw luminosity
contributions (in W) accumulated per bin, before the distance-, pixel-, and
wavelength-bin-width calibration that `calibrateAndWrite()` applies just before writing the
corresponding output bundle.

**Naming.** Every dataset name starts with the `flux_` prefix and the owning instrument's
`instrumentName` (as used for its regular output bundles), followed by the output type (`sed`,
`ifu`, `lc`, or `stm`) and, for the flux datasets, the flux component. The wavelength
and time axes are shared by all output types of a given instrument, so they carry only the prefix
and the instrument name:

- `flux_<instrument>_wavelength`, `flux_<instrument>_wavelength_width` — present if the instrument
  produces an SED, IFU, or STM.
- `flux_<instrument>_time`, `flux_<instrument>_time_width` — present if the instrument produces an
  LC or STM.
- `flux_<instrument>_<type>_<component>` — one dataset per flux component actually recorded for
  that output type.

**Components.** Depending on configuration, a recorder tracks either a single `total`
component, or the full breakdown `transparent`, `primary_direct`, `primary_scattered`,
`secondary_direct`, `secondary_scattered`, `secondary_transparent` — never both at once. If
scattering levels are tracked separately, `primary_scattered_<n>` (one-based) is added for
each level. If polarization is recorded, `stokes_q`, `stokes_u`, and `stokes_v` hold the
Stokes elements of the total flux, for all four output types; if the full component
breakdown and polarization are both recorded together, SED and LC additionally carry
`<component>_stokes_q/u/v` for each of the six components (IFU and STM never do). If
statistics are recorded, `stats_0` through `stats_4` hold the sums of the k-th power of
each contributing packet's weight, one set per output type, not per component.

**Light curve weighting.** Because an LC bin is spectrally integrated and therefore has no
single characteristic wavelength, every `flux_<instrument>_lc_<component>` dataset is
accompanied by a wavelength-weighted twin, `flux_<instrument>_lc_<component>_weighted`, needed
to convert the calibrated result between flux-per-wavelength and photon-count styles.

**Datasets** — shown here for two instruments: `i`, an `SEDInstrument`, and `j`, a
`LightCurveInstrument`, both with component tracking and statistics enabled, secondary
emission present, and no polarization or scattering-level breakdown; `W` is `i`'s number of
wavelength bins and `T` is `j`'s number of time bins:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `flux_i_wavelength` | (W) | 64-bit float | `instrument = "i"`, `description = "characteristic wavelength of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `flux_i_wavelength_width` | (W) | 64-bit float | `instrument = "i"`, `description = "effective width of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `flux_i_sed_transparent` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_primary_direct` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "primary_direct"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_primary_scattered` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "primary_scattered"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_secondary_direct` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_direct"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_secondary_scattered` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_scattered"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_secondary_transparent` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_i_sed_stats_0` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "stats_0"`, `quantity = ""`, `unit = ""` |
| `flux_i_sed_stats_1` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "stats_1"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_j_time` | (T) | 64-bit float | `instrument = "j"`, `description = "characteristic time of the bin"`, `quantity = "time"`, `unit = "s"` |
| `flux_j_time_width` | (T) | 64-bit float | `instrument = "j"`, `description = "width of the bin"`, `quantity = "time"`, `unit = "s"` |
| `flux_j_lc_transparent` | (T) | 64-bit float | `instrument = "j"`, `type = "lc"`, `component = "transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `flux_j_lc_transparent_weighted` | (T) | 64-bit float | `instrument = "j"`, `type = "lc"`, `component = "transparent"`, `unit = "W m"` |

`stats_0` is a plain, dimensionless count of contributing packet histories, hence the empty
`quantity`/`unit`. `stats_1` has the same dimension as the flux components themselves.
`stats_2` through `stats_4` and the `_weighted` datasets have no established SKIRT
quantity-type identifier, so they carry no `quantity` attribute and their `unit` is set
directly to `"W2"`, `"W3"`, and `"W4"` (matching SKIRT's own `"m3"`-style unit-string
convention) and `"W m"` respectively. IFU and STM datasets follow the same component and
naming rules as SED and LC, but with dimensions (W, Ny, Nx) and (W, T) respectively, where
Ny and Nx are the instrument's pixel counts.

## Iteration history

Wraps the `IterationHistory` object proposed in the
[Iteration history](iteration-history/01-introduction.md) note, which holds two kinds of series:
scalar series, each holding the values of a global quantity over its most recent iterations, and
cell windows, each holding a running sum per spatial cell. This part is present whenever the
history holds at least one series, and is written at every checkpoint, including `Setup`, since the
aggregates record the initial medium state at the end of setup.

**Series keys.** In the history, a series is identified by a simulation item pointer and an
integer id. A pointer is meaningless in another run, so the checkpoint identifies the item by its
path in the simulation item hierarchy: the sequence of child indices leading from the root item to
the item, such as `0/3/1`, stored as a string. Because a resumed run uses the same ski file, the
same path leads to the corresponding item. The checkpoint also stores the item's type, so that a
mismatch is detected when resuming.

**Naming.** Each scalar series is stored as a dataset `history_scalar_<n>`, and each cell window as
a dataset `history_window_<n>`, where `<n>` is a zero-based index in the order in which the
history lists the series. The index serves only to make the names unique; the attributes identify
the series.

**Scalar series.** The dataset holds the value of each slot, from the current iteration (index 0)
back to the depth of the series, with NaN for a slot whose value was not set. Its attributes record
everything needed to recreate the series:

| Attribute | Type | Description |
| --- | --- | --- |
| `item_path` | string | Path of the item on whose behalf the series is kept (see above). |
| `item_type` | string | Type of that item, for verification. |
| `id` | 32-bit integer | The id chosen by the client. |
| `description` | string | The description of the series. |
| `quantity` | string | The quantity of the values, or empty if dimensionless. |
| `lifetime` | string | `Loop` or `Simulation`. |
| `aggregate_medium` | 32-bit integer | Index of the medium component of the aggregation rule. Aggregates only. |
| `aggregate_custom` | 32-bit integer | Index of the custom state variable of the aggregation rule. Aggregates only. |

Values are stored in SI units, as used internally by SKIRT, irrespective of the `quantity`.

**Cell windows.** The dataset holds the running sum for each spatial cell, so that it has the same
length `M` as the Medium state datasets, also after dynamic grid refinement has appended cells. Its
attributes are `item_path`, `item_type`, `id`, `description`, and `lifetime`, as for a scalar
series, plus `num_iterations`, the number of iterations accumulated since the last reset.

**Loop state.** Two further attributes on the checkpoint bundle record the state of the history
itself:

| Attribute | Type | Description |
| --- | --- | --- |
| `history_loop` | string | The current loop: `None`, `Primary`, `Secondary`, or `Merged`. |
| `history_iteration` | 32-bit integer | The one-based index of the current iteration within that loop. |

`history_loop` distinguishes a merged loop from a secondary loop, which the `when` attribute does
not.

**Datasets** — shown here for a history with two scalar series of depth 2 and one cell window
over `M` cells:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `history_scalar_0` | (2) | 64-bit float | `item_path`, `item_type`, `id`, `description`, `quantity`, `lifetime` |
| `history_scalar_1` | (2) | 64-bit float | as above, plus `aggregate_medium`, `aggregate_custom` |
| `history_window_0` | (M) | 64-bit float | `item_path`, `item_type`, `id`, `description`, `lifetime`, `num_iterations` |
