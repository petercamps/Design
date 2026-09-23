# Tree-based spatial grids

## Motivation

This section proposes a significant restructuring of SKIRT's hierarchical tree classes
and their helpers, including the classes that define cell subdivision policies. This
change invalidates any existing ski file that configures an octree or binary tree spatial
grid. This is annoying, even if PTS provides a procedure to automatically upgrade ski
files to the new structure. However, the additional capabilities unlocked by this change
seem to outweigh the nuisances.

The key objectives are to:

- Combine multiple cell subdivision policies; for example refining a previously
  stored tree topology with an extra criterion or refining multiple regions in the domain
  with their own distinct criteria.

- Enable new type of cell subdivision policies, for example based on an imported scalar
  field (other than density) or employing information from neighboring cells.

- Dynamically refine the grid after each primary or secondary iteration, based on the 
  calculated radiation field or quantities in the (standard or custom) medium state.

- Increase performance of path segment generation by providing a specific implementation 
  for each tree type (octree or binary tree).

## Static tree construction

### Replaced or adjusted classes

- **Grid**: `TreeSpatialGrid`, `FileTreeSpatialGrid`, `PolicyTreeSpatialGrid`

- **Node**: `TreeNode`, `BinTreeNode`, `OctTreeNode`

- **Policy**: `TreePolicy`, `DensityTreePolicy`, `NestedDensityTreePolicy`, `SiteListTreePolicy`

- **Probe**: `TreeSpatialGridTopologyProbe`

### Grid classes

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

### Policy classes

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
|  &emsp;`SiteListTreePolicy` | `numExtraLevels` |
|  &emsp;`BoxTreePolicy` | `minX`, ..., `maxZ`, `policy` |
|  &emsp;`TopologyTreePolicy` | `filename` |

**Material properties**. There now is a seperate policy for each material type and
property. This clearly separates the various independent criteria, and makes it easy to
add another criterion (say, electron optical depth) without changing existing policies.

**Site list**. The `SiteListTreePolicy` carries over unchanged. It locates a site list
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

New policies can be easily added. Two examples are listed in the following table, but
others can and will be devised in the future.

| Policy | Properties |
| --- | --- |
| `TreePolicy` | |
|  &emsp;`ParticleFieldTreePolicy` | `filename`, `maxFraction` |
|  &emsp;`ResolvedSpheresTreePolicy` | `filename`, `numBins` |

The `ParticleFieldTreePolicy` imports a density field defined by a list of smoothed
particles, each with their position and smoothing length plus some "mass" in arbitrary
units. The field can reflect a property of the medium other than those actually used in
the simulation, or the particles can be positioned "by hand" in strategic positions.

The `ResolvedSpheresTreePolicy` reads a list of spheres defined by their position and
radius, and ensures that the grid resolves each sphere with at least `numBins` in each
spatial direction - again limited to the global `maxLevel`.

## Dynamic tree refinement

To be completed.

## Implementation

To be completed.
