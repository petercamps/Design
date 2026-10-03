# Features

## Replaced or adjusted classes

- **Grid**: `TreeSpatialGrid`, `FileTreeSpatialGrid`, `PolicyTreeSpatialGrid`

- **Node**: `TreeNode`, `BinTreeNode`, `OctTreeNode`

- **Policy**: `TreePolicy`, `DensityTreePolicy`, `NestedDensityTreePolicy`, `SiteListTreePolicy`

- **Probe**: `TreeSpatialGridTopologyProbe`

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
to `minLevel` and never beyond `maxLevel`.

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
|  &emsp;`DustDensityTreePolicy` | `maxFraction` |
|  &emsp;`DustOpticalDepthTreePolicy` | `maxOpticalDepth`, `wavelength` |
|  &emsp;`DustDispersionTreePolicy` | `maxDispersion` |
|  &emsp;`ElectronDensityTreePolicy` | `maxFraction` |
|  &emsp;`GasDensityTreePolicy` | `maxFraction` |
|  &emsp;`SiteListTreePolicy` | `filename`, `numExtraLevels` |
|  &emsp;`BoxTreePolicy` | `minX`, ..., `maxZ`, `policy` |
|  &emsp;`TopologyTreePolicy` | `filename` |

**Material properties**. There now is a separate policy for each material type and
property. This clearly separates the various independent criteria, and makes it easy to
add another criterion (say, electron optical depth) without changing existing policies.

**Site list**. The `SiteListTreePolicy` carries over unchanged. It uses a site list
given as position coordinates in an input file or, if the filename property is empty,
offered by one of the media components in the medium system. In a first step the tree is
subdivided in such a way that each leaf node contains at most one of the sites in the
list. Subsequently each of these leaf nodes is further subdivided a fixed number of
times, as configured by the user.

**Regional refinement**. The previous `NestedDensityTreePolicy` is replaced by the much
more powerful `BoxTreePolicy`. Next to a regular bounding box, this new policy holds an
arbitrary nested policy to which it defers only for nodes intersecting the box.
Listed alongside one or more ordinary material property policies covering the full
domain, `BoxTreePolicy` reproduces the same result as `NestedDensityTreePolicy` for the
typical case where the box-gated policy's criteria are stricter and therefore dominate
wherever they overlap. The same mechanism now works for any kind of policy, and for more
than one nested box, without a dedicated class for each combination.

**Tree topology**. `TopologyTreePolicy` replaces `FileTreeSpatialGrid`. It loads a
topology previously recorded by the `TreeSpatialGridTopologyProbe` from a file, but now
as one policy among others rather than a separate spatial grid class. When configured as
the sole policy, it operates as before - assuming that the `minLevel`..`maxLevel` range
is sufficiently wide. It is now possible, however, to refine a previously recorded grid
by configuring other policies alongside it. Note that the `TreeSpatialGridTopologyProbe`
output remains unchanged; only its implementation is adjusted to the new tree classes.

The following new policies are added right away; others can be devised in the future.

| Policy | Properties |
| --- | --- |
| `TreePolicy` | |
|  &emsp;`ParticleFieldTreePolicy` | `filename`, `maxFraction` |
|  &emsp;`ResolvedSpheresTreePolicy` | `filename`, `numBins`, `reach`, `importNumBins`, `importReach` |
|  &emsp;`AdaptiveMeshTreePolicy` | `filename` |
|  &emsp;`CellTreePolicy` | `filename` |
|  &emsp;`GasThermalEnergyTreePolicy` | `maxFraction` |

The `ParticleFieldTreePolicy` imports a density field defined by a list of smoothed
particles, each with their position and smoothing length plus some "mass" in arbitrary
units. The field can reflect a property of the medium other than those actually used in
the simulation, or the particles can be positioned "by hand" in strategic positions.

The `ResolvedSpheresTreePolicy` reads a list of spheres defined by their position and
radius, and ensures that the grid resolves each sphere with at least `numBins` cells
across its radius in each spatial direction, enforced out to `reach` radii from the
sphere's center rather than only within the sphere itself. The optional `importNumBins`
and `importReach` flags read a per-sphere override for either property from extra columns
in the same file, for spheres that need a different resolution or reach than the rest.

`AdaptiveMeshTreePolicy` and `CellTreePolicy` ensure the tree is never coarser than an
imported adaptive mesh or cell mesh, respectively: any node whose extent is coarser than
the imported mesh's leaf at its center is refined. If `filename` is given, that policy
imports its own snapshot to refine against; if it is left empty, the policy instead uses
whichever adaptive mesh or cell mesh the medium system has already imported, avoiding a
second copy of the same file that could drift out of sync with the actual medium.

`GasThermalEnergyTreePolicy` refines cells holding more than `maxFraction` of the total
gas thermal energy, `n T V`, mirroring the density-fraction policies above but for a
gas-physics quantity instead.
