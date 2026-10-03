# Data model

## Links

Checkpoints rely on one HDF5 concept beyond those introduced in the
[HDF5 input](hdf5-input/01-introduction.md) note. Membership in a group is not intrinsic to the
object it contains, but implemented through a separate **link** that the group owns and that
points to a named object elsewhere in the file. A **hard link**, the normal case, keeps an object
alive as long as at least one hard link to it still exists. Several groups can therefore hold
hard links to the same object, which is how a checkpoint refers to unchanged data stored by an
earlier checkpoint without copying it.

## Checkpoint bundle

A checkpoint bundle is a bundle of bundles: its HDF5 group holds, as direct members, some
or all of the four checkpoint bundles described in the following sections (Spatial grid,
Medium state, Radiation field, Recorded fluxes) — never a dataset of its own. Which of the
four are actually present depends on what data the simulation has available at that point
(see Checkpoint probe behavior in the Features chapter).

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
programmatic access. The remaining attributes record the ski
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

## Spatial grid

Captures the grid's structure, as introduced under Checkpoint bundles in the Features
chapter. A simulation with a medium has exactly one spatial grid, so this bundle is present in
every such simulation, identified by its `grid_type` attribute — the grid's SKIRT class name
converted to underscore style, e.g. `OctTreeSpatialGrid` becomes `oct_tree_spatial_grid`. Beyond that
attribute, the bundle's contents depend on the grid type: tree grids need their topology
stored, grids with cuboidal, axis-aligned cells additionally (or instead) get a linear cell
list enabling visualization or sampling without reconstructing the grid (tree, AMR, and
Cartesian grids), and Voronoi/Tetra grids get their site or vertex positions; other grid
types carry no datasets at all.

**`TreeSpatialGrid`.** *Topology*: stored as one row per tree node — row `i` of `is_leaf`
and `parent_id` describes the node with id `i`, the root always being id `0`. `parent_id`
gives the id of that node's parent (`-1` for the root), making the hierarchy explicit rather
than implied by storage order. A node's parent always has a smaller id than the node itself,
since nothing can be subdivided into children before it exists, so replaying rows `0` to
`Nn - 1` in order always reaches a parent before its children — regardless of when, during
the run, any given node was actually subdivided. The format therefore does not depend on the
tree having been built in a single top-down pass. `tree_type` records whether each nonleaf
node splits into 2 children (`BinTree`) or 8 (`OctTree`), needed to map a node's children —
the rows that name it as their `parent_id`, in ascending id order — onto the fixed geometric
split of its box. *Linear cell list*: yes, as described below.

**`AdaptiveMeshSpatialGrid` and `CartesianSpatialGrid`.** *Topology*: none. An AMR grid's
structure comes wholesale, and deterministically, from its own text/HDF5 input bundle; a
Cartesian grid's structure is fully determined by the ski file — neither needs to be
stored again here. *Linear cell list*: yes, as described below, for visualization and
sampling.

**`VoronoiMeshSpatialGrid` and `TetraMeshSpatialGrid`.** *Topology*: none — their sites or
vertices are not organized hierarchically, so there is no topology to speak of. *Linear
cell list*: none either, since neither grid type is cuboidal (see Unsupported features in
Features). What needs preserving instead is simply the site or vertex positions themselves,
to avoid resampling them; this bundle is therefore just a Text column file bundle (see the
[HDF5 input](hdf5-input/03-data-model.md) data model) — `x`, `y`, and `z` columns — with the `grid_type`
attribute added.

**`Cylinder`/`Sphere` grids.** *Topology*: none, since these grids are fully parametric.
*Linear cell list*: none either, since neither grid family is cuboidal (see Spatial grid
types under Unsupported features in Features). This bundle carries no datasets at all, just
the standard bundle attributes and `grid_type`.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `grid_type` | string | The grid's SKIRT class name, converted to underscore style (see above). |
| `tree_type` | string | `OctTree` or `BinTree`. Tree grids only. |

**Datasets** — shown here for an octree with `Nn` nodes (leaves and nonleaves combined) and
`M` leaf cells:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `is_leaf` | (Nn) | boolean | `description = "true for a leaf node"`. Tree grids only. |
| `parent_id` | (Nn) | 32-bit integer | `description = "id of the parent node; -1 for the root"`. Tree grids only. |
| `min` | (M, 3) | 64-bit float | `description = "minimum corner of the cell's bounding box"`, `quantity = "length"`, `unit = "m"` |
| `max` | (M, 3) | 64-bit float | `description = "maximum corner of the cell's bounding box"`, `quantity = "length"`, `unit = "m"` |

`min` and `max` are present for tree, AMR, and Cartesian grids; `is_leaf`, `parent_id`, and
`tree_type` for tree grids only. The row index into `min`/`max` matches the cell index `m`
used throughout the Medium state and Radiation field bundles; for tree grids, `m` is assigned
by scanning `is_leaf` in row (id) order and numbering the leaves in the order encountered,
exactly as SKIRT itself does when the grid is freshly constructed.

## Medium state

Wraps the `MediumState` data member of `MediumSystem`, which holds the complete per-cell
medium state. For each spatial cell, this includes a set of **common** variables shared by
all medium components, and, for each medium component, a set of **specific** variables —
some always present, some requested only by certain material mixes. A material mix can
also request any number of **custom** variables, each with its own human-readable
description and physical quantity type.
`MediumState` also supports "aggregate" cells used to judge convergence across
iterations; per Unsupported features (Iteration history) in the Features chapter, the checkpoint
probe does not store these, so this bundle covers only the `M` real spatial cells.

Every dataset carries an `M`-length first dimension, one entry per spatial cell. `volume`
is always present; `bulk_velocity` and `magnetic_field` are present only if requested by at
least one medium component. `number_density_<h>` is always present for every medium
component `h` (0-based); `metallicity_<h>` and `temperature_<h>` are present only if
requested by that component's material mix; `custom_<h>_<k>` is present once for each
custom variable `k` (0-based, in declaration order) that component requests. This
index-suffix naming follows the same convention SKIRT's `MetallicityProbe`,
`TemperatureProbe`, and `CustomStateProbe` already use for their own per-component output
today (`<h>_Z`, `<h>_T`, `<h>_customstate`).

**Attributes**

None other than the standard bundle attributes.

**Datasets** — the names and the number of per-component and custom datasets come from
the simulation's configuration, not a fixed schema. Shown here for a two-component medium —
component 0 a plain dust mix, component 1 an ionized gas mix requesting two custom
variables:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `volume` | (M) | 64-bit float | `description = "volume"`, `quantity = "volume"`, `unit = "m3"` |
| `bulk_velocity` | (M, 3) | 64-bit float | `description = "bulk velocity"`, `quantity = "velocity"`, `unit = "m/s"` |
| `magnetic_field` | (M, 3) | 64-bit float | `description = "magnetic field"`, `quantity = "magneticfield"`, `unit = "T"` |
| `number_density_0` | (M) | 64-bit float | `component = 0`, `description = "number density"`, `quantity = "numbervolumedensity"`, `unit = "1/m3"` |
| `number_density_1` | (M) | 64-bit float | `component = 1`, `description = "number density"`, `quantity = "numbervolumedensity"`, `unit = "1/m3"` |
| `metallicity_1` | (M) | 64-bit float | `component = 1`, `description = "metallicity"`, `quantity = ""`, `unit = ""` |
| `temperature_1` | (M) | 64-bit float | `component = 1`, `description = "temperature"`, `quantity = "temperature"`, `unit = "K"` |
| `custom_1_0` | (M) | 64-bit float | `component = 1`, `description = "helium abundance"`, `quantity = ""`, `unit = ""` |
| `custom_1_1` | (M) | 64-bit float | `component = 1`, `description = "hydrogen neutral fraction"`, `quantity = ""`, `unit = ""` |

Every dataset carries a `description` attribute with a short human-readable label, a
`quantity` attribute giving SKIRT's physical-quantity-type identifier, and a `unit`
attribute reflecting the default SI unit for the quantity type, because that's what SKIRT
uses internally. For a dimensionless quantity, both `quantity` and `unit` are empty
strings. The per-component datasets also carry a `component` attribute repeating the
zero-based index.

## Radiation field

Wraps the `MediumSystem` data members that hold the radiation field accumulated from
primary and, if applicable, secondary sources, at each spatial cell and each bin of
the configured radiation-field wavelength grid. This bundle is present only if the
simulation has a radiation field; `secondary` is present only if the simulation
also has a secondary radiation field.

The stored values are the raw, unnormalized quantity SKIRT accumulates per photon packet,
i.e. `L·Δs`, the packet's luminosity times its path length through the cell, summed over
every packet contributing to the bin. To reproduce the mean intensity `J_λ` reported by
`RadiationFieldProbe`, this quantity must still be divided by 4π times the cell volume
(Medium state's `volume` dataset) and the bin's effective width:

```
J_λ = (L·Δs) / (4π · V · Δλ)
```

Keeping the raw, undivided form lets a resumed run add further packets on top of what a
checkpoint already reflects.

**Attributes**

None other than the standard bundle attributes.

**Datasets** — shown here for a medium with `M` cells and `W` radiation-field wavelength bins and both a
primary and a secondary radiation field:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `wavelength` | (W) | 64-bit float | `description = "characteristic wavelength of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `width` | (W) | 64-bit float | `description = "effective width of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `primary` | (M, W) | 64-bit float | `description = "radiation field accumulated from primary sources"`, `unit = "W m"` |
| `secondary` | (M, W) | 64-bit float | `description = "radiation field accumulated from secondary sources"`, `unit = "W m"` |

`primary` and `secondary` carry no `quantity` attribute, since this raw, unnormalized form
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

**Naming.** Every dataset name starts with the owning instrument's `instrumentName` (as
used for its regular output bundles), followed by the output type (`sed`, `ifu`, `lc`, or
`stm`) and, for the flux datasets themselves, the flux component. The wavelength and time
axes are shared by all output types of a given instrument, so they carry only the
instrument name:

- `<instrument>_wavelength`, `<instrument>_wavelength_width` — present if the instrument
  produces an SED, IFU, or STM.
- `<instrument>_time`, `<instrument>_time_width` — present if the instrument produces an
  LC or STM.
- `<instrument>_<type>_<component>` — one dataset per flux component actually recorded for
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
single characteristic wavelength, every `<instrument>_lc_<component>` dataset is
accompanied by a wavelength-weighted twin, `<instrument>_lc_<component>_weighted`, needed
to convert the calibrated result between flux-per-wavelength and photon-count styles.

**Attributes**

None other than the standard bundle attributes.

**Datasets** — shown here for two instruments: `i`, an `SEDInstrument`, and `j`, a
`LightCurveInstrument`, both with component tracking and statistics enabled, secondary
emission present, and no polarization or scattering-level breakdown; `W` is `i`'s number of
wavelength bins and `T` is `j`'s number of time bins:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `i_wavelength` | (W) | 64-bit float | `instrument = "i"`, `description = "characteristic wavelength of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `i_wavelength_width` | (W) | 64-bit float | `instrument = "i"`, `description = "effective width of the bin"`, `quantity = "wavelength"`, `unit = "m"` |
| `i_sed_transparent` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_primary_direct` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "primary_direct"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_primary_scattered` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "primary_scattered"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_secondary_direct` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_direct"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_secondary_scattered` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_scattered"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_secondary_transparent` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "secondary_transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `i_sed_stats_0` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "stats_0"`, `quantity = ""`, `unit = ""` |
| `i_sed_stats_1` | (W) | 64-bit float | `instrument = "i"`, `type = "sed"`, `component = "stats_1"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `j_time` | (T) | 64-bit float | `instrument = "j"`, `description = "characteristic time of the bin"`, `quantity = "time"`, `unit = "s"` |
| `j_time_width` | (T) | 64-bit float | `instrument = "j"`, `description = "width of the bin"`, `quantity = "time"`, `unit = "s"` |
| `j_lc_transparent` | (T) | 64-bit float | `instrument = "j"`, `type = "lc"`, `component = "transparent"`, `quantity = "bolluminosity"`, `unit = "W"` |
| `j_lc_transparent_weighted` | (T) | 64-bit float | `instrument = "j"`, `type = "lc"`, `component = "transparent"`, `unit = "W m"` |

`stats_0` is a plain, dimensionless count of contributing packet histories, hence the empty
`quantity`/`unit`. `stats_1` has the same dimension as the flux components themselves.
`stats_2` through `stats_4` and the `_weighted` datasets have no established SKIRT
quantity-type identifier, so they carry no `quantity` attribute and their `unit` is set
directly to `"W2"`, `"W3"`, and `"W4"` (matching SKIRT's own `"m3"`-style unit-string
convention) and `"W m"` respectively. IFU and STM datasets follow the same component and
naming rules as SED and LC, but with dimensions (W, Ny, Nx) and (W, T) respectively, where
Ny and Nx are the instrument's pixel counts.
