# Tree-based spatial grids

## Motivation

This chapter proposes a significant restructuring of SKIRT's hierarchical tree classes
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
  field (other than density), an imported grid, or a list of regions to be resolved.

- Increase performance of path segment generation by providing a specific implementation 
  for each tree type (octtree or binary tree).

## Features

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


## Implementation

### Construction

`TreeSpatialGrid` builds its tree level by level, the same way `DensityTreePolicy` does
today: starting from a one-node list holding the root, each pass evaluates every node at
the current level, subdivides the ones that need it, and appends their children to the
end of the same list, so the newly appended range becomes the next level. Subdivision
only ever appends, so a node's position in the list is a stable, level-ordered id — the
list doubles as the tree's ownership list and its implicit breadth-first queue.

The two phases per level stay split as today, for the same reason: evaluating whether a
node needs subdivision is read-only and can be expensive — sampling the density field is
the typical case — so it runs in parallel across the level's nodes, using SKIRT's
existing `Parallel` machinery exactly as `DensityTreePolicy::constructTree()` does now.
Subdividing a flagged node mutates the tree (new children, updated neighbor links) and
stays sequential. The one real change is what gets evaluated per node: instead of a
single policy's `needsSubdivide()`, it is now the configured list of policies, tested in
order and short-circuiting on the first one that asks for subdivision. Since
`TreeSpatialGrid` now owns that combined list, it drives the loop itself, rather than
each policy implementing its own as today's `TreePolicy` subclasses do. A policy narrows
to a per-node test that this shared loop calls into.

Nodes stay pointer-based and are allocated in a std::deque during construction — growing
a tree incrementally is what that representation is good at, and evaluation does not care
whether nodes are reached through pointers or indices. Once construction finishes, the
tree is converted to the flat, index-linked array `BinTreeSpatialGrid` and
`OctTreeSpatialGrid` use for path segment generation (see below), establishing neighbor
links for every node in a single top-down pass over the now-complete tree. The
pointer-based nodes can then be discarded.

### Policies 

Many policies fit the shared per-node test directly, since their subdivision criterion is
purely a property of the individual cell — e.g. a sampled density — with no dependency on
other nodes. Some policies need a closer look.

The owning `TreeSpatialGrid` class provides a way for a policy's setup code to obtain the
domain extent and the splitting convention (2 or 8 children per split), so that a
policy's implementation can use this information. Furthermore, with each
`needsSubdivide()` call, the policy is passed the bounding box and level of the node to
be evaluated.

`SiteListTreePolicy` must subdivide nodes until every leaf holds at most one site, then
refine each of those leaves — and everything below them — a fixed number of further
levels. To recast this as a per-node test, the policy calculates at setup the isolation
box for each site in the list, defined as the bounding box of the node where this site
first becomes isolated (no other sites are inside the node). This is a cheap, sequential,
geometry-only walk to find the extent of the node that site-only splitting would first
leave alone. That walk needs the raw site positions, so the policy builds a temporary
`BoxSearch` over them, only for this one-time setup computation. The per-node test itself
queries a second `BoxSearch`, built over the resulting isolation boxes, which is the only
structure that needs to survive setup.

The per-node test then becomes a single query: how many of the precomputed per-site
isolation boxes does the node's own box overlap? Since tree nodes are always either
disjoint or strictly nested, a box overlaps two or more isolation boxes exactly when it
still spans multiple, not-yet-separated sites; it overlaps exactly one when it lies at or
below the point where some site became isolated; and it overlaps none when it has no
relation to any site at all. Two or more overlaps means the node needs subdividing
outright, matching the "more than one site" rule. Exactly one overlap means it needs
subdividing only if its level is still less than that isolation box's own level plus
`numExtraLevels`. No overlaps means this policy has nothing to say about the node.

`TopologyTreePolicy` parses the recorded topology file — unchanged, still the depth-first
sequence of subdivision flags written by `TreeSpatialGridTopologyProbe` — at setup into a
small in-memory tree that mirrors it. Given a node's box and level, the per-node test
descends the parsed tree from its root, `level` times, choosing at each step the child
whose sub-box contains the query box's center. The domain extent and splitting convention
make this descent exact, without needing to compare boxes by floating-point equality. If
the descent reaches a recorded node at that depth, the test answers yes if that node is
marked subdivided in the recording and no otherwise; if it runs out of recorded depth
first — because another policy in the list kept subdividing past where this recording
stops — it answers no. Each call repeats this descent independently from the root, so the
test needs no state carried between calls and no particular calling order.

`ResolvedSpheresTreePolicy` builds a `BoxSearch` at setup, bulk-loaded with the spheres'
bounding boxes — the same pattern `ParticleSnapshot` already uses for its own smoothed
particles. The per-node test queries `entitiesFor(box)` for candidate spheres, verifies
each with an exact box-sphere intersection test, and subdivides if any of them is still
under-resolved in this node. No precomputation walk is needed here, unlike the site list:
each sphere's required cell size follows directly from its own properties, with nothing
that depends on the tree's evolving structure.

`AdaptiveMeshTreePolicy` and `CellTreePolicy` need no comparable treatment: given a node,
the per-node test asks the imported snapshot for the cell at the node's center and
compares that cell's extent to the node's. There is no cross-node bookkeeping or ordering
concern to design around: each node's test only looks at its own center against an
already-built, read-only snapshot.

### Path segment generation

`BinTreeSpatialGrid` stores its tree as a flat array of small, fixed-size nodes that
refer to each other by index rather than by pointer, avoiding pointer chasing and virtual
function calls. Each node holds its own extent, its cell index, the index of its first
child (the second child is stored consecutively), and the index of its neighbor across
each of its six walls. This data structure is established once, for the whole tree,
before actual path segment generation begins.

The path segment generator follows the neighbor links directly when a path crosses a
wall, descending into the neighbor only if it still has children of its own. A top-down
search from the root is needed only to locate a path's first cell, and as a fallback for
the rare case where rounding error leaves a neighbor link inconsistent. Per-path
quantities — the reciprocal of each direction component, and which wall it can cross —
are computed once rather than for each step, and the handful of nodes a path could move
to next are prefetched while the current step is still being finished.

`OctTreeSpatialGrid` follows the same approach, adapted to eight children per node.
