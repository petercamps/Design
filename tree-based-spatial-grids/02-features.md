# Features

## Replaced or adjusted classes

- **Grid**: `TreeSpatialGrid`, `FileTreeSpatialGrid`, `PolicyTreeSpatialGrid`

- **Node**: `TreeNode`, `BinTreeNode`, `OctTreeNode`

- **Policy**: `TreePolicy`, `DensityTreePolicy`, `NestedDensityTreePolicy`, `SiteListTreePolicy`

- **Probe**: `TreeSpatialGridTopologyProbe`

- **Support**: `MediumSystem` (component lists per material type), `CellGeometry` and
  `CellMedium` (new `CellMeshInterface`), `CellSnapshot` (cell lookup), `TextInFile` (reading
  without declared columns), `SpinFlipAbsorptionMix` and `XRayIonicGasMixFamily` (inserting
  `GasMix`); new helpers `TreeNodeEvaluation` and the `Prefetch` namespace

## Grid classes

| Grid | Properties |
| --- | --- |
| `BoxSpatialGrid` | `minX`, `maxX`, `minY`, `maxY`, `minZ`, `maxZ` |
| &emsp;`TreeSpatialGrid` | `policies`, `minLevel`, `maxLevel` |
| &emsp;&emsp;`BinTreeSpatialGrid` | |
| &emsp;&emsp;`OctTreeSpatialGrid` | |

`BoxSpatialGrid` is an existing abstract class that remains untouched and is the base for
spatial tree classes that implement a cuboidal domain. `TreeSpatialGrid` is the common
base for the new tree spatial grid classes. It bears the same name as the class being
replaced but has a significantly adjusted role.

The `policies` property takes a list of tree subdivision policies, discussed below.
During construction, a node is subdivided as soon as any one policy in the list asks for
it. This makes the policies freely combinable. For example, a grid can use density
criteria and the positions of an imported medium's particles at the same time. It also
makes refining an existing grid with an extra criterion trivial — just add one more
policy to the list.

As before, the `minLevel` and `maxLevel` properties define the range of subdivision
levels within which subdivision policies play a role. The tree will always subdivide up
to `minLevel` and never beyond `maxLevel`. The defaults are also as before: levels 3 to 7
for an octree and 9 to 21 for a binary tree. With an empty policy list, the tree is
subdivided uniformly up to `minLevel`. The default spatial grid of the medium system for a
three-dimensional model is an `OctTreeSpatialGrid` with a single `DensityTreePolicy`.

The cells of a tree grid are numbered level by level, in the order in which the nodes are
created. The cell numbering therefore depends only on the tree structure, not on the policies
that produced it (see the tree topology policy below).

`BinTreeSpatialGrid` and `OctTreeSpatialGrid` implement the features specific to each
tree type, including path segment generation. The respective node classes (previously
`BinTreeNode` and `OctTreeNode`) are inlined in each implementation.

## Policy classes

The table lists the policies offering capabilities that were available before, adjusted
to the new formalism where multiple policies can be easily combined. They are discussed
in more detail below the table.

| Policy | Properties |
| --- | --- |
| `TreePolicy` | |
|  &emsp;`DensityTreePolicy` | `materialType`, `maxFraction` |
|  &emsp;`OpticalDepthTreePolicy` | `materialType`, `maxOpticalDepth`, `wavelength` |
|  &emsp;`DispersionTreePolicy` | `materialType`, `maxDispersion` |
|  &emsp;`SiteListTreePolicy` | `filename`, `numExtraLevels` |
|  &emsp;`BoxTreePolicy` | `minX`, ..., `maxZ`, `policy` |
|  &emsp;`TopologyTreePolicy` | `filename` |

**Material properties.** There now is a separate policy for each material property: the
fraction of the total mass contained in a cell, the optical depth across a cell, and the
dispersion of the density within a cell. This clearly separates the various independent
criteria, and makes it easy to add another criterion without changing existing policies.

Each of these policies has a `materialType` enumeration property that selects the material
type to which its criterion applies: `Dust`, `Electrons`, or `Gas`, matching the material types
that SKIRT distinguishes internally. The `OpticalDepthTreePolicy` also offers `All`, which applies
the criterion to the combined optical depth of all media. To apply a criterion to more than one
material type, the user lists a policy for each type. Unlike the dust-only criteria of today, the
optical depth and dispersion criteria thus become available for electrons and gas as well. For
these material types, the opacity is evaluated for the default state of the material mix, because
the medium state is not yet known when the grid is constructed.

The list of enumeration values offered by MakeUp is the same for every simulation, because SMILE
does not support conditions on individual enumeration values. The default value, however, depends
on the media in the simulation, through a conditional value expression. For example,
`DustMix:Dust;GasMix:Gas;Electrons` selects dust if the simulation has dust, and otherwise gas or
electrons; for the optical depth, `DustMix:Dust;All` selects dust if present, and otherwise all
media. If a policy selects a material type that is not present in the simulation, setup reports a
fatal error. For these expressions to work, every gas mix and gas mix family inserts the name
`GasMix`; `SpinFlipAbsorptionMix` and `XRayIonicGasMixFamily` were adjusted to do so, like the
other gas mixes.

**Site list.** The `SiteListTreePolicy` carries over unchanged, apart from the new
`filename` property. It uses a site list given as position coordinates (three columns) in an
input file or, if the `filename` property is empty, offered by the first medium component in
the medium system that offers one, either itself (an imported medium) or through its geometry
(a geometric medium with an imported geometry). The `filename` property is required if no such
medium is configured. Sites outside of the domain are ignored. In a first step the tree is
subdivided in such a way that each leaf node contains at most one of the sites in the
list. Subsequently each of these leaf nodes, including the empty ones, is further subdivided a
fixed number of times, as configured by the user. Because the tree is always subdivided up to
`minLevel`, the extra levels are counted from `minLevel` for a site isolated at a lower level.
When it is the only policy, the policy produces the same grid as in SKIRT 9.

**Regional refinement.** The previous `NestedDensityTreePolicy` is replaced by the much
more powerful `BoxTreePolicy`. Next to a regular bounding box, this new policy holds an
arbitrary nested policy to which it defers only for nodes intersecting the box.
Listed alongside one or more ordinary material property policies covering the full
domain, `BoxTreePolicy` reproduces the same result as `NestedDensityTreePolicy` for the
typical case where the box-gated policy's criteria are stricter and therefore dominate
wherever they overlap. The same mechanism now works for any kind of policy, and for more
than one nested box, without a dedicated class for each combination.

As for `NestedDensityTreePolicy`, a node intersects the box if it touches it (borders
included), and a warning is issued if the box is not fully inside the domain. Unlike
`NestedDensityTreePolicy`, `BoxTreePolicy` has no subdivision levels of its own; the
`minLevel`..`maxLevel` range of the grid applies to the whole domain. A
`NestedDensityTreePolicy` configuration that relies on different level ranges in different
regions, for example a lower maximum level outside the box than inside it, can thus not be
reproduced; the policies covering the full domain must instead use less strict criteria.

**Tree topology.** `TopologyTreePolicy` replaces `FileTreeSpatialGrid`. It loads a
topology previously recorded by the `TreeSpatialGridTopologyProbe` from a file, but now
as one policy among others rather than a separate spatial grid class. When configured as
the sole policy, it operates as before — assuming that the `minLevel`..`maxLevel` range
is sufficiently wide. It is now possible, however, to refine a previously recorded grid
by configuring other policies alongside it. Note that the `TreeSpatialGridTopologyProbe`
output keeps its format, the same sequence of values, so that existing topology files remain
usable; only its implementation is adjusted to the new tree classes. The
[HDF5 output](hdf5-output/04-implementation.md) design note adds a column information line to
the header of the file, which readers that skip header lines ignore.

The tree type of the grid must match the recorded tree: if the number of children per node in
the file differs from that of the grid, setup reports a fatal error naming the grid type to use.
If the recorded tree is deeper than the grid's `maxLevel`, the policy logs a warning. Because the
cells are numbered level by level, a grid reproduced from a recorded topology has the same cells
in the same order as the original grid. This is required, for example, to read per-cell data
written by a previous simulation, as for the initial level populations of the
`NonLTELineGasMix`. In SKIRT 9, the `FileTreeSpatialGrid` numbered its cells depth-first, so
that the cell order differed from that of the original grid.

The following new policies are added right away; others can be devised in the future.

| Policy | Properties |
| --- | --- |
| `TreePolicy` | |
|  &emsp;`ParticleFieldTreePolicy` | `filename`, `maxFraction`, `smoothingKernel` |
|  &emsp;`ResolvedSpheresTreePolicy` | `filename`, `numBins`, `reach`, `importNumBins`, `importReach` |
|  &emsp;`AdaptiveMeshTreePolicy` | `filename` |
|  &emsp;`CellTreePolicy` | `filename` |
|  &emsp;`GasThermalEnergyTreePolicy` | `maxFraction` |

The `ParticleFieldTreePolicy` imports a density field defined by a list of smoothed
particles, each with their position and smoothing length plus some "mass" in arbitrary
units. The field can reflect a property of the medium other than those actually used in
the simulation, or the particles can be positioned "by hand" in strategic positions. A node is
subdivided if it contains more than `maxFraction` of the total mass of the field. The input
file has columns for the position, the smoothing length, and the mass; a column header, if
present, must specify a mass unit for the last column, but the unit is otherwise irrelevant.
The smoothing kernel is configurable through the `smoothingKernel` property, with a cubic
spline kernel as the default, as for the imported particle media.

The `ResolvedSpheresTreePolicy` reads a list of spheres defined by their position and
radius, and ensures that the grid resolves each sphere with at least `numBins` cells
across its radius in each spatial direction, enforced out to `reach` radii from the
sphere's center rather than only within the sphere itself. Specifically, a node is subdivided
if it intersects a sphere enlarged by its reach, and if the largest width of the node exceeds
the radius of that sphere divided by its number of bins. The defaults are 10 bins and a reach of
1. The optional `importNumBins` and `importReach` flags read a per-sphere override for either
property from extra columns in the same file (in that order, after the radius), for spheres that
need a different resolution or reach than the rest: a positive imported value overrides the
configured value, while zero or a negative value selects the configured value. Spheres with a
radius that is not positive are ignored.

`AdaptiveMeshTreePolicy` and `CellTreePolicy` ensure the tree is never coarser than an
imported adaptive mesh or cell mesh, respectively: any node whose extent is coarser than
the imported mesh's leaf at its center is refined. If `filename` is given, that policy
imports its own snapshot to refine against; if it is left empty, the policy instead uses
whichever adaptive mesh or cell mesh the medium system has already imported, avoiding a
second copy of the same file that could drift out of sync with the actual medium.

More precisely, a node is "coarser" than a mesh leaf if it is wider along any of the coordinate
axes. Only the mesh leaf at the center of the node is considered; for a mesh whose cells line up
with the nodes of the tree (e.g. an adaptive mesh that is split in two along each axis, with the
same domain as an octree grid), the grid then reproduces the mesh exactly, and otherwise the
criterion is approximate. An adaptive mesh imported by the policy itself uses the domain of the
grid, and its data values are ignored; a cell list imported by the policy uses only the first
six columns (the cell corners). When cells in a cell list overlap, the cell listed first in the
file is used, as for the other properties of a cell snapshot. Without a file name, the policy
uses the mesh of the first medium component that offers one, either itself (an adaptive mesh or
cell medium) or through its geometry; the `filename` property is required if no such medium is
configured.

`GasThermalEnergyTreePolicy` refines cells holding more than `maxFraction` of the total
gas thermal energy, `n T V`, mirroring the `DensityTreePolicy` above but for a
gas-physics quantity instead. Because the tree is constructed before the medium state is
initialized, the policy uses the temperature imported by the gas medium components (i.e. an
imported medium with the `importTemperature` flag). Gas medium components that do not import a
temperature are ignored, and setup logs a warning listing them; if none of them imports a
temperature, setup reports a fatal error. The policy is offered only if the simulation has gas.
A limitation remains to be resolved: an imported medium that also imports variable mix
parameters does not report its imported temperature (`ImportedMedium::hasTemperature()` returns
false for it), so that such a medium, for example one with an `XRayIonicGasMixFamily`, is ignored.
